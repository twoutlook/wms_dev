# FrmINASN_DEdit 入库管理->入库通知单明细
  FrmINASN_DEdit

*已加了[留样数量：]

* 頁面填單筆, 保存後可以繼續加或返回.
* 料号 右的Search 是現有功能
  * 目前[查看] 筆數 66212
  * 單選後, 回填到 料號 品名 規格
    * 數量必填


```
     <td class="InputLabel" style="width: 13%; height: 25px;">
                            <asp:Label ID="Label3" runat="server" Text="*" ForeColor="Red"></asp:Label>
                            <asp:Label ID="lblCINVCODE" runat="server" Text="<%$ Resources:Lang, FrmInbill_CinvCode %>"></asp:Label>：
     </td>
     <td  style="width: 20%" colspan="3">
         <asp:TextBox ID="txtCINVCODE" runat="server" CssClass="NormalInputText" Width="99%"
             MaxLength="50"></asp:TextBox>
   
         <asp:Literal ID="ltSearch" runat="server"></asp:Literal>
         <span class="requiredSign">*</span>
     </td>
                        
```

```
    protected void grdINASN_D_RowDataBound(object sender, GridViewRowEventArgs e)
    {

      this.OpenFloatWin(linkModify, BuildRequestPageURL("FrmINASN_DEdit.aspx?Flag=1&ids=" + strKeyID + "&ErpCode=" + txtCERPCODE.Text.Trim() + "&InType=" + txtITYPE.SelectedValue.Trim(), SYSOperation.Modify, strKeyID), Resources.Lang.FrmINASN_DEdit_Content1, "INASN_D", 1000, 500);
    }
```
