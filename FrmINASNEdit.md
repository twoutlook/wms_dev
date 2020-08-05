# FrmINASNEdit

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
