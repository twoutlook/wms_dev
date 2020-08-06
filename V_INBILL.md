
# before 
```

SELECT ib.id, dbo.Fun_GetOperatorInfo(ib.ccreateownercode, '1') AS ccreateownercode, ib.dcreatetime, dbo.Fun_GetOperatorInfo(ib.cauditperson, '1') AS cauditperson, ib.daudittime, ib.cticketcode, ib.cstatus, sp.flag_name AS status_name, 
                  ib.casnid, ia.cticketcode AS asncode, ib.dindate, ib.cmemo, ib.cerpcode, it.typename, ib.ddefine3, ib.debittime, ib.debitowner, ib.itype, CASE WHEN ib.WORKTYPE = 1 THEN ib.palletcode ELSE '' END AS palletcode, ib.worktype, 
                  parameter.flag_name AS worktypeName, ib.operationtype
FROM     dbo.INBILL AS ib WITH (NOLOCK) LEFT OUTER JOIN
                  dbo.INASN AS ia WITH (NOLOCK) ON ib.casnid = ia.id LEFT OUTER JOIN
                  dbo.INTYPE AS it WITH (NOLOCK) ON ib.itype = it.cerpcode LEFT OUTER JOIN
                  dbo.SYS_PARAMETER AS sp WITH (NOLOCK) ON ib.cstatus = sp.flag_id AND sp.flag_type = 'INBILL' LEFT OUTER JOIN
                  dbo.SYS_PARAMETER AS parameter ON ia.WORKTYPE = parameter.flag_id AND parameter.flag_type = 'WorkType'
WHERE  (ia.cticketcode IS NOT NULL)

```



# after 1
```

SELECT ib.id, dbo.Fun_GetOperatorInfo(ib.ccreateownercode, '1') AS ccreateownercode, ib.dcreatetime, dbo.Fun_GetOperatorInfo(ib.cauditperson, '1') AS cauditperson, ib.daudittime, ib.cticketcode, ib.cstatus, sp.flag_name AS status_name, 
                  ib.casnid, ia.cticketcode AS asncode, ib.dindate, ib.cmemo, ib.cerpcode, it.typename, ib.ddefine3, ib.debittime, ib.debitowner, ib.itype, CASE WHEN ib.WORKTYPE = 1 THEN ib.palletcode ELSE '' END AS palletcode, ib.worktype, 
                  parameter.flag_name AS worktypeName, ib.operationtype, ib.ProprietorId
FROM     dbo.INBILL AS ib WITH (NOLOCK) LEFT OUTER JOIN
                  dbo.INASN AS ia WITH (NOLOCK) ON ib.casnid = ia.id LEFT OUTER JOIN
                  dbo.INTYPE AS it WITH (NOLOCK) ON ib.itype = it.cerpcode LEFT OUTER JOIN
                  dbo.SYS_PARAMETER AS sp WITH (NOLOCK) ON ib.cstatus = sp.flag_id AND sp.flag_type = 'INBILL' LEFT OUTER JOIN
                  dbo.SYS_PARAMETER AS parameter ON ia.WORKTYPE = parameter.flag_id AND parameter.flag_type = 'WorkType'
WHERE  (ia.cticketcode IS NOT NULL)
```
# after 2

```
SELECT ib.id, dbo.Fun_GetOperatorInfo(ib.ccreateownercode, '1') AS ccreateownercode, ib.dcreatetime, dbo.Fun_GetOperatorInfo(ib.cauditperson, '1') AS cauditperson, ib.daudittime, ib.cticketcode, ib.cstatus, sp.flag_name AS status_name, 
                  ib.casnid, ia.cticketcode AS asncode, ib.dindate, ib.cmemo, ib.cerpcode, it.typename, ib.ddefine3, ib.debittime, ib.debitowner, ib.itype, CASE WHEN ib.WORKTYPE = 1 THEN ib.palletcode ELSE '' END AS palletcode, ib.worktype, 
                  parameter.flag_name AS worktypeName, ib.operationtype, ib.ProprietorId, dbo.Base_Proprietor.ProprietorCode
FROM     dbo.INBILL AS ib WITH (NOLOCK) LEFT OUTER JOIN
                  dbo.Base_Proprietor ON ib.ProprietorId = dbo.Base_Proprietor.Id LEFT OUTER JOIN
                  dbo.INASN AS ia WITH (NOLOCK) ON ib.casnid = ia.id LEFT OUTER JOIN
                  dbo.INTYPE AS it WITH (NOLOCK) ON ib.itype = it.cerpcode LEFT OUTER JOIN
                  dbo.SYS_PARAMETER AS sp WITH (NOLOCK) ON ib.cstatus = sp.flag_id AND sp.flag_type = 'INBILL' LEFT OUTER JOIN
                  dbo.SYS_PARAMETER AS parameter ON ia.WORKTYPE = parameter.flag_id AND parameter.flag_type = 'WorkType'
WHERE  (ia.cticketcode IS NOT NULL)


```
