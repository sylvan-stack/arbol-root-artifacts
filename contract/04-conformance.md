---
role: authored
spec: RCT-4
status: accepted
---
# Root Contract · 04 Conformance

The *conformance suite* of the Root Contract is one program that checks a Trunk Build against this contract. It starts the Build's program as `arbol-root` and a person would, in a home folder of its own that holds a Root record for the test, with a stand-in that plays a Driver from a script and a database server of its own with a database made from `layout.sql`, and it judges what the Trunk prints, leaves in its folder and writes in the tables. For example, `rct-conformance.py --trunk "$ARBOL_BUILD_DIR/bin/arbol-trunk-trivial" --root-contract 1` runs every scenario against one Build of the Trivial Trunk. This specification states where the suite lives and how it is started, what it sets up for each scenario, the stand-in Driver, the scenarios and what each judges, and how a Trunk Build's check runs it. It matters because a Trunk that passes it can be started, listed, stopped and settled by every Root manager of its version and shown by its backend without either knowing which Trunk it is; and because version 1 accepted each step on tests and isolated harnesses while the installed application was seldom run from end to end, and a test fixture dropped tables in whatever database it reached.

It follows the conventions of the [Driver Contract](../../arbol/driver-contract/README.md#conventions) and the additions in this chapter's [README](README.md#conventions). It is to the Trunks what [DRV-5](../../arbol/driver-contract/05-conformance.md) is to the Drivers. Its forms are technical choices made on 15 October 2026 ([RCT-D26](#decisions), RCT-D27). The owner answered the open questions on 15 October 2026, among them that a Trunk Build, like a Driver, is tested in a separate test folder of its own (RCT-D29), and the specification was accepted the same day.

## Scope

Covered: where the suite lives and how it is started, the home, the database server and the Root record it makes, the stand-in Driver, shorter waits and a Trunk that hangs on purpose, the scenarios, how a Trunk Build's check runs the suite.

Not covered: a Trunk's own unit tests, which each Trunk chooses; checks of the Root manager and of the Arbol backend, which their specifications and the journeys and drills of the Architecture name; the suite of the Driver Contract ([DRV-5](../../arbol/driver-contract/05-conformance.md)).

## Behaviour

### Where it lives and how it is started

The suite lives in the `arbol-root` repository, in the folder `contract/conformance`, beside `contract/1/layout.sql`: it belongs to the Root that gives the contract, as the Driver Contract's suite lives in `arbol` (DRV-5). It is typed Python that uses only the standard library, and it reads the tables with `psql`.

```sh
python3.14 ~/repos/sylvan-stack/readonly/arbol-root/contract/conformance/bin/rct-conformance.py \
  --trunk "$ARBOL_BUILD_DIR/bin/arbol-trunk-trivial" --root-contract 1
python3.14 …/rct-conformance.py --trunk "…" --root-contract 1 --only R3,R8
```

| Option | Meaning |
|---|---|
| `--trunk <command>` | What starts the Trunk Build's own program, not the launcher. The suite adds `--root rct` and the command, as the launcher does (RCT-0) |
| `--root-contract <version>` | The version to check. Its `layout.sql` comes from the checkout the suite runs from |
| `--only <scenarios>` | Only these scenarios |

It ends with the exit status 0 when every scenario passed, 1 when one failed, and 2 when its input was wrong or the programs of PostgreSQL (`initdb`, `pg_ctl`, `createdb`, `psql`) are missing, which it names.

### What it sets up for every scenario

| What | How |
|---|---|
| A home of its own | A new temporary folder is the Trunk's home folder. Every file of RCT-0 is reached from there, so the Root record, the settings and the database server the Trunk finds are the test's own. The Keychain is the Mac's own: the suite relies on there being no item for its role `rct`, so that the Trunk reads no password, and it writes none |
| A database server of its own | `initdb` into the home, with trust for the role `rct`, started with `pg_ctl` and listening only on a socket in a short folder under `/tmp`, because the path of a socket may not be longer than 103 bytes. It is stopped and removed after the scenario. The installed server (ARC-0) is never used |
| A Root record and a database of the test's own | `<home>/Library/Application Support/Arbol/roots/rct/root.json` with `"rootContract"` set to the version under test and `"database": "arbol_rct"`, and the empty folder `exchanges/` beside it (mode 0700); the database made with `createdb` and the version's `layout.sql` run with `psql`; `<home>/.arbol/database.json` with the socket's address, in the form `postgresql://rct@%2Ftmp%2Frct-1a2b` (RCT-0) |
| A minimal environment | The Trunk gets only `HOME`, `PATH`, `LANG` and the variables of the test, as a Driver does in its suite (DRV-5), so that no `PG…` variable or other setting of the owner's shell reaches it |
| The stand-in Driver | First on the search path, under the names `arbol-claude-driver` and `arbol-codex-driver` (below) |
| The starter | The suite starts `run --detach` as `arbol-root` does: detached from the terminal in a process group of its own, with standard input from `/dev/null`, reading one line ([RCT-2](02-starting-stopping-and-settling.md#how-the-root-manager-uses-a-trunk)). Every start names the Driver, the Subscription and the model, as Arbol's starts do (ARC-0), so that no scenario depends on a Trunk's own settings ([RCT-0](00-overview.md#a-trunks-own-settings)), of which the home of the test has none |
| Shorter waits | `ARBOL_TEST_TIME_SCALE=<n>` in the environment, 10 unless a scenario says otherwise. The Trunk and the stand-in divide by it the waits of this contract that do not depend on how fast a program starts or a connection is made: the 10 seconds of the heartbeat, the 5 seconds of a wait in the work loop other than a database call, the 60 seconds and the one-second look of the Trunk's watch on itself, the 5 seconds between SIGTERM and SIGKILL, the second of `control`, the 5 and 60 seconds of the database tries while working, before a run and at the end, the 18 seconds of a stop, the minute added to the idle limit, and the Driver's own waits, the idle limit among them (DRV-5), and the waits a Trunk's own specification sets inside them, such as the time it gives its Driver to end after a stop. They do not divide the 15 seconds of the answer to a start, the 10 seconds of the acceptance's work with the database, the 5 seconds of one database call and the 2 seconds of the Keychain, because starting a program and a first connection take as long in a test as anywhere; nor limits a request names, nor the size limits of `events.jsonl` |
| A Trunk that hangs on purpose | `ARBOL_TEST_HANG=1` in the environment: the Trunk's work stops for ever once it has written the first event of its first Driver run to `events.jsonl`, and its watch on itself goes on ([RCT-1](01-the-exchange-folder.md#the-trunk-watches-itself)) |

<!-- sources:
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/clock.py#scaled
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/run.py#drive
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/watch.py#Watch
arbol-trunk:trunks/trivial/tests/test_time_scale.py#TimeScaledWaits
arbol-trunk:trunks/trivial/tests/test_watch.py#DeliberateHang
-->

### The stand-in Driver

One program for every Trunk, part of the suite: `contract/conformance/standin/driver.py` ([RCT-D26](#decisions)). It takes a Driver's command line (DRV-2): an optional `-v <build>` first, `run` with `--request <file>`, `--control` and the other options, `runs stop <run>`, and `info`, which names contract 1. It plays the script of the scenario named in `<home>/standin/scenario`, writes events as a Driver does, stops a run as a Driver does, and appends to `<home>/standin/received.jsonl` one record for each start, when that start ends: the launcher name it was started by, its arguments, the request file as it read it, every line of its control input, every line it wrote on its standard output, and every signal it got ([RCT-D33](#decisions)). Where a scenario says so, it holds a run open after its `started` event until the suite writes `<home>/standin/release` ([RCT-D34](#decisions)), and it starts a child in a process group of its own, as a Driver starts its vendor tool, and ends that child when it gets SIGTERM, as a Driver ends the tool on a stop. A run with `"tools": "none"` and no chat session, such as a title, gets one `text` event, "Stand-in title", and completes, unless the scenario makes it fail.

### The scenarios

| ID | Scenario | The suite judges | Covers |
|---|---|---|---|
| R1 | A plain answer in the background: the stand-in writes text, a `shell` call, more text and usage, and completes | One acceptance line, and when it came the folder, the chat, the open Exchange and the message at position 0 were there; the process lived on after the suite closed its end of the pipe; the stand-in was started through its launcher name with `--control`; `n` from 1 without gaps, every event as the stand-in wrote it, the end last; the rows at their positions, the run of the kind `agent` with its usage and its `started` event; the Exchange completed; the folder gone | RCT-0.4, 0.6, 0.14, 1.1, 1.2, 1.5, 2.1, 2.4, 2.9, 2.20, 3.1, 3.2, 3.3 |
| R2 | The same in the foreground, a failed and a cancelled one, Ctrl-C, and a standard output that nothing reads | Standard output is the acceptance line and then the file's lines; the exit status is 0, 1 and 2; after SIGINT to the group no end is written and `settle` saves the Exchange as interrupted; with standard output never read, the Exchange completes | RCT-2.5, 2.8, 2.19 |
| R3 | Refusals: an unknown option, no message, a Root identifier that is not one word, such as `../rct`, an unknown Root, a version not in the Build's list, a working folder that is relative or missing, a busy chat, `--replaces` with a message, a stopped database server | One error line with the code of RCT-2, the exit status 1, within 15 seconds; no folder, no row, and the stand-in never started | RCT-0.3, 0.5, 2.2, 2.13, 2.17 |
| R4 | A repeat: the same Exchange identifier once while the first start is accepting, once after it has ended, and once after a first start that died before the database had it; two starts of it at the same moment; and a start whose starter closed its end of the pipe before reading the line, then repeated | `in_progress` while the first accepts, and for one of the two at the same moment; `"duplicate": true` with the first Exchange after it ended; a new Exchange after the dead start, whose folder is gone; after the closed pipe, `"duplicate": true` on the repeat, the stand-in started once and the Exchange completed | RCT-2.3, 2.18, 2.19 |
| R5 | A chat goes on: a second message with `--chat`, a message after a failed and after a cancelled Exchange, a message without `--cwd`, and two messages to one chat at the same moment | The numbers 2 and 3; each accepted; the working folder is the folder the suite started the Trunk from; of the two at the same moment one is accepted and the other refused with `chat_busy` | RCT-0.4, 0.5, 2.17 |
| R6 | Signs of life: the stand-in says nothing for 60 seconds (divided), with a tool call open, then with a question open | The heartbeat rises at least every 10 seconds (divided); `waitingFor` is `driver`, `tool`, `answer`; `since` moves when it changes | RCT-1.3 |
| R7 | A Trunk that hangs: `ARBOL_TEST_HANG=1`, while the stand-in's child runs in a process group of its own | Within 60 seconds of the last heartbeat, the look of the watch and the 5 seconds to SIGKILL, all divided, and a second for margin, nothing of the Trunk's group lives and the stand-in's child is gone; no end was written; `settle` saves the Exchange as interrupted | RCT-1.4, 2.6, 2.12 |
| R8 | Stop: a stop line while text streams, while a tool call runs, while a question is open, and one written while the folder still has its dot name ([RCT-D35](#decisions)) | The stand-in got the line unchanged; the end `cancelled` in the folder within 18 seconds (divided); the running call saved as interrupted; the Exchange cancelled; the folder gone; for the early stop, no run started | RCT-1.7, 2.7 |
| R9 | An answer to a question | The stand-in got the answer line unchanged, and the run went on; the answer is a line of `events.jsonl`, and the question's row holds the question and the answer | RCT-1.7, 1.11 |
| R10 | The termination signal in the middle of a run, while the stand-in's child runs in a process group of its own | No end written; the stand-in got SIGTERM; nothing of the group and not the child lives; the folder stays; `settle` saves the rows, the open call and the run as interrupted, and the Exchange as interrupted | RCT-2.8, 2.12 |
| R11 | Failures: the stand-in ends with `rate_limited` and a time to wait, refuses with `not_signed_in`, and is missing from the search path; and the suite makes the Exchange's folder read-only after the acceptance line | The Exchange failed with the stand-in's error object whole, `retryAfterSeconds` included; with `driver_failed` when it was missing; the stand-in started once, and with the route the request named; for the read-only folder, the Exchange failed with `folder_unwritable` in the database, and `settle` then removed the folder | RCT-2.9, 2.10, 2.20 |
| R12 | The database away: the suite stops its server in the middle of a run and starts it again; then right after the acceptance line, for longer than 60 seconds (divided); then until after the end, for longer than 60 seconds (divided) | The lines go on arriving meanwhile, and the rows are saved, in order, once it is back; in the second part the Exchange completes, or fails with `database_unavailable` when the Trunk had to read before its run, and never hangs; in the third the Trunk ends with `end` in its record and its folder kept, and `settle`, once the server is back, saves the Exchange as completed | RCT-2.11, 2.12 |
| R13 | Settling at its edges: an unknown Exchange, one whose process lives, the same Exchange twice, two settles of it at the same moment, a folder whose Exchange the database does not have, a folder whose Exchange has ended in the database, a folder of an Exchange that a message sent again has replaced, and a last line without a newline | `none`; `alive`; the second finds nothing; of the two at the same moment, both end with the exit status 0 and the tables are as after one settle; the folder removed with nothing written; the state in the database printed and nothing changed; for the replaced Exchange, its runs saved and no row in `messages` or `tool_calls`; the unfinished line skipped; no Exchange that had ended changed | RCT-1.9, 2.12, 3.4 |
| R14 | Sending again: the last Exchange once it has ended, then a next message; then one that is not the last, one that is open, one already replaced, and one that does not exist | The new Exchange has the next number and the same message, and the old one is replaced by it, has no rows in `messages` and `tool_calls`, and keeps its rows in `runs`; the request of the message sent again and that of the next message, in `received.jsonl`, hold nothing of the replaced Exchange; `cannot_replace`, `cannot_replace`, `cannot_replace` and `not_found` | RCT-2.13, 3.4, 3.9 |
| R15 | Size limits: one tool call streams 2 MiB of output; then the stand-in streams text until `events.jsonl` would pass 256 MiB | In the first, about 1 MiB of that call's output in the file, every other line written, and the Exchange completed; in the second, the file at most 256 MiB and the closing lines, every line whole, `n` without gaps, every saved tool call with its `tool_started` in the file, the Exchange failed with `events_file_full`, and what was finished before saved | RCT-1.6, 1.12 |
| R16 | No table changed: after every scenario | The tables, columns, indexes and checks of the database are those `layout.sql` made, and no row belongs to an Exchange the scenario did not run | RCT-3.1 |
| R17 | Characters PostgreSQL cannot store: a U+0000 in a `shell` result, an emoji whose two halves come in two `text` events, and a `tool_finished` that says both `ok` and `interrupted` | U+FFFD in their place in the rows and nothing else changed; the file as the stand-in wrote it; the flags saved as sent; the Exchange completed | RCT-3.10 |
| R18 | A Driver that falls silent, at the default scale of 10, about 96 seconds: the stand-in gives no event for its idle limit and a minute (divided), with nothing open | The stand-in got a stop and the Exchange failed with `driver_silent` | RCT-2.21 |
| R19 | Small calls: a start without a small-call route; one with a route whose small call completes; one whose small call fails | Without a route, no run with `"tools": "none"`; a small call that ran was a run with `"tools": "none"` on the route handed, and left a `smallRun` line and a `runs` row of the kind `small` with its usage; the Exchange completed in every part. A Trunk that makes no small call passes when every part completes | RCT-1.11, 2.22 |

<!-- sources:
arbol-root:contract/conformance/rct_conformance/scenarios/r4.py
arbol-root:contract/conformance/rct_conformance/scenarios/r13.py
arbol-root:contract/conformance/rct_conformance/scenarios/r14.py
arbol-root:contract/conformance/rct_conformance/registry.py
arbol-root:contract/conformance/rct_conformance/scenarios/r6.py
arbol-root:contract/conformance/rct_conformance/scenarios/r7.py
-->

### How a Trunk Build's check runs it

A Trunk's `arbol-build.json` names the suite as its check of the kind `contract`, once for each version in its `rootContracts`. It runs the suite from `~/repos/sylvan-stack/readonly/arbol-root`, the clone of the last merged `main` ([BLD-D18](../../arbol/builds/00-builds-and-versions.md#decisions)). The check starts the Build's own program, not the launcher, as every check does (BLD-0):

```json
{
  "component": "arbol-trunk/trivial",
  "rootContracts": [1],
  "checks": [
    { "name": "contract-1", "kind": "contract",
      "command": "\"$ARBOL_BUILD_DIR/<its Python>\" ~/repos/sylvan-stack/readonly/arbol-root/contract/conformance/bin/rct-conformance.py --trunk \"$ARBOL_BUILD_DIR/<its entry>\" --root-contract 1" }
  ]
}
```

`<its Python>` is the interpreter the Build carries and `<its entry>` the program its record names; both belong to the Trunk's own specification. A Build that does not pass every scenario for every version it lists is `failed` (BLD-0). A machine without the programs of PostgreSQL cannot check a Trunk Build.

## Requirements

The suite:

| ID | The suite must… | Origin | Verified by |
|---|---|---|---|
| RCT-4.1 | start a Trunk Build's own program as its starter would, in a home folder of its own with a Root record, a database server and a database of the test's own, with the stand-in Driver and a minimal environment, and need no network, no account and no quota | New | unit: `Setup.test_setup_makes_the_home_the_server_the_root_record_and_the_database`, `Setup.test_the_starter_starts_run_detach_as_arbol_root_does`, `Setup.test_the_trunk_gets_only_home_path_lang_and_the_tests_own_variables`, `Setup.test_the_owners_postgresql_settings_reach_neither_the_suite_nor_the_trunk`, `Setup.test_closing_leaves_nothing_behind`, `Setup.test_the_stand_in_is_what_the_trunk_starts_as_its_driver`, the eleven tests of `StandIn`, `Cli.test_without_initdb_on_the_search_path_it_ends_with_2_and_names_it`, `Cli.test_wrong_input_ends_with_2_and_says_what_is_wrong`, in `contract/conformance/tests/` (SYL-170); the three tests of `Starts` (SYL-173) |
| RCT-4.2 | make each scenario's database from the `layout.sql` of the version under test, and judge only by what the Trunk prints, its exit status, the Exchange folder, the tables and `received.jsonl` | New | unit: `Setup.test_setup_makes_the_home_the_server_the_root_record_and_the_database`, `Setup.test_a_command_runs_to_its_end`, `Readers.test_readers_skip_dot_folders_and_an_unfinished_last_line`, the four tests of `R16`, `Runner.test_reports_each_scenario_and_runs_r16_after_each`, `Cli.test_a_whole_run_fails_while_scenarios_are_not_implemented`, in `contract/conformance/tests/` (SYL-170) |
| RCT-4.3 | have a scenario for every requirement of RCT-0 to RCT-3 that binds a Trunk and names `contract` as its check | New | unit |

A Trunk:

| ID | A Trunk must… | Origin | Verified by |
|---|---|---|---|
| RCT-4.4 | pass every scenario for every version in its `rootContracts`, as a check of the kind `contract` of every Build | New | contract |
| RCT-4.5 | divide by the number in `ARBOL_TEST_TIME_SCALE` the waits this specification lists as divided, and stop its work for ever once it has written the first event of its first Driver run, keeping its watch on itself, when `ARBOL_TEST_HANG` is `1` | New | contract: R6 and R7 (SYL-176); unit: `TimeScaledWaits`, `DeliberateHang`, `Watching`, `Loop`, `StoppingTheRun` and `Termination` in `trunks/trivial/tests/` of `arbol-trunk` |

## Interface

```ts
// rct-conformance.py --trunk <command> --root-contract <version> [--only <ids>]
//   exit status 0: every scenario passed; 1: one failed; 2: wrong input, or PostgreSQL's programs missing

// <home>/standin/scenario: the scenario and its part, such as "R8 tool"
// <home>/standin/received.jsonl: one line for each start of the stand-in
// <home>/standin/release: written by the suite to let a run that the stand-in holds open go on; naming a part removes it
interface StandInRecord {
  scenario: string;                    // "R8"
  part: string;                        // "tool"; "" when the scenario has one part
  launcher: string;                    // the name it was started by, "arbol-claude-driver" or "arbol-codex-driver"
  arguments: string[];                 // ["run", "--control", "--request", "/…/run-1.json"]
  request: unknown;                    // the request file as the stand-in read it (DRV-2 RunRequest)
  control: unknown[];                  // every line of its control input, in order
  events: string[];                    // every line it wrote on its standard output, without the newline, in order
  signals: string[];                   // ["SIGTERM"]
  workingDirectory: string;
}
```

## Known problems in version 1

Not to be reproduced.

- **Tests instead of the installed product.** Each step was accepted on tests and isolated harnesses while the installed application was seldom run from end to end; the first real cutover found 24 defects in two days.
- **A test that reached real data.** A fixture dropped every schema with a test name in whatever database it reached, and a scratch database alone did not keep sockets and launch records apart.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:product-spec/12-lessons-and-invariants.md#1. The architecture grew faster than the installed product was exercised`
- `arbol_deprecated_v1:product-spec/12-lessons-and-invariants.md#Validation practice`

## Decisions

| ID | Question | Proposed default | Outcome |
|---|---|---|---|
| RCT-D26 | Where does the suite live, and what does it start in place of a Driver and of the database server? | In the `arbol-root` repository, `contract/conformance`, beside the layout it checks against. One stand-in Driver, part of the suite, serves every Trunk, because it stands for the Driver Contract and not for one vendor tool. Each scenario gets a PostgreSQL server of its own in a temporary folder, reached only through a socket, so no test can touch a Root in use. The alternative: a scratch database on the installed server, which needs no `initdb` but puts tests and chats on one server | Decided 15 October 2026, a technical choice: the proposed default |
| RCT-D27 | How can the suite check waits of 60 seconds, and a Trunk whose work hangs? | Two variables that every Trunk honours. `ARBOL_TEST_TIME_SCALE`, 10 by default, divides the waits of this contract, as for a Driver (DRV-5), except those that depend on starting a program and connecting to the database, which a test cannot make faster. `ARBOL_TEST_HANG=1` stops the Trunk's work once its first Driver run has given an event, while its watch on itself goes on. Without the second, the watch could be checked only by a drill. The alternative: divide every wait by 20, which leaves less than a second to start a Trunk and connect, so a correct Build would fail on a busy machine | Decided 15 October 2026, a technical choice: the proposed default |
| RCT-D29 | ARC-1 says a test runs on a folder and a database named with options (ARC-D37). May a Trunk Build, which belongs to no Root, be tested in a home folder of its own instead, as a Driver is (DRV-5)? | Yes. The home folder leads the Trunk to a Root record, a settings file and a database server that are all the test's own, so no Trunk needs options for tests, and no test can reach the chats of a Root in use; ARC-1 then says that its rule is for the Builds of a Root. If no: every Trunk takes options that name another Root folder and another database settings file, and the suite passes them | Decided 15 October 2026 by the owner: Trunks and Drivers are both tested in a separate test folder of their own ("Can we test both Trunks and Drivers in separate test folder?"); Drivers already are (DRV-5). A Trunk Build is tested in a temporary home of its own, and ARC-D37's options stay for a Root's own Builds. The live checks are the one exception, because they need the real test account kept in the Driver's test state folder (TRV-0) |
| RCT-D32 | How does the suite run while only some of its scenarios exist, and how does R16 know what the scenario changed? | A whole run lists every scenario of the table above and fails each one that is not written yet, and `--only` refuses to choose one, so that no Trunk Build passes against half a suite. R16 compares PostgreSQL's catalogue of the database, read right after `layout.sql` made it and before any Trunk started, with the catalogue after the scenario, and counts as run only the Exchanges that the scenario's acceptance lines named. The Trunk's search path is the stand-in's folder and then `/usr/bin:/bin:/usr/sbin:/sbin`, and `LANG` is `en_US.UTF-8`. The stand-in writes each start's record when that start ends, naming the launcher it was started by. A suite stopped with SIGINT or SIGTERM removes the scenario's home and ends with 2, as the Driver Contract's suite does (DRV-5). The alternative: run only the scenarios written so far, which a Build of a half-built Trunk would pass | Decided 16 October 2026, a technical choice (SYL-170) |
| RCT-D33 | How does a scenario judge that `events.jsonl` holds every event exactly as the Driver wrote it? | The stand-in records in each start's `StandInRecord`, as `events`, every line it wrote on its standard output, and the scenario compares each with the `event` of the line of `events.jsonl` that carries it. The alternative: the scenario rebuilds the events from the script it named, which cannot see a field the stand-in adds, such as `at` | Decided 16 October 2026, a technical choice (SYL-171) |
| RCT-D34 | How does a scenario keep an Exchange open for as long as it needs, such as the open Exchange of a busy chat in R3? | The stand-in's script holds the run open after its `started` event until the suite writes `<home>/standin/release`, taking control lines and signals meanwhile, and the suite writes it once it has judged what it needed. The alternative: a silence of a fixed length, which makes every run of the scenario wait for it, and fails when the judging takes longer | Decided 16 October 2026, a technical choice (SYL-172) |
| RCT-D35 | How does R8 write a stop line while the folder still has its dot name, when that name lasts only as long as the Trunk takes to write three small files? | The suite plays a writer that found the folder under its dot name: from the moment it starts the Trunk it waits for `<exchange>/control`, at most 15 seconds (RCT-1.8), and appends the line. Meanwhile it holds every acceptance with a lock on `chats` (`LOCK TABLE chats IN SHARE MODE` in an open transaction of a `psql` session of its own), which lets the tables be read and no row be written, and releases it once the line is written. So the line is in `control` before the acceptance ends and before the Trunk could start its run, however fast the Trunk is, and the early stop is judged on every run. The alternative: write the line as soon as the folder is seen, which a fast Trunk often outruns by starting its run first, so the scenario would pass or fail by chance | Decided 18 October 2026, a technical choice (SYL-173) |
