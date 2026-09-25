# Ronin66

Each box lives in `machines/<box-name>/` with its note and an `assets/` folder for
screenshots.

## Tracker

```dataview
TABLE os AS "OS", difficulty AS "Difficulty", status AS "Status", date AS "Date"
FROM "OffensiveIndex/01-CTF/ronin66/environments"
WHERE file.name != "README"
SORT date DESC
```