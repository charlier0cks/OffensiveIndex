# HackTheBox

Each box lives in `machines/<box-name>/` with its note and an `assets/` folder for
screenshots. Challenges go under `challenges/`.

[Cap (Easy)](machines/cap/cap.md)

## Machine Tracker

```dataview
TABLE os AS "OS", difficulty AS "Difficulty", status AS "Status", date AS "Date"
FROM "OffensiveIndex/01-CTF/hackthebox/machines"
SORT date DESC
```

## Rooted count

```dataview
LIST
FROM "OffensiveIndex/01-CTF/hackthebox/machines"
WHERE status = "rooted"
```