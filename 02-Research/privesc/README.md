# Privilege Escalation

- [Linux privesc](linux/README.md)
- [Windows privesc](windows/README.md)

```dataview
TABLE category AS "Category"
FROM "OffensiveIndex/02-Research/privesc"
WHERE file.name != "README"
SORT file.name ASC
```