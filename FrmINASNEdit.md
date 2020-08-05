# FrmINASNEdit

## 新增編輯貨主, 配合 小仲 控件
###  


* FrmINASNEdit_Print
* FrmINASNEdit_UnionPallet

```
  protected void btnNew_Click(object sender, EventArgs e)
    {
        this.btnNew.Enabled = false;
        SaveToDB(sender);
        this.btnNew.Enabled = true;
    }
```

在
```
    protected void grdINASN_D_RowDataBound(object sender, GridViewRowEventArgs e)
    {
          //DOING 【入庫通知單】FrmINASNList-> 【入庫通知單-編輯】FrmINASNEdit -> FrmINASN_DEdit
          this.OpenFloatWin(linkModify, BuildRequestPageURL("FrmINASN_DEdit.aspx?Flag=1&ids=" + strKeyID + "&ErpCode=" + txtCERPCODE.Text.Trim() + "&InType=" + txtITYPE.SelectedValue.Trim(), SYSOperation.Modify, strKeyID), Resources.Lang.FrmINASN_DEdit_Content1, "INASN_D", 1000, 500);
                   
             
    }
```
