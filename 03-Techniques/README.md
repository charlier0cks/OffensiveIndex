# Techniques

Atomic, reusable notes, one technique per file. Each answers *"how does this work
and how do I run it"* independent of any single box.
## Index

```dataview
TABLE category AS "Category", mitre AS "MITRE", tools AS "Tools"
FROM "OffensiveIndex/03-Techniques"
WHERE type = "technique"
SORT category ASC, file.name ASC
```