# AGENTS.md

## 1. Purpose and authority

This file defines durable engineering rules for agents and contributors implementing **M18 E2E Acceptance**.

`SPEC.md` is the product source of truth. This file governs implementation technique where the specification is silent. In a conflict, follow `SPEC.md`. Do not silently reinterpret or weaken a requirement to make implementation easier.

Repository visibility MUST remain `public`.

## 2. Working principles

1. Read `SPEC.md` before changing code, tests, workflows, or documentation.
2. Make the smallest coherent change that satisfies the applicable requirement.
3. Preserve user data and pre-existing export destinations on failures before replacement.
4. Never solve schema errors by deleting, replacing, or silently recreating an existing database.
5. Keep behavior deterministic and suitable for subprocess testing.
6. Do not add features from the non-goals list without an approved specification change.
7. Do not claim commands, tests, CI, settings, or repository operations were executed unless they actually were.
8. If a requested change cannot be reconciled with `SPEC.md`, stop and surface the conflict instead of inventing a requirement.

## 3. Required architecture

Maintain these responsibilities:

```text
rock_albums/
    __init__.py     package metadata only; no database or CLI side effects
    __main__.py     calls the CLI entry point and raises SystemExit
    cli.py          parser, coordination, terminal output, exit mapping
    storage.py      SQLite lifecycle, schema, transactions, and CRUD queries
    exporting.py    CSV serialization and safe destination replacement
```

A separate validation or domain module MAY be added if it materially improves clarity. Avoid unnecessary layers for this small application.

### 3.1 CLI layer

- Expose `main(argv: list[str] | None = None) -> int`.
- Build arguments with the standard library.
- Keep `sys.exit` and `SystemExit` in `__main__.py`; return integer statuses from `main`.
- Send normal command output to stdout and errors to stderr.
- Ensure the first error line begins with `error:`.
- Do not perform SQL or CSV serialization directly in CLI handlers.
- Catch only failures that can be translated into defined user-facing errors.
- Do not catch `KeyboardInterrupt`, `SystemExit`, or `BaseException`.
- Do not leak expected tracebacks to users.

### 3.2 Validation

- Normalize artist and title exactly once before storage.
- Apply the trimming and prohibited-character rules from `SPEC.md`.
- Treat malformed or non-positive IDs as usage errors before invoking a write.
- Require command options exactly as specified; do not silently accept duplicate occurrences by using the last value.
- Validate that export output does not identify the database path before destination replacement.
- For path identity, compare normalized absolute paths and use `os.path.samefile` when both paths exist. Handle same-file check failures without bypassing the required safety check.
- Do not introduce arbitrary album-text or export-path length limits without a specification change.

### 3.3 Storage layer

- Own all production `sqlite3` imports and SQL statements in the storage boundary. Tests MAY use `sqlite3` directly to create and inspect fixtures.
- Open connections per operation or per explicit short-lived storage context; do not use a module-global connection.
- Ensure connections and cursors are released deterministically.
- Use bound parameters for every SQL data value.
- Do not interpolate values into SQL, including integer IDs.
- Use explicit transactions for initialization and mutations.
- Roll back failed mutations and commit only complete operations.
- Use `PRAGMA user_version` exactly as defined by the schema contract.
- Treat unknown or ambiguous schemas as errors; never attempt destructive recovery.
- Do not use `INSERT OR REPLACE`, because replacement can delete and recreate rows.
- Return records using explicit `ORDER BY id ASC`; never rely on SQLite's incidental row order.
- Keep SQL schema text centralized so implementation and tests cannot drift across production copies.
- Storage functions MUST return records or raise focused application exceptions. They MUST NOT print or write CSV files.

### 3.4 CSV export layer

- Use the standard-library `csv` module with explicit delimiter, quote, quoting, encoding, and line-terminator behavior matching `SPEC.md`.
- Open text files with UTF-8 encoding and newline handling that prevents platform newline translation from changing the specified `\n` records.
- Do not construct CSV rows through string joining or manual quote replacement.
- Serialize the header and records to a temporary file in the destination directory.
- Close the temporary file before replacement.
- Replace an existing regular destination only after serialization and close have completed successfully.
- Use `os.replace` or an equivalent standard-library atomic replacement primitive; do not copy partial data over the destination.
- Clean up temporary files on handled failures without deleting the pre-existing destination.
- Do not recursively create a missing destination parent.
- Do not print from the export layer.
- Do not add spreadsheet-formula prefixes or otherwise change stored text during export.

## 4. Command contract discipline

The CLI grammar, exact success messages, list format, CSV representation, and exit statuses in `SPEC.md` are compatibility contracts.

- Do not add decorative text, colors, prompts, logging, or progress output.
- Do not print database initialization messages.
- Do not change list or CSV headers, delimiters, ordering, or line endings.
- Do not print the export destination path as part of the defined success line.
- Do not put errors on stdout.
- Do not infer deletion by artist or title; deletion is by generated ID.
- Do not suppress duplicate entries.
- Do not add implicit environment-variable configuration unless the specification is revised.

When modifying parser or command behavior, add or update subprocess tests for stdout, stderr, exit status, and filesystem effects.

## 5. Dependency policy

- Production and test code MUST use only the Python 3.14 standard library.
- Do not add runtime, development, test, formatting, migration, CSV, or CLI framework dependencies.
- Do not introduce a lockfile for an empty dependency set.
- If a future requirement genuinely needs a third-party dependency, revise `SPEC.md` explicitly before adding it and document the reason and security implications.
- GitHub Actions are CI dependencies. Pin every action to an immutable full commit SHA and include a nearby comment with the reviewed release version.
- Do not download or execute arbitrary scripts in CI.

## 6. Python implementation rules

- Target CPython 3.14 and use clear type annotations for public internal functions.
- Prefer straightforward standard-library code over metaprogramming or abstraction frameworks.
- Keep import-time behavior side-effect free. Importing the package MUST NOT parse arguments, open a database, write a CSV file, or print.
- Use `pathlib.Path` for ordinary filesystem-path handling while passing supported path representations to standard-library APIs.
- Use context managers or `try/finally` for resources, transaction cleanup, and temporary-file cleanup.
- Define focused application exceptions for unsupported schemas, missing records, database failures, invalid export destinations, and export I/O failures.
- Preserve exception chaining internally where useful, but render concise user-facing errors.
- Avoid broad `except Exception` unless it is at a narrowly defined CLI boundary and maps only documented operational failures while preserving interrupts and system-exit behavior.
- Keep functions small enough that transaction, output replacement, and error paths are apparent during review.
- Do not expose temporary export filenames in normal output or unnecessary error details.

## 7. Testing rules

Use `unittest`. Tests are part of the product and MUST be deterministic, isolated, and runnable in any order.

### 7.1 Unit tests

Unit tests SHOULD cover:

- Text normalization and validation.
- Delete-ID validation.
- Database/output path identity checks.
- Schema initialization and version checks.
- Storage add, list, and delete behavior.
- Transaction rollback on expected failures.
- CSV serialization, quoting, UTF-8 text, and line endings.
- Export replacement and cleanup behavior.
- CLI argument-to-status mapping where direct invocation is clearer than subprocess use.

### 7.2 Integration tests

Subprocess tests MUST:

- Invoke the application with `sys.executable -m rock_albums`.
- Use an explicit database path inside `tempfile.TemporaryDirectory`.
- Use export paths inside test-controlled temporary directories.
- Capture stdout and stderr as text.
- Assert exact output where `SPEC.md` defines exact output.
- Inspect exported files as bytes where encoding or line endings are under test.
- Parse representative exported files with the standard-library `csv` module to verify field preservation.
- Verify persistence by invoking separate processes.
- Avoid relying on the developer's home directory, default database, network, locale-specific error details, or test execution order.

### 7.3 Required failure-path coverage

Tests MUST demonstrate that:

- Validation failures do not insert rows or replace export destinations.
- Missing deletes do not remove another row.
- Unsupported schema versions are not overwritten.
- Database operational failures do not produce success output or expected-user tracebacks.
- Duplicate records remain separate rows.
- IDs are ordered and are not reused after normal deletion.
- Export filesystem failures do not produce success output or expected-user tracebacks.
- A pre-existing output remains unchanged when export fails before replacement.
- Temporary export files are cleaned up on handled failures.
- Export to the database file, including a detectable alias, is rejected without database corruption.
- Commas, quotes, and non-ASCII album text are serialized correctly.
- Empty exports contain the header and no data rows.

Do not weaken assertions merely to make a failing implementation pass. Fix the implementation or obtain an explicit specification revision if the contract is wrong.

## 8. CI rules

The workflow at `.github/workflows/tests.yml` MUST remain simple and auditable.

- Trigger on `push` and `pull_request` only as required.
- Do not use `pull_request_target`.
- Declare minimal permissions, including `contents: read`.
- Set a finite `timeout-minutes` value.
- Use an Ubuntu GitHub-hosted runner and Python 3.14.
- Pin checkout and Python setup actions to reviewed full commit SHAs.
- Run `python -m unittest discover -s tests -v` from the repository root.
- Do not require secrets, service containers, external databases, package publication, or network services beyond obtaining the runner and pinned actions.
- Do not hide failures with `continue-on-error`, shell `|| true`, or equivalent constructs.
- Do not make the required test job conditional on changed file paths.

The documented local test command and the CI test command MUST exercise the same suite.

## 9. Security rules

- Treat all CLI values, filesystem paths, and database contents as untrusted data.
- Use SQL parameter binding without exceptions.
- Never invoke a shell with album values, IDs, database paths, or export paths.
- Use `csv`, not handcrafted escaping, for exported fields.
- Never replace the SQLite database with an export file.
- Do not add network calls, telemetry, analytics, or crash reporting.
- Do not print credentials, environment dumps, temporary-file paths, or unrelated filesystem information in errors.
- Do not change filesystem permissions, follow a privilege-escalation path, or attempt to manage user accounts.
- Do not recursively create database or export parent directories.
- Do not repair unknown databases destructively.
- Do not mutate album text to address behavior in downstream spreadsheet applications; preserve data and document that CSV consumers must treat values as untrusted.
- Keep GitHub Actions free of secrets and grant no write permission unless a future specification explicitly requires it.
- Review any future action SHA or dependency change as a supply-chain-sensitive modification.

## 10. Repository hygiene

Do not commit generated artifacts, including:

- `rock_albums.sqlite3` or other SQLite database files.
- SQLite journal, WAL, or shared-memory files.
- Generated CSV exports or temporary export files.
- `__pycache__`, `.pyc`, coverage output, temporary directories, or editor state.
- Credentials, tokens, local environment files, or test databases.

Maintain an appropriate `.gitignore`. Do not add a license on the assumption that public visibility selects one; licensing requires an explicit project decision.

README examples MUST match the real interface and `SPEC.md`, including the export command and CSV contract. Documentation changes are required whenever user-visible behavior changes.

## 11. Change protocol

For every change:

1. Identify the applicable requirement IDs or specification sections.
2. Preserve the boundaries between CLI, validation, storage, and CSV export.
3. Add or update tests before considering the change complete.
4. Run the complete standard-library test suite under Python 3.14 when execution is available.
5. Review exact stdout, stderr, and exit statuses for changed CLI paths.
6. Review transaction and rollback behavior for changed persistence paths.
7. Review temporary-file cleanup, path identity checks, and destination preservation for changed export paths.
8. Update README and source-of-truth documents if the approved contract changes.
9. Inspect the diff for databases, CSV exports, temporary files, caches, secrets, mutable action tags, and unrelated edits.

Do not edit `SPEC.md` solely to describe accidental implementation behavior. Product-contract changes must be deliberate.

## 12. Definition of done

A milestone or change is done only when:

- Its implementation conforms to `SPEC.md`.
- Required tests exist and pass on CPython 3.14.
- The full suite remains runnable with the documented command.
- CLI output and exit statuses match the contract.
- Database operations are parameterized, transactional, and non-destructive on failure.
- CSV output matches the encoding, quoting, ordering, and line-ending contract.
- Export replacement is safe, temporary files are cleaned up, and the database cannot be selected as the destination.
- CI remains pinned, minimal-permission, secret-free, and passing.
- README instructions are accurate.
- No generated database, CSV, cache, credential, or unrelated artifact is included.
- No unapproved dependency or feature has been introduced.
