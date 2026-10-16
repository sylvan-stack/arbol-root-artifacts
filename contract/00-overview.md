---
role: authored
spec: RCT-0
status: accepted
---
# Root Contract · 00 Overview

The *Root Contract* is what every [Arbol Trunk](../../arbol/GLOSSARY.md#arbol-trunk) of a [Root](../../arbol/GLOSSARY.md#arbol-root) has to do, and what the Root gives every Trunk, so that the Root manager `arbol-root` can start any Trunk and list, stop and settle any [Chat Exchange](../../arbol/GLOSSARY.md#chat-exchange), and the Arbol backend can follow and show any Exchange, without knowing which Trunk ran it. For example, `arbol-trunk run "What does build.sh do?"` starts the Trivial Trunk in the current Root: it makes a folder for the Exchange, writes the message to the Root's database, and the answer arrives first in that folder and then in the database, in a form that is the same whichever Trunk ran it. This specification states who takes part, how Trunks are named and kept, how versions of the contract make Trunk Builds and Roots fit together, the command `arbol-trunk` and how it picks a Build, what is handed to a Trunk, and how a Trunk finds its Root's folder, its database and its Drivers. It matters because in version 1 the boundary between the Root and its Trunks grew into a product of its own: an assignment handshake in ten steps, an old and a new runtime generation side by side, and a register of 21 contract questions, each with its own accepted record.

It follows the conventions of the [Driver Contract](../../arbol/driver-contract/README.md#conventions) and the additions in this chapter's [README](README.md#conventions). The owner asked for it on 15 October 2026 (SYL-168) and decided its points the same day ([RCT-D1](#decisions) to RCT-D6). The owner answered the open questions on 15 October 2026, among them that no setting of a Root narrows which Builds run (RCT-D28), and the specification was accepted the same day. It is the contract that the architecture called the *Trunk Contract* until 15 October 2026 ([ARC-0](../../arbol/architecture/00-components-and-boundaries.md#what-any-trunk-must-offer)); the formats that [ARC-2](../../arbol/architecture/02-chat-exchange-across-components.md) and [ARC-3](../../arbol/architecture/03-stuck-trunks.md) leave to it are fixed in [RCT-1](01-the-exchange-folder.md) to [RCT-3](03-the-chat-tables.md).

## Scope

Covered: the parties, Trunks and their code names, versions and what fits together, the command `arbol-trunk` and how it picks a Build, what is handed to a Trunk, how a Trunk finds its Root's folder and database, where a Trunk keeps its own settings, how a Trunk starts a Driver.

Not covered: the folder of an Exchange ([RCT-1](01-the-exchange-folder.md)); accepting, stopping, ending and settling ([RCT-2](02-starting-stopping-and-settling.md)); the chat tables ([RCT-3](03-the-chat-tables.md)); the conformance suite ([RCT-4](04-conformance.md)); when `arbol-root` lists, stops and settles, and what it prints, which belong to the [Root manager's specifications](../specs/README.md); what a Trunk does inside an Exchange, such as the history it sends the agent and whether it makes titles, which belongs to each Trunk's own specification, for the first Trunk the [Trivial Trunk's](../../arbol-trunk/trunks/trivial/specs/README.md); the Driver's run ([DRV-2](../../arbol/driver-contract/02-inference.md)); Builds and the registry ([BLD-0](../../arbol/builds/00-builds-and-versions.md)).

## Behaviour

### The parties

| | |
|---|---|
| A Root | A named environment such as First Sapling, with the identifier `sapling` ([ARC-1](../../arbol/architecture/01-roots.md)). Its folder, `~/Library/Application Support/Arbol/roots/sapling/`, holds its record `root.json` and the folder `exchanges/`. Its database is `arbol_sapling`. Its record names the one version of this contract it uses |
| A Trunk | A program that runs one Chat Exchange in one process, from the accepted message to its end, and writes the chat tables. Each has a *code name*, such as `trivial` |
| The Root manager | `arbol-root`, whose Builds are made for one Root. It starts Trunks, lists the Exchanges in progress from their folders, stops them, and has a Trunk Build settle an Exchange whose process is gone. It never opens the database (ARC-0). Its Builds carry the `layout.sql` of the Root's contract version ([RCT-3](03-the-chat-tables.md)), beside the Root's own layout for Arbol's tables, as `layout/chat-tables.sql` and `layout/root-tables.sql` ([RTM-1](../specs/01-builds-layouts-and-migration.md#what-a-build-carries)) |
| The Arbol backend | The process of a Root's `arbol` Build that serves the [Leather Bag's](../../arbol/GLOSSARY.md#leather-bag) windows. It reads the chat tables with SQL, follows the answer that is still arriving in the folder of an Exchange, and writes the Leather Bag's answers to the agent's questions there (ARC-0) |

Who writes and who reads what this contract fixes:

| What | Where | Written by | Read by |
|---|---|---|---|
| The Root's record | `root.json` in the Root's folder | `arbol-roots` (ARC-1), which sets the Root's contract version when it creates the Root | The launcher `arbol-trunk`, every Trunk, the Root manager |
| An Exchange in progress: its record and the arriving answer | `record.json` and `events.jsonl` in the Exchange's folder ([RCT-1](01-the-exchange-folder.md)) | The Trunk | The Root manager (the record only), the Arbol backend |
| Stop, and answers to the agent's questions | `control` in the Exchange's folder | The Root manager (stop), the Arbol backend (answers) | The Trunk |
| The chat tables | The Root's database ([RCT-3](03-the-chat-tables.md)) | The Trunks; the title also by Arbol | The Arbol backend, the Trunks |
| The table layout of the chat tables | `contract/<version>/layout.sql` in the `arbol-root` repository | By hand, in the repository | `arbol-roots`, which runs it from `layout/chat-tables.sql` of the Root's `arbol-root` Build (RTM-1) |
| A Trunk's own settings, if it has any | `~/.arbol/arbol-trunk/<code name>.json` ([below](#a-trunks-own-settings)) | The owner | That Trunk |

### Trunks and their code names

A Trunk is named by a *code name*, never by a number ([RCT-D2](#decisions)). A code name is one lower-case word of 1 to 32 letters and digits that starts with a letter, such as `trivial` or `advanced`, and it is never one of the command words of this contract, `run`, `settle` and `list`.

The first Trunk is the *Trivial Trunk*, `trivial`. It makes one Driver run with the parameters it is given, and has little logic. It is the most stable of all Trunks, there so that Arbol keeps working when every other Trunk fails. It works in every Root and is the default Trunk of every Root: a command that names no Trunk gets it. A Root is therefore marked working only while a working Build of `trivial` fits it ([below](#versions-what-fits-together)).

All Trunks are kept in one repository, `arbol-trunk`, one folder for each ([RCT-D4](#decisions)). Each folder is a component of its own for `arbol-builds`, named `arbol-trunk/<code name>`, with its own build description and its own count of build numbers (BLD-0). A Build is named like `arbol-trunk/trivial@4`, and its record lies in the registry under `builds/arbol-trunk/<code name>/<build>/`. One launcher, `arbol-trunk`, serves every Trunk. No code is shared between Trunks.

```text
~/repos/sylvan-stack/arbol-trunk/        the repository of every Trunk
  trunks/
    trivial/                             the Trivial Trunk, the component arbol-trunk/trivial
      arbol-build.json                   its build description, with "rootContracts": [1]
    advanced/                            another Trunk, whose Builds are numbered on their own
~/Library/Application Support/Arbol/builds/
  arbol-trunk/                           one folder for each Trunk, not a component of its own
    trivial/
      3/   build.json                    the Build arbol-trunk/trivial@3
      4/   build.json                    the Build arbol-trunk/trivial@4
~/Artifacts/arbol-trunk/                 the Trunks' Artifact Corpus
  trunks/trivial/specs/                  the Trivial Trunk's own specifications
  shared/specs/                          specifications that several Trunks share
```

### Versions: what fits together

The version of this contract is a whole number, and the first one is 1. It rises when a command, an option, a file, a field, a kind of line, a table or a column is removed or changes its meaning. Contract version 1 has no additions; how a table or a column is added within a version, and whether that raises the version, is decided with the first addition ([RCT-3](03-the-chat-tables.md#additions), RCT-D24). Only this contract has version numbers; a Trunk has a code name, and its Builds get their numbers from `arbol-builds` ([RCT-D3](#decisions)).

| Who | Says | Where | Example |
|---|---|---|---|
| A Root | The one version it uses, for as long as it exists. Several Roots may use the same one | `rootContract` in `root.json` | `"rootContract": 1` |
| A Trunk Build | Every version it implements | `rootContracts` in the Trunk's `arbol-build.json`, copied into the Build's record `build.json` | `"rootContracts": [1]` |
| A Build of `arbol-root` | Nothing of its own: it is made for one Root and carries the `layout.sql` of that Root's version | | |

A Trunk Build *fits* a Root when its list holds the Root's version. A Root runs the newest working Build of a Trunk that fits. Roots that use the same version share every Build that fits one of them, so a new Root that keeps the chat tables needs no Trunk built again (ARC-D40). A Root whose Trunks need another version is a new Root ([ARC-1](../../arbol/architecture/01-roots.md)).

`arbol-roots create <identifier> --root-contract <version>` writes a new Root's version into its `root.json`; without the option it takes the version of the Root named with `--from`, and 1 for a Root that grows from none. No command changes it afterwards. `arbol-roots ready` marks a Root working only while a working Build of `trivial` fits it, so the default Trunk works in every working Root. Pruning keeps, for every version a retained Root uses, the newest working Build of each Trunk that fits it (BLD-0).

For example, First Sapling and a second Root made from it use version 1, and a later Root uses version 2:

| Build of `trivial` | `rootContracts` | First Sapling and the second Root (version 1) | The later Root (version 2) |
|---|---|---|---|
| 3 | `[1]` | fits | — |
| 4 | `[1, 2]` | fits, and runs: the newest that fits | fits |
| 5 | `[2]` | — | fits, and runs |

The later Root could be marked working only once Build 4 or 5 had passed its checks.

### The command `arbol-trunk`

One command starts every Trunk ([RCT-D5](#decisions)):

```sh
arbol-trunk run "What does build.sh do?"                  # the Trivial Trunk: its newest working Build that fits the current Root
arbol-trunk advanced run "What does build.sh do?"         # the newest working Build of the Trunk advanced that fits
arbol-trunk advanced -v 8f run "What does build.sh do?"   # Build 8f of advanced, when it fits
arbol-trunk --root sapling run "What does build.sh do?"    # in First Sapling rather than the current Root
arbol-trunk list                                          # every Trunk and its Builds, for the current Root
```

Its form is `arbol-trunk [--root <root>] [<code name>] [-v <build>] <command> [<options>] [<arguments>]`:

| Part | Meaning | Without it |
|---|---|---|
| `--root <root>` | The Root, by its identifier | The current Root: the newest Root in the state `working` (ARC-1) |
| `<code name>` | The Trunk | `trivial` |
| `-v <build>` | That Build of the Trunk | The newest working Build of the Trunk that fits the Root |
| `<command>` | `run` or `settle` ([RCT-2](02-starting-stopping-and-settling.md)), passed to the Build; or `list`, which the launcher answers itself | Refused |

`arbol-trunk` is a launcher that `arbol-builds` writes, and it holds no Trunk code. It reads the Root's `root.json` and the Build records, picks the Build and starts its program in its own place, as every launcher does (BLD-0). It hands the Build `--root <root>` followed by the command and everything after it, unchanged, so a Build always knows its Root, also when the Root was not named. `--root` is recognised only before the code name, and a Build refuses a second one. No setting of the Root narrows which Builds may run: the Trunks a Root runs are those whose Builds list its Root Contract version, as `arbol-trunk list` and `arbol-root trunks` show them ([RCT-D28](#decisions)).

The launcher refuses a call, with the refusal of BLD-0 (one line `{ "error": { "code", "message" }, "available": [...] }` and exit status 1), for an unknown Root (`unknown_root`), an unknown Trunk (`unknown_trunk`), and a named Build that does not fit or a Trunk with no working Build that fits (`no_fit`); `available` lists the runnable Builds of that Trunk that fit. Build numbers of different Trunks are never compared (BLD-0). A Root identifier that is not one lower-case word of letters and digits, and a call that names no Root while no Root is working, are refused with `unknown_root` too, with a message that says which, and a call without a command with `invalid_input`; how the launcher reads a call and the Roots is in BLD-0 ([BLD-D17](../../arbol/builds/00-builds-and-versions.md#decisions)).

`arbol-trunk list` prints the current Root, or the one named with `--root`, with every Trunk and its Builds, and marks the Build that `run` would start. With a code name, only that Trunk:

```json
{
  "root": "sapling",
  "rootContract": 1,
  "trunks": [
    { "trunk": "trivial", "default": true,
      "builds": [ { "build": "4", "status": "working", "rootContracts": [1, 2], "fits": true, "runnable": true, "chosen": true },
                  { "build": "3", "status": "working", "rootContracts": [1], "fits": true, "runnable": true, "chosen": false } ] },
    { "trunk": "advanced", "default": false,
      "builds": [ { "build": "8f", "status": "working", "rootContracts": [2], "fits": false, "runnable": true, "chosen": false } ] }
  ]
}
```

`arbol-root` starts every Trunk with the same command, always naming the Root, the Trunk, the Build and the Exchange: `arbol-trunk --root sapling trivial -v 4 run --detach --exchange <id> --request <file>` ([RCT-2](02-starting-stopping-and-settling.md#how-the-root-manager-uses-a-trunk)).

### What is handed to a Trunk

The command `run` hands a Trunk everything an Exchange needs. Short values are options; anything long goes in a *request file*, a JSON file the caller writes and names with `--request`, never in an argument, as with a Driver (DRV-0). An option given beside the request file replaces that field of the file; `--driver`, `--subscription`, `--model` and `--effort` each replace only their own field of `route`. The Trunk reads the file once, before it answers, and the caller removes it after the answer.

| What | Option | Field of the request file | When it is not handed |
|---|---|---|---|
| The Root | `--root sapling`, which the launcher always adds | — | The Build refuses with `invalid_input` |
| The route: the Driver, the Subscription, the model and the effort ([ARC-0](../../arbol/architecture/00-components-and-boundaries.md#brain-recipes-and-the-resolved-route)) | `--driver claude`, `--subscription anthropic`, `--model sonnet`, `--effort high` | `route` | The Trunk's own specification says what it uses, such as a setting of its own ([below](#a-trunks-own-settings)), or that it refuses the start with `invalid_input`. Arbol always hands the Driver, the Subscription and the model (ARC-0) |
| The small-call route, for small calls such as a title (ARC-0) | — | `smallCallRoute` | The Trunk makes no small text call |
| The working folder: where the agent works | `--cwd /Users/sam/repos/shop` | `workingFolder` | The folder the Trunk was started from ([RCT-D6](#decisions)) |
| The chat | `--chat <identifier>` | `chat` | A new chat |
| The Exchange | `--exchange <identifier>` | `exchange` | The Trunk makes an identifier |
| The Leather Bag's message | The last argument | `message` | Refused, except when the message is sent again |
| Sending again ([RCT-2](02-starting-stopping-and-settling.md#sending-a-message-again)) | `--replaces <identifier>` | `replaces` | — |
| Running in the background ([RCT-2](02-starting-stopping-and-settling.md#in-the-background-or-in-the-foreground)) | `--detach` | — | The Trunk stays in the foreground |

The identifiers of a chat and of an Exchange are UUIDs, written in lower case with hyphens, such as `7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b`. The caller may choose them; otherwise the Trunk makes them ([RCT-D8](#decisions)). A chat identifier that the Root's database does not know starts a new chat with that identifier.

A request file that Arbol writes for the second message of a chat, which `arbol-root start` hands on without opening it ([RTM-0](../specs/00-the-commands.md#starting-a-trunk)):

```json
{
  "chat": "c3d4e5f6-0718-4293-a4b5-c6d7e8f90a1b",
  "message": "Add a --clean option to it.",
  "route": { "driver": "claude", "subscription": "anthropic", "model": "sonnet", "effort": "high" },
  "smallCallRoute": { "driver": "claude", "subscription": "anthropic", "model": "haiku" },
  "workingFolder": "/Users/sam/repos/shop"
}
```

The working folder is an absolute path to a folder that exists; any other is refused with `invalid_input`. By hand, the folder of the shell is used:

```sh
cd ~/repos/shop && arbol-trunk run "What does build.sh do?"    # the agent works in ~/repos/shop
```

Observed in version 1: a chat kept an ordered list of folders, chosen when it was created, and each accepted message recorded the folders it was sent with. A chat with no folder could not run a Claude turn at all: it failed at once.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:product-spec/07-agent-context.md#Working Directory and additional directories`

### How a Trunk finds its Root's folder and database

1. The Root's folder is `~/Library/Application Support/Arbol/roots/<root>/`, for a Root identifier that is one lower-case word of letters and digits (ARC-1); any other, such as `../sapling`, is refused with `invalid_input`. The Trunk reads `root.json` there: `database` names the Root's database and `rootContract` the version. A Root without a readable `root.json` is refused with `unknown_root`, and one whose version is not in the Build's list with `contract_mismatch`.
2. The address of the database server is `address` in `~/.arbol/database.json` (ARC-1): a PostgreSQL connection address whose user is the database role, such as `postgresql://arbol@127.0.0.1:5432`, or `postgresql://rct@%2Ftmp%2Frct-1a2b` for a server that listens only on a socket in the folder `/tmp/rct-1a2b`.
3. The password is the Keychain item with the service `com.sylvan-stack.arbol.database` and, as its account, the role that the address names, read with `security find-generic-password -s com.sylvan-stack.arbol.database -a arbol -w` and given at most 2 seconds. The exit status 44 of that command means that there is no such item, and the Trunk then connects without a password. Any other failure, or no answer within 2 seconds, is refused with `database_unavailable`, and the message names the Keychain.
4. It connects to that address and to the database that `root.json` names, which takes the place of any database the address names, with the client of [RCT-3](03-the-chat-tables.md#the-layout-is-one-file-of-sql).

Every file of these steps is reached from the home folder; only the Keychain is the Mac's own. A test therefore gives the Trunk a home folder of its own, and the Root record, the server and the database it finds are all the test's own ([RCT-4](04-conformance.md), [RCT-D7](#decisions)).

### A Trunk's own settings

A Trunk that has settings of its own keeps them in one file, `~/.arbol/arbol-trunk/<code name>.json`, such as `~/.arbol/arbol-trunk/trivial.json`, as every Arbol component keeps its settings in `~/.arbol` ([DRV-0](../../arbol/driver-contract/00-overview-and-boundary.md#the-settings-file)). The owner writes it, and only that Trunk reads it. It lies outside every Root, so the Trunk's Builds read the same file in every Root, and it is reached from the home folder, so a test's home folder of its own keeps it apart ([RCT-4](04-conformance.md)). What the file holds, and what the Trunk does without it, is each Trunk's own specification; the Trivial Trunk's one setting is the Driver it uses when it is handed none ([TRV-D7](../../arbol-trunk/trunks/trivial/specs/00-the-trivial-trunk.md#decisions)).

### How a Trunk starts a Driver

A Trunk starts a Driver run through the Driver's launcher, `arbol-claude-driver` or `arbol-codex-driver`, found on the search path, as `run` with `--control` and a request file (DRV-2), and `arbol-classifier` through its launcher in the same way. It may name a Driver Build with `-v`. `arbol-root` starts a Trunk with the folder of the launchers, `~/.local/bin` unless `arbol-root` is given another with `--bin-dir` (RTM-0), first on the search path, so that a Trunk started from the Arbol backend finds the same Drivers as one started from a shell, and a test can put a stand-in there instead ([RCT-4](04-conformance.md)). For example:

```sh
arbol-claude-driver run --control --request /…/exchanges/7f3a2c1e-…/run-1.json
```

## Requirements

A Trunk:

| ID | A Trunk must… | Origin | Verified by |
|---|---|---|---|
| RCT-0.1 | list in the `rootContracts` of its build description every version of this contract it implements, and no other | New | unit |
| RCT-0.2 | have a code name of one lower-case word of 1 to 32 letters and digits that starts with a letter and is no command word of this contract | New | unit |
| RCT-0.12 | be kept in the folder `trunks/<code name>/` of the `arbol-trunk` repository as the component `arbol-trunk/<code name>`, and share no code with another Trunk | New | unit |
| RCT-0.3 | take its Root only from `--root`, refuse a Root identifier that is not one lower-case word of letters and digits with `invalid_input` and a Root without a readable `root.json` with `unknown_root`, and refuse a Root whose `rootContract` is not in its list with `contract_mismatch` | New | contract |
| RCT-0.4 | take the route, the small-call route, the working folder, the chat, the Exchange and the message from the options and the request file of `run`, an option replacing only its own field of the file, and read the request file once, before it answers | New | contract: R1 (SYL-171); R5 comes with SYL-173 |
| RCT-0.5 | let the agent work in the folder named with `--cwd`, or else in the folder it was started from, and refuse a folder that is not absolute or does not exist with `invalid_input` | New | contract: R3 and R5, which come with SYL-172 and SYL-173; so far the Trivial Trunk's unit tests in `trunks/trivial/tests/` of `arbol-trunk`: `RequestFile.test_the_working_folder_is_the_cwd_option_else_the_folder_it_was_started_from`, `RequestFile.test_refusals_are_invalid_input` (SYL-171) |
| RCT-0.6 | find its Root's database only through `root.json`, `~/.arbol/database.json` and the Keychain item `com.sylvan-stack.arbol.database`, every file reached from the home folder, connect to the address with the database that `root.json` names, and connect without a password when the Keychain has no such item | New | contract: R1 (SYL-171); also the Trivial Trunk's unit tests in `trunks/trivial/tests/` of `arbol-trunk`: `TheRoot`, `TheDatabase` and `Address` (SYL-171) |
| RCT-0.16 | keep its own settings, if it has any, only in the file `~/.arbol/arbol-trunk/<code name>.json`, reached from the home folder | New | unit: the Trivial Trunk's unit tests in `trunks/trivial/tests/` of `arbol-trunk`: `Start.test_a_start_reads_no_settings_file_but_its_own`, `WholeExchange.test_a_whole_start_reads_no_settings_file_but_its_own` (SYL-171) |
| RCT-0.15 | refuse with `database_unavailable`, naming the Keychain, when reading the Keychain item fails otherwise than with no such item or gives no answer within 2 seconds | New | unit |
| RCT-0.14 | start a Driver only through its launcher found on the search path, as `run` with `--control` and a request file, and `arbol-classifier` only through its launcher found on the search path | New | contract: R1 (SYL-171); also the Trivial Trunk's unit tests in `trunks/trivial/tests/` of `arbol-trunk`: `Launcher`, `Run.test_it_starts_run_control_request_without_v_and_reads_every_event_line` (SYL-171) |

The launcher `arbol-trunk`:

| ID | The launcher must… | Origin | Verified by |
|---|---|---|---|
| RCT-0.7 | take `[--root <root>] [<code name>] [-v <build>]` before the command, take the current Root when no Root and `trivial` when no Trunk is named, and start the newest working Build of that Trunk whose `rootContracts` hold the Root's `rootContract`, or the named Build when it fits (RCT-D28) | New | unit: `tests/test_builds.py` of `arbol`: `builds.trunk-launch`, `builds.trunk-selection`, `builds.trunk-named-build`, `builds.trunk-first-build` (SYL-179) |
| RCT-0.8 | hand the Build `--root <root>` followed by the command and every later argument unchanged | New | unit: `tests/test_builds.py` of `arbol`: `builds.trunk-launch`, `builds.trunk-selection`, `builds.trunk-pass-through` (SYL-179) |
| RCT-0.9 | refuse an unknown Root, an unknown Trunk and a Build that does not fit with the refusal of BLD-0, listing in `available` the runnable Builds that fit, and never compare the build numbers of different Trunks | New | unit: `tests/test_builds.py` of `arbol`: `builds.trunk-refusals`, `builds.trunk-no-fit`, `builds.trunk-named-build` (SYL-179) |
| RCT-0.10 | answer `arbol-trunk list` from `root.json` and the Build records alone, with every Trunk, its Builds, their `rootContracts` and whether each is runnable, and the Build that `run` would start | New | unit: `tests/test_builds.py` of `arbol`: `builds.trunk-list`, `builds.trunk-list-chosen` (SYL-179) |

`arbol-roots`:

| ID | `arbol-roots` must… | Origin | Verified by |
|---|---|---|---|
| RCT-0.11 | write into a new Root's `root.json`, as `rootContract`, the version named with `--root-contract`, by default the version of the Root named with `--from` and else 1, and never change it afterwards | New | unit |
| RCT-0.13 | mark a Root `working` only while a working Build of `trivial` lists the Root's `rootContract` | New | unit |

## Interface

```ts
// arbol-trunk [--root <root>] [<code name>] [-v <build>] run [--request <file>] [--exchange <id>] [--chat <id>]
//             [--replaces <id>] [--cwd <folder>] [--driver <d>] [--subscription <s>] [--model <m>] [--effort <e>]
//             [--detach] ["<message>"]
interface StartRequest {               // the request file; an option given beside it replaces its field
  exchange?: string;                   // a UUID the caller chose, "7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b"; absent: the Trunk makes one
  chat?: string;                       // a UUID; one the database does not know starts a new chat with it; absent: a new chat
  replaces?: string;                   // send again: the Exchange whose answer is deleted and whose message runs again (RCT-2)
  message?: string;                    // the Leather Bag's message; absent exactly when replaces is given
  route?: Partial<Route>;              // ARC-0; Arbol always hands driver, subscription and model
  smallCallRoute?: Route;              // ARC-0; absent: the Trunk makes no small text call
  workingFolder?: string;              // absolute; absent: the folder the Trunk was started from
}

interface Route {                      // as in ARC-0
  driver: string;                      // "claude" or "codex": the Trunk starts arbol-claude-driver or arbol-codex-driver
  subscription: string;                // the Subscription's identifier (DRV-1), "anthropic"
  model: string;                       // "sonnet"
  effort?: string;                     // "high"
}

// arbol-trunk [--root <root>] [<code name>] list
interface TrunkList {
  root: string;                        // the Root's identifier, "sapling"
  rootContract: number;                // the version of this contract the Root uses, 1
  trunks: Array<{
    trunk: string;                     // the code name, "trivial"
    default: boolean;                  // true for trivial, the default Trunk of every Root
    builds: Array<{
      build: string;                   // the build number, "4"
      status: "building" | "failed" | "working" | "withdrawn";   // as in BLD-0
      rootContracts: number[];         // [1, 2]
      fits: boolean;                   // the list holds the Root's version
      runnable: boolean;               // its program is in place: the launcher starts it when it is named and fits (BLD-0)
      chosen: boolean;                 // the Build that run would start
    }>;
  }>;
}

// a refusal of the launcher, as in BLD-0; a refusal of a Trunk Build has no "available"
interface Refusal {
  error: { code: string; message: string };   // "unknown_root", "unknown_trunk", "no_fit", "invalid_input" (BLD-D17)
  available?: string[];                // the runnable Builds of the Trunk that fit, ["4", "3"]
}
```

The fields this contract adds to the records of [ARC-1](../../arbol/architecture/01-roots.md#interface) and [BLD-0](../../arbol/builds/00-builds-and-versions.md#interface):

```ts
// arbol-roots create <identifier> --name <display name> [--from <root>] [--root-contract <version>]
interface RootRecord {                 // root.json
  // …
  rootContract: number;                // the one version of the Root Contract this Root uses, 1; never changes
}

interface BuildDescription {           // trunks/<code name>/arbol-build.json in the arbol-trunk repository
  component: string;                   // "arbol-trunk/trivial"
  // …
  rootContracts?: number[];            // required for a Trunk: the versions of the Root Contract it implements, [1]
}

interface BuildRecord {                // builds/arbol-trunk/<code name>/<build>/build.json
  // …
  rootContracts?: number[];            // a Trunk: copied from its build description
}
```

## Known problems in version 1

Not to be reproduced.

- **A boundary that became a product.** Handing an Exchange from the Root to a Trunk took an assignment handshake in ten steps, with runtime generations and durable ownership records.
- **A register of contract questions.** The stable contract between them had a register of 21 design questions, each with an accepted record, before a second Trunk existed.
- **No folder, no answer.** A chat without a folder failed at once.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:grafts/arbol-trunks/component-model.md#Exchange assignment and durable ownership`
- `arbol_deprecated_v1:grafts/arbol-trunks/decisions.md#Stable-contract design register`
- `arbol_deprecated_v1:product-spec/12-lessons-and-invariants.md#Anti-goals`

## Decisions

| ID | Question | Proposed default | Outcome |
|---|---|---|---|
| RCT-D1 | What is the contract that a Root shares with all its Trunks called, and where are the specifications of Trunks kept? | The Root Contract, kept with the Root in the `arbol-root` corpus, `contract/`. A Trunk's own specifications go in `arbol-trunk/trunks/<code name>/specs/`, and specifications that several Trunks share in `arbol-trunk/shared/specs/` | Decided 15 October 2026 by the owner: "root has to share same contract with all its child trunks". It replaces the name Trunk Contract that ARC-0, ARC-2, ARC-3, BLD-0 and the Glossary used until 15 October 2026 |
| RCT-D2 | How are Trunks named, and which Trunk runs when none is named? | By a code name, never by a number. The first Trunk is the Trivial Trunk, `trivial`: one Driver run with the parameters it is given, and little logic. It is the most stable of all Trunks, so that Arbol keeps working when every other Trunk fails, it must work in every Root, and it is the default Trunk of every Root | Decided 15 October 2026 by the owner: the proposed default. The form of a code name, one lower-case word of 1 to 32 letters and digits that starts with a letter, is a technical choice made the same day. That `arbol-roots ready` marks a Root working only while a working Build of `trivial` fits it is a technical consequence drawn the same day |
| RCT-D3 | What has a version number: the Root Contract, a Trunk, or both? | Only the Root Contract. A Trunk has a code name, and its Builds get numbers from `arbol-builds`. Each Trunk Build lists the Root Contract versions it implements; a Root uses exactly one version, and several Roots may use the same one; a Root runs the newest working Build of a Trunk whose list holds the Root's version | Decided 15 October 2026 by the owner: the proposed default. It replaces the Trunk Build fields `contract` and `chatTablesOf` of BLD-0 and ARC-1. That a Root's record then names its version, set by `arbol-roots create`, in place of the field `chatTablesOf` whose form ARC-D40 left open, and the names `rootContracts` and `rootContract`, are a technical consequence drawn the same day |
| RCT-D4 | Where is the code of the Trunks kept? | In one repository, `arbol-trunk`, one folder for each Trunk, `trunks/<code name>/`. Each Trunk's Builds are numbered on their own, and no code is shared between Trunks | Decided 15 October 2026 by the owner: the proposed default. That the component of a Trunk is named `arbol-trunk/<code name>`, that its Builds lie in the registry under `builds/arbol-trunk/<code name>/`, and that one launcher serves every Trunk, are a technical choice made the same day |
| RCT-D5 | How is a Trunk started? | One command for every Trunk: `arbol-trunk run "prompt"` runs the default Trunk's newest working Build, `arbol-trunk advanced run "prompt"` the Trunk `advanced`, and `arbol-trunk advanced -v 8f run "prompt"` its Build 8f. A code name is never a command word. `arbol-trunk list` shows every Trunk and its Builds. `arbol-trunk` is a launcher written by `arbol-builds` and holds no Trunk code. `arbol-root` starts Trunks with the same command, always with an exact `-v`. `--root <name>` chooses the Root, otherwise the current Root | Decided 15 October 2026 by the owner: the proposed default. That `--root` comes first, that the launcher hands the Build `--root <root>` before the command, and that the commands of contract version 1 are `run`, `settle` and `list`, are a technical choice made the same day |
| RCT-D6 | In which folder does the agent work? | In the folder the Trunk is given as a parameter, or else in the folder it was started from | Decided 15 October 2026 by the owner: the proposed default. The option `--cwd`, named as a Driver names it (DRV-2), is a technical choice made the same day |
| RCT-D7 | How does a Trunk find its Root's folder, its database and its Drivers? | From `--root` alone: `root.json` names the database and the version, `~/.arbol/database.json` the server, and the Keychain the password, with none when there is no item. Drivers and `arbol-classifier` by their launchers on the search path, which `arbol-root` sets. Every file is reached from the home folder, so a test can give the Trunk a home of its own ([RCT-D29](04-conformance.md#decisions)). The alternative: options that name the Root's folder, the settings file and each program, which every Trunk would have to take | Decided 15 October 2026, a technical choice: the proposed default |
| RCT-D8 | Who chooses the identifiers of a chat and of an Exchange, and what are they? | The caller may choose both, and otherwise the Trunk makes them. Both are UUIDs. A caller that chose the Exchange's identifier can start again after it lost the answer, without making a second Exchange ([RCT-2](02-starting-stopping-and-settling.md#a-repeat-after-a-lost-answer)). The alternative: the Trunk always makes them, and a caller that lost the answer cannot tell whether its message was accepted | Decided 15 October 2026, a technical choice: the proposed default |
| RCT-D28 | ARC-0 let a Root's settings narrow which Builds of a Trunk may run, until 15 October 2026. Should that come back, now that a Root runs the newest working Build whose list holds its version? | No. A Build that must not run is withdrawn with `arbol-builds withdraw`, which keeps it out of every Root. If yes: the Root manager's settings list the allowed Builds, and `arbol-trunk` used by hand ignores that list | Decided 15 October 2026 by the owner: "Roots shouldn't limit anything explicitly. They only have list of compatible trunks". No setting narrows which Builds run: the Trunks a Root runs are those whose Builds list its Root Contract version, as `arbol-root trunks` shows them. `arbol-root` reads no settings file, and `arbol-root.json` is not made (RTM-D3) |
