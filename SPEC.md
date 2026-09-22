# SPEC.md

## 1. Authority and status

This document is the product and acceptance source of truth for **M18 E2E Acceptance**. Requirement keywords **MUST**, **MUST NOT**, **SHOULD**, and **MAY** are normative.

Where implementation details are not fixed here, `AGENTS.md` governs implementation practice. If the documents conflict, this specification takes precedence.

- Project ID: `4df60922-ca40-439a-96d7-850f30d539a7`
- Repository visibility: `public`
- Application type: local command-line application

## 2. Product intent

Provide a small, reliable command-line application for maintaining a persistent list of favourite rock albums. A user can add an album, list stored albums, delete an album, and export all albums to a CSV file. Album data persists between invocations in SQLite.

The application treats every stored entry as a rock album; genre is not stored separately.

## 3. Scope

### 3.1 In scope

1. Add an album with an artist and album title.
2. List all stored albums with stable numeric IDs.
3. Delete one album by its numeric ID.
4. Export all stored albums to a user-selected CSV file.
5. Persist records in a user-selected or default SQLite database file.
6. Provide deterministic command output and exit statuses suitable for people and automation.
7. Include automated tests.
8. Run the tests in GitHub Actions on Python 3.14.

### 3.2 Non-goals

The initial release MUST NOT add requirements for:

- Editing an existing record.
- Searching, filtering, user-selected sorting, pagination, or interactive menus.
- Storing genre, year, label, rating, cover art, or external identifiers.
- Duplicate detection or uniqueness enforcement.
- User accounts, authentication, synchronization, networking, or cloud storage.
- CSV import or export formats other than the required CSV format.
- A graphical or web interface.
- Compatibility with databases other than SQLite.
- Supporting out-of-band modification of the application-owned schema.
- Spreadsheet-specific formula rewriting or sanitization; exported text preserves the stored album data.
- An installed shell command or packaging for a package index; module invocation is the guaranteed interface.

## 4. Runtime and platform

- The supported runtime MUST be CPython 3.14.x.
- The application MUST run from the repository root as `python -m rock_albums`.
- Production code MUST use only the Python standard library.
- The application MUST operate without network access.
- UTF-8-compatible artist and title text MUST be supported through Python's normal Unicode handling.
- CI MUST run on a GitHub-hosted Ubuntu runner with Python 3.14.

## 5. Command-line interface

### 5.1 Command grammar

The public interface is:

```text
python -m rock_albums [--database PATH] add --artist ARTIST --title TITLE
python -m rock_albums [--database PATH] list
python -m rock_albums [--database PATH] delete ID
python -m rock_albums [--database PATH] export --output CSV_PATH
python -m rock_albums --help
```

`--database` is a global option and appears before the subcommand. If omitted, it MUST default to `rock_albums.sqlite3` in the current working directory.

The application MUST NOT prompt for missing arguments. Invalid or missing arguments MUST be reported as command-line usage errors.

### 5.2 Add

`add` MUST:

1. Require `--artist` and `--title` exactly once each.
2. Remove leading and trailing whitespace from both values before storage.
3. Reject either value if it becomes empty after trimming.
4. Reject values containing tab, carriage-return, line-feed, or NUL characters because list output is tab-separated and line-oriented.
5. Insert one row in a transaction.
6. Allow duplicate artist/title pairs.
7. Print exactly the following success line to standard output, followed by a newline:

```text
Added album <ID>.
```

`<ID>` is the generated positive integer database ID.

### 5.3 List

`list` MUST print a tab-separated table to standard output. The first line MUST be:

```text
ID\tARTIST\tTITLE
```

Each record MUST then appear on its own line in ascending ID order:

```text
<ID>\t<ARTIST>\t<TITLE>
```

If there are no albums, only the header line MUST be printed. Listing MUST NOT modify album records, although opening a new database may initialize its schema.

### 5.4 Delete

`delete` MUST:

1. Accept one positive integer ID.
2. Delete only the row whose ID exactly matches the argument.
3. Perform the deletion in a transaction.
4. Print exactly the following success line to standard output, followed by a newline:

```text
Deleted album <ID>.
```

If the ID does not exist, the command MUST leave the database unchanged, print an error to standard error, and exit with operational failure status `1`.

### 5.5 Export

`export` MUST:

1. Require `--output CSV_PATH` exactly once.
2. Read all albums from one consistent database view in ascending ID order.
3. Create a CSV file at the selected path containing the header fields `ID`, `ARTIST`, and `TITLE` in that order.
4. Include one record for every stored album and no additional records.
5. Preserve artist and title text exactly as stored.
6. Produce a header-only CSV file when the database contains no albums.
7. Leave album records unchanged, although opening a new database may initialize its schema.
8. Print a success line only after the complete CSV file has been installed at the destination.

The success line MUST be:

```text
Exported 1 album.
```

when exactly one album is exported, and otherwise:

```text
Exported <COUNT> albums.
```

`<COUNT>` is the non-negative number of exported album records.

#### 5.5.1 CSV representation

The exported file MUST:

- Be encoded as UTF-8 without a byte-order mark.
- Use a comma as the delimiter.
- Use `"` as the quote character.
- Escape an embedded quote by doubling it.
- Quote fields when required by the standard-library `csv` minimal-quoting behavior.
- Use `\n` as the record terminator.
- End every record, including the final record, with `\n`.
- Represent IDs as base-10 integers without additional formatting.

The first record MUST therefore serialize exactly as:

```text
ID,ARTIST,TITLE
```

CSV serialization MUST use the Python standard-library `csv` module rather than handcrafted delimiter escaping.

#### 5.5.2 Destination behavior

- The output path MUST be interpreted as a local filesystem path.
- The application MUST NOT recursively create a missing output parent directory.
- If a regular destination file exists, the completed export MUST replace it.
- Replacement MUST occur only after the complete new CSV has been written successfully to a temporary sibling file.
- A database read, CSV write, close, or replacement failure MUST leave a pre-existing destination file unchanged when replacement has not completed.
- Temporary export files MUST be removed on handled failure paths.
- The output path MUST NOT identify the database file itself. Paths resolving to the same existing file, including aliases detectable through normal path resolution or same-file checks, MUST be rejected as input-validation errors before replacement.
- An output directory, missing parent directory, permission failure, or other filesystem export failure MUST be reported as an operational failure with status `1`.

### 5.6 Help, errors, and exit statuses

- Successful commands and explicit help MUST exit with status `0`.
- Usage and input-validation errors MUST exit with status `2`.
- Database failures, unsupported schemas, missing delete targets, and CSV filesystem failures MUST exit with status `1`.
- Errors MUST be written to standard error, and the first error line MUST begin with `error:`.
- A failed command MUST NOT print its success line.
- Expected user or operational errors MUST NOT emit a Python traceback.
- Help text MUST identify the `add`, `list`, `delete`, and `export` subcommands and the `--database` option.
- Export help MUST identify the required `--output` option.

## 6. Persistence model

### 6.1 Database ownership and lifecycle

The application owns its SQLite schema. On the first invocation against a new usable database path, it MUST create the database and schema as part of a transaction. SQLite may create the database file, but the application MUST NOT recursively create a missing parent directory.

Connections MUST be closed on all normal and handled-error paths. Separate application invocations using the same database path MUST observe committed changes from earlier invocations.

### 6.2 Schema version 1

SQLite `PRAGMA user_version` MUST be used, with initial schema version `1`.

The version 1 logical schema is:

```sql
CREATE TABLE albums (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    artist TEXT NOT NULL CHECK (length(artist) > 0),
    title TEXT NOT NULL CHECK (length(title) > 0)
);
```

Required behavior:

- A new database with no `albums` table MUST be initialized transactionally and assigned `user_version = 1`.
- A database with `user_version = 1` MUST be used without destructive recreation.
- A database with an unsupported nonzero `user_version` MUST be rejected with exit status `1`.
- If `user_version = 0` but an `albums` object already exists, the application MUST fail rather than overwrite or silently adopt an unknown schema.
- No migration beyond schema version 1 is required.
- IDs MUST NOT be reused after deletion under normal application operation.

### 6.3 Transaction behavior

- Schema initialization, add, and delete operations MUST be atomic.
- Failed writes MUST be rolled back.
- List and export queries MUST explicitly order records by `id ASC`.
- SQL values MUST be passed through parameterized statements.
- No command may construct SQL by interpolating artist, title, ID, or database path values.

## 7. Component and trust boundaries

The implementation MUST preserve these boundaries:

1. **CLI boundary:** parses arguments, coordinates operations, formats terminal output, maps known failures to exit statuses, and performs no direct SQL.
2. **Validation boundary:** normalizes and validates artist/title values and validates command inputs before persistence or export.
3. **Storage boundary:** owns SQLite connection handling, schema initialization, transactions, and parameterized SQL.
4. **CSV export boundary:** owns CSV serialization and safe destination replacement; it performs no SQL.
5. **Filesystem boundary:** database and output paths are local user input. The process uses the invoking user's permissions and MUST NOT attempt privilege changes.

Album text is data only. It MUST never be interpreted by this application as SQL, shell syntax, a filename, or executable content. The application MUST NOT invoke a shell or make network requests.

## 8. Security and data integrity

- All SQL data parameters MUST be bound parameters.
- Expected database and export exceptions MUST be converted to concise CLI errors without tracebacks.
- Existing database content MUST NOT be deleted to repair a version or schema mismatch.
- Export MUST NOT overwrite or replace the selected SQLite database.
- CSV fields MUST be serialized through the standard-library `csv` module.
- The application MUST NOT collect telemetry or transmit album data.
- The application MUST NOT embed credentials or require repository secrets.
- Tests and CI MUST use temporary database and export files and MUST NOT modify a developer's default database.
- SQLite files, journals, generated CSV fixtures, caches, and other runtime artifacts MUST NOT be committed.

CSV consumers are responsible for treating exported album values as untrusted data. Filesystem access controls and backup policy remain the responsibility of the invoking user.

## 9. Interfaces

### 9.1 Public interface

The command-line interface in section 5 is the only required public application interface.

### 9.2 Internal interface expectations

- `rock_albums.__main__` MUST terminate using the integer returned by the CLI entry point.
- The CLI entry point SHOULD have the form `main(argv: list[str] | None = None) -> int` so it can be tested without mutating global argument state.
- Storage operations MUST return domain data or raise defined application-level exceptions; they MUST NOT print directly.
- CSV export operations MUST accept album records as data and MUST NOT import or query SQLite directly.
- The CSV exporter MUST return completion information or raise a defined application-level export exception; it MUST NOT print directly.

No HTTP, plugin, extension, or stable Python library API is required.

## 10. Dependencies

### 10.1 Runtime

There MUST be no third-party runtime dependencies. Expected standard-library facilities include:

- `argparse`
- `csv`
- `os`
- `pathlib`
- `sqlite3`
- `sys`
- `tempfile`

### 10.2 Testing

Tests MUST use the standard-library `unittest` framework. `tempfile`, `subprocess`, and related standard-library modules MAY be used. No external test runner is required.

### 10.3 CI

GitHub Actions may use reviewed marketplace actions for checkout and Python setup. Actions MUST be pinned to immutable full commit SHAs, with a nearby comment naming the corresponding reviewed release version.

## 11. Required repository artifacts

The repository MUST contain at least:

```text
SPEC.md
AGENTS.md
README.md
rock_albums/
    __init__.py
    __main__.py
    cli.py
    storage.py
    exporting.py
tests/
.github/workflows/tests.yml
```

A focused validation or domain module MAY be introduced when it preserves the boundaries in this specification.

The README MUST provide Python 3.14 prerequisites, examples for all four commands, the default database location, the CSV format and replacement behavior, test instructions, and the exit-status contract. It MUST not contradict this specification.

## 12. Automated testing requirements

Tests MUST cover at least:

1. Fresh database initialization and schema version.
2. Empty list output.
3. Adding and listing one album.
4. Multiple albums listed in ascending ID order.
5. Persistence across separate CLI processes.
6. Deleting an existing album.
7. Deleting a missing ID without mutating existing rows.
8. Duplicate artist/title pairs being accepted.
9. Whitespace trimming.
10. Rejection of empty or line-breaking artist/title values.
11. Invalid and non-positive delete IDs.
12. Unsupported schema-version failure without destructive changes.
13. Operational database errors returning status `1` without a traceback.
14. Empty-database export producing exactly the CSV header.
15. Export of multiple albums in ascending ID order.
16. CSV handling of commas, quotes, and non-ASCII text.
17. Replacement of a pre-existing regular output file.
18. Export failure preserving a pre-existing destination when replacement has not completed.
19. Rejection when output identifies the database file.
20. Export failure for a missing output parent or invalid destination.
21. Exact success messages, list header, stdout/stderr routing, and relevant exit statuses.

Integration tests MUST invoke `python -m rock_albums` in subprocesses with explicit temporary `--database` and export paths.

## 13. GitHub Actions requirements

`.github/workflows/tests.yml` MUST:

- Run on pushes and pull requests.
- Use a GitHub-hosted Ubuntu runner.
- Install or select CPython 3.14.
- Use minimal permissions, including `contents: read`.
- Use no secrets and no `pull_request_target` trigger.
- Run from a clean checkout.
- Execute:

```text
python -m unittest discover -s tests -v
```

- Fail when any test fails.
- Include a finite job timeout.

## 14. Milestones

### M1 — CLI and persistence foundation

Create the package entry point, argument parser, validation, SQLite schema initialization, and storage operations.

### M2 — Command and export behavior

Complete add, list, delete, and CSV export behavior with the specified formatting, ordering, transactions, atomic output replacement, errors, and exit statuses.

### M3 — Automated verification

Add unit and subprocess integration tests covering success paths, validation, persistence, CSV encoding, destination safety, and operational failures.

### M4 — CI and documentation

Add the Python 3.14 GitHub Actions workflow and user-facing README, then verify repository hygiene and all acceptance criteria.

## 15. Acceptance criteria

The product is accepted only when all of the following are true:

1. On CPython 3.14, `python -m rock_albums --database <fresh-temp-path> list` exits `0`, creates schema version 1, and prints exactly `ID\tARTIST\tTITLE` plus a newline.
2. Adding `--artist "Black Sabbath" --title "Paranoid"` to a fresh database exits `0`, reports ID `1`, and persists the normalized values.
3. Adding `--artist "Pink Floyd" --title "The Wall"` next reports ID `2`; listing prints IDs `1` then `2` in the required tab-separated format.
4. A separate process using the same database path observes both records.
5. Deleting ID `1` exits `0`, prints `Deleted album 1.`, and subsequent listing contains ID `2` but not ID `1`.
6. Deleting an absent ID exits `1`, writes an `error:` message only to standard error, and does not change remaining records.
7. Invalid text and invalid IDs fail with status `2` and do not perform a write.
8. Duplicate artist/title records can be added as distinct IDs.
9. Exporting an empty database exits `0`, prints `Exported 0 albums.`, and creates a UTF-8 file containing exactly `ID,ARTIST,TITLE\n`.
10. Exporting the two example records exits `0`, prints `Exported 2 albums.`, and writes the header followed by IDs `1` and `2` in ascending order.
11. Artists and titles containing commas, quotes, and non-ASCII text are represented according to the specified CSV rules and retain their stored values when parsed with the standard-library `csv` module.
12. A successful export replaces an existing regular output file only after the complete new CSV is ready.
13. An export filesystem failure exits `1`, emits no success line or traceback, and does not alter a pre-existing destination when replacement has not occurred.
14. Attempting to export to the database file is rejected with status `2` without replacing or corrupting the database.
15. An unsupported nonzero schema version fails safely without replacing or clearing database content.
16. `python -m unittest discover -s tests -v` passes under Python 3.14 from the repository root.
17. GitHub Actions runs the same suite successfully on pushes and pull requests using Python 3.14.
18. No third-party runtime or test dependency is needed.
19. The repository visibility is `public`.
