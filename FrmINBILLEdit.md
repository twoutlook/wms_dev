# FrmINBILLEdit 入庫單 

## TODO 08/06
入库单明细页面新增物料明细	表头新增货主 物料明细栏位新增批次号字段


## aspx
  *  <asp:GridView ID="grdINBILL_D" 
  *  OnRowDataBound="grdINBILL_D_RowDataBound"

## cs
### 單頭
  * txtCAUDITPERSON, ShowData()
    * EF:INBILL
      * ProprietorId
      
      ```
      
        // https://docs.microsoft.com/en-us/dotnet/csharp/linq/perform-left-outer-joins
        DBContext db = new DBContext();
        var query = from inbill in db.INBILL
                    join goodsowner in db.Base_Proprietor on inbill.ProprietorId equals goodsowner.Id into gj
                    from subpet in gj.DefaultIfEmpty()
                    where inbill.id == this.KeyID.Trim()
                    select new {
                        inbill.id,
                        inbill.dcreatetime,
                        inbill.daudittime,
                        inbill.worktype,
                        inbill.dindate,
                        inbill.cdefine1,
                        inbill.cdefine2,
                        inbill.itype,
                        inbill.palletcode,
                        inbill.debitowner,
                        inbill.debittime,
                        inbill.ddefine3,


                        inbill.casnid, inbill.cauditperson,
                        inbill.ccreateownercode, inbill.cerpcode,
                        inbill.cmemo, inbill.creatertype,
                        inbill.cstatus, inbill.cticketcode,
                        inbill.cvender, inbill.cvendercode, GoodsOwnerCode = subpet.AbbreviationName ?? String.Empty };


      
      ```
      
### 單身
  *  GridBind()
   * EF:V_INBILL_D
    * 
  *
  *
