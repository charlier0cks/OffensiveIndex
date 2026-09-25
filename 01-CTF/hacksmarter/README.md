# HackSmarter

Each box/range lives in `machines/<box-name>/` with its note and an `assets/` folder for
screenshots.

## Tracker

```dataview
TABLE os AS "OS", difficulty AS "Difficulty", status AS "Status", date AS "Date"
FROM "OffensiveIndex/01-CTF/hacksmarter/machines"
WHERE file.name != "README"
SORT date DESC
```