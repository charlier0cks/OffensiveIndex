# Windows Privilege Escalation

Token abuse, service misconfig, unquoted paths, AlwaysInstallElevated, DLL hijacking, etc.

```dataview
LIST
FROM "OffensiveIndex/02-Research/privesc/windows"
WHERE file.name != "README"
```