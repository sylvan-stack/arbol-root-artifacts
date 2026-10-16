---
role: authored
spec: RCT-1
status: accepted
---
# Root Contract · 01 The Exchange folder

The *Exchange folder* is the folder a Trunk keeps for one [Chat Exchange](../../arbol/GLOSSARY.md#chat-exchange) while it runs, such as `~/Library/Application Support/Arbol/roots/sapling/exchanges/7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b/`. It is the whole interface between a running Trunk and everything else ([ARC-2](../../arbol/architecture/02-chat-exchange-across-components.md)): the Root manager reads the record in it to list what runs, the Arbol backend follows the answer in it as it arrives, and a stop or an answer to the agent's question reaches the Trunk through it. This specification states where the folder is and how it comes and goes, the record `record.json` with the Trunk's signs of life, how a Trunk watches itself, the file `events.jsonl` that holds the answer as it arrives and its size limits, the file `control`, and what a reader can tell from what it finds. It matters because version 1 held the answer still arriving and the running usage only in process memory, which a crash lost, and gave no sign of life at all, so that a silent turn looked exactly like a busy one.

It follows the conventions of the [Driver Contract](../../arbol/driver-contract/README.md#conventions) and the additions in this chapter's [README](README.md#conventions). It fixes the formats that ARC-2 and [ARC-3](../../arbol/architecture/03-stuck-trunks.md) leave to this contract. Its forms are technical choices made on 15 October 2026 ([RCT-D9](#decisions) to RCT-D13). The owner answered the chapter's open questions on 15 October 2026, and the specification was accepted the same day.

## Scope

Covered: where the folder is, how it comes and goes, the record, the signs of life, the Trunk's watch on itself, the arriving answer and its size limits, stop and answers, what a folder tells a reader.

Not covered: what the record means for acceptance, a stop, the end and settling ([RCT-2](02-starting-stopping-and-settling.md)); how lines of the arriving answer become rows ([RCT-3](03-the-chat-tables.md#how-lines-become-rows)); how often Arbol looks from outside and when it ends an Exchange by force (ARC-3, [RTM-0](../specs/00-the-commands.md)); the Driver's own events ([DRV-2](../../arbol/driver-contract/02-inference.md)).

## Behaviour

### Where the folder is

```text
~/Library/Application Support/Arbol/roots/sapling/exchanges/
  7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b/    one Exchange in progress; the name is its identifier
    record.json                            who runs it, its route and folder, its signs of life, its end
    events.jsonl                           the answer as it arrives, one JSON object per line
    control                                stop, and the Leather Bag's answers to the agent's questions
  .9b3a2c1e-5c6d-4f70-9a1b-2c3d4e5f6a7b.new/   a folder being made or removed: every reader skips a name that starts with a dot
```

The folder has the mode 0700 and its files 0600, because they hold the conversation. A Trunk may keep files of its own in it, such as the request file of its Driver run; no other program reads them, and they go with the folder.

### How the folder comes and goes

1. The Trunk makes the folder `.<exchange>.new` and writes into it `record.json` without `acceptedAt`, an empty `events.jsonl` and an empty `control`.
2. It renames the folder to `<exchange>`. When either step finds a folder of that name already there, another start of the same Exchange came first: the Trunk removes what it made and refuses with `in_progress` ([RCT-2](02-starting-stopping-and-settling.md#a-repeat-after-a-lost-answer)).
3. Once the database holds the Exchange, it adds `acceptedAt` to the record ([RCT-2](02-starting-stopping-and-settling.md#accepting-a-message)).
4. Once the database holds the Exchange's end, it renames the folder to `.<exchange>.old` and deletes it.

A folder named by an Exchange's identifier therefore always holds a whole record ([RCT-D12](#decisions)).

### The record

```json
{
  "rootContract": 1,
  "root": "sapling",
  "exchange": "7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b",
  "chat": "c3d4e5f6-0718-4293-a4b5-c6d7e8f90a1b",
  "trunk": "trivial",
  "build": "4",
  "process": { "pid": 48213, "startedAt": "2026-10-15T09:30:00.000Z" },
  "route": { "driver": "claude", "subscription": "anthropic", "model": "sonnet", "effort": "high" },
  "smallCallRoute": { "driver": "claude", "subscription": "anthropic", "model": "haiku" },
  "workingFolder": "/Users/sam/repos/shop",
  "acceptedAt": "2026-10-15T09:30:00.412Z",
  "signsOfLife": { "heartbeat": 41, "waitingFor": "tool", "since": "2026-10-15T09:31:02.000Z" }
}
```

| Rule | Detail |
|---|---|
| One writer | Only the Trunk writes the record |
| Replaced whole | The Trunk writes `record.json.new` in the folder and renames it over `record.json`, so a reader never sees half a record |
| Who runs it never changes | `root`, `exchange`, `chat`, `trunk`, `build`, `process`, `route`, `smallCallRoute` and `workingFolder` keep the values of the first record. A newer Build of the Root manager, or another Root, never takes the Exchange over (ARC-2) |
| The process | `pid` is the Trunk's process identifier, and `startedAt` the time the system gives as that process's start, as the command `ps -o lstart -p <pid>` prints it. A record's process lives when a process with that identifier exists and was started at that time, give or take two seconds, as a Driver tells the owner of a run ([DRV-6](../../arbol/driver-contract/06-state-and-simultaneous-calls.md)). The Trunk leads a process group with the same identifier, which holds the Trunk and the processes it starts itself ([RCT-2](02-starting-stopping-and-settling.md#one-process-group)) |
| The end | When the Exchange ends, the record gets `end`, the same object as the last line of `events.jsonl` (below) |

### Signs of life

The record carries the two facts of ARC-3 under `signsOfLife`, rewritten whenever one of them changes ([RCT-D11](#decisions)):

| Field | What it is | Example |
|---|---|---|
| `heartbeat` | A counter, 0 in the first record, that the Trunk raises by one from the loop that does its work, at least every 10 seconds, whatever it waits for. No timer raises it, so a Trunk whose work has hung stops raising it | `41` |
| `waitingFor` | What the Trunk waits for now (below) | `"tool"` |
| `since` | Since when, in UTC | `"2026-10-15T09:31:02.000Z"` |

| `waitingFor` | The Trunk waits for | Example |
|---|---|---|
| `answer` | The Leather Bag's answer to a question of the agent, between `ask_opened` and `ask_closed` (DRV-3) | "Which branch should I push to?" |
| `tool` | A tool call between its `tool_started` and its `tool_finished` | `shell` running `make` for 2 h 10 min |
| `driver` | A Driver run that goes on, with no question and no tool call open, or one that it is starting | the agent writes its answer |
| `small-call` | A small call (ARC-0), while no run of the agent goes on | a title, after the answer is complete |
| `database` | The database: at acceptance, to save what is finished, and at the end ([RCT-2](02-starting-stopping-and-settling.md#when-the-database-cannot-be-reached)) | the server is restarting |

When several are true, the Trunk gives the first of them in the order of this table. Its own steps count as the wait they serve: starting a run as `driver`, saving rows and ending as `database`. Every wait in the Trunk's work has a time limit of at most 5 seconds, after which the loop comes round and raises the heartbeat (ARC-3).

### The Trunk watches itself

A part of the Trunk's process that does nothing else, such as a thread of its own, looks at the heartbeat every second. When the work has not raised it for 60 seconds, that part ends the Trunk's process group as [RCT-2](02-starting-stopping-and-settling.md#one-process-group) says: SIGTERM first, so that each Driver in the group stops its run and ends the vendor tool and the agent's commands, and SIGKILL 5 seconds later to whatever is still alive. It writes nothing. The Exchange is then one whose process is gone, and settling saves it as interrupted ([RCT-2](02-starting-stopping-and-settling.md#settling)). This works while Arbol is not running ([ARC-3](../../arbol/architecture/03-stuck-trunks.md)).

For example, a defect makes the Trunk's work wait for ever on a lock while the agent runs `make`: 60 seconds after the last heartbeat the Driver gets SIGTERM, ends `claude` and the `make`, and exits; the Trunk is gone 5 seconds later, and `arbol-root list` shows the Exchange as interrupted.

The 60 seconds are counted with the clock that macOS calls `CLOCK_UPTIME_RAW`, which stands still while the Mac sleeps, so waking the Mac makes no Exchange look hung.

### The answer as it arrives: `events.jsonl`

The Trunk appends to `events.jsonl` one line for each event of the agent's Driver runs, exactly as the Driver wrote it ([DRV-2](../../arbol/driver-contract/02-inference.md)), under `event` and with a number `n` that counts from 1 without gaps; one line for each answer of the Leather Bag that it hands to a run; one line for the end of each small call; and, last, one line with the end ([RCT-D9](#decisions)). An Exchange with one run, in which the agent writes, reads a file and answers:

```json
{"n":1,"event":{"runId":"7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b.1","seq":1,"at":"2026-10-15T09:30:01.204Z","type":"started","continuity":"fresh","subscriptionId":"anthropic","model":"sonnet","effort":"high","toolVersion":"2.1.294","driverBuild":"b"}}
{"n":2,"event":{"runId":"7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b.1","seq":2,"at":"2026-10-15T09:30:03.020Z","type":"text_delta","text":"Let me "}}
{"n":3,"event":{"runId":"7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b.1","seq":3,"at":"2026-10-15T09:30:03.310Z","type":"text","text":"Let me read it."}}
{"n":4,"event":{"runId":"7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b.1","seq":4,"at":"2026-10-15T09:30:03.400Z","type":"tool_started","callId":"call_1","name":"shell","input":{"command":"cat build.sh"}}}
{"n":5,"event":{"runId":"7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b.1","seq":5,"at":"2026-10-15T09:30:03.900Z","type":"tool_finished","callId":"call_1","ok":true,"result":{"output":"#!/bin/sh\nset -e\n…","exitCode":0}}}
{"n":6,"event":{"runId":"7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b.1","seq":6,"at":"2026-10-15T09:30:09.700Z","type":"text","text":"It compiles the three packages and copies them to dist/."}}
{"n":7,"event":{"runId":"7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b.1","seq":7,"at":"2026-10-15T09:30:09.800Z","type":"finished","outcome":"completed","usage":{"model":"sonnet","inputTokens":5120,"outputTokens":812},"producedOutput":true,"unfinishedCalls":[]}}
{"n":8,"smallRun":{"runId":"7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b.t","seq":3,"at":"2026-10-15T09:30:10.900Z","type":"finished","outcome":"completed","usage":{"model":"haiku","inputTokens":310,"outputTokens":9},"producedOutput":true,"unfinishedCalls":[]}}
{"n":9,"end":{"state":"completed","at":"2026-10-15T09:30:11.050Z"}}
```

When the agent asks a question, the Leather Bag's answer is a line of its own, between the question's `ask_opened` and `ask_closed`:

```json
{"n":14,"answer":{"type":"answer","askId":"ask_1","answer":{"type":"answers","answers":{"branch":["main"]}}}}
```

| Rule | Detail |
|---|---|
| The whole answer, not only fragments | Finished blocks and tool calls are in the file too, and each line is in the file before the row it carries is in the database ([RCT-3](03-the-chat-tables.md#how-lines-become-rows)). So while the folder exists, the file alone holds the answer so far, and settling needs nothing but the folder |
| The Leather Bag's answers | Each `answer` line the Trunk hands to a run is appended as `{"n", "answer"}`, exactly as it was in `control`, so that the answer is kept with its question ([RCT-3](03-the-chat-tables.md#how-lines-become-rows)) |
| Of a small call, only its end | A small call, such as a title, is no part of the answer. Of its Driver run the Trunk appends only the `finished` event, as `{"n", "smallRun"}`, so that its usage is kept. A classification through `arbol-classifier` is no Driver run and writes nothing here |
| The end, last | `{"n": …, "end": {"state", "at", "error"?}}`, with the state `completed`, `failed` or `cancelled` and, for `failed`, the cause ([RCT-2](02-starting-stopping-and-settling.md#the-end)). Nothing follows it |
| One writer, whole lines | Only the Trunk writes the file. Each line is appended whole, with one write, and ends with a newline. A last line without a newline is unfinished, and every reader skips it |
| Readers ignore what they do not know | A line with none of `event`, `answer`, `smallRun` and `end`, an event type and a field that a reader does not know, as a Driver's caller does (DRV-0) |
| Read from the start | A window opened late, opened again or opened twice reads the same file from its first line, so it sees the whole answer so far (ARC-2) |

Two limits keep the file in bounds ([RCT-D10](#decisions)):

| Limit | What happens |
|---|---|
| 1 MiB of output for one tool call | Once the `tool_output_delta` lines of one call in the file pass 1 MiB, the Trunk writes no further `tool_output_delta` line for that call. Every other line is written as before, and the call's `tool_finished` brings its result, which the Driver cuts to 200 KiB (DRV-2). For example, an agent that runs `cat` on a log of 1 GiB, or leaves a development server printing for hours, puts about 1 MiB of that output into the file, and the Exchange goes on |
| 256 MiB for the whole file | When the next line would take the file past 256 MiB, the Trunk stops its run as for a stop line and ends the Exchange as failed with the cause `events_file_full`. From then on it writes only a `tool_finished` or `ask_closed` whose `tool_started` or `ask_opened` is in the file, `usage`, `finished`, `smallRun` and the end; these may take the file a little past the limit. What was finished before is kept |

Where the Trunk leaves lines out, the Driver's `seq` has gaps in the file; `n` never has.

Observed in version 1: the text not yet sealed, the running usage, rate-limit pauses and the Driver's copy of the last 300 seconds of events, kept for replay, lived only in process memory; a crash or a reboot lost them, and a window opened late missed the unsealed text. A queue of 256 notifications crashed the Codex Driver at the 257th, and a spool on disk had no limit.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:product-spec/03-turn-execution.md#Stored versus live-only, and how a window reconciles them`
- `arbol_deprecated_v1:product-spec/03-turn-execution.md#Survival and recovery`
- `arbol_deprecated_v1:product-spec/12-lessons-and-invariants.md#3. Durable bookkeeping on the streaming path`

### Stop and answers: `control`

A stop and the Leather Bag's answers reach the Trunk as lines of the Driver's own *control input*: the standard input that a Driver run keeps open for stop and answers (DRV-2, DRV-3). They are one JSON object per line ([RCT-D13](#decisions)):

```json
{"type":"stop"}
{"type":"answer","askId":"ask_1","answer":{"type":"answers","answers":{"branch":["main"]}}}
```

| Rule | Detail |
|---|---|
| Two writers | The Root manager writes `stop`; the Arbol backend writes `answer` (ARC-2) |
| Whole lines | A writer opens the file for appending and writes one line of at most 64 KiB, ending with a newline, with one write call, so two writers never mix their lines |
| Only the Trunk makes it | A writer never creates the file. A writer that finds no `<exchange>/control` but finds the folder `.<exchange>.new` waits until it is renamed into place, at most 15 seconds, and then appends its line. A writer that finds neither stops there: the Exchange has ended, and a stop of an ended Exchange succeeds and changes nothing (ARC-3) |
| Read every second | The Trunk reads the lines that are new at least once a second, and always before it starts a run, and hands each `stop` and `answer` line, unchanged, to the control input of the run it concerns; an answer goes to the run whose question it is. It never writes to the file |
| Nothing else | The Trunk ignores a line it does not know |

What a stop leads to is in [RCT-2](02-starting-stopping-and-settling.md#stop).

### What a folder tells a reader

A reader such as `arbol-root list` or the Arbol backend can tell the state of an Exchange from its folder and its record alone:

| In the folder | The record's process | The Exchange is… | What follows |
|---|---|---|---|
| A record without `acceptedAt` | lives | being accepted | The starter waits for the answer ([RCT-2](02-starting-stopping-and-settling.md#accepting-a-message)) |
| A record without `acceptedAt` | is gone | probably never accepted | Settling looks it up in the database, and removes the folder when it is not there |
| A record without `end` | lives | in progress; its signs of life say more | — |
| A record with `end` | lives | ending: its rows are being saved | The Trunk removes the folder |
| A record with `end` | is gone | ended, and not yet saved, because the database was away | Settling saves it with that state; until then a window shows the answer from the file, marked as not yet saved (ARC-D53) |
| A record without `end` | is gone | interrupted | Settling saves what the file holds, as interrupted |
| No folder | — | ended and saved, or never accepted | The chat tables say which |

## Requirements

A Trunk:

| ID | A Trunk must… | Origin | Verified by |
|---|---|---|---|
| RCT-1.1 | keep one folder for each Exchange at `exchanges/<exchange>/` in its Root's folder, with the mode 0700 and files of the mode 0600, make it under the name `.<exchange>.new` and rename it into place only when `record.json`, `events.jsonl` and `control` are in it, and remove it by renaming it to `.<exchange>.old` before deleting it | New | contract: R1 (SYL-171); also the Trivial Trunk's unit tests in `trunks/trivial/tests/` of `arbol-trunk`: `Folder` (SYL-171) |
| RCT-1.2 | write `record.json` in the form of `ExchangeRecord`, replace it only by renaming a whole file over it, and never change `root`, `exchange`, `chat`, `trunk`, `build`, `process`, `route`, `smallCallRoute` or `workingFolder` after the first record | New | contract: R1 (SYL-171); also the Trivial Trunk's unit tests in `trunks/trivial/tests/` of `arbol-trunk`: `Folder.test_the_record_is_replaced_whole_and_who_runs_it_never_changes` (SYL-171) |
| RCT-1.3 | raise `signsOfLife.heartbeat` from the loop that does its work at least every 10 seconds and rewrite the record each time, keep `waitingFor` and `since` true, giving the first that holds of `answer`, `tool`, `driver`, `small-call` and `database`, and give no wait in that loop more than 5 seconds | New | contract |
| RCT-1.4 | end its process group, with SIGTERM and 5 seconds later SIGKILL, and write nothing, when its work has not raised the heartbeat for 60 seconds, counted with `CLOCK_UPTIME_RAW` by a part of its process that does nothing else | New | contract, drill |
| RCT-1.5 | append every event of the agent's Driver runs to `events.jsonl` exactly as the Driver wrote it, except as RCT-1.6 and RCT-1.12 say, in a line `{"n", "event"}` with `n` counting from 1 without gaps, each line whole with one write, before it saves anything that comes from it, and the end as the last line | New | contract: R1 (SYL-171); also the Trivial Trunk's unit tests in `trunks/trivial/tests/` of `arbol-trunk`: `Events` (SYL-171) |
| RCT-1.11 | append each `answer` line it hands to a run as `{"n", "answer"}`, and the `finished` event of each small call's run as `{"n", "smallRun"}`, and nothing else of a small call | New | contract: R19 for the small calls (SYL-171); R9 for the answers comes with SYL-173 |
| RCT-1.6 | when the next line would take `events.jsonl` past 256 MiB, stop its run, end the Exchange as failed with `events_file_full`, and from then on write only closing lines whose start is in the file, `usage`, `finished`, `smallRun` and the end | New | contract |
| RCT-1.12 | write no further `tool_output_delta` line for a tool call once that call's `tool_output_delta` lines in `events.jsonl` have passed 1 MiB, and go on writing every other line | New | contract |
| RCT-1.7 | make `control` empty with the folder, never write to it, read the new lines at least once a second and before it starts each run, and hand each `stop` and `answer` line unchanged to the control input of the run it concerns | Changed | contract |

The Root manager and the Arbol backend, as writers of `control`:

| ID | A writer of `control` must… | Origin | Verified by |
|---|---|---|---|
| RCT-1.8 | append only whole lines of the Driver's control input, of at most 64 KiB, each with one write on a file opened for appending, never create the file, and wait up to 15 seconds while only `.<exchange>.new` exists | New | unit |

Every reader of an Exchange folder:

| ID | A reader must… | Origin | Verified by |
|---|---|---|---|
| RCT-1.9 | skip a folder whose name starts with a dot and a last line of `events.jsonl` that does not end with a newline, and ignore every field, line and event type it does not know | New | unit |
| RCT-1.10 | take a record's process as living only when a process with its `pid` exists and was started at its `startedAt`, give or take two seconds | New | unit |

## Interface

```ts
interface ExchangeRecord {             // record.json, written by the Trunk alone and replaced whole
  rootContract: number;                // the version of this contract the record is written in, 1
  root: string;                        // the Root's identifier, "sapling"
  exchange: string;                    // the Exchange's identifier, a UUID; also the folder's name
  chat: string;                        // the chat's identifier, a UUID
  trunk: string;                       // the Trunk's code name, "trivial"
  build: string;                       // the build number of the Trunk Build that accepted it, "4"; "0" outside a Build, as in development (BLD-0)
  process: { pid: number; startedAt: string };   // the Trunk's process; it leads a process group with the same identifier
  route: Partial<Route>;               // as it was handed (RCT-0)
  smallCallRoute?: Route;              // as it was handed; absent: none
  workingFolder: string;               // "/Users/sam/repos/shop"
  acceptedAt?: string;                 // absent until the database holds the Exchange
  signsOfLife: SignsOfLife;
  end?: End;                           // present once the Exchange has ended
}

interface SignsOfLife {                // ARC-3, made exact
  heartbeat: number;                   // 0 at first; raised by 1 at least every 10 seconds; only grows
  waitingFor: "answer" | "tool" | "driver" | "small-call" | "database";   // the first that holds, in this order
  since: string;                       // since when it waits for that, UTC
}

type EventsLine =                      // one line of events.jsonl
  | { n: number; event: RunEvent }     // an event of an agent's Driver run, exactly as the Driver wrote it (DRV-2)
  | { n: number; answer: ControlLine } // an answer line the Trunk handed to a run, exactly as it was in control
  | { n: number; smallRun: RunEvent }  // the finished event of a small call's run, exactly as the Driver wrote it
  | { n: number; end: End };           // the last line

interface End {
  state: "completed" | "failed" | "cancelled";
  at: string;                          // UTC
  error?: ExchangeError;               // present exactly when state is "failed" (RCT-2)
}

type ControlLine =                     // one line of control, as the Driver's control input takes it (DRV-2)
  | { type: "stop" }                                    // written by the Root manager
  | { type: "answer"; askId: string; answer: Answer };  // written by the Arbol backend; Answer as in DRV-3
```

Every time has the form that JavaScript's `toISOString` writes, such as `2026-10-15T09:30:00.000Z`, as in BLD-0.

## Known problems in version 1

Not to be reproduced.

- **Live state only in memory.** The text not yet sealed, the running usage and the Driver's copy of recent events for replay were lost on a crash or a reboot.
- **No sign of life.** There was no heartbeat and no time limit; a scan called a chat stalled after 15 minutes and did nothing about it.
- **Buffers without a sound limit.** A full queue crashed a Driver, and a spool on disk could grow without end.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:product-spec/03-turn-execution.md#Stored versus live-only, and how a window reconciles them`
- `arbol_deprecated_v1:product-spec/03-turn-execution.md#Failures and what the Leather Bag sees`
- `arbol_deprecated_v1:product-spec/12-lessons-and-invariants.md#3. Durable bookkeeping on the streaming path`

## Decisions

| ID | Question | Proposed default | Outcome |
|---|---|---|---|
| RCT-D9 | What does `events.jsonl` hold: only the fragments of the answer, or everything that arrives? | Every event of the agent's Driver runs, exactly as the Driver wrote it, numbered from 1, finished blocks and tool calls included; the Leather Bag's answers to the agent's questions; the end of each small call; and one last line with the end. Finished parts still go to the database as well (ARC-D42). A reader then needs only the file while the folder exists, and settling needs only the folder, also when the database was away. The alternative: fragments only, as ARC-2 worded it until 15 October 2026, with a second file for the finished parts the database could not take, so two places to read and to settle from | Decided 15 October 2026, a technical choice: the proposed default |
| RCT-D10 | How large may `events.jsonl` grow, and what happens at the limit? | 1 MiB of streamed output for each tool call, beyond which that call's output is left out of the file and the Exchange goes on; and 256 MiB for the whole file, at which the run is stopped and the Exchange fails with `events_file_full`, everything finished before being kept. The first keeps a command that prints without end, such as a development server, from filling the file, so nothing is ended for printing much (ARC-D61); the second is a safety net that an ordinary Exchange never reaches. The alternative: one limit of 256 MiB for everything, at which the output of one long command ends the Exchange | Decided 15 October 2026, a technical choice: the proposed default |
| RCT-D11 | Where are the signs of life kept? | In `record.json`, under `signsOfLife`, rewritten with each raise, because the Root manager reads only the record of an Exchange (ARC-0). The Trunk's own steps count as the wait they serve, so the five values of ARC-3 suffice. The alternative: a file of their own, which the Root manager would read beside the record | Decided 15 October 2026, a technical choice: the proposed default |
| RCT-D12 | How does a folder appear and go without a reader seeing half of it? | It is made under a name that starts with a dot and renamed into place once its three files are in it, and renamed to a dot name again before it is deleted. Every reader skips dot names. The alternative: a mark in the record that says it is whole, which every reader would have to check | Decided 15 October 2026, a technical choice: the proposed default |
| RCT-D13 | In what form do stop and answers reach the Trunk, when two programs write `control`? | As the Driver's own control lines (DRV-2), so a Trunk hands them on unchanged. Each writer appends a whole line of at most 64 KiB with one write; the Trunk reads at least once a second. The alternative: a form of the contract's own, which every Trunk would translate for each Driver | Decided 15 October 2026, a technical choice: the proposed default |
