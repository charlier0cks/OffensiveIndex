# Research

Deep-dive research organized by domain.

## Domains
- [Active Directory](active-directory/README.md)
- [Web](web/README.md)
- [Privilege Escalation](privesc/README.md)
- [C2 & Evasion](c2-and-evasion/README.md)
- [Initial Access](initial-access/README.md)
- [Post-Exploitation](post-exploitation/README.md)
- [Cloud](cloud/README.md)

## All research notes

```dataview
TABLE category AS "Category", tags AS "Tags"
FROM "OffensiveIndex/02-Research"
WHERE file.name != "README"
SORT file.name ASC
```