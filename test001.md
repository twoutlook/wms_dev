# [新增]入庫通知單
  * ERP 單號: ERP001
  * 貨主: 貨主1001
  * [保存]
  
# [新增]入庫通知單明細
  * 選擇料號: 0000070300001   NOET: 只顯示 貨主的關聯料號 , 9筆
    * 數量: 98
    * 留樣數量: 2  TODO  本欄位 限 医疗产业, 由配置 500003	為 2, 判斷為 :医疗产业
    * [保存]
  * 選擇料號: 070812-71904580  
    * 數量: 20
    * [保存]
   
  # [生成]入庫單
   * 在入庫通知單列表找到,在[編輯]右邊欄位為[生成]
     * INASNList.aspx
     
     * OnClick="lbtnCreateInBill_Click"
     ```
      <ItemTemplate>
         <asp:LinkButton ID="lbtnCreateInBill" runat="server" CausesValidation="false" CommandArgument='<%#Eval("ID") %>'
            CommandName="" Text="<%$ Resources:Lang, Common_GenerateBtn %>" OnClick="lbtnCreateInBill_Click" OnClientClick="return confirmprocess();">  
         </asp:LinkButton>
      </ItemTemplate>
     ```
     
     * cs: "lbtnCreateInBill_Click"
     ```
     protected void lbtnCreateInBill_Click(object sender, EventArgs e)
     {
        List<string> SparaList = new List<string>();
        try
        {
            string InAsnId = (sender as LinkButton).CommandArgument;
            INTYPE INASN_type = GetINTYPEByID(InAsnId, "INASN");
            if (INASN_type.Is_Query == 0)
            {
                string InBill_Id = Guid.NewGuid().ToString();
                //20130731134649 存在修改中的料时，不允许生成出库单
                var Asn = new INASN();
                Asn.id = InAsnId;

                #region 调用存储过程
                SparaList.Add("@P_InAsn_id:" + InAsnId.Trim());
                SparaList.Add("@P_UserNo:" + WmsWebUserInfo.GetCurrentUser().UserNo);
                SparaList.Add("@P_InBill_Id:" + InBill_Id.Trim());
                SparaList.Add("@P_IsTemporary:" + "0");
                SparaList.Add("@P_ReturnValue:" + "");
                SparaList.Add("@INFOTEXT:" + "");
                string[] Result = DBHelp.ExecuteProc("Proc_CreateInBill", SparaList);
                if (Result.Length == 1)//调用存储过程有错误
                {
                    this.Alert(Result[0].ToString());
                }
                else if (Result[0] == "0")
                {
                    //生成成功 跳转到入库单页面
                    this.WriteScript("PopupFloatWinMax('" + BuildRequestPageURL("FrmINBILLEdit.aspx", SYSOperation.Modify, InBill_Id) + "','" + Resources.Lang.FrmLogSystemList_Msg04 + "','INBILL');");
                }
                else
                {
                    this.Alert(Result[1]);
                }
                #endregion
            }
            else
            {
                //此入库通知单类型为仅查询，不能生成入库单
                Alert(Resources.Lang.FrmINASNList_MSG18 + "！");
            }
        }
        catch (Exception)
        {
        }
    }
     ```
   
   * SP, Stored Procedure, [dbo].[Proc_CreateInBill]
'''
USE [NuoXin]
GO
/****** Object:  StoredProcedure [dbo].[Proc_CreateInBill]    Script Date: 2020/8/7 15:02:11 ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO


ALTER  PROCEDURE [dbo].[Proc_CreateInBill]
(
	@P_InAsn_id VARCHAR(50),
	@P_UserNo VARCHAR(50),
	@P_InBill_Id VARCHAR(50),
	@P_IsTemporary    VARCHAR(20),  --是否暂存单据  0: 非暂存   1：暂存单据
	@P_ReturnValue INTEGER OUT,
	@INFOTEXT VARCHAR(5000) OUT
)
AS
BEGIN
	DECLARE @P_Count INTEGER,@P_InBill INTEGER,@P_CountTemp INTEGER,@P_CTICKETCODE VARCHAR(100);
	DECLARE @P_cstatus VARCHAR(50),@p_GUID VARCHAR(50),@vAsnCode VARCHAR(100),@vRetMsg VARCHAR(500),@vItype VARCHAR(50);
	DECLARE @vErpCode VARCHAR(500),@vConFig VARCHAR(50)
	DECLARE @Max INTEGER,@Cnt INTEGER;
	DECLARE @workType varchar(20);
	declare @asnLineId nvarchar(10),@billCount int;
	declare @inasn_d_qty float,@inasn_d_qty_Total float;--通知单数量，已入总数量

	declare @P_Inassit_id nvarchar(50);
	declare @M_BillCoun    float; 
	DECLARE  @P_Inassit_ids  nvarchar(50);

	DECLARE @P_Total_ASNDQty DECIMAL(18,6),
			@P_Total_BillDQty DECIMAL(18,6);


	BEGIN TRY
		SET @P_ReturnValue = 0
		set @INFOTEXT = '';

		--获取入库通知单相关信息
		SELECT @P_cstatus = ia.cstatus, 
			   @vAsnCode = ia.cticketcode,
			   @vItype = ia.itype,
			   @vErpCode = ia.cerpcode,
			   @workType = ia.WORKTYPE
		FROM InAsn ia with(nolock) 
		WHERE ia.id = @P_InAsn_id;

		--验证入库通知单是否有明细
		SELECT @P_Count = COUNT(*) FROM InAsn_d iad with(nolock) WHERE iad.id = @P_InAsn_id;
		IF(@P_Count = 0) BEGIN
			SET @INFOTEXT = dbo.Fun_GetResourceValue('Proc_CreateInBill_MSG2',@P_UserNo)+'!';--入库通知单没有明细不能生成
			GOTO END_OF_PROC;
		END

		--非未处理状态时判断明细状态
		IF(@P_cstatus != '0') BEGIN
			--判断明细状态是否存在补单中的
			SELECT @P_Count = COUNT(1) FROM inasn_d iad with(nolock) WHERE iad.cstatus IN ('0','4') AND iad.id = @P_InAsn_id AND EXISTS (SELECT 1 FROM dbo.INASN a WITH(NOLOCK) WHERE a.id = iad.id AND a.cstatus='0');
			IF(@P_Count =0) BEGIN 
				--只有存在未处理或补单中状态的明细的状态为未处理的入库通知单才能生成入库单
				SET @INFOTEXT = dbo.Fun_GetResourceValue('Proc_CreateInBill_MSG1',@P_UserNo)+'!';
				GOTO END_OF_PROC;
			END;
		END;

		SELECT @p_GUID = NEWID()

		--判断 P_OutBill_Id 传入值是否是监时表中批次的GUID
		SELECT @P_CountTemp = COUNT(*) 
		FROM TEMP_SELECTASND temp with(nolock)
		WHERE temp.asn_id = @P_InAsn_id 
		  AND temp.guid = @P_InBill_Id;

		--新增验证逻辑：判断每个项次的料号是否出完，页面中验证即可
		IF object_id('tempdb..#tempASND') is not null DROP table #tempASND

		SELECT ROW_NUMBER() over(order by userno) as 'RowNum', temp.asn_id,temp.asn_ids,temp.guid 
		into #tempASND   
		FROM TEMP_SELECTASND temp with(nolock) 
		WHERE temp.asn_id = @P_InAsn_id 
		  AND temp.guid = @P_InBill_Id;

		SET @cnt=1;
		select @max=count(1) from #tempASND;

		WHILE(@cnt<=@max)
		BEGIN
			IF object_id('tempdb..#temp_row') is not null DROP table #temp_row
			select * into #temp_row from #tempASND where RowNum = @cnt
			set @cnt = @cnt + 1;

			 --declare @inasn_d_qty float,@inasn_d_qty_Total float;--通知单数量，已入总数量
			 --select @inasn_d_qty = iquantity from inasn_d where ids  in ( select asn_ids from #temp_row ) --获取入库通知单明细项次的数量

			 --获取已经生出入库单明细，但未完成的个数
			select @billCount = count(m.id),@P_Total_BillDQty = ISNULL(SUM(d.iquantity),0)
			from inbill_d d with(nolock)
			inner join inbill m with(nolock) on d.id = m.id
			where d.asn_d_ids in ( select asn_ids from #temp_row ) --获取已生出入库单明细的总数量
			  and m.cstatus != 1
			GROUP BY d.id;

			SELECT @P_Total_ASNDQty = ISNULL(SUM(b.iquantity),0) 
			FROM dbo.INASN a with(nolock)
			INNER JOIN dbo.INASN_D b with(nolock) ON a.id = b.id
			WHERE ids IN (select asn_ids from #temp_row ) 
			  AND a.cstatus != 1;
		    
			IF(@P_Total_ASNDQty = @P_Total_BillDQty) begin
				select @asnLineId = lineid  from inasn_d where ids  in ( select asn_ids from #temp_row )
				--项次
				set @INFOTEXT = @INFOTEXT+ dbo.Fun_GetResourceValue('Proc_CreateInBill_MSG8',@P_UserNo)+ISNULL(@asnLineId,'')+dbo.Fun_GetResourceValue('Proc_CreateInBill_MSG3',@P_UserNo)+';';--已经生出入库单且未完成，不可以再生成入库单
			end
		END;

		if len(@INFOTEXT)>0 
		begin
			GOTO END_OF_PROC;
		end
   
		--验证入库通知单是否有未生成入库单明细的入库通知单明细
		IF(@P_CountTemp > 0 ) begin
			SELECT @P_Count = COUNT(*) 
			FROM
			(
				SELECT iad.ids, 
					   ISNULL(SUM(iad.iquantity),0)as InAsn_Qty,
					   (
						   SELECT ISNULL(SUM(ibd.iquantity),0) as iquantity 
						   FROM InBill_d ibd with(nolock)
						   INNER JOIN InBill ib with(nolock) ON ibd.id = ib.id
						   WHERE ib.casnid = @P_InAsn_id 
							 AND ibd.asn_d_ids = iad.ids 
						) as Inbill_Qty
				FROM InAsn_d iad with(nolock)
				WHERE iad.id = @P_InAsn_id
				  AND iad.cstatus IN ('0','4')
				GROUP BY iad.ids
			)InAsn_d_View
			INNER JOIN temp_selectasnd temp with(nolock) ON InAsn_d_View.ids = temp.asn_ids
			WHERE (InAsn_Qty - Inbill_Qty) > 0
			  AND temp.asn_id = @P_InAsn_id
			  AND temp.guid = @P_InBill_Id;
		end
		ELSE begin
			SELECT @P_Count=COUNT(*) 
			 FROM
			 (
				 SELECT iad.ids,
						ISNULL(SUM(iad.iquantity),0) as InAsn_Qty,
						ISNULL(
							(
								SELECT ISNULL(SUM(ibd.iquantity),0) as iquantity 
								FROM InBill_d ibd with(nolock) 
								INNER JOIN InBill ib with(nolock) ON ibd.id = ib.id
								WHERE ib.CASNID = @P_InAsn_id 
								  AND ibd.asn_d_ids = iad.ids
							),0) as InBill_Qty
				FROM InAsn_d iad with(nolock)
				WHERE iad.id = @P_InAsn_id
				  AND iad.cstatus IN ('0','4')
				GROUP BY iad.ids
			)newTable
			WHERE (InAsn_Qty - InBill_Qty) > 0;
		end

		IF(@P_Count = 0) BEGIN
			 --           入库通知单没有未生成出库单的明细不能生成 或 选择的出库单明细都已生成了出库单
			SET  @INFOTEXT =dbo.Fun_GetResourceValue('Proc_CreateInBill_MSG4',@P_UserNo)+ '!';
			GOTO END_OF_PROC;
		END 

		SELECT @P_InBill = COUNT(*) FROM InBill ib with(nolock) WHERE ib.id = @P_InBill_Id;

		IF(@P_InBill = 0)
		BEGIN
			--获取入库单单据号
			SELECT @P_CTICKETCODE=[dbo].[Fun_CreateNo]('INBILL')
			--生成,入库单表头
			SET @INFOTEXT = dbo.Fun_GetResourceValue('Proc_CreateInBill_MSG5',@P_UserNo)+'['+@P_CTICKETCODE+']'+dbo.Fun_GetResourceValue('Proc_CreateInBill_MSG7',@P_UserNo)+'';

			--暂存单据时，转化为平库
			IF @P_IsTemporary = '1' BEGIN
			    set @workType = '0';
			END

			--生成入库单表头
			insert into InBILL
			(
				id,
				ccreateownercode,
				dcreatetime,
				cauditperson,
				daudittime,
				cticketcode,
				cstatus,
				casnid,
				dindate,
				cmemo,
				cerpcode,
				cdefine1,
				cdefine2,
				idefine5,
				itype,
				cvendercode,
				cvender,
				creatertype,
				worktype,
				operationtype,
				IsTemporary
			 )
			 select
				 @P_InBill_Id,
				 @P_UserNo,
				 GETDATE(),
				 null,
				 null,
				 @P_CTICKETCODE,
				 '0',
				 @P_InAsn_id,
				 GETDATE(),
				 ia.cmemo,
				 ia.cerpcode,
				 ia.CDEFINE1,
				 ia.CDEFINE2,
				 null,
				 ia.itype,
				 ia.CVENDERCODE,
				 ia.CVENDER,
				 '2',
				 @workType,
				 1,
				 @P_IsTemporary  --是否暂存单据
			from InAsn ia with(nolock)
			where ia.id = @P_InAsn_id;
		END

		--判断获取0SN管理 1批次管理
		select @vConFig = fi.config_value from sys_config fi where fi.code='000002';

		--生成入库单表体
		--生成,入库单表体
		SET @INFOTEXT = dbo.Fun_GetResourceValue('Proc_CreateInBill_MSG5',@P_UserNo)+'['+@P_CTICKETCODE+']'+dbo.Fun_GetResourceValue('Proc_CreateInBill_MSG6',@P_UserNo)+'';
		if(@P_CountTemp > 0 ) BEGIN
			if(@vConFig = 0) begin
				--SN条码管理
				--根据选择的入库通知单明细生成入库单表体
				Insert into InBill_d
				(
					ids,
					id,
					cstatus,
					cinvcode,
					cinvname,
					iquantity,
					cinvbarcode,
					cerpcodeline,
					cmemo,
					dindate,
					cinpersoncode,
					iasnline,
					cdefine1,
					cdefine2,
					ddefine3,
					ddefine4,
					idefine5,
					asn_d_ids,
					lineid,
					AsndLineID --项次
				)
				select
					newid() ids,
					@P_InBill_Id,
					'0' cstatus,
					newTable.cinvcode,
					cinvname,
					(InAsn_Qty-Inbill_Qty)surplus_Qty,
					cinvbarcode,
					cerpcodeline,
					newTable.cmemo,
					GETDATE(),
					@P_UserNo,
					null,
					null,
					null,
					null,
					null,
					null,
					newTable.ids,
					ROW_NUMBER() over(order by lineid),
					lineid --项次
				from
				(
					select lineid,ids,cinvcode,cinvname,cerpcodeline,cinvbarcode,cmemo,InAsn_Qty,Inbill_Qty
					from
					(
						SELECT iad.lineid, iad.ids,iad.cinvcode,iad.cinvname,iad.cinvbarcode,iad.cerpcodeline,iad.cmemo,
							   isnull(sum(iad.iquantity),0) InAsn_Qty,
							   (
								   select isnull(sum(ibd.iquantity),0)iquantity 
								   from InBill_d ibd with(nolock) 
								   inner join InBill ib with(nolock) on ibd.id = ib.id
								   where  ib.casnid = @P_InAsn_id 
									 and ibd.asn_d_ids = iad.ids
							  )Inbill_Qty
						from InAsn_d iad
						where iad.id = @P_InAsn_id
						  and iad.cstatus in ('0','4')
						group by iad.ids,iad.cinvcode,iad.cinvname,iad.cerpcodeline,iad.cinvbarcode,iad.cmemo,iad.lineid
					)InAsn_d_View
					inner join temp_selectasnd temp with(nolock) on InAsn_d_View.ids = temp.asn_ids
					where (InAsn_Qty - Inbill_Qty) > 0
					  and temp.asn_id = @P_InAsn_id
					  and temp.guid = @P_InBill_Id
				)newTable;
			end
			ELSE BEGIN --批次管理
				select Row_Number() OVER ( ORDER BY lineid asc) RowID,NEWID() inbill_d_ids,ids,cinvcode,cinvname,cerpcodeline,cinvbarcode,cmemo,InAsn_Qty,Inbill_Qty,cbatch,datecode,lineid 
				INTO #ii 
				from (
					select iad.lineid, iad.ids,iad.cinvcode,iad.cinvname,iad.cinvbarcode,iad.cerpcodeline,iad.cmemo,iad.cbatch,iad.datecode,
						   isnull(sum(iad.iquantity),0)InAsn_Qty,
						   isnull(
							(select isnull(sum(ibd.iquantity),0)iquantity 
							from InBill_d ibd with(nolock) 
							inner join InBill ib with(nolock) on ibd.id = ib.id
							where ib.casnid = @P_InAsn_id and ibd.asn_d_ids = iad.ids)
							,0)Inbill_Qty
					from InAsn_d iad
					where iad.id = @P_InAsn_id and iad.cstatus in ('0','4')
					group by iad.ids,iad.cinvcode,iad.cinvname,iad.cerpcodeline,iad.cinvbarcode,iad.cmemo,iad.cbatch,iad.datecode,iad.lineid
				)InAsn_d_View
				inner join temp_selectasnd temp with(nolock)  on InAsn_d_View.ids = temp.asn_ids
				where (InAsn_Qty - Inbill_Qty) > 0
					and temp.asn_id = @P_InAsn_id 
					and temp.guid = @P_InBill_Id

				--保存入库单明细
				Insert into InBill_d
				(
					ids,
					id,
					cstatus,
					cinvcode,
					cinvname,
					iquantity,
					cinvbarcode,
					cerpcodeline,
					cmemo,
					dindate,
					cinpersoncode,
					iasnline,
					cdefine1,
					cdefine2,
					ddefine3,
					ddefine4,
					idefine5,
					asn_d_ids,
					LineID,
					AsndLineID --项次
				)
				SELECT 
					inbill_d_ids,
					@P_InBill_Id,
					'0',
					cinvcode,
					cinvname,
					InAsn_Qty - Inbill_Qty,
					cinvbarcode,
					cerpcodeline,
					cmemo,
					GETDATE(),
					@P_UserNo,
					null,
					null,
					null,
					null,
					null,
					null,
					ids,
					RowID,
					lineid --项次
				FROM #ii
           
				insert into inbill_d_sn
				(
					id, 
					inbill_id, 
					inbill_d_ids, 
					sn_code,
					PalletCode,
					datecode,
					cinvcode, 
					quantity, 
					createtime, 
					createowner, 
					lastvpdtime,
					lastvpdowner, 
					worktype, 
					istransed,
					transedtime, 
					union_id,
					scan_ids, 
					sntype,
					line_qty
				)
				SELECT 
					NEWID(),
					@P_InBill_Id,
					inbill_d_ids,
					null,
					cbatch,
					datecode,
					cinvcode,
					InAsn_Qty - Inbill_Qty,
					GETDATE(),
					@P_UserNo,
					GETDATE(), 
					@P_UserNo,
					0,
					0,
					null,
					null,
					null,
					3,
					null 
				FROM #ii 
				WHERE cbatch IS NOT NULL 
				  AND datecode IS NOT null;				
			END

			--修改入库通知单选择的明细料号为补单中
			update  ind
			SET    ind.cstatus = '4',
				   ind.manual = 1
			FROM inasn_d ind
			where ind.id = @P_InAsn_id 
			  and ind.cstatus in ('0','4')
			  and exists (
				  select 1 
				  from temp_selectasnd tp,inasn_d isd 
				  where tp.asn_ids = isd.ids
					and tp.asn_id = ind.id 
					and tp.guid = @P_InBill_Id 
					and isd.cinvcode = ind.cinvcode
					and isd.cerpcodeline = ind.cerpcodeline
			  );

			--更新入库通知单表头补单标记
			UPDATE dbo.INASN SET specil_return = 0 WHERE id = @P_InAsn_id;

			--删除临时表数据
			DELETE temp_selectasnd 
			WHERE asn_id = @P_InAsn_id
			  AND guid = @P_InBill_Id
		END
		ELSE BEGIN
			--理论上else不应该会执行，以下为废代码
			if(@vConFig = 0) BEGIN
				--SN条码管理
				--根据入库通知单明细生成入库单表体
				Insert into InBill_d
				(
					ids,
					id,
					cstatus,
					cinvcode,
					cinvname,
					iquantity,
					cinvbarcode,
					cerpcodeline,
					cmemo,
					dindate,
					cinpersoncode,
					iasnline,
					cdefine1,
					cdefine2,
					ddefine3,
					ddefine4,
					idefine5,
					asn_d_ids,
					lineid,
					AsndLineID --项次
				)
				select
					newid() ids,
					@P_InBill_Id,
					'0' cstatus,
					newTable.cinvcode,
					cinvname,
					(InAsn_Qty-Inbill_Qty)surplus_Qty,
					cinvbarcode,
					cerpcodeline,
					newTable.cmemo,
					GETDATE(),
					@P_UserNo,
					null,
					null,
					null,
					null,
					null,
					null,
					newTable.ids,
					ROW_NUMBER() over(order by lineid asc),
					lineid --项次
				from
				(
					select  iad.lineid,iad.ids,iad.cinvcode,iad.cinvname,iad.cerpcodeline,iad.cinvbarcode,iad.cmemo,
							isnull(sum(iad.iquantity),0)InAsn_Qty,
							isnull((select isnull(sum(ibd.iquantity),0)iquantity from InBill_d ibd inner join InBill ib on ibd.id = ib.id
							where ib.casnid = @P_InAsn_id and ibd.asn_d_ids = iad.ids),0)InBill_Qty
					from InAsn_d iad with(nolock)
					where iad.id = @P_InAsn_id
					  and iad.cstatus in ('0','4')
					group by iad.ids,iad.cinvcode,iad.cinvname,iad.cerpcodeline,iad.cinvbarcode,iad.cmemo,iad.lineid
				)newTable
				where (InAsn_Qty - InBill_Qty) > 0;
			END
			ELSE BEGIN--批次管理
				select Row_Number() OVER ( ORDER BY lineid asc) RowID, NEWID() inbill_d_ids,ids,newTable.cinvcode, cinvname,(InAsn_Qty-Inbill_Qty)surplus_Qty,
							  cinvbarcode,cerpcodeline,newTable.cmemo,newTable.cbatch,newTable.datecode,lineid INTO #iie
				from
				( select  iad.lineid,iad.ids,iad.cinvcode,iad.cinvname,iad.cerpcodeline,iad.cinvbarcode,iad.cmemo,iad.cbatch,iad.datecode,
								 isnull(sum(iad.iquantity),0)InAsn_Qty,
								 isnull((select isnull(sum(ibd.iquantity),0)iquantity from InBill_d ibd inner join InBill ib  on ibd.id = ib.id
								  where ib.casnid = @P_InAsn_id and ibd.asn_d_ids = iad.ids),0)InBill_Qty
						  from InAsn_d iad  where iad.id = @P_InAsn_id and iad.cstatus in ('0','4')
						  group by iad.ids,iad.cinvcode,iad.cinvname,iad.cerpcodeline,iad.cinvbarcode,iad.cmemo,iad.cbatch,iad.datecode,iad.lineid
				)newTable
				where (InAsn_Qty-InBill_Qty) > 0
				--保存入库单明细
				Insert into InBill_d
				(
					ids,
					id,
					cstatus,
					cinvcode,
					cinvname,
					iquantity,
					cinvbarcode,
					cerpcodeline,
					cmemo,
					dindate,
					cinpersoncode,
					iasnline,
					cdefine1,
					cdefine2,
					ddefine3,
					ddefine4,
					idefine5,
					asn_d_ids,
					LineID,
					AsndLineID --项次
				)
				SELECT 
					inbill_d_ids,
					@P_InBill_Id,
					'0',
					cinvcode,
					cinvname,
					surplus_qty,
					cinvbarcode,
					cerpcodeline,
					cmemo,
					GETDATE(), 
					@P_UserNo,
					null,
					null,
					null,
					null,
					null,
					NULL,
					ids,
					RowID,
					lineid  --项次
				FROM #iie

				insert into inbill_d_sn
				(
					id, 
					inbill_id, 
					inbill_d_ids, 
					sn_code,
					datecode,
					cinvcode, 
					quantity, 
					createtime, 
					createowner, 
					lastvpdtime,
					lastvpdowner, 
					worktype, 
					istransed,
					transedtime, 
					union_id,
					scan_ids, 
					sntype,line_qty
				)
				SELECT 
					NEWID(),
					@P_InBill_Id,
					inbill_d_ids,
					cbatch,
					datecode,
					cinvcode,
					surplus_qty,
					GETDATE(),
					@P_UserNo,
					GETDATE(),
					@P_UserNo,
					0,
					0,
					null,
					null,
					null,
					3,
					null 
				FROM #iie 
				WHERE cbatch IS NOT NULL 
				  AND datecode IS NOT null;
			END

			--修改所有明细状态为4补单中
			UPDATE inasn_d  
			SET cstatus = '4',manual = 1
			WHERE id = @P_InAsn_id 
			  AND cstatus = '0'

			--更新入库通知单表头补单标记
			UPDATE dbo.INASN SET specil_return = 0 WHERE id = @P_InAsn_id;
		END;

		--要去计算入库通知单明细中各料号的入库数量是否都生成入库单明细了，
		--已生成完的料号不应再生成入库单明细
		SET @INFOTEXT = '';
		select @p_count = COUNT(*) from InBill_d ibd where ibd.id = @P_InBill_Id;

		IF(@p_count = 0)
		begin
			delete from InBill  where id = @P_InBill_Id;
		end
		ELSE BEGIN
			SELECT ROW_NUMBER() OVER (ORDER BY lineid asc) RowID,tbd.cinvcode INTO #cc FROM inbill_d tbd, inbill t WHERE t.id = tbd.id AND t.id=@P_InBill_Id
    
			SET @cnt = 0;
		
			SELECT @max = COUNT(1) FROM #cc

			WHILE(@cnt<@max) BEGIN
				IF object_id('tempdb..#ccrow') is not null DROP table #ccrow
				SELECT * INTO #ccrow FROM #cc WHERE #cc.RowID = @Cnt + 1

				SET @vRetMsg = [dbo].[Fun_check_cinvcodebyAsn](@vAsnCode, (SELECT  cinvcode FROM #ccrow), '0', '', '');
				IF @vRetMsg <> 'OK' BEGIN
					--生成ERROR消息
					SET @INFOTEXT = @vRetMsg;
					GOTO  END_OF_PROC;
				END

				--檢查料號是否存在相反稅別的庫存CQ 2014-12-19 16:23:14-------------------------------------
				IF(@vItype = '101') BEGIN
					SET @vRetMsg = [dbo].[Fun_Check_PO_Part]((SELECT cinvcode FROM #ccrow));

					IF @vRetMsg <> 'OK' BEGIN
						--生成ERROR消息
						SET @INFOTEXT = @vRetMsg;
						GOTO  END_OF_PROC;
					END 
				END

				--檢查料號是否存在相反稅別的庫存CQ 2014-12-19 16:23:14------------------------------------
				SET @cnt = @cnt + 1;
			END 
		END

		delete from  Temp_Selectasnd where asn_id = @P_InAsn_id and userno = @P_UserNo;

		END_OF_PROC:

		IF @INFOTEXT<>'' BEGIN 
			SET @P_ReturnValue = 1 
		END
	END TRY
	BEGIN CATCH
		SET @P_ReturnValue=1
		SET @INFOTEXT = @INFOTEXT +ERROR_MESSAGE()+ dbo.Fun_GetResourceValue('Common_GenerationFailure',@P_UserNo)+' ! ';--生成失败
	END CATCH
END





















'''
   
   * 系統會在現有的[入庫通知單]TAB顯示剛才生成的[入庫單]
     * 单据号：012008070011
     * 入库通知单号：InAsn2008070005
      
   * TODO 這裡的[貨主]應該要帶過來. 
     * source code:
  
