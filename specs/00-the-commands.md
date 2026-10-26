---
role: authored
spec: RTM-0
status: accepted
---
# Root manager · 00 The commands

`arbol-root` is the program that manages the [Arbol Trunks](../../arbol/GLOSSARY.md#arbol-trunk) of one [Root](../../arbol/GLOSSARY.md#arbol-root). Each command does one thing and ends: it lists the Trunks that fit the Root, starts a Trunk for a new [Chat Exchange](../../arbol/GLOSSARY.md#chat-exchange), lists the Exchanges in progress, stops one, or has a Trunk [Build](../../arbol/GLOSSARY.md#build) *settle* an Exchange whose process is gone, which means saving what its folder holds. For example, `arbol-root --root sapling list` prints every Exchange in progress in First Sapling with its heartbeat, and `arbol-root --root sapling stop 7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b` stops one of them. This specification states where a command takes its Root from, the commands `trunks`, `start`, `list`, `stop` and `settle` with what each prints, its exit status and its refusals, which settings the Root manager has, and how a test points it at other folders. It matters because in version 1 Arbol Root was a resident service that did three jobs at once, so restarting it touched every chat, and a chat whose turn had died stayed "running" until one of about fifteen repair scripts was run.

It follows the conventions of the [Driver Contract](../../arbol/driver-contract/README.md#conventions) and the additions in this chapter's [README](README.md). The Root manager is one party of the [Root Contract](../contract/README.md), version 1, and uses its terms: it starts Trunks as [RCT-2](../contract/02-starting-stopping-and-settling.md#how-the-root-manager-uses-a-trunk) says, reads the records of [RCT-1](../contract/01-the-exchange-folder.md) and writes stop lines into their `control` file. The owner decided on 15 October 2026 that the Root manager starts Trunks with the one command `arbol-trunk` and an exact `-v` ([RTM-D1](#decisions)). The other forms are technical choices made the same day. The owner answered the open questions on 15 October 2026, among them that a Root's settings narrow nothing, so the Root manager has no settings ([RCT-D28](../contract/00-overview.md#decisions), [RTM-D3](#decisions)), and the specification was accepted the same day.

## Scope

Covered: the Root of a command, the command line and its replies, the Trunks that fit, starting a Trunk, the listing, stopping, settling, settings, tests.

Not covered: what a Build of `arbol-root` carries, its table layouts and its migration script ([RTM-1](01-builds-layouts-and-migration.md)); the folder of an Exchange, the commands of a Trunk and what settling saves ([RCT-1](../contract/01-the-exchange-folder.md), [RCT-2](../contract/02-starting-stopping-and-settling.md)); choosing a Build of `arbol-root` and naming a Root to its launcher ([BLD-0](../../arbol/builds/00-builds-and-versions.md#running-a-build)); when Arbol lists, stops and settles, what it shows, and how it asks every Root for its Exchanges, which belong to Arbol's own specification ([ARC-1](../../arbol/architecture/01-roots.md#discovery-across-roots)).

## Behaviour

### The Root of a command

Every Build of `arbol-root` is made for one Root (BLD-0), and its launcher picks that Build. A command works on the Root that the `root` of the Build's own record `build.json` names. A program started outside any Build, as in development, needs the option `--root <root>` and reports the build number `0` (BLD-0). A Root's identifier is one lower-case word of letters and digits, such as `sapling` (ARC-1); any other, such as `../sapling`, is refused with `invalid_input`. The Root's folder is `~/Library/Application Support/Arbol/roots/<root>/`. When that folder holds no `root.json`, every command is refused with `unknown_root`.

```sh
arbol-root --root sapling list     # First Sapling's newest working Build of arbol-root lists First Sapling's Exchanges
arbol-root -v 5 list              # Build 5 of arbol-root, in the Root its record names
```

The replies the Root manager makes itself, those of `list`, `stop` and `settle`, name the Root as `root`; so do the reply of `trunks` and the acceptance line of `start`, which it passes on from `arbol-trunk`. A refusal has the form below and names no Root. The listing also names the build number and the package version of the `arbol-root` Build that made it, as `rootBuild` and `rootPackageVersion`, so the Root and the Build of its manager are never taken for each other (ARC-0, BLD-0). A newer Build of the same Root lists, stops and settles the Exchanges that an older Build started, because it knows them only from their records.

<!-- sources:
arbol-root:src/arbol_root/model.py#identity
arbol-root:src/arbol_root/model.py#build_record
arbol-root:src/arbol_root/model.py#object_file
arbol-root:tests/test_commands.py#Commands
-->

### The command line

Implementation choices (RTM-D14): the program accepts its common options before or after the command, in either value form, and refuses duplicate options, missing values and surplus arguments. Development requires an explicit Root; it never chooses the current Root itself. A packaged program reads `build.json` beside its own package, independent of caller environment variables. A malformed Build record is `invalid_input`, never development mode. A missing, nonregular, unreadable or malformed Root record is `unknown_root`; an object must name the selected Root. These reads open no settings file.


A Trunk is named by its *code name*, such as `trivial` or `advanced` ([RCT-0](../contract/00-overview.md#trunks-and-their-code-names)).

| Command | What it does | Prints | Exit status |
|---|---|---|---|
| `trunks [<code name>]` | Lists the Trunks and their Builds, for this Root | The reply of `arbol-trunk list` (RCT-0) | 0; 1 refused |
| `start [<code name>] [-v <build>] --exchange <id> --request <file> [<options of run>]` | Starts a Trunk for a new Exchange | The Trunk's acceptance line or refusal (RCT-2) | 0 accepted; 1 refused |
| `list` | Lists the Exchanges in progress | `Listing` | 0; 1 refused |
| `stop [--force] <exchange>` | Stops one Exchange | `StopResult` | 0 when the Exchange has ended: its record has `end`, its folder is gone, or its processes are gone; 1 refused |
| `settle [<exchange>]` | Has every Exchange whose process is gone settled, or the one named | `SettleResult` | 0 when every one was settled; 1 otherwise |

Options are written `--name value` or `--name=value`, after the command. Each command prints one JSON document on standard output; standard error is for a person only. A refusal is `{ "error": { "code", "message" } }` with the exit status 1, as with `arbol-roots` and `arbol-builds` (ARC-1, BLD-0). A refusal that comes from `arbol-trunk` is passed on as it came, its `available` included.

| Code | When |
|---|---|
| `invalid_input` | An unknown command or option; an Exchange identifier that is missing or not a UUID; `start` without `--exchange` or `--request`, or with a second argument; a Root identifier that is not one lower-case word; no `--root` outside a Build; `--root-dir` without `--bin-dir`; a `--root` given to a Build's program that differs from its record |
| `unknown_root` | The Root's folder holds no `root.json` |
| `no_answer` | `start`: no line that is a JSON object came from the Trunk within 20 seconds, or its output ended without one |
| `not_ended` | `stop`: something of the Exchange's process group still lives 5 seconds after SIGKILL |
| `trunk_unavailable` | `arbol-trunk` could not be started from the folder of the launchers, or its `list` or `settle` ended without a line that is a JSON object; the message says which |
| `no_fit` | Passed on from `arbol-trunk`; and the Root manager's own when `arbol-trunk list` marks no Build of the Trunk `chosen` ([below](#starting-a-trunk)) |
| `unknown_trunk` | Passed on from `arbol-trunk` ([RCT-0](../contract/00-overview.md#the-command-arbol-trunk)) |
| The codes of RCT-2 | Passed on from the Trunk: as the reply of `start`, and in the results of `settle` |

<!-- sources:
arbol-root:src/arbol_root/cli.py#parse
arbol-root:src/arbol_root/cli.py#main
arbol-root:bin/arbol-root
arbol-root:bin/entry.py
-->

### The Trunks that fit

`arbol-root trunks [<code name>]` runs `arbol-trunk --root <root> [<code name>] list` and prints its first UTF-8 line that is a JSON object unchanged, including whitespace and extra refusal fields (RTM-D14). It forwards the child's standard error to standard error. A reply with an error or a nonzero child status returns 1; otherwise it returns 0. Diagnostic lines and non-object JSON lines are not replies. No object line is `trunk_unavailable`. The launcher already knows which Builds fit the Root and which one `run` would start (RCT-0), so the Root manager reads no Build record itself.

```sh
arbol-root --root sapling trunks

<!-- sources:
arbol-root:src/arbol_root/cli.py#trunks
arbol-root:tests/standin_trunk.py
arbol-root:tests/test_commands.py#Commands.test_trunks_forwards_reply_bytes_exact_arguments_and_launcher_path
arbol-root:scripts/verify_build.py#qualify
-->

# {"root":"sapling","rootContract":1,"trunks":[{"trunk":"trivial","default":true,"builds":[{"build":"4","status":"working","rootContracts":[1,2],"fits":true,"runnable":true,"chosen":true}]}]}
```

### Starting a Trunk

A *request file* is the JSON file of [RCT-0](../contract/00-overview.md#what-is-handed-to-a-trunk) that holds the message, the route, the chat and the working folder. The caller writes it, and names the Exchange with `--exchange`, a UUID it chooses, so that it can start the same Exchange again when it lost the answer ([RCT-D8](../contract/00-overview.md#decisions)). `arbol-root start` hands the file to the Trunk without opening it:

1. The Trunk is the one whose code name is given as the one argument of `start`, and otherwise `trivial`. A second argument is refused with `invalid_input`, because the message is only in the request file. Without `-v`, the Root manager runs `arbol-trunk --root <root> <code name> list` and takes the Build marked `chosen`. When none is marked, it refuses with `no_fit`, and `available` lists the Builds of that Trunk that fit and are `runnable`, which run only when they are named (BLD-0). A refusal of the launcher, such as `unknown_trunk`, is printed as it came.
2. It starts `arbol-trunk --root <root> <code name> -v <build> run --detach --exchange <id> --request <file>`, followed unchanged by the options of `run` that the caller gave: `--chat`, `--replaces`, `--cwd`, `--driver`, `--subscription`, `--model` and `--effort`. It starts it in its own working folder, in the way RCT-2 states. `--exchange` takes the place of any `exchange` in the request file (RCT-0).
3. It waits for the Trunk's one line at most 20 seconds: the 15 seconds within which a Trunk answers (RCT-2), and 5 for starting the launcher and the Build. It then removes the request file and prints the line unchanged.
4. When no line that is a JSON object came within 20 seconds, or the Trunk's output ended without one, it refuses with `no_answer`. It leaves the request file and the process alone. The caller learns what happened by starting the same Exchange again, which answers `"duplicate": true` when the message was accepted (RCT-2).

```sh
arbol-root --root sapling start --exchange 7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b --request /…/start.json
#   runs: arbol-trunk --root sapling trivial -v 4 run --detach --exchange 7f3a2c1e-… --request /…/start.json
# {"accepted":true,"duplicate":false,"root":"sapling","exchange":"7f3a2c1e-…","chat":"c3d4e5f6-…","number":1,"trunk":"trivial","build":"4"}
```

Decided ([RTM-D1](#decisions)): the Root manager names the exact Build it starts, so the Build it chose and the Build that runs are always the same.

### The listing

`arbol-root list` reads every folder in `exchanges/` whose name does not start with a dot, reads only `record.json` in it, and checks the record's process as RCT-1 says. It reads a process's start time with `/bin/ps`, in a clean environment and the C locale, as a Driver does ([DRV-D36](../../arbol/driver-contract/06-state-and-simultaneous-calls.md#decisions)). It starts nothing and settles nothing, so a listing every 30 seconds (ARC-3) stays quick and needs no database ([RTM-D2](#decisions)).

Each Exchange gets a *state*, from the rows of [RCT-1](../contract/01-the-exchange-folder.md#what-a-folder-tells-a-reader):

| The record | Its process | `state` | What the caller does with it |
|---|---|---|---|
| has no `acceptedAt` | lives | `accepting` | Waits |
| has no `acceptedAt` | is gone | `unaccepted` | Has it settled: settling removes the folder when the database does not have the Exchange |
| has no `end` | lives | `running` | Shows it with its signs of life |
| has `end` | lives | `ending` | Waits |
| has `end` | is gone | `unsaved` | Shows the answer from the folder, marked as not yet saved, and has it settled once the database is back ([ARC-D53](../../arbol/architecture/02-chat-exchange-across-components.md#decisions)) |
| has no `end` | is gone | `interrupted` | Has it settled |

```json
{
  "root": "sapling",
  "rootBuild": "6",
  "rootPackageVersion": "1.0.0",
  "exchanges": [
    { "root": "sapling", "exchange": "7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b", "chat": "c3d4e5f6-0718-4293-a4b5-c6d7e8f90a1b",
      "trunk": "trivial", "build": "4",
      "process": { "pid": 48213, "startedAt": "2026-10-15T09:30:00.000Z", "alive": true },
      "state": "running",
      "signsOfLife": { "heartbeat": 41, "waitingFor": "tool", "since": "2026-10-15T09:31:02.000Z" } }
  ],
  "unreadable": []
}
```

The listings of several Roots are combined by putting their `exchanges` together, because each entry names its Root. The form is the same in every Root and every Build of `arbol-root`: no field is removed or given another meaning. A caller ignores a field it does not know, and shows a value it does not know, such as a new `waitingFor`, as it is given. A folder that is gone by the time its record is read is left out, because its Exchange has just ended. A folder whose record is there but cannot be read as a record is named in `unreadable` and does not stop the listing. A Root whose listing is refused or fails is reported by its caller for that Root alone (ARC-1).

### Stopping

`arbol-root stop <exchange>`:

1. When the Exchange has no folder, but `.<exchange>.new` exists, it waits up to 15 seconds for the folder to be renamed into place (RCT-1). When there is still no folder `<exchange>`, the Exchange has ended or is unknown, and the stop succeeds with `"ended": "none"` (ARC-3).
2. When the record's process is gone, it writes no stop line, has the Exchange settled at once, and adds what settling printed, as `settling`.
3. Otherwise it appends `{"type":"stop"}` to `control` and reads the record every 100 milliseconds. Once the record has `end`, or the folder is gone, the Exchange has ended (RCT-2). When the record's process is gone without an `end`, it has the Exchange settled at once, as in step 2.
4. When the Exchange has not ended after 20 seconds, it ends it by force ([ARC-D60](../../arbol/architecture/03-stuck-trunks.md#decisions)).

Ending by force, with `stop --force <exchange>` or after those 20 seconds:

1. It sends SIGTERM to the process group that the record's process leads, only while that process lives (RCT-2).
2. 5 seconds later it sends SIGKILL to the group, when anything of it still lives. When something of it still lives 5 seconds after that, it refuses with `not_ended`.
3. It has the Exchange settled ([below](#settling)) and adds what settling printed, as `settling`.

The 20 seconds and the 5 seconds are counted with the clock that macOS calls `CLOCK_UPTIME_RAW`, which stands still while the Mac sleeps (ARC-3).

The Root manager reads only the record, and a Trunk removes its folder soon after it has written its end. A stop therefore reports what the Root manager could see ([RTM-D6](#decisions)). `forced` is true when it sent the process group a signal.

| `ended` | Meaning |
|---|---|
| `completed`, `failed`, `cancelled`, `interrupted` | When settling printed one of these: that state, which the database now holds. Otherwise the state of the `end` it read in the record, or `interrupted` when the record had no `end` and settling was refused, for example because the database is away |
| `saved` | The folder went while the stop ran, before its `end` could be read, or settling printed `none`: the Exchange ended and its rows are saved, or it was never accepted. The chat tables say which |
| `none` | There was no folder when the stop began: the Exchange had ended before, or is unknown |

```sh
arbol-root --root sapling stop 7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b
# {"root":"sapling","exchange":"7f3a2c1e-…","ended":"cancelled","forced":false}
arbol-root --root sapling stop --force 6e2f1a0b-3c4d-4e5f-8a6b-7c8d9e0f1a2b
# {"root":"sapling","exchange":"6e2f1a0b-…","ended":"interrupted","forced":true,"settling":{"exchange":"6e2f1a0b-…","settled":"interrupted"}}
```

Observed in version 1: stop ended the agent's whole process group, with a forced kill after 2 seconds and 15 seconds in total, and cancel doubled as repair: on a chat that was marked running although nothing ran, it reset the chat to idle.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:product-spec/03-turn-execution.md#Cancel`

### Settling

`arbol-root settle` has every Exchange whose record's process is gone settled, one after another: those in the states `unaccepted`, `unsaved` and `interrupted`. `arbol-root settle <exchange>` settles only that one, and the Trunk refuses with `alive` when its process lives.

Each is settled by `arbol-trunk --root <root> trivial -v <build> settle <exchange>`, with the Build that `arbol-trunk --root <root> trivial list` marks `chosen` ([RTM-D5](#decisions)). The Trivial Trunk is the most stable Trunk, and every Trunk Build that fits the Root can settle every Exchange of the Root, also one that another Trunk ran ([RCT-D20](../contract/02-starting-stopping-and-settling.md#decisions)). When no Build of `trivial` is marked `chosen`, `settle` settles nothing and refuses with `no_fit`, as `start` does. After a refusal with `database_unavailable`, it tries no further Exchange, because the database is away for all of them. Two settles of one Exchange may run at the same time, such as the one `stop` runs and one that Arbol runs after a listing; neither fails because of the other (RCT-2).

`arbol-root settle` without an Exchange also removes what a Trunk that was killed can leave under a dot name, which every reader skips (RCT-1): every folder `.<exchange>.old`, which a Trunk makes only once the database holds the Exchange's end, and every folder `.<exchange>.new` more than 15 seconds old whose record's process is gone or that has no record, whose Exchange never reached the database (RCT-2). A file that is gone already is no error.

| Who runs `settle` | When |
|---|---|
| Arbol | When its backend starts, when it sees the database return (ARC-D53), and after a listing that shows an Exchange whose process is gone |
| `arbol-root stop` | After it ended an Exchange's processes, or found them gone |

```sh
arbol-root --root sapling settle
# {"root":"sapling","results":[{"exchange":"7f3a2c1e-…","settled":"interrupted"},{"exchange":"9b3a2c1e-…","error":{"code":"database_unavailable","message":"…"}}]}
```

A Root with nothing to settle prints `"results": []` with the exit status 0. Decided ([RTM-D4](#decisions)): `list` never settles; Arbol runs `settle`.

Observed in version 1: when a turn's end could not be recorded, the chat showed "running" for ever, and about fifteen one-off repair scripts, each keyed to single turns, were the way out.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:product-spec/12-lessons-and-invariants.md#2. One turn's failure could block everything else`

### Settings

The Root manager reads no settings file (RTM-D3). ARC-0 and BLD-0 gave its file, `~/.arbol/roots/<root>/arbol-root.json` (ARC-1), two jobs. The first, the default Trunk, is the Trivial Trunk in every Root ([RCT-D2](../contract/00-overview.md#decisions)). The second, narrowing which Builds may run, is gone: a Root limits nothing, and the Trunks it runs are those whose Builds list its Root Contract version, as `trunks` shows them (RCT-D28). The file is therefore not made, and its name is kept for the first setting the Root manager needs. For example, `~/.arbol/roots/sapling/` holds only `arbol.json`.

<!-- sources:
arbol-root:tests/test_commands.py#Commands.test_no_settings_or_exchange_payload_is_opened
-->

### Tests

The Root manager opens no database, so a test points it only at folders, with two options, as a Driver takes `--state-dir` ([ARC-1](../../arbol/architecture/01-roots.md#a-place-for-tests), [ARC-D37](../../arbol/architecture/01-roots.md#decisions), [RTM-D7](#decisions)):

| Option | In place of | What it names |
|---|---|---|
| `--root-dir <folder>` | `~/Library/Application Support/Arbol/roots/<root>/` | The Root's folder, with its `root.json` and `exchanges/` |
| `--bin-dir <folder>` | `~/.local/bin` | The folder of the launchers. `arbol-trunk` is started from it, with the folder first on the search path (RCT-0) |

`--root-dir` is refused without `--bin-dir`. A real Trunk finds its Root's folder and database through the home folder alone (RCT-0), so a test that named only another Root folder could still start a Trunk on the Root in use. A test therefore puts a stand-in `arbol-trunk` in the folder it names with `--bin-dir`. `--bin-dir` also serves an owner who installed the launchers elsewhere.

```sh
"$ARBOL_BUILD_DIR/bin/arbol-root" list --root-dir /tmp/rtm-1a2b/sapling --bin-dir /tmp/rtm-1a2b/bin
```

<!-- sources:
arbol-root:tests/test_commands.py
arbol-root:scripts/check_build.py#main
arbol-root:pyproject.toml
-->

## Requirements

The Root manager:

| ID | The Root manager must… | Origin | Verified by |
|---|---|---|---|
| RTM-0.1 | take its Root from the `root` of its own Build record, or from `--root` only when it runs outside a Build, refuse with `invalid_input` a Root identifier that is not one lower-case word and with `unknown_root` a Root whose folder holds no `root.json`, and name that Root in every reply of `list`, `stop` and `settle` | New | Root identity/refusal part: unit `root.identity`, `root.build-identity`, `root.input`; list/stop/settle replies remain SYL-182/SYL-183 |
| RTM-0.2 | end every command, keep no file between two commands, read in an Exchange folder only `record.json`, write there only stop lines in `control`, remove only the folders that `settle` removes, open no request file and no database, and start no program but `arbol-trunk`, and `/bin/ps` in a clean environment and the C locale to read a process's start time | Changed | trunks/program boundary: unit `root.no-settings`, `root.trunks`, `root.trunk-unavailable`; no settings, request or database read. Exchange commands remain SYL-182/SYL-183 |
| RTM-0.3 | on `trunks`, print the reply of `arbol-trunk --root <root> [<code name>] list` unchanged | New | unit: `root.trunks`, `root.trunks-refusal`; private integration: `root.private-build` in scripts/verify_build.py (SYL-181) |
| RTM-0.4 | on `start`, take the Exchange's identifier only from `--exchange`, which the caller has to give, and start the Build named with `-v`, or else the one that `arbol-trunk … list` marks `chosen`, of the Trunk named as the one argument or else `trivial`, as `arbol-trunk --root <root> <code name> -v <build> run --detach --exchange <id> --request <file>` followed unchanged by the caller's `--chat`, `--replaces`, `--cwd`, `--driver`, `--subscription`, `--model` and `--effort`, in its own working folder, and refuse with `no_fit` when no Build is marked `chosen` | New | unit |
| RTM-0.5 | on `start`, print the Trunk's one line unchanged, with the exit status 0 for an acceptance and 1 for a refusal, remove the request file once it has the line, and refuse with `no_answer`, leaving the file and the process alone, when no line that is a JSON object came within 20 seconds or the output ended without one | New | unit |
| RTM-0.6 | on `list`, give every folder of `exchanges/` whose name does not start with a dot, from its record alone, with its Root, Exchange, chat, Trunk, Build, process and whether it lives, its state, its signs of life and its end, name its own build number and package version as `rootBuild` and `rootPackageVersion`, leave out a folder that is gone before its record is read, and name in `unreadable` every folder whose record is there but cannot be read | New | unit |
| RTM-0.7 | keep the form of the listing in every Root and every Build, removing no field and changing the meaning of none | New | unit |
| RTM-0.8 | on `list`, start nothing and settle nothing | New | unit |
| RTM-0.9 | on `stop`, append one stop line only for a record whose process lives, read the record every 100 milliseconds until it has `end`, the folder is gone or its process is gone, have the Exchange settled at once when its process is gone without an `end`, and end the Exchange by force when it has not ended after 20 seconds | Changed | unit, drill |
| RTM-0.10 | end an Exchange by force with SIGTERM to the process group of a record whose process lives, SIGKILL 5 seconds later to whatever of the group still lives, refuse with `not_ended` when something of it lives 5 seconds after that, and otherwise have the Exchange settled | Changed | unit, drill |
| RTM-0.11 | treat the stop of an Exchange without a folder as success, with `"ended": "none"`, and have an Exchange whose process is gone settled at once, without a stop line | Kept | unit |
| RTM-0.12 | count the 20 seconds and the 5 seconds of a stop with `CLOCK_UPTIME_RAW` | New | unit |
| RTM-0.13 | on `settle`, settle every Exchange whose record's process is gone, or only the one named, one after another, each through `arbol-trunk --root <root> trivial -v <build> settle <exchange>` with the Build that `arbol-trunk --root <root> trivial list` marks `chosen`, refuse with `no_fit` when none is marked, try no further Exchange after a refusal with `database_unavailable`, and, when no Exchange is named, remove every folder `.<exchange>.old` and every folder `.<exchange>.new` more than 15 seconds old whose record's process is gone or that has no record | New | unit |
| RTM-0.14 | read no settings file | New | unit: `root.no-settings` (SYL-181) |
| RTM-0.15 | take the Root's folder from `--root-dir` and the folder of the launchers from `--bin-dir` when they are given, start `arbol-trunk` from that folder with the folder first on the search path, and refuse `--root-dir` without `--bin-dir` | New | unit: `root.identity`, `root.trunks`, `root.input` (SYL-181) |
| RTM-0.16 | refuse with `trunk_unavailable` when `arbol-trunk` cannot be started, or its `list` or `settle` ends without a line that is a JSON object | New | list reply/start failure part: unit `root.trunk-unavailable`; settle remains SYL-183 |

RTM-0.7 is checked against a listing kept in the repository, to which a later Build may only add fields. RTM-0.10's `not_ended` and RTM-0.12's clock are checked inside the program's own process, because no stand-in outlives SIGKILL or makes the Mac sleep.

## Interface

```ts
// arbol-root [--root <root>] [-v <build of arbol-root>] <command> [<options>]
//   the launcher takes --root and -v (BLD-0); a program outside a Build takes --root after the command, like any option
//   options of every command, for tests: --root-dir <folder> --bin-dir <folder>

// arbol-root … start [<code name>] [-v <build>] --exchange <id> --request <file>
//   [--chat <id>] [--replaces <id>] [--cwd <folder>] [--driver <d>] [--subscription <s>] [--model <m>] [--effort <e>]

// arbol-root … list
interface Listing {
  root: string;                        // the Root's identifier, "sapling"
  rootBuild: string;                   // the build number of the arbol-root Build that listed, "6"; "0" outside a Build (BLD-0)
  rootPackageVersion: string;          // the package version of that Build, "1.0.0" (BLD-0)
  exchanges: ListedExchange[];
  unreadable: string[];                // names of folders in exchanges/ whose record is there but could not be read
}

interface ListedExchange {
  root: string;                        // the Root again, so that listings of several Roots can be put together
  exchange: string;                    // a UUID, the folder's name
  chat: string;                        // a UUID
  trunk: string;                       // the code name of the Trunk that accepted it, "trivial"
  build: string;                       // the build number of that Trunk Build, "4"
  process: { pid: number; startedAt: string; alive: boolean };   // alive: as RCT-1 tells it
  state: "accepting" | "unaccepted" | "running" | "ending" | "unsaved" | "interrupted";
  signsOfLife: {                       // from the record (RCT-1)
    heartbeat: number;                 // 41
    waitingFor: string;                // in contract version 1 "answer", "tool", "driver", "small-call" or "database"
    since: string;                     // "2026-10-15T09:31:02.000Z"
  };
  end?: {                              // from the record, once it has one (RCT-1)
    state: string;                     // in contract version 1 "completed", "failed" or "cancelled"
    at: string;
    error?: { code: string; message: string };   // as in the record, whole: a Driver's error has more fields (DRV-4)
  };
}

// arbol-root … stop [--force] <exchange>
interface StopResult {
  root: string;                        // "sapling"
  exchange: string;
  ended: "completed" | "failed" | "cancelled" | "interrupted" | "saved" | "none";
  forced: boolean;                     // true: it sent the process group a signal
  settling?: SettleOutcome;            // when it had the Exchange settled: what settling printed
}

// arbol-root … settle [<exchange>]
interface SettleResult {
  root: string;
  results: SettleOutcome[];            // one for each Exchange, in the order they were settled
}

type SettleOutcome =
  | Settled                            // what arbol-trunk … settle printed (RCT-2)
  | { exchange: string; error: { code: string; message: string } };   // a refusal of the Trunk (RCT-2), "trunk_unavailable" or "no_fit"
```

The forms of `signsOfLife` and `end` are the listing's own, taken from the records of contract version 1; a Build for a Root of a later version gives its records in these forms. `trunks` prints `TrunkList` and `start` prints `Accepted` or a refusal, as the Root Contract gives them ([RCT-0](../contract/00-overview.md#interface), [RCT-2](../contract/02-starting-stopping-and-settling.md#interface)). `Settled` is that of RCT-2. Every time has the form that JavaScript's `toISOString` writes, as in BLD-0.

## Known problems in version 1

Not to be reproduced.

- **A resident Root with three jobs.** Arbol Root was the control plane, the gateway of every window and the place where chats were run, so a restart of Root touched everything.
- **Stop that doubled as repair.** Cancel reset a chat to idle when it was marked running although nothing ran, so a stop could not be trusted to mean that something had stopped.
- **Repair by script.** About fifteen one-off programs, keyed to single turns, were the way out of a chat whose turn had died.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:product-spec/12-lessons-and-invariants.md#6. Many resident services with strict, version-coupled contracts`
- `arbol_deprecated_v1:product-spec/03-turn-execution.md#Cancel`
- `arbol_deprecated_v1:product-spec/12-lessons-and-invariants.md#2. One turn's failure could block everything else`

## Decisions

| ID | Question | Proposed default | Outcome |
|---|---|---|---|
| RTM-D1 | How does the Root manager start a Trunk for a new Exchange, and with which Build? | With the one command of every Trunk and an exact `-v`: `arbol-trunk --root <root> <code name> -v <build> run --detach --exchange <id> --request <file> …`. The Build is the one the caller names, or else the one `arbol-trunk list` marks `chosen`. The caller names the Exchange with `--exchange`, so that it can start the same Exchange again when it lost the answer; the Root manager hands the request file on without opening it, and waits 20 seconds for the one line. The alternative: start `arbol-trunk run` without `-v` and let the launcher choose, so the Root manager learns the Build only from the acceptance line | Decided 15 October 2026 by the owner: the Root manager starts Trunks with the same command `arbol-trunk` and an exact `-v` (SYL-168, decision 9). Asking `arbol-trunk list` for the Build, the `--exchange` the caller gives, the request file it never opens and the 20 seconds are a technical choice made the same day |
| RTM-D2 | What does a listing say, so that Arbol can put the listings of several Roots together? | One JSON document per Root: the Root, the build number and package version of the `arbol-root` Build that listed, and for each Exchange its Root, chat, Trunk, Build, process, one of six states, signs of life and end, all read from the records alone. The form is the same in every Root and is never changed, only added to. A listing starts nothing and settles nothing. The alternative: the records as they are, which every caller would have to read with the rules of the Root Contract version of each Root | Decided 15 October 2026, a technical choice: the proposed default |
| RTM-D3 | What does `arbol-root.json` hold? | Nothing, as long as no setting narrows which Builds run, the proposed answer of RCT-D28: the Root manager reads no settings file, and the file is not made. The default Trunk is the Trivial Trunk in every Root (RCT-D2). If the owner answers RCT-D28 with yes, the file holds the allowed Builds of each Trunk, such as `{"builds": {"advanced": ["8f"]}}`; `start` refuses any other with `no_fit`, and without `-v` it takes the newest allowed Build that fits, refusing only when there is none. This row is answered together with RCT-D28 | Decided 15 October 2026 by the owner, together with RCT-D28: "Roots shouldn't limit anything explicitly. They only have list of compatible trunks". The file holds nothing and is not made: the Root manager reads no settings file, and the Trunks a Root runs are those whose Builds list its Root Contract version, as `arbol-root trunks` shows them |
| RTM-D4 | When is an Exchange whose process is gone settled? | When `arbol-root settle` runs: Arbol runs it when its backend starts, when it sees the database return, and after a listing that shows such an Exchange; `arbol-root stop` runs it after ending processes. `list` never settles, so a listing every 30 seconds needs no database. The alternative: every listing settles what it finds, so each listing waits up to 10 seconds for every such Exchange while the database is away | Decided 15 October 2026, a technical choice: the proposed default |
| RTM-D5 | Which Trunk Build settles an Exchange? | The Build of the Trivial Trunk that `arbol-trunk list` marks `chosen`, whichever Trunk ran the Exchange; when none is marked, nothing is settled. The Trivial Trunk is the most stable Trunk, and every Build that fits the Root settles alike (RCT-D20). The alternative: the Build that ran the Exchange, which may be the very Build whose defect ended it, or one that was removed | Decided 15 October 2026, a technical choice: the proposed default |
| RTM-D6 | What does `stop` report, when the Root manager reads only the record and the folder goes soon after the end? | When it had the Exchange settled, the state settling printed. Otherwise the state of the end it read; `interrupted` when the record had no end and settling was refused; `saved` when the folder went before it could read the end, or settling printed `none`, which means the Exchange ended and its rows are saved, or it was never accepted; `none` when there was no folder. With what settling printed, when it had the Exchange settled. ARC-3 had sketched `ended` without `saved`. The alternative: the Root manager also reads the last line of `events.jsonl`, which ARC-0 leaves to the Arbol backend | Decided 15 October 2026, a technical choice: the proposed default |
| RTM-D7 | How does a test point the Root manager at another folder and another database? | With options, as ARC-1 says for a Root's own Builds: `--root-dir` for the Root's folder and `--bin-dir` for the folder of the launchers, the second required with the first. The Root manager opens no database, so it has no option for one; the Trunks it starts find theirs through their home folder (RCT-0), and a test gives it a stand-in `arbol-trunk` instead. The alternative: a home folder of the test's own, as RCT-D29 decided for Trunks, which ARC-D37 does not provide for a Root's own Builds | Decided 15 October 2026, a technical choice: the proposed default, which keeps ARC-D37 |
| RTM-D14 | How are command identity, option edges and Trunk replies made precise? | Common options before/after the command, no duplicates; own adjacent Build record with strict component/number/Root/version checks, never caller environment identity; missing/damaged/nonregular Root record is unknown_root; first UTF-8 JSON-object line forwarded byte for byte and child stderr preserved, success only with no error and status 0. No settings reads or current-Root fallback. The alternative is permissive last-value options and trusting caller Build variables, which can silently select another identity | Decided 26 October 2026, technical choice for SYL-181: the proposed default |
