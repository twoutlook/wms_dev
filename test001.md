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
   
   
   
   * 系統會在現有的[入庫通知單]TAB顯示剛才生成的[入庫單]
     * 单据号：012008070011
     * 入库通知单号：InAsn2008070005
      
   * TODO 這裡的[貨主]應該要帶過來. 
     * source code:
  
