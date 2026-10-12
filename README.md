# timetable-cli

Your weekly class timetable in the terminal. A command-line tool in plain Python with no dependencies. The timetable is saved in `timetable.json`.

## Running it

You need Python 3.8 or newer.

```
python3 -m timetable show monday
python3 -m timetable week
python3 -m timetable now
python3 -m timetable export
python3 -m timetable filter --day monday --start 09:00 --end 12:00
python3 -m timetable add --day friday --subject "Maths" --start 09:00 --end 10:00 --room "Room 101"
python3 -m unittest discover tests
```

On Windows, use `python` instead of `python3`.

## Commands

| Command | Does |
|---|---|
| `show <day>` | One day's classes |
| `week` | The whole week, Monday to Sunday |
| `now` | What's on right now, and what's next |
| `export [--output FILE]` | Writes a `timetable.ics` calendar file |
| `filter [--day --start --end --room]` | Classes matching a day, time range and/or room |
| `add --day --subject --start --end --room` | Adds a class |
| `diff <first> <second>` | Compares two timetable files and reports what changed |
| `merge <first> <second> [--output OUT] [--force]` | Combines two timetables into one with conflict detection |

## How it's supposed to work

- Classes are always listed in order of start time, however they were added.
- Day names ignore capital letters: `Monday`, `monday` and `MONDAY` are the same. A name that isn't a day, like `mondy`, is an error that lists the valid days, for both `show` and `add`.
- Times are 24-hour `HH:MM`. `add` refuses a time that doesn't exist, like `25:99`, and a class that ends before it starts.
- `add` refuses a class that overlaps another one on the same day, and says which one it clashes with.
- Each class shows how long it is, in minutes.

## What's on now

```
python3 -m timetable now
```

Looks at today's date, works out which class is running at this moment, and which one starts next, with a countdown in minutes. If nothing is running it says so, and if the day is finished it says that too.

## Exporting to a calendar

```
python3 -m timetable export
python3 -m timetable export --output timetable.ics
```

Writes an `.ics` file (default name `timetable.ics`) containing every class as a weekly recurring event, so you can import your timetable into Google Calendar, Apple Calendar, Outlook and the like. Each class becomes one `VEVENT` with the subject as the title and the room as the location, repeating every week.

Because `timetable.json` only stores a weekday and a time rather than a date, each weekday is mapped onto a fixed reference date in one week, and that date carries the weekly repeat rule. The same timetable always exports the same file.

## Filtering

```
python3 -m timetable filter --day monday --start 09:00 --end 12:00
python3 -m timetable filter --room "Lab 1"
python3 -m timetable filter --day monday --room "Lab 1" --start 09:00 --end 14:00
```

`filter` shows only the classes matching `--day`, `--start`/`--end` (a time range) and/or `--room`. Every option is optional and they combine: only the classes that satisfy all the given filters are shown. The time range is half-open, so a class matches when it overlaps the range; a class that only touches a boundary (for example one ending exactly at `--start`, or starting exactly at `--end`) is not included. `--day` ignores capital letters and `--room` matches exactly. If nothing matches it just says so.
## Comparing two timetables

```bash
python3 -m timetable diff timetable.json timetable-new.json
```

`diff` takes exactly two timetable JSON files and reports what changed between them. It prints three sections — `Added:` (classes only in the second file), `Removed:` (classes only in the first file) and `Modified:` (classes present in both but with changed fields, listed as `Field: old -> new`). Classes are matched by their day and subject rather than their position in the JSON, so reordering entries is never reported as a change. The output is sorted by weekday and start time.

```text
Timetable changes

Added:
  Tuesday 14:00-15:00 — Operating Systems — Room 204

Removed:
  Wednesday 10:00-11:00 — Mathematics — Room 101

Modified:
  Monday 09:00-10:30 — Data Structures
    Room: Lab 1 -> Lab 2
```

When the timetables are identical it prints `No changes detected. The timetables are identical.` The command exits with status `0` when the files match and non-zero when they differ or an input is missing/invalid, so it can be used in scripts. Neither input file is modified.

## Merging two timetables

```bash
python3 -m timetable merge timetable.json timetable-new.json --output merged.json
```

`merge` combines two timetable JSON files into a single merged timetable file without modifying either input file.

- **Duplicate handling**: Identical classes (same subject, start time, end time, and room on the same day) are deduplicated so only one instance is kept in the output.
- **Conflict detection**: Classes occurring on the same day that overlap in time (evaluated using half-open intervals `[start, end)`) are detected as conflicts. If conflicts exist, the operation halts with exit code `1`, prints all conflicting pairs clearly to stderr, and avoids producing a merged file.
- **Deterministic ordering**: The output merged timetable is deterministically sorted by canonical weekday order (Monday to Sunday) and start time.
- **Output file safety**: The output path cannot match either input path. If the output file already exists, `merge` will refuse to overwrite it unless `--force` is provided.

## Version history and restore

Every command that successfully rewrites `timetable.json` (`add`, `merge --output`, and `restore`) first saves a snapshot of the timetable as it was *before* the change, under a `.timetable-history` directory next to the timetable file (for example `./.timetable-history/`). Snapshots are skipped entirely when a command fails validation, so a rejected `add` never creates one.

```bash
python3 -m timetable history list
python3 -m timetable history list --limit 5
python3 -m timetable history show snap-20260101120000000000-000000
python3 -m timetable history restore snap-20260101120000000000-000000
python3 -m timetable restore snap-20260101120000000000-000000
```

- **Listing**: `history list` prints each snapshot's id, creation time and operation (`add`, `merge`, `edit`, `restore-previous`) in newest-first order, skipping any snapshot whose files are missing or whose checksum no longer matches. `--limit` caps how many are shown (positive integer).
- **Inspecting**: `history show <id>` prints the snapshot's timetable in the usual day view without touching the active file.
- **Restoring**: `history restore <id>` (or the shorter `restore <id>`) replaces the active timetable with the snapshot. Before overwriting, the current contents of the active timetable are saved as a recoverable snapshot with operation `restore-previous`, so nothing is ever lost by restoring. A missing or corrupt snapshot raises an error and leaves the active timetable untouched. The write is atomic, so a failure mid-write cannot leave a truncated file.
- **Snapshot layout**: each snapshot is a directory `<id>/` containing `timetable.json` and `.metadata.json`. The metadata records the id, creation time, source timetable path, operation, and a SHA-256 checksum of the timetable bytes; `list`/`show`/`restore` verify the checksum before trusting the data.
- **Retention**: the newest `TIMETABLE_HISTORY_LIMIT` snapshots are kept (default `10`). Set the environment variable to a positive integer to change it, e.g. `$env:TIMETABLE_HISTORY_LIMIT = "20"` (PowerShell) or `TIMETABLE_HISTORY_LIMIT=20 python3 -m timetable history list` (bash). Older snapshots are pruned after each save and restore; `restore-previous` snapshots are protected from pruning so the pre-restore state stays recoverable.

## Code

- `timetable/loader.py`: reading and writing `timetable.json`
- `timetable/display.py`: printing a day or the week, plus time parsing
- `timetable/now.py`: working out what's on right now
- `timetable/export.py`: writing the `.ics` calendar file
- `timetable/filter.py`: reusable filtering by day, time range and room
- `timetable/diff.py`: comparing two timetable files
- `timetable/merge.py`: combining timetables with deduplication and conflict detection
- `timetable/cli.py`: the commands
- `tests/`: tests, run with `python3 -m unittest discover tests`


## Contributing

Fork the repo, make your changes on a new branch, and open a pull request. Run the tests first.

If you find a bug, open an issue with the steps to reproduce it, what you expected, and what happened instead.

Part of Source Start by CSI SPIT. MIT licensed.
