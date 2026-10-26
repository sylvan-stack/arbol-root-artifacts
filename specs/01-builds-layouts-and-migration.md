---
role: authored
spec: RTM-1
status: accepted
---
# Root manager · 01 Builds, layouts and migration

Each [Build](../../arbol/GLOSSARY.md#build) of `arbol-root` is made for one [Root](../../arbol/GLOSSARY.md#arbol-root) and carries three files of SQL for it. The *chat tables* file creates the tables of the Root's Root Contract version. The *own tables* file creates the tables that only this Root has, such as the record of where each migrated chat came from. The *migration script*, for a Root that grew from another, brings the chats of the older Root into the new one. For example, Build 7 of `arbol-root`, made for the second Root, holds `layout/chat-tables.sql`, `layout/root-tables.sql` and `layout/migrate.sql`, and `arbol-roots migrate <second>` runs all three with `psql`. This specification states how the `arbol-root` repository keeps these files for every Root, how a Build learns which Root it is made for and what it carries, how `arbol-roots` runs the files, the Root's own layout, the form of a migration script, what becomes of an Exchange that was open, the record of where a chat came from, and the checks a Build passes. It matters because the last upgrade of version 1 tried to convert all of history in place, behind a gate that was still open at its 137th checkpoint, and an earlier cutover cut 34 records short in place.

It follows the conventions of the [Driver Contract](../../arbol/driver-contract/README.md#conventions) and the additions in this chapter's [README](README.md). It gives the forms that [ARC-1](../../arbol/architecture/01-roots.md#moving-chats-between-roots) leaves to the Root, for the second, trivial Root that the first thin path creates and migrates into ([What is built first](../../arbol/architecture/01-roots.md#what-is-built-first), [ARC-D39](../../arbol/architecture/01-roots.md#decisions)). The owner decided on 15 October 2026 that the table layout is a plain file of SQL in the `arbol-root` repository, and that `arbol-roots` runs SQL files with `psql` ([RTM-D8](#decisions)). The other forms are technical choices made the same day. The owner answered the open questions on 15 October 2026, leaving to the agent whether the first thin path also brings single chats over again and checks for chats that went on in the older Root, which it does not ([RTM-D12](#decisions)), and the specification was accepted the same day.

## Scope

Covered: the repository, which Root a Build is made for, what a Build carries, how `arbol-roots` runs the files, the Root's own layout, the migration script, an Exchange that was open, where a migrated chat came from, the checks of a Build.

Not covered: the chat tables themselves ([RCT-3](../contract/03-the-chat-tables.md)); the commands of `arbol-root` ([RTM-0](00-the-commands.md)); creating a Root, and when `arbol-roots` makes a database and runs the files, which belong to [ARC-1](../../arbol/architecture/01-roots.md#the-tray-menu-and-arbol-roots) and Arbol's own specification; Arbol's own tables, such as Drafts, which Arbol's own specification adds to the Root's own layout; bringing one chat over again and marking copies whose original went on, which RTM-D12 leaves for later.

## Behaviour

### The repository

One repository, `arbol-root`, holds the program and the SQL of every Root. The Root Contract's files are shared by every Root of a version; each Root has a folder of its own:

```text
~/repos/sylvan-stack/arbol-root/
  arbol-build.json            the build description; it names no Root
  package.json                the package version, "1.0.0" (BLD-0)
  src/arbol_root/             the program of RTM-0
  tests/                      the checks of a Build, with the stand-in arbol-trunk and the fictional chats they use
  scripts/build.py            copies the files of the Root a Build is made for into the Build
  contract/
    1/layout.sql              the chat tables of Root Contract version 1 (RCT-3)
    conformance/              the Root Contract's conformance suite (RCT-4)
  roots/
    sapling/layout.sql         First Sapling's own tables
    <second>/layout.sql          the second Root's own tables, migrated_chats among them
    <second>/migrate.sql         brings the chats of First Sapling into the second Root
```

<!-- sources:
arbol-root:arbol-build.json
arbol-root:package.json
arbol-root:roots/sapling/layout.sql
arbol-root:scripts/package_build.py#package
arbol-root:README.md
-->

### Which Root a Build is made for

`arbol-builds build` is told the Root ([RTM-D9](#decisions)):

```sh
arbol-roots create <second> --name "<its loud name>" --from sapling    # its root.json: "rootContract": 1, "grewFrom": "sapling"
arbol-builds build arbol-root --root <second>                    # Build 7 of arbol-root, made for the second Root
arbol-roots migrate <second>                                     # runs the three files of Build 7 (below)
```

`--root` on `build` names the Root a Build of `arbol` or `arbol-root` is made for. `arbol-builds` writes it as `root` into the Build's record when the build starts, so the build commands and the checks find it there. `arbol-build.json` names no Root, because one repository serves every Root. [BLD-0](../../arbol/builds/00-builds-and-versions.md#what-a-repository-says-about-its-build) states this since 15 October 2026 (RTM-D9); before, it read the Root from `arbol-build.json`.

<!-- sources:
arbol-root:scripts/package_support.py#destination
arbol-root:scripts/build.py#main
arbol-root:src/arbol_root/model.py#build_record
-->

### What a Build carries

The last build command, `scripts/build.py`, reads `root` from the record in `ARBOL_BUILD_DIR`, then that Root's `root.json` for its `rootContract` and its `grewFrom` (RCT-0, ARC-1), and copies three files unchanged:

| Into the Build | From the repository | When |
|---|---|---|
| `layout/chat-tables.sql` | `contract/<rootContract>/layout.sql` | Always |
| `layout/root-tables.sql` | `roots/<root>/layout.sql` | Always |
| `layout/migrate.sql` | `roots/<root>/migrate.sql` | When the Root grew from another |

A file it has to copy that is missing makes the build fail. RTM-D15 fixes the implementation boundary: the final build command validates the adjacent Build record, the matching Root object, a positive integer Root Contract and an optional valid different `grewFrom` identifier, then reads all required regular source files before writing any layout. It removes a stale `migrate.sql` for a Root with no lineage. Packaging copies the standard-library program, tests, scripts and the entire contract/roots fixtures with checksum-pinned standalone CPython 3.14.8 for macOS arm64 or x86_64. The last command copies the selected SQL and writes a SHA-256 manifest of the complete image; mutable `build.json`, `build.log`, `last-start` and interpreter bytecode caches are excluded. The unit runner verifies that manifest and its own Build identity, then runs program/build-script tests and the existing conformance-suite tests with the packaged interpreter. Unit tests never recursively package another interpreter. Candidate packaging may prepare an empty absolute folder outside the source for development (build number 0); only a numbered Build can run the final SQL-copy command and qualification. No package check uses an owner Root or the installed database. The program of `arbol-root` never runs these files and opens no database (RTM-0): `arbol-roots` alone runs them on a Root's database (ARC-1), and the checks of the Build run them only on a database server of their own ([below](#the-checks-of-a-build)). The program is typed Python that uses only the standard library, as the conformance suite is (RCT-4), and the Build carries the pinned CPython that BLD-0 gives every Build.

```text
~/Library/Application Support/Arbol/builds/arbol-root/7/
  build.json               "root": "<second>"
  bin/arbol-root           the program of RTM-0
  layout/
    chat-tables.sql        contract/1/layout.sql
    root-tables.sql        roots/<second>/layout.sql
    migrate.sql            roots/<second>/migrate.sql
```

<!-- sources:
arbol-root:scripts/build.py#copy_layout
arbol-root:scripts/package_build.py#archive
arbol-root:scripts/package_build.py#package
arbol-root:scripts/package_support.py#metadata
arbol-root:scripts/package_support.py#write_manifest
arbol-root:scripts/python-distribution.json
arbol-root:.python-version
arbol-root:tests/test_layout_copy.py
arbol-root:tests/test_package.py
-->

### How `arbol-roots` runs the files

`arbol-roots` takes the files from the Root's newest working Build of `arbol-root` (ARC-1) and runs them with `psql`:

```sh
psql --no-psqlrc --set ON_ERROR_STOP=1 --single-transaction --file …/7/layout/chat-tables.sql postgresql://arbol@127.0.0.1:5432/arbol_<second>
psql --no-psqlrc --set ON_ERROR_STOP=1 --single-transaction --file …/7/layout/root-tables.sql postgresql://arbol@127.0.0.1:5432/arbol_<second>
cd <an empty folder of its own, mode 0700> && \
psql --no-psqlrc --set ON_ERROR_STOP=1 --set from_database=arbol_sapling --set from_root=sapling \
     --file …/7/layout/migrate.sql postgresql://arbol@127.0.0.1:5432/arbol_<second>
```

| Command of `arbol-roots` | Runs |
|---|---|
| `migrate <root>` | Only while the Root is `preparing`; for a Root in any other state it is refused and changes nothing. It drops the Root's database and makes it again, empty, then runs all three files, the third only for a Root that grew from another. So the migration can run again as often as needed (ARC-1). Afterwards it removes the folder it ran the script in, which holds copies of the chats |
| `layout <root>` | The first two, on a Root in use: they add what is missing and change nothing else |

`arbol-roots` starts `psql` with only `HOME`, `PATH` and `LANG` of its own environment, so that no `PG…` variable of the owner's shell reaches it, and with the password of the Keychain item of ARC-1, when there is one, in `PGPASSWORD`; `\connect` uses the same password. `psql` ends with the exit status 0 when every statement succeeded, and with 3 after the first that failed, having printed the file, the line and the error, for example `psql:migrate.sql:35: ERROR: relation "migrated_chats" does not exist`. Any other status, such as 1 for an error of `psql` itself or 2 for a lost connection, is a failure too.

Runtime application (RTM-D25) holds the existing catalogue mutation lock from selection through completion and a shared Build-use guard while reading/running its files, so removal/pruning cannot take the selected files away. It selects the newest runnable working, unpruned `arbol-root` Build for the explicit Root; an absent layout directory is ineligible. An incomplete or mismatched selected Build is refused before any database change, rather than silently choosing older SQL. Preflight requires regular files, their SHA-256 entries in the copied package manifest, and matching Root, Build, Root Contract and lineage metadata. All required files, both database names and an executable psql are checked before DROP. The target database cannot also belong to another retained Root or be the older Root’s database.

`layout` applies to working or failed retained Roots; a preparing Root uses `migrate`. Neither command changes root.json or marks a Root ready/failed. A success returns one JSON object with root, rootBuild and the ordered files run. Layout files each use a transaction; migration uses its own two transactions. A failed later file can leave earlier successful files committed. Failure names psql’s status/file/line/error, bounded to 4096 characters with any password removed. Maintenance calls retain their 30-second limit; each file has a 60-second limit and no automatic retry. A failed migration remains preparing and can be run again from an empty database. Migration copy files live only in a fresh mode-0700 temporary directory, removed on success, SQL failure or interruption. While that directory is owned, the command handles SIGINT and SIGTERM as a structured operation_failed refusal: the subprocess runner terminates and reaps its own psql child, the directory is removed, then the prior handlers are restored. Other commands keep their existing signal behavior; the real CLI boundary is checked with both signals, not inferred from an injected exception.

The prescribed script passes database names through psql’s `\connect` variables. Because psql can treat a database argument as a connection string, preflight refuses a source or target name containing `=`, beginning `postgres://` or `postgresql://`, or equal to `-`, before any database change; it does not reinterpret such a name or connect elsewhere (RTM-D26). Ordinary quoted/Unicode database names remain names. See the [PostgreSQL 17 psql connection rules](https://www.postgresql.org/docs/17/app-psql.html#APP-PSQL-META-COMMAND-CONNECT).

<!-- sources:
arbol:src/arbol_roots/sql.py#selected_sql
arbol:src/arbol_roots/sql.py#databases
arbol:src/arbol_roots/sql.py#database_client
arbol:src/arbol_roots/sql.py#layout
arbol:src/arbol_roots/sql.py#migrate
arbol:src/arbol_roots/database.py#Database.file
arbol:src/arbol_roots/database.py#Database._execute
arbol:tests/test_roots_sql.py
arbol:scripts/roots_sql_drill.py#drill
-->

### The Root's own layout

`roots/<root>/layout.sql` holds the tables that only this Root has: Arbol's own tables, which Arbol's own specification states, and, for a Root that grew from another, `migrated_chats` ([below](#where-a-migrated-chat-came-from)). It follows the rules of the chat tables' file (RCT-3): every statement can run again, and every table and column has a comment. In the first thin path the file of the first Root may hold no statement at all.

A later Build of the same Root may only add to this file what the Root's older Builds can ignore (ARC-1): a new table, and on a table that exists a column that may be NULL or has a default, or an index that is not unique. It adds no check, no unique index and no column that is NOT NULL without a default to a table that exists, because the writes of the older Builds would then fail. Each addition is a statement of its own that can run again, such as `ALTER TABLE drafts ADD COLUMN IF NOT EXISTS pinned boolean NOT NULL DEFAULT false`. A statement that an earlier Build carried is never edited: on a Root in use, `IF NOT EXISTS` would skip the edit, and a new database would differ from the one in use. Any other change needs a new Root (ARC-1). The chat tables of a Root Contract version have no additions in version 1 ([RCT-D24](../contract/03-the-chat-tables.md#decisions)).

<!-- sources:
arbol-root:roots/sapling/layout.sql
arbol-root:scripts/check_layout.py#run_layout
arbol-root:scripts/sql_statements.py#preserves
arbol-root:scripts/layout_compatibility.py#check_previous
arbol-root:scripts/layout_compatibility.py#additions
arbol-root:tests/test_layout_check.py
arbol-root:tests/test_layout_compatibility.py
arbol-root:roots/jensen/layout.sql
-->

### The migration script

`roots/<root>/migrate.sql` brings the chats of the Root this Root grew from into this Root's database, in one step ([RTM-D10](#decisions)):

1. `arbol-roots` runs it with `psql` connected to the new database, with the variables `from_database`, the older Root's database, and `from_root`, its identifier, in an empty folder of its own.
2. It switches to the older database with `\connect` and reads it in one transaction that is `REPEATABLE READ` and `READ ONLY`: every table shows the same moment, the *snapshot*, and nothing there can be changed. It writes each table with `\copy … TO '<table>.copy' WITH (FORMAT binary)` into the folder.
3. It switches back to the new database and, in one transaction, reads each file into its table, writes `migrated_chats` and the Exchanges that were open, and commits. A failure anywhere leaves no row in the new database, and the older one was never written (ARC-1).

```sh
ls /…/migrate-1a2b/      # the folder after a run; arbol-roots removes it
# chats.copy  exchanges.copy  messages.copy  runs.copy  tool_calls.copy
```

A table that did not change is copied as it is: the binary form keeps every byte of every text and `jsonb` value, and every identifier stays the same. A table that changed is read with a `SELECT` that converts its rows into the new form. The second, trivial Root of the first thin path has none: both Roots use Root Contract version 1, so all five chat tables are copied as they are. Arbol's own tables, such as Drafts, are copied in the same way once Arbol's own specification has them. The whole script for that Root is in the [Interface](#interface).

<!-- sources:
arbol-root:scripts/check_migration.py#run_migration
arbol-root:scripts/check_migration.py#invoke
arbol-root:scripts/migration_protocol.py#protocol
arbol-root:scripts/migration_fixtures.py#seed
arbol-root:tests/fixtures/migration-root/migrate.sql
arbol-root:tests/test_migration_protocol.py
arbol-root:roots/jensen/migrate.sql
-->

### An Exchange that was open

An Exchange that is open in the older database at the snapshot is written into the new one as `interrupted`, ended at the time of the snapshot, with the rows its Trunk had saved by then. Its process and its folder stay in the older Root, where it goes on; the migration starts nothing (ARC-1).

For example, the chat "What build.sh does" has three Exchanges in First Sapling when the second Root is migrated: the first completed, the second failed and was replaced by the third, and the third is open. In the second Root the first two are as they were, and the third is interrupted at the moment First Sapling was read, with the message and the one block of the answer its Trunk had saved. In First Sapling the third goes on, and its Trunk ends it there.

<!-- sources:
arbol-root:scripts/check_migration.py#judge
arbol-root:scripts/check_migration.py#normalized
arbol-root:tests/test_migration_check.py#MigrationCheck
-->

### Where a migrated chat came from

The migration writes one row of `migrated_chats` for each chat it brings over ([RTM-D11](#decisions)). The table is in the Root's own layout, not among the chat tables, because no Trunk writes or reads it ([RCT-D25](../contract/03-the-chat-tables.md#decisions)).

```text
chat_id   from_root  from_chat  ended_through  migrated_at
c3d4e5f6  sapling     c3d4e5f6   2              2026-10-15 11:02:00.000+00
d4e5f607  sapling     d4e5f607   1              2026-10-15 11:02:00.000+00
```

`ended_through` says how far the original had got: the number of its last Exchange that had ended at the snapshot, 0 when none had. A later check can tell that the original went on in the older Root when it has an ended Exchange with a higher number (ARC-1, RTM-D12).

<!-- sources:
arbol-root:scripts/check_migration.py#judge
arbol-root:tests/fixtures/migration-root/layout.sql
arbol-root:tests/test_migration_check.py#MigrationCheck
-->

### The checks of a Build

`arbol-build.json` names three checks. Each starts the Build's own program or runs the Build's own files, not through a launcher (BLD-0); `layout` also runs the files of an earlier Build of the same Root:

| Check | What it judges |
|---|---|
| `unit` | The program's tests in its own process, among them `scripts/build.py` run on a repository of its own; the commands of RTM-0, run with `--root-dir` and `--bin-dir` on a temporary Root folder and a stand-in `arbol-trunk`, and with them the Root manager's part of RCT-1.8 to RCT-1.10 and RCT-2.14; and the tests of the conformance suite in `contract/conformance/` (RCT-4.1 to RCT-4.3). The stand-in plays the launcher and a Trunk from a script: it answers `list`, writes one line for `run`, makes and removes Exchange folders, ends its process group on SIGTERM or ignores it, and records every start with its arguments |
| `layout` | On a database server of its own: `chat-tables.sql` and `root-tables.sql` run twice, and the second run changes nothing; every table and column of both files has a comment. When an earlier working Build of the same Root still has its files in the registry, the newest of them is compared: its files are run on one database and this Build's files after them, and this Build's files alone on another. Every table, column, index and check that the earlier files made stays as it was, none of those tables gains a check, a unique index or a column that is NOT NULL without a default, and both databases end with the same tables, columns, indexes, checks and comments, compared as `pg_dump --schema-only` prints them |
| `migration` | For a Root that grew from another, on a database server of its own: it makes the older Root's database from `contract/<its rootContract>/layout.sql` and `roots/<older root>/layout.sql` of the same checkout, fills it with fictional chats, one of them with an open Exchange, and runs `migrate.sql` as `arbol-roots` does. It judges that the older database's rows are unchanged; that every row of a table that did not change, and every Exchange that had ended, is the same in the new one, the text byte for byte; that the open Exchange is `interrupted` at the snapshot; and that `migrated_chats` has one row for each chat with the right `ended_through`. A run whose older database is missing, and one whose new database lacks a table, end with the exit status 3 and leave no row. A second run on a database made empty again gives the same rows, apart from the time of the snapshot in `migrated_chats` and in the Exchange that was open. For a Root that grew from none it passes at once |

Implementation choices (RTM-D22/D23): the two checks reuse this repository's conformance Server and catalogue reader, never another component's code. Preflight names all five required programs before creating a server; a no-lineage migration passes before needing a server. Each check owns and removes its temporary server, short socket directory and private home. SQL clients explicitly name that socket/database, use absolute tool paths, and the migration invocation has only HOME/PATH/LANG, an empty mode-0700 working directory, ON_ERROR_STOP and the two private database/Root variables. The existing 60-second SQL-client bound applies; no owner PG variables or credentials are used. Build input comes from its validated record and copied-layout manifest; the older Root's identifier/version is validated once from its header and the packaged checkout supplies its version/layout files.

Layout comparison uses both the logical PostgreSQL catalogue and pg_dump --schema-only --no-owner --no-privileges, with a fixed psql restrict key when the client supports that option. It requires unchanged earlier schema objects and comments, rejects new constraints/unique indexes on existing tables and new NOT NULL columns without defaults, and requires identical old-then-new/fresh schemas. Earlier complete SQL statements must remain in order, allowing only whitespace/case/comment changes outside quoted values; literal values and COMMENT statements retain their contents. The specified plain table/index/column/comment DDL is accepted; client metadata commands and data-writing statements are refused. This keeps layouts within their prescribed job and makes the statement-preservation rule independently checkable. New tables may have their own constraints; additions to older tables obey the narrower rule above. The comparison uses the highest lower hexadecimal Build with a readable matching working/unpruned record and both layout files; missing/damaged/ineligible records are not a qualified baseline, and the report identifies the baseline it used.

Migration qualification uses the prescribed two-connection protocol, one REPEATABLE READ READ ONLY source transaction and one destination transaction; connection/file metadata commands must keep both databases on the private server and every copy file in the private working directory. It checks the actual row invariants against fictional data, source immutability, the snapshot interval and identical migrated_at/open ended_at times, all unchanged tables and byte-exact text/JSON values, and a fresh repeated run excluding only snapshot times. The fixture contains two chats, a replaced failed Exchange, completed/cancelled history and an open Exchange, with emoji/U+FFFD/CRLF/tab/backslash/trailing spaces and whole nested JSON. Missing-source and missing-destination-table runs must exit 3 and leave the destination empty. A private READ ONLY write refusal and protocol validation establish the source transaction rule; planted protocol/data/rollback faults must fail qualification. Check code and fictional SQL fixtures are packaged; the owner's production Root identifier is never inferred from a fixture.

A database server of its own is made with `initdb` in a temporary folder, listens only on a socket in a short folder under `/tmp`, and is stopped and removed afterwards, as the Root Contract's suite does ([RCT-4](../contract/04-conformance.md#what-it-sets-up-for-every-scenario), [RTM-D13](#decisions)). The installed server is never used, so no check can reach the chats of a Root in use (ARC-1). A check that does not find `initdb`, `pg_ctl`, `createdb`, `psql` or `pg_dump` on the search path fails and names the program.

<!-- sources:
arbol-root:scripts/check_build.py#main
arbol-root:scripts/package_support.py#verify
arbol-root:scripts/verify_build.py#qualify
arbol-root:contract/conformance/tests/test_cli.py
arbol-root:contract/conformance/tests/support.py
arbol-root:contract/conformance/tests/test_layout.py
arbol-root:contract/conformance/README.md
arbol-root:tests/test_commands.py
arbol-root:tests/test_layout_copy.py
arbol-root:tests/test_package.py
arbol-root:arbol-build.json
arbol-root:scripts/sql_support.py#BuildInput
arbol-root:scripts/sql_support.py#private_server
arbol-root:scripts/sql_support.py#tools
arbol-root:scripts/check_layout.py#main
arbol-root:scripts/check_migration.py#main
arbol-root:scripts/check_migration.py#check_migration
arbol-root:scripts/verify_sql_build.py#qualify
arbol-root:tests/test_sql_support.py
arbol-root:tests/test_layout_check.py
arbol-root:tests/test_layout_compatibility.py
arbol-root:tests/test_migration_check.py
arbol-root:tests/test_migration_protocol.py
arbol-root:tests/program_support.py#development
arbol-root:tests/management_support.py#Fixture
-->

## Requirements

An `arbol-root` Build:

| ID | An `arbol-root` Build must… | Origin | Verified by |
|---|---|---|---|
| RTM-1.1 | be made for the Root named with `arbol-builds build arbol-root --root <root>`, and carry `layout/chat-tables.sql`, a copy of `contract/<rootContract>/layout.sql` for its Root's version, `layout/root-tables.sql`, a copy of `roots/<root>/layout.sql`, and, exactly when its Root grew from another, `layout/migrate.sql`, a copy of `roots/<root>/migrate.sql` | New | unit: `root.layout-copy`, `root.layout-lineage`, `root.layout-metadata`, `root.package`, `root.package-integrity`; private integration: `root.private-build` (SYL-181) |
| RTM-1.2 | fail to build when a file it has to carry is missing from the repository | New | unit: `root.layout-missing`, `root.layout-lineage`; private integration: `root.private-build` missing-layout Build (SYL-181) |
| RTM-1.3 | have the checks `unit`, `layout` and `migration`, and run the last two only on a PostgreSQL server of the check's own in a temporary folder, listening only on a socket, never on the installed server | New | unit: `root.sql-private-server`, `root.sql-cleanup`, `root.sql-tools`, `root.sql-input`, `root.package-integrity`; private Build integration: `root.private-sql-builds`, with named unit/layout/migration and packaged missing-tool probes (SYL-184) |

The Root's own layout:

| ID | The Root's own layout must… | Origin | Verified by |
|---|---|---|---|
| RTM-1.4 | be one file, `roots/<root>/layout.sql`, of statements that can run again, with a comment on every table and column, holding Arbol's own tables and, for a Root that grew from another, `migrated_chats` | New | unit check `layout`: `root.layout-idempotence`, `root.layout-not-repeatable`, `root.layout-comments`, `root.layout-ddl-boundary`; First Sapling carries its empty own layout. Jensen Root’s exact `roots/jensen/layout.sql` and `migrate.sql` pass the actual private Build path in `root.private-sql-builds --root jensen` (SYL-184) |
| RTM-1.5 | change in a later Build of the same Root only by additions its older Builds can ignore, which are a new table and, on a table that exists, a column that may be NULL or has a default and an index that is not unique, and never by editing a statement that an earlier Build carried | Changed | unit check `layout`: `root.layout-compatible-additions`, `root.layout-edited-statement`, `root.layout-required-column`, `root.layout-new-constraints`, `root.layout-baseline`, `root.layout-statements`; packaged prior-Build and planted-fault probes in `root.private-sql-builds` (SYL-184) |

A migration script:

| ID | A migration script must… | Origin | Verified by |
|---|---|---|---|
| RTM-1.6 | be one file, `roots/<root>/migrate.sql`, for the one step from the Root its Root grew from, run by `psql` connected to the new database with the variables `from_database` and `from_root`, and write files only in the folder it is run in | New | unit check `migration`: `root.migration-protocol`, `root.migration-protocol-refusal`, `root.migration-whole`, `root.migration-origin`; private Build integration `root.private-sql-builds` (SYL-184). Runtime: `roots.sql-selection`, `roots.sql-preflight`, `roots.sql-pruning`, `roots.sql-databases`, `roots.migrate`, `roots.migrate-failure`, `roots.migrate-refusal`, `roots.migrate-interruption`; explicit installed-server `roots.runtime-sql` (SYL-185) |
| RTM-1.7 | read the older database only in one transaction that is `REPEATABLE READ` and `READ ONLY` | Changed | unit check `migration`: `root.migration-protocol`, `root.migration-protocol-refusal`, `root.migration-whole` (validated READ ONLY/REPEATABLE READ source protocol, server-side write refusal and unchanged source rows/schema; SYL-184) |
| RTM-1.8 | write the new database in one transaction, so that a failure leaves no row there | Changed | unit check `migration`: `root.migration-whole` (one destination transaction, missing-source and missing-table status 3, no target rows), `root.migration-protocol-refusal`; runtime rollback/status/preparing behavior: `roots.migrate-failure` and explicit installed-server `roots.runtime-sql` (SYL-185); real-Root drill remains SYL-189 |
| RTM-1.9 | copy every row of a table that did not change, with the same identifiers and every text and `jsonb` value as it was, except as RTM-1.10 says | Changed | unit check `migration`: `root.migration-fixture`, `root.migration-whole`, `root.migration-text-fault` (all five unchanged tables, identifiers, Unicode/CRLF/tab/backslash/trailing spaces and nested JSON; SYL-184) |
| RTM-1.10 | write every Exchange that was open at the snapshot as `interrupted`, ended at the time of the snapshot, and start nothing | New | unit check `migration`: `root.migration-whole`, `root.migration-open-fault`, `root.migration-snapshot-fault` (matching interruption/snapshot interval, source unchanged; SYL-184). A running original process staying in its Root is a SYL-189 drill |
| RTM-1.11 | write one row of `migrated_chats` for every chat it brings over | New | unit check `migration`: `root.migration-whole`, `root.migration-provenance-fault` (one row per chat, from_root/from_chat/ended_through and shared snapshot; SYL-184) |

RTM-1.1 and RTM-1.2 are checked by the tests of `scripts/build.py` in the check `unit`, because a build that fails runs no check; `arbol-builds` checks its own `--root` (BLD-0).

## Interface

```ts
// arbol-builds build arbol-root --root <root>
interface BuildDescription {           // arbol-build.json at the top of the arbol-root repository
  component: "arbol-root";
  build: string[];                     // the build commands; the last, scripts/build.py, copies the three files into layout/
  entry: string;                       // "bin/arbol-root"
  checks: Array<{ name: "unit" | "layout" | "migration"; kind: "unit"; command: string }>;
  // no "root": the Root is named with --root on build (RTM-D9)
}
```

The own tables of the second Root, `roots/<second>/layout.sql`, as the first thin path needs them:

```sql
-- The second Root's own tables (RTM-1).
-- Kept in the arbol-root repository as roots/<second>/layout.sql; the second Root's arbol-root Builds carry it as layout/root-tables.sql.
-- arbol-roots runs it after the chat tables: psql --no-psqlrc --set ON_ERROR_STOP=1 --single-transaction --file root-tables.sql
-- Every statement can run again: on a database that already has everything, the file changes nothing.
-- Arbol's own tables, such as Drafts, are added to this file by Arbol's specification.

CREATE TABLE IF NOT EXISTS migrated_chats (
  chat_id        uuid        PRIMARY KEY REFERENCES chats (id) ON DELETE RESTRICT,
  from_root      text        NOT NULL,
  from_chat      uuid        NOT NULL,
  ended_through  integer     NOT NULL CHECK (ended_through >= 0),
  migrated_at    timestamptz NOT NULL
);
COMMENT ON TABLE migrated_chats IS 'One row for each chat that a migration brought into this Root: where it came from and how far the original had got. Written once, by the migration script.';
COMMENT ON COLUMN migrated_chats.chat_id IS 'The chat in this Root, for example c3d4e5f6-0718-4293-a4b5-c6d7e8f90a1b.';
COMMENT ON COLUMN migrated_chats.from_root IS 'The identifier of the Root it came from, for example sapling.';
COMMENT ON COLUMN migrated_chats.from_chat IS 'The chat''s identifier in that Root; the same as chat_id for a chat copied as it was.';
COMMENT ON COLUMN migrated_chats.ended_through IS 'How far the original had got when it was read: the number of its last Exchange that had ended, 0 when none had.';
COMMENT ON COLUMN migrated_chats.migrated_at IS 'When the older database was read: the moment of the snapshot the copy shows.';
```

The migration script of the second Root, `roots/<second>/migrate.sql`, whole:

```sql
-- The second Root's migration script (RTM-1): brings the chats of First Sapling into the second Root's database.
-- Kept in the arbol-root repository as roots/<second>/migrate.sql; the second Root's arbol-root Builds carry it as layout/migrate.sql.
-- arbol-roots runs it with psql, connected to the second Root's database just made from the second Root's layouts, in an empty folder of
-- its own (mode 0700) that it removes afterwards:
--   psql --no-psqlrc --set ON_ERROR_STOP=1 --set from_database=arbol_sapling --set from_root=sapling \
--        --file migrate.sql postgresql://arbol@127.0.0.1:5432/arbol_<second>
-- First Sapling and the second Root use Root Contract version 1, so every chat table is copied as it is.

\set to_database :DBNAME

-- 1. Read First Sapling's tables in one transaction that cannot write, so the copy is one moment and nothing there changes.
\connect :from_database
BEGIN ISOLATION LEVEL REPEATABLE READ READ ONLY;
SELECT now() AS snapshot \gset
\copy (SELECT * FROM chats) TO 'chats.copy' WITH (FORMAT binary)
\copy (SELECT * FROM exchanges) TO 'exchanges.copy' WITH (FORMAT binary)
\copy (SELECT * FROM messages) TO 'messages.copy' WITH (FORMAT binary)
\copy (SELECT * FROM tool_calls) TO 'tool_calls.copy' WITH (FORMAT binary)
\copy (SELECT * FROM runs) TO 'runs.copy' WITH (FORMAT binary)
COMMIT;

-- 2. Write the second Root's tables in one transaction, so that a failure leaves no row.
\connect :to_database
BEGIN;
\copy chats FROM 'chats.copy' WITH (FORMAT binary)
\copy exchanges FROM 'exchanges.copy' WITH (FORMAT binary)
\copy messages FROM 'messages.copy' WITH (FORMAT binary)
\copy tool_calls FROM 'tool_calls.copy' WITH (FORMAT binary)
\copy runs FROM 'runs.copy' WITH (FORMAT binary)

-- Where each chat came from and how far it had got, counted before open Exchanges change.
INSERT INTO migrated_chats (chat_id, from_root, from_chat, ended_through, migrated_at)
SELECT c.id, :'from_root', c.id, coalesce(max(e.number) FILTER (WHERE e.state <> 'open'), 0), :'snapshot'
FROM chats c LEFT JOIN exchanges e ON e.chat_id = c.id
GROUP BY c.id;

-- An Exchange still open in First Sapling is only history here: interrupted, at the moment of the snapshot.
UPDATE exchanges SET state = 'interrupted', ended_at = :'snapshot' WHERE state = 'open';
COMMIT;
```

`\copy` takes its file name as written, without `psql` variables, so the script writes its files by bare names into the folder `psql` runs in.

Both files were run on PostgreSQL 17.10 on 15 October 2026, in a throwaway server with no network, against fictional chats of First Sapling: two chats, an Exchange replaced by another, a failed one, a cancelled one and one still open, and messages with an emoji, U+FFFD, a CRLF, a tab, a backslash and trailing spaces. Both layouts, run twice, changed nothing the second time, and every column of `migrated_chats` had its comment. The migration ended with the exit status 0, and First Sapling's rows were unchanged. Every row of `chats`, `messages`, `tool_calls` and `runs`, and every Exchange that had ended, was the same in the second Root, the text byte for byte. The open Exchange was interrupted at the snapshot, and `migrated_chats` had one row for each chat, with `ended_through` 2 and 1. A run whose older database did not exist, and one whose new database lacked `migrated_chats`, ended with the exit status 3 and left no row. A second run on a database made new gave the same rows of `chats`, `messages`, `tool_calls` and `runs`, and a write in a `READ ONLY` transaction was refused. The run used the superuser `postgres` over a socket; the role `arbol`, a connection over TCP and a password were not tried. The installed server was not used.

## Known problems in version 1

Not to be reproduced.

- **Conversion as a release gate.** The replacement was to take over only after all historical execution state had been converted. The gate was still open at its 137th checkpoint.
- **History rewritten during an upgrade.** An earlier cutover cut 34 oversized records short in place and kept the originals in a backup table.
- **Tests that could reach real data.** A test fixture dropped every schema with a test name in whatever database it reached.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:product-spec/12-lessons-and-invariants.md#8. Proving replaced delivering`
- `arbol_deprecated_v1:product-spec/12-lessons-and-invariants.md#5. Everyday operations cost in proportion to total history`
- `arbol_deprecated_v1:product-spec/12-lessons-and-invariants.md#Validation practice`

## Decisions

| ID | Question | Proposed default | Outcome |
|---|---|---|---|
| RTM-D8 | Where does the `arbol-root` repository keep the layouts, the migration scripts and the conformance suite, and what does a Build carry? | `contract/<version>/layout.sql` and `contract/conformance/` for the Root Contract, shared by every Root of a version; `roots/<root>/layout.sql` and `roots/<root>/migrate.sql` for each Root. A Build carries copies of its Root's files under fixed names in `layout/`, so `arbol-roots` finds them without knowing the repository. The alternative: one branch of the repository for each Root, which keeps the files of one Root apart from another's but makes every fix to the program a merge into each branch | Decided 15 October 2026 by the owner: the table layout is a plain `layout.sql` in the `arbol-root` repository, and `arbol-roots` runs SQL files with `psql` (SYL-168, decision 5). The folders and the names in `layout/` are a technical choice made the same day |
| RTM-D9 | How does `arbol-builds build arbol-root` know which Root it builds for? | It is told: `arbol-builds build arbol-root --root <second>` writes `"root": "<second>"` into the record when the build starts, and `arbol-build.json` names no Root, because one repository serves every Root. The build reads the Root's `root.json` for its Root Contract version and the Root it grew from. It is the one change this specification needs in BLD-0, and it holds for `arbol` too. The alternative: `arbol-build.json` names the Root, and the owner edits the file before each build for another Root | Decided 15 October 2026, a technical choice: the proposed default |
| RTM-D10 | `psql` talks to one database at a time. How does a migration script read the older database? | It switches to it with `\connect`, reads every table in one transaction that is `REPEATABLE READ` and `READ ONLY`, so the copy shows one moment and nothing there can change, and writes each table in binary into the folder it runs in. Then it switches back and writes the new database in one transaction. `arbol-roots` runs it in an empty folder of its own and removes the folder afterwards. The alternative: the extension `postgres_fdw`, which reads the older tables without files but has to be created by a superuser and leaves a server definition in the new database | Decided 15 October 2026, a technical choice: the proposed default |
| RTM-D11 | Where is the record of where a migrated chat came from, and what does it hold? | A table `migrated_chats` in the Root's own layout (RCT-D25). For each chat: the Root and the chat it came from, the number of the original's last Exchange that had ended, and when the original was read. A migrated chat keeps its identifier. A later check can then tell a copy whose original went on: the original has an ended Exchange with a higher number. The alternative: the time of the original's last change instead of the number, which a clock set back would mislead | Decided 15 October 2026, a technical choice: the proposed default |
| RTM-D12 | After the second Root is made from First Sapling, a chat can go on in First Sapling. Does the first thin path already build what deals with that: the second Root noticing such a chat, and bringing that one chat over again? | No. The first migration brings the whole of First Sapling over once, and writes for each chat where it came from and how far it had got (`migrated_chats`), so both can be added later without changing that record. In practice: a chat you go on with in First Sapling after the second Root was made stays at its old state in the second Root, and the second Root does not tell you. ARC-1.19, ARC-1.20, the one-chat half of ARC-1.16 and a script that can run for one chat (ARC-1) are then not met in the first thin path; when they are built, they go into `roots/<root>/migrate.sql` and the Root's own layout. The alternative: build them now, as the owner had the check built first to see how it works (ARC-D50). Each time the second Root's backend starts or its chat list opens, `arbol-roots migrate <second> --check` marks such a chat as out of date, and the window offers to bring it over again: that replaces the second Root's copy, or adds First Sapling's chat as a second chat when you also wrote in the second Root. The cost: a `--check` and a one-chat form of `migrate.sql`, and a column for the mark, all written before any migration has run | Decided 15 October 2026: the owner left it to the agent ("Not sure yet. You decide"), which chose the proposed default |
| RTM-D13 | Which checks does an `arbol-root` Build pass, and on which database? | Three: `unit` for the commands, against a stand-in `arbol-trunk`; `layout`; and `migration` for a Root that grew from another. The last two each run on a PostgreSQL server of their own in a temporary folder, listening only on a socket, as the Root Contract's suite does (RCT-D26). No check reaches the installed server. The alternative: a scratch database on the installed server, which needs no `initdb` but puts tests and chats on one server | Decided 15 October 2026, a technical choice: the proposed default |
| RTM-D15 | How is the Root-specific standard-library Build delivered and checked? | Two stages: package the checksum-pinned standalone CPython and all manager/contract test assets, then scripts/build.py reads the numbered record and private or owner HOME Root record, validates safe identity/version/lineage, preloads required SQL, copies it unchanged and records a file manifest. Unit checks verify integrity and run both test families using the Build interpreter. No dependency on another component's Python package, recursive packaging in unit tests, SQL execution or installed-server access | Decided 26 October 2026, technical choice for SYL-181: the proposed default; layout/migration checks remain SYL-184 |
| RTM-D16 | How do the existing conformance tests run unchanged in purpose inside a package, including a path with spaces and a private HOME? | Quote every argument of their test-only Trunk command. Package an exact copy of the authoritative RCT-3 document under contract/conformance/specifications/03-the-chat-tables.md; its tests compare layout.sql with that document’s SQL Interface. Prefer that packaged oracle, otherwise the development corpus at HOME/Artifacts/arbol-root/contract/03-the-chat-tables.md; missing both is a failure, never a skip. Packaging uses its source’s embedded oracle or the development corpus and requires one. The private Build probe supplies a copy of the owner’s authored document in its private HOME. No owner database or chat data is copied as an oracle | Decided 26 October 2026, technical choice after the first SYL-181 packaged qualification exposed unquoted paths and a skipped HOME-bound oracle |
| RTM-D22 | How are layout checks repeatable and compatible with earlier Builds? | Private socket-only Server plus mandatory pg_dump; compare logical catalogue and stable schema dumps, require every old statement/object/comment unchanged, and allow only prescribed additive DDL. Select the newest eligible lower-numbered same-Root working Build, preserving the distinction between a qualified baseline and a damaged/missing record. No owner server or shell PG settings enter the check | Decided 26 October 2026, technical choice for SYL-184: reusing Root-owned helpers and directly checking RTM-1.4/1.5 rather than inferring compatibility from a successful SQL run |
| RTM-D23 | How does migration qualification prove the first thin path without touching a Root in use? | Validate copied Build/lineage inputs and the two private-database/file protocol, then run the exact packaged script on rich fictional rows. Verify source immutability, byte-exact unchanged rows, snapshot interruption/provenance, rollback/status-3 failures and repeatability; use strict minimal client environment and private cleanup. A Root without lineage passes at once. Production Root SQL paths await the owner name, while generic fixtures have explicitly fictional identities | Decided 26 October 2026, technical choice for SYL-184: preserving the accepted script and RTM-D10/D11/D13 with inspectable private evidence |
| RTM-D24 | Which display name and identifier should the second production Root use? | Jensen Root, identifier `jensen`; its files are `roots/jensen/layout.sql` and `roots/jensen/migrate.sql`. SYL-184 qualifies those files in a disposable home/registry; the actual Root is created in SYL-189 | Decided by the owner on 26 October 2026: “let’s call it Jensen Root”; lowercase identifier follows that chosen name |
| RTM-D25 | How does arbol-roots apply qualified SQL without losing its selected Build or mutating a wrong database? | Serialize with the catalogue lock and shared Build-use guard, preflight copied SQL/manifest and both Root/database identities, then use one minimal-environment psql file runner. Layout accepts working/failed Roots; migrate accepts preparing only. Keep Root metadata unchanged, return root/rootBuild/files on success, and report bounded redacted failures with no automatic retry | Decided 26 October 2026, technical choice before SYL-185 implementation; maintenance 30 seconds and each SQL file 60 seconds, private migration-copy cleanup on every exit |
| RTM-D26 | What if a recorded database name has a form that psql interprets as connection parameters? | Refuse the ambiguous names before DROP or any layout file: equals sign, postgres URI prefix or the positional reuse sentinel. Never reinterpret the Root’s database field or let the copied script choose another server. Other names retain PostgreSQL’s identifier semantics | Decided 26 October 2026, technical choice for the prescribed two-connection script; PostgreSQL 17’s documented psql connection rules checked |
