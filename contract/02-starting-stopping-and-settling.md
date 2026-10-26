---
role: authored
spec: RCT-2
status: accepted
---
# Root Contract · 02 Starting, stopping and settling

This specification follows a [Chat Exchange](../../arbol/GLOSSARY.md#chat-exchange) through the hands of its Trunk. A Trunk *accepts* a message when its folder is in place and the message is in the Root's database, and says so to whoever started it; it stops when it is told to; it *ends* the Exchange by itself; and when its process is gone, a Trunk Build *settles* the Exchange, which means it saves what the folder holds. For example, `arbol-root` runs `arbol-trunk --root sapling trivial -v 4 run --detach --exchange 7f3a2c1e-… --request /…/start.json`, reads the one line `{"accepted": true, …}` and returns, and the Trunk goes on alone until the end. This specification states acceptance and refusal, a repeat after a lost answer, running in the background, the process group, stop, the termination signal, the end and its causes, a Driver that falls silent, small calls, what happens while the database is away, settling, sending a message again, and how the Root manager uses all of this. It matters because in version 1 the end of a turn could be refused by another service, a failure before the agent started could leave no end at all, "stopped" was shown before anything had stopped, and a duplicate that arrived after the end started the agent a second time.

It follows the conventions of the [Driver Contract](../../arbol/driver-contract/README.md#conventions) and the additions in this chapter's [README](README.md#conventions). It makes exact what [ARC-2](../../arbol/architecture/02-chat-exchange-across-components.md) and [ARC-3](../../arbol/architecture/03-stuck-trunks.md) say of these steps, and carries the rules of ARC-3 and [ARC-0](../../arbol/architecture/00-components-and-boundaries.md#small-model-calls) for a silent Driver and for small calls. The owner decided on 15 October 2026 what sending a message again does ([RCT-D15](#decisions)); the other forms are technical choices made the same day. The owner answered the open questions on 15 October 2026, that the replaced answer is deleted (RCT-D16) and that telling a silent Driver from a command left in the background is left to future Trunks (RCT-D30), and the specification was accepted the same day.

## Scope

Covered: accepting a message and refusing one, a repeat after a lost answer, the background and the foreground, the process group, stop, the termination signal, the end and its causes, a Driver that falls silent, small calls, a database that cannot be reached, settling, sending a message again, how the Root manager uses a Trunk.

Not covered: the files of the folder ([RCT-1](01-the-exchange-folder.md)); the rows ([RCT-3](03-the-chat-tables.md)); when the Root manager lists, settles and ends an Exchange by force, which ARC-3 and [RTM-0](../specs/00-the-commands.md) state; what a Trunk does with the answer of a small call, such as a title, which each Trunk's own specification states; a message that steers a running agent (ARC-D54).

## Behaviour

### Accepting a message

1. The Trunk reads its options and request file and the Root's `root.json` ([RCT-0](00-overview.md#what-is-handed-to-a-trunk)).
2. When it was handed an Exchange identifier, it looks for it before anything else; a start of an identifier that exists already is a repeat ([below](#a-repeat-after-a-lost-answer)).
3. It makes the Exchange's folder, with a record that has no `acceptedAt` ([RCT-1](01-the-exchange-folder.md#how-the-folder-comes-and-goes)).
4. In one transaction it writes the chat when the chat is new, with an insert that does nothing when another start wrote it a moment before; the Exchange with the state `open` and the next number in its chat; and the Leather Bag's message as the message at position 0 ([RCT-3](03-the-chat-tables.md)).
5. It adds `acceptedAt` to the record.
6. It writes one line on standard output, the *acceptance line*:

```json
{"accepted":true,"duplicate":false,"root":"sapling","exchange":"2d4f6a8c-1b3e-4c5d-8e7f-9a0b1c2d3e4f","chat":"c3d4e5f6-0718-4293-a4b5-c6d7e8f90a1b","number":2,"trunk":"trivial","build":"4"}
```

From that line on, the Exchange is the Trunk's ([ARC-2](../../arbol/architecture/02-chat-exchange-across-components.md)). The Trunk answers within 15 seconds of its start, accepted or refused. All its work with the database for the acceptance, the lookup of step 2 included, ends within 10 seconds of its start; what is still outstanding then is refused with `database_unavailable` ([RCT-D19](#decisions)).

A refusal is one line on standard output and the exit status 1, for example `{"error":{"code":"chat_busy","message":"The chat has an Exchange in progress: 6e2f…"}}`. It leaves no folder and no row, starts nothing, and the Leather Bag's [Draft](../../arbol/GLOSSARY.md#draft) stays in the composer (ARC-2).

| Code | When |
|---|---|
| `invalid_input` | An unknown option or command, a second `--root`, a part of the route that the Trunk's own specification needs and finds nowhere else ([RCT-0](00-overview.md#what-is-handed-to-a-trunk)), a Root identifier that is not one lower-case word of letters and digits (ARC-1), no message, an identifier that is not a UUID, a working folder that is not absolute or does not exist; with `--replaces`, a message, or a `--chat` that is not the chat of the replaced Exchange |
| `unknown_root` | The Root has no readable `root.json` ([RCT-0](00-overview.md#how-a-trunk-finds-its-roots-folder-and-database)) |
| `contract_mismatch` | The Root's `rootContract` is not in the Build's `rootContracts` |
| `database_unavailable` | The work with the database did not end within 10 seconds of the start, or the Keychain failed ([RCT-0](00-overview.md#how-a-trunk-finds-its-roots-folder-and-database)) |
| `chat_busy` | The chat has an Exchange that is still open, or another message to the same chat was accepted at the same moment; the message names the open Exchange. One Exchange at a time in a chat (ARC-D54) |
| `in_progress` | Another start of the same Exchange identifier is still accepting it. The caller asks again after 15 seconds, the longest an acceptance takes |
| `not_found` | `--replaces` names an Exchange that does not exist |
| `cannot_replace` | `--replaces` names an Exchange that is not the last of its chat, has not ended, or was replaced already ([below](#sending-a-message-again)) |
| `trunk_failed` | The Trunk could not make its folder, for example because the disk is full, or failed otherwise; the message says what |

Two first messages to one new chat, or two messages to one chat, that arrive at the same moment meet in the database at step 4: it takes one, and the other is refused with `chat_busy` ([RCT-3](03-the-chat-tables.md#the-tables)).

Observed in version 1: acceptance stored the message and its options in one step, and the agent was started only after that, which worked. The reply carried the session, the Exchange and the chosen provider.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:product-spec/03-turn-execution.md#Lifecycle of one Exchange`

<!-- sources:
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/accept.py#accept_message
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/database.py#PostgresConnection.accept
arbol-root:contract/conformance/rct_conformance/scenarios/r14.py#refused
-->

### A repeat after a lost answer

A caller that chose the Exchange's identifier can start the same Exchange again when it lost the answer, for example when `arbol-root` was killed before it read the acceptance line ([RCT-D17](#decisions)). The Trunk looks for the identifier before it makes any identifier of its own:

| The identifier is… | The Trunk |
|---|---|
| in the database | answers with that Exchange's acceptance and `"duplicate": true`, and starts nothing, also when that Exchange has ended long ago |
| only in the Root's folder, as `<exchange>` or `.<exchange>.new`, and the record's process lives or the folder was made less than 15 seconds ago | refuses with `in_progress` |
| only in the Root's folder otherwise: the start that made it died before the database had the Exchange | removes that folder, as settling does, and goes on as a new Exchange |
| nowhere | goes on as a new Exchange |

Two starts of the same identifier at the same moment cannot both make the folder: the one that finds `.<exchange>.new` or `<exchange>` there already removes what it made and refuses with `in_progress` ([RCT-1](01-the-exchange-folder.md#how-the-folder-comes-and-goes)). A start that finds the Exchange in the database only when it writes it answers as a repeat.

A repeat is not sending a message again: a repeat names the same Exchange and starts nothing; sending again names a new one.

Observed in version 1: repeating the same submission identity returned the same Exchange. Two defects followed from the order of the steps: fresh identifiers were made before the repeat was recognised, so a byte-identical repeat of an edit was refused as a conflict; and a repeat that arrived after the end started the agent again.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:grafts/arbol-trunks/m8-installed-acceptance.md#2026-09-01 — Installed failed-Turn acceptance exposed exact-duplicate edit defect; source follow-up committed`
- `arbol_deprecated_v1:grafts/arbol-trunks/m8-installed-acceptance.md#2026-09-01 continuation — duplicate edit redrive diagnosis and fix`

<!-- sources:
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/accept.py#find_prior
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/claim.py#claim_folder
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/claim.py#root_lock
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/process.py#process_alive
arbol-trunk:trunks/trivial/tests/test_repeat.py#Repeats
arbol-root:contract/conformance/rct_conformance/scenarios/r4.py
-->

### In the background, or in the foreground

With `--detach`, the Trunk points its standard input, output and error at `/dev/null` after the acceptance line, ignores the hang-up signal (SIGHUP), and goes on alone ([RCT-D14](#decisions)). It needs neither its starter, nor Arbol, nor the Root manager, nor the database in order to finish (ARC-0). Its starter reads the one line, decides from it alone, and never waits for the process, which ends with the Exchange.

Without `--detach`, for use by hand, the Trunk stays where it was started. After the acceptance line it also writes every line it appends to `events.jsonl` on standard output, and it ends with the Exchange, with the exit status of a Driver's run (DRV-2): 0 for completed, 1 for failed, 2 for cancelled. A refusal ends with 1, and a repeat with 0 at once.

```sh
cd ~/repos/shop && arbol-trunk run "What does build.sh do?"
# {"accepted":true,"duplicate":false,"root":"sapling","exchange":"…","chat":"…","number":1,"trunk":"trivial","build":"4"}
# {"n":1,"event":{"runId":"…","seq":1,"type":"started",…}}
# …
# {"n":9,"end":{"state":"completed","at":"2026-10-15T09:30:11.050Z"}}
```

Standard output never holds up or ends the Exchange. A line that standard output cannot take at once, for example because nothing reads it, is left out there; the file keeps it. A write that fails, for example because the starter was killed before it read the acceptance line, or because the lines go to `head`, ends nothing: the Trunk writes nothing more to standard output and goes on. A starter that lost the acceptance line learns what happened by a repeat (above).

<!-- sources:
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/run.py#run_exchange
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/run.py#run_accepted
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/events.py#EventsFile._append
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/output.py
-->

### One process group

A Trunk leads a process group of its own, whose identifier is the Trunk's process identifier. Every process it starts itself for the Exchange, its Driver runs and `arbol-classifier` among them, stays in that group. `arbol-root` starts it detached from the terminal and from `arbol-root` itself, in a process group of its own, as the system call `setsid` does, and an interactive shell gives each command line a group that its first program leads. A Trunk that leads its group when it starts stays in it, so that Ctrl-C reaches a Trunk in the foreground; one that does not makes itself the leader of a new group.

A Driver keeps the vendor tool, and with it the agent's commands, in process groups of their own, and ends them itself when its run is stopped, also on an interrupt or termination signal (DRV-2). The Trunk therefore *ends its group* in two steps: SIGTERM to the group, upon which each Driver in it stops its run and ends what the run started, and 5 seconds later SIGKILL to whatever in the group is still alive. The Trunk's watch on itself ([RCT-1](01-the-exchange-folder.md#the-trunk-watches-itself)) and the termination signal (below) end the group this way, and so does `arbol-root stop --force` (ARC-3). A Driver killed before it could end the vendor tool leaves that to the next call of the same Driver, which ends it (DRV-6, ARC-3). A program that moved itself out of those groups is out of everyone's reach (DRV-D33).

For example, the agent runs `make` when the group is ended: the Driver gets SIGTERM, asks `claude` to stop, ends it and the `make` within a few seconds, and exits; the SIGKILL 5 seconds after the SIGTERM finds nothing left.

<!-- sources:
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/detach.py#lead_process_group
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/termination.py#end_group
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/driver.py#capability_query
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/driver.py#DriverRun.start
arbol-trunk:trunks/trivial/tests/test_capabilities.py#Capabilities.test_capability_queries_and_the_run_launcher_stay_in_the_callers_process_group
arbol-root:contract/conformance/rct_conformance/scenarios/r7.py
-->

### Stop

1. The Root manager appends `{"type":"stop"}` to the Exchange's `control` ([RCT-1](01-the-exchange-folder.md#stop-and-answers-control)).
2. Within a second the Trunk hands the line to its run's control input, and stops a small call that runs. Before a run has started, it starts none.
3. The Driver stops the run within 15 seconds ([DRV-2](../../arbol/driver-contract/02-inference.md)): tool calls that were running finish as interrupted, and the run's `finished` says `cancelled`.
4. The Trunk writes the end `cancelled` to its folder, within 18 seconds of the stop line in all, and then ends as below.

A window shows "stopping" from the request, and "stopped" only once the end line of `events.jsonl` says `cancelled` (ARC-2, ARC-3). When the run ends otherwise before the stop reaches it, the Exchange ends with that outcome, also within the 18 seconds. The 18 seconds leave room inside the 20 seconds after which `arbol-root stop` ends the Exchange by force (ARC-D60).

### The termination signal

A Trunk that receives SIGTERM ends its group as above and writes no end ([RCT-D18](#decisions)). This is how a stop that was not obeyed and `arbol-root stop --force` end an Exchange (ARC-3): settling then saves it as interrupted, because the Trunk never confirmed the stop, unless the Trunk had already written its end, which then stands. In contract version 1 a Trunk treats SIGINT, and in the foreground also SIGHUP, as SIGTERM, so Ctrl-C and a closed terminal leave an Exchange to be settled as interrupted.

For example, `arbol-root stop --force` sends SIGTERM to the group of an Exchange whose Trunk has not obeyed a stop: the Driver stops its run, the Trunk exits without an end, and `arbol-trunk … settle` saves every block and tool call that had arrived, and the Exchange as interrupted.

<!-- sources:
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/termination.py#install_handlers
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/cli.py#main
-->

### The end

Every accepted Exchange ends exactly once, on every path, including a failure before the agent started and a defect of the Trunk itself. The Trunk ends it by itself, and no other component accepts, confirms or records the end (ARC-2):

1. It appends the end line to `events.jsonl` and puts the same `end` into `record.json` ([RCT-1](01-the-exchange-folder.md)).
2. It writes the rows not yet saved and the Exchange's final state in one transaction, changing the Exchange only while it is still `open` ([RCT-3](03-the-chat-tables.md#how-lines-become-rows)).
3. It removes its folder.

| State | When | Cause in `error` |
|---|---|---|
| `completed` | The agent's work ended normally. With one run: the run's `finished` said `completed` | — |
| `failed` | Anything else that was not asked for | Always (below) |
| `cancelled` | A stop line, or the Leather Bag declined the agent's question (DRV-3) | — |
| `interrupted` | Only through settling: the process was gone before it ended | — |

| Cause | When |
|---|---|
| A code of [DRV-4](../../arbol/driver-contract/04-errors-and-limits.md), such as `rate_limited`, `not_signed_in` or `context_exceeded` | The run failed. The Driver's whole error object is kept, `retryAfterSeconds` included |
| `driver_failed` | The Driver could not be started, or its process ended without a `finished` event |
| `driver_silent` | The Trunk ended a run that had fallen silent ([below](#a-driver-that-falls-silent)) |
| `events_file_full` | `events.jsonl` reached its limit ([RCT-1](01-the-exchange-folder.md#the-answer-as-it-arrives-eventsjsonl)) |
| `database_unavailable` | The Trunk needed to read the database before a run and could not for 60 seconds ([below](#when-the-database-cannot-be-reached)) |
| `folder_unwritable` | A write to the Exchange's folder failed (below) |
| `trunk_failed` | A defect in the Trunk, or a row the database refused for what it holds ([RCT-3](03-the-chat-tables.md#how-lines-become-rows)); the message says what |

A cause of the contract has the form `{"code", "message"}`. A reader shows a code it does not know with its message.

A run that ends with `rate_limited` ends the Exchange as failed with that error, for example `{"code":"rate_limited","message":"…","retryable":true,"retryAfterSeconds":1800}`. The Trunk starts no run on another route: Arbol offers to send again with what its Brain Recipe picks next (ARC-0, ARC-2).

When a write to the Exchange's folder fails, for example because the disk is full, the end cannot go to the folder first. The Trunk then stops its run as on a stop, writes the end `failed` with the cause `folder_unwritable` and every row it can still make straight to the database, and leaves the folder as it is; settling later adds what the file holds and removes the folder.

A failed, cancelled or interrupted Exchange concerns its own chat only, and the chat takes a new message as soon as the Exchange has ended (ARC-2).

Observed in version 1: the end of a turn had to be accepted by a separate service against a closed list; on 28 September 2026 "usage, then failure" was not on the list, and a real failure could not be recorded. An error while the agent's context was prepared ended the supervising task without closing the Exchange, and a stop then failed until a restart. A stopped Exchange was stored with no mark that it was stopped, so it looked like a short completed one.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:product-spec/12-lessons-and-invariants.md#2. One turn's failure could block everything else`
- `arbol_deprecated_v1:product-spec/03-turn-execution.md#Known problems and lessons`
- `arbol_deprecated_v1:product-spec/03-turn-execution.md#Cancel`

<!-- sources:
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/run.py#run_accepted
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/run.py#drive
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/end.py#end_of_run
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/end.py#save_end
arbol-trunk:trunks/trivial/tests/test_run.py#WholeExchange
arbol-root:contract/conformance/rct_conformance/scenarios/r11.py
-->

### A Driver that falls silent

A Driver ends a silent vendor tool by itself, but the Driver's own process can hang too (ARC-3). A Trunk therefore ends a Driver run that has given no event for one minute longer than its idle limit, 15 minutes unless the run's request set another (DRV-4), counted with `CLOCK_UPTIME_RAW` as in [RCT-1](01-the-exchange-folder.md#the-trunk-watches-itself), while no tool call and no question is open and, where its own specification says how it tells one, no command left in the background. It stops the run, ending the Driver's process when it does not stop, and ends the Exchange as failed with the cause `driver_silent`.

A Trunk cannot see a command that the agent left running in the background: the Driver gives no event for it, possibly for hours (DRV-3). Telling such a command from a Driver that has hung is left to future Trunks, each in its own specification ([RCT-D30](#decisions)). A Trunk that tells none, such as the Trivial Trunk, counts the silence whenever no tool call and no question is open, so a command left running in the background that gives no event for 16 minutes ends the run as failed with `driver_silent`.

For example, a run that has written half its answer and then gives nothing for 16 minutes, with no tool call and no question open, ends as failed with `driver_silent`.

### Small calls

A Trunk makes a small text call, such as a title, only when it was handed a small-call route, as a Driver run on that route with `--tools none` and no chat session; a classification goes through `arbol-classifier` (ARC-0). A small call that is missing, slow or fails never fails the Exchange: the Trunk gives it a time limit of its own and goes on without the result. Of a small call's run, only its `finished` event goes into `events.jsonl` ([RCT-1](01-the-exchange-folder.md#the-answer-as-it-arrives-eventsjsonl)), so that its usage is kept. What a Trunk does with the answer is in its own specification.

For example, a title call that fails with `rate_limited` leaves the chat without a title for now, keeps the run's usage, and the Exchange completes.

### When the database cannot be reached

| When | What happens |
|---|---|
| At acceptance | All work with the database for the acceptance ends within 10 seconds of the start; what is still outstanding then is refused with `database_unavailable`. Nothing is started, no folder stays, and the Draft stays |
| Before a run starts | When the Trunk reads something from the database for a run, such as the chat's earlier messages, it tries every 5 seconds for 60 seconds, with `waitingFor` `database`, and then ends the Exchange as failed with the cause `database_unavailable` |
| While the agent works | The answer keeps arriving in `events.jsonl`. What could not be saved stays in the file; the Trunk tries again every 5 seconds and saves it, in order, once the database is back |
| At the end | The end is in the folder first. The Trunk tries to save for 60 seconds, then ends and leaves its folder, with `end` in its record. Settling saves the Exchange later with that state (ARC-D53) |

Every call to the database has a time limit of 5 seconds, to connect and for each statement, so the heartbeat goes on meanwhile ([RCT-D19](#decisions)). A Trunk never needs the database in order to finish, and nothing the agent produced is lost because the database was away (ARC-2).

For example, while the database server restarts for 20 seconds in the middle of an answer, the answer keeps arriving in the file and in the window, and the rows of those 20 seconds are saved, in order, once the server is back.

<!-- sources:
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/saver.py#Saver
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/database.py#within_limit
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/database.py#transaction
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/end.py#save_end
arbol-trunk:trunks/trivial/tests/test_saver.py#Saving
arbol-trunk:trunks/trivial/tests/test_database.py#Limit
arbol-trunk:trunks/trivial/tests/test_end.py#SavingTheEnd
arbol-root:contract/conformance/rct_conformance/scenarios/r12.py
-->

### Settling

*Settling* saves what the folder of an Exchange holds once its process is gone. It is the command `settle` of a Trunk Build:

```sh
arbol-trunk --root sapling trivial -v 4 settle 7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b

<!-- sources:
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/claim.py#root_lock
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/process.py#process_alive
arbol-trunk:trunks/trivial/tests/test_settle.py#SavingSettlement
arbol-root:contract/conformance/rct_conformance/scenarios/r13.py
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/database.py#PostgresConnection.end
arbol-root:contract/conformance/rct_conformance/scenarios/r11.py#folder_failure
arbol-root:contract/conformance/rct_conformance/scenarios/r12.py#through_the_end
-->

# {"exchange":"7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b","settled":"interrupted"}
```

1. When the Exchange has no folder, there is nothing to do: `"settled": "none"`, exit status 0.
2. When the record's process lives, it refuses with `alive` and changes nothing.
3. It opens the database; when that does not answer within 10 seconds, it refuses with `database_unavailable`, and the folder stays.
4. When the database does not have the Exchange, it was never accepted: settling removes the folder, `"settled": "none"`.
5. It saves every row that `events.jsonl` holds and the tables lack, by the rules of [RCT-3](03-the-chat-tables.md#how-lines-become-rows): a tool call without its `tool_finished` as interrupted, a run without its `finished` as interrupted, with its last usage. For an Exchange that a message sent again has replaced, it writes no row in `messages` or `tool_calls`, because those were deleted on purpose ([RCT-D16](#decisions)); its `runs` are saved, so its tokens count.
6. It writes the final state, only while the Exchange is still `open`: the state of the `end`, when the record or the file has one, and otherwise `interrupted`, ended at the time of the last line of `events.jsonl` that carries one (the `at` of an event or a `smallRun`), or at `acceptedAt` when the file has no such line.
7. It removes the folder, a folder that another settle removed first being no error, and prints the state the Exchange now has in the database, whoever wrote it.

Settling changes no Exchange that has ended, and it can run twice: the second finds no folder. Two settles of one Exchange may also run at the same time, such as the one of `arbol-root stop` and one Arbol runs after a listing: a settle whose folder the other removed first counts it as removed and prints the state the Exchange then has in the database, with the exit status 0. Because RCT-3 fixes how lines become rows, every Trunk Build that implements the Root's version can settle every Exchange of the Root, whichever Trunk ran it ([RCT-D20](#decisions)). Which Build the Root manager starts for it, and when, is in its own specification (ARC-2, ARC-3).

<!-- sources:
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/command.py#parse
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/settle.py
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/database.py#PostgresConnection.exchange_state
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/database.py#PostgresConnection.end
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/rows.py#RowMaker
-->

### Sending a message again

The Leather Bag can send the message of a chat's last Exchange again, once that Exchange has ended, for example after a failure or a rate limit. The previous answer and its tool calls are discarded, and everything runs again from the same message ([RCT-D15](#decisions)).

```sh
arbol-trunk --root sapling trivial -v 4 run --detach --replaces 2d4f6a8c-1b3e-4c5d-8e7f-9a0b1c2d3e4f --exchange 9b3a2c1e-5c6d-4f70-9a1b-2c3d4e5f6a7b --request /…/start.json
```

| | |
|---|---|
| What runs | A new Exchange, with the next number in the chat, whose message the Trunk copies from the replaced Exchange. The command takes no message of its own, and refuses one with `invalid_input`, as it refuses a `--chat` that is not the replaced Exchange's chat |
| What disappears from the chat | The replaced Exchange's answer and tool calls: they are never shown in the chat and never sent to an agent again ([RCT-3](03-the-chat-tables.md#how-a-chat-is-read)). In one transaction with the acceptance, after it has copied the message, the Trunk marks the replaced Exchange as replaced by the new one and deletes its rows in `messages` and `tool_calls` ([RCT-D16](#decisions)). Its row in `exchanges`, marked replaced, and its rows in `runs` stay, because a chat keeps the token usage of each run ([RCT-D21](03-the-chat-tables.md#decisions)), so the tokens it spent still count |
| What is not undone | Changes the agent already made in the working folder. If the first answer wrote two files and ran a database migration, the files and the migration are still there, and the new run may write the files and run the migration again |
| Which Exchange | Only the chat's last Exchange that was not replaced, and only once it has ended. Otherwise `cannot_replace`. Branches and edited messages come later ([RCT-D21](03-the-chat-tables.md#decisions)) |
| Which route | The one the new Exchange is handed. Arbol may hand the route the replaced Exchange ran with, or the next one its Brain Recipe picks (ARC-0) |

Observed in version 1: Retry hid the failed Exchange and everything below it and opened a new Exchange at the same place with the same text and images; the agent got nothing of the failed run, and its changes to files were not undone and could be repeated.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:product-spec/03-turn-execution.md#Retry and Continue from failure point`

<!-- sources:
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/accept.py#find_replacement
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/database.py#checked_replacement
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/database.py#PostgresConnection.replacement
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/database.py#PostgresConnection.accept
arbol-trunk:trunks/trivial/tests/test_replacement.py#ReplacingAMessage
arbol-root:contract/conformance/rct_conformance/scenarios/r14.py
-->

### How the Root manager uses a Trunk

| What | How |
|---|---|
| Start | `arbol-trunk --root <root> <code name> -v <build> run --detach --exchange <id> --request <file>`, naming the Exchange with `--exchange`, which takes the place of any in the file, detached from the terminal in a process group of its own (the system call `setsid`), with standard input from `/dev/null` and the folder of the launchers first on the search path ([RCT-0](00-overview.md#how-a-trunk-starts-a-driver)). It reads one line, waiting at most 20 seconds, removes the request file and returns what the line says |
| Stop | It appends `{"type":"stop"}` to `control`. The Exchange has ended, for the 20 seconds of `arbol-root stop`, once its record has `end` or its folder is gone |
| Stop by force | It sends SIGTERM to the process group of a record whose process lives, and SIGKILL after 5 seconds to what is still alive (ARC-3) |
| Settle | `arbol-trunk --root <root> trivial -v <build> settle <exchange>`, with the Build of `trivial` that `arbol-trunk list` marks `chosen` (RTM-0) |

When it does each, how long it waits for the acceptance line and which Build it starts are stated in [RTM-0](../specs/00-the-commands.md).

## Requirements

A Trunk:

| ID | A Trunk must… | Origin | Verified by |
|---|---|---|---|
| RCT-2.1 | count a message as accepted only when its folder is in place and the chat, the Exchange with the state `open` and the message are in the database, written in one transaction, and only then write the acceptance line | Changed | contract: R1 (SYL-171); also the Trivial Trunk's unit tests in `trunks/trivial/tests/` of `arbol-trunk`: `Acceptances`, `AcceptingAMessage` (SYL-171) |
| RCT-2.2 | answer a start within 15 seconds, end all work with the database for the acceptance within 10 seconds of the start, and refuse with one error line and the exit status 1, leaving no folder and no row and starting nothing, in every case of the table of refusals | Changed | contract: R3 (SYL-172), R4 and R14 (SYL-175), covering every refusal; unit in the Trivial Trunk: `WholeStart`, `InvalidInput`, `NoDriver`, `TheRootTheKeychainAndTheFolder`, `TheDatabaseAtAcceptance`, `Repeats` and `ReplacingAMessage` |
| RCT-2.17 | refuse a message with `chat_busy` exactly when its chat has an open Exchange or another message to that chat is accepted at the same moment | Changed | contract: R3, a chat with an open Exchange (SYL-172), and R5, two messages at the same moment (SYL-173); also the Trivial Trunk's unit tests in `trunks/trivial/tests/` of `arbol-trunk`: `Acceptances.test_a_chat_with_an_open_exchange_is_busy_and_names_it` (SYL-171), `TheDatabaseAtAcceptance.test_a_chat_with_an_open_exchange_is_busy_and_the_refusal_names_it` (SYL-172) |
| RCT-2.3 | look for a handed Exchange identifier in the database and in the Root's folder, under its name and under `.<exchange>.new`, before it makes any identifier; answer with that Exchange's acceptance and `"duplicate": true` when the database has it; refuse with `in_progress` while another start of it lives or is less than 15 seconds old; and remove a folder whose start died before the database had it and go on as a new Exchange | Changed | contract: R4 (SYL-175); unit: `Repeats` in `trunks/trivial/tests/test_repeat.py` of `arbol-trunk` |
| RCT-2.18 | when two starts of the same Exchange identifier meet, let only the one that renamed its folder into place go on, and refuse the other with `in_progress` after removing what it made | New | contract: R4 (SYL-175); unit: `Repeats.test_two_starts_of_one_identifier_leave_only_the_winners_folder_and_acceptance` and `Repeats.test_another_process_cannot_claim_while_the_root_is_locked_and_its_wait_is_bounded` |
| RCT-2.4 | with `--detach`, point its standard input, output and error at `/dev/null` after the acceptance line, ignore SIGHUP, and finish the Exchange without its starter, Arbol, the Root manager or the database | Changed | contract: R1 (SYL-171); also the Trivial Trunk's unit tests in `trunks/trivial/tests/` of `arbol-trunk`: `Background` (SYL-171); drill: SYL-187 |
| RCT-2.5 | without `--detach`, write every line it appends to `events.jsonl` on standard output too, and end with the exit status 0 for completed, 1 for failed and 2 for cancelled | New | contract: R2 (SYL-174); also the Trivial Trunk’s unit `WholeExchange.test_foreground_exit_status_follows_the_exchange_end` |
| RCT-2.19 | let standard output never hold up, change or end the Exchange: leave out a line it cannot take at once, and write nothing more there after a write has failed | New | contract: R2 (SYL-174) and R4 (SYL-175), including a starter that closes its pipe before acceptance; unit: `ForegroundOutput` in `trunks/trivial/tests/test_output.py` of `arbol-trunk` |
| RCT-2.6 | lead a process group of its own and keep in it every process it starts itself for the Exchange | New | contract: R7 (SYL-176); unit: `Capabilities.test_capability_queries_and_the_run_launcher_stay_in_the_callers_process_group`, `Background` and `Termination` in `trunks/trivial/tests/` of `arbol-trunk` |
| RCT-2.7 | on a stop line, hand the stop to its run and stop a small call that runs, start no run that had not started, and write its end to its folder within 18 seconds of the line, `cancelled` unless the run had ended otherwise before the stop reached it | Changed | contract: R8 (SYL-173); also the Trivial Trunk's unit tests in `trunks/trivial/tests/` of `arbol-trunk`: `StoppingTheRun`, `WholeExchange.test_after_a_stop_line_before_the_run_no_run_starts_and_the_exchange_is_cancelled`, `WholeExchange.test_a_stop_line_while_a_tool_call_runs_cancels_the_exchange_and_saves_the_call_as_interrupted` (SYL-173); the Trivial Trunk makes no small call (TRV-0.8) |
| RCT-2.8 | on SIGTERM, on SIGINT, and in the foreground on SIGHUP, end its process group with SIGTERM and 5 seconds later SIGKILL, and write no end | New | contract: R2 and R10 (SYL-174); also the Trivial Trunk’s unit `Termination` and `Background` |
| RCT-2.9 | end every accepted Exchange exactly once, on every path, with no other component accepting or confirming the end | Changed | contract: R1 (SYL-171) and R11 (SYL-177); unit: `WholeExchange` in `trunks/trivial/tests/test_run.py` of `arbol-trunk`; installed drill remains SYL-187 |
| RCT-2.20 | end an Exchange in this order: the end in its folder, then the rows not yet saved and the final state in one transaction that changes only an open Exchange, then the folder removed; and when the folder cannot be written, end it as failed with `folder_unwritable` straight in the database and leave the folder | New | contract: R1 (SYL-171) and R11 (SYL-177), including a folder made read-only during a tool call and later settled; unit: `SavingTheEnd`, `WholeExchange` and `RejectedRows`; installed drill remains SYL-187 |
| RCT-2.10 | end an Exchange whose run met a rate limit as failed with the Driver's error kept whole, and start no run on another route | Changed | contract: R11 (SYL-177); unit: `WholeExchange.test_a_failed_run_keeps_the_drivers_error_whole` and `WholeExchange.test_a_refusal_can_be_the_only_event_and_no_other_route_is_tried`, in `trunks/trivial/tests/test_run.py` of `arbol-trunk` |
| RCT-2.21 | end a Driver run that gave no event for one minute longer than its idle limit, counted with `CLOCK_UPTIME_RAW`, while no tool call or question was open and, where its own specification says how it tells one, no command left in the background (RCT-D30), and end the Exchange as failed with `driver_silent` | New | contract |
| RCT-2.22 | make a small text call only on the small-call route it was handed, as a Driver run with `--tools none` and no chat session, make classifications only through `arbol-classifier`, and never fail an Exchange because a small call is missing, slow or fails | New | contract: R19 (SYL-171); also the Trivial Trunk's unit tests in `trunks/trivial/tests/` of `arbol-trunk`: `WholeExchange.test_with_a_small_call_route_no_small_call_is_made_and_the_route_stays_as_handed` (SYL-171) |
| RCT-2.11 | when the database cannot be reached, keep what is not saved in `events.jsonl` and try again every 5 seconds while the agent works, try for 60 seconds to read what a run needs and then end the Exchange as failed with `database_unavailable`, and at the end try for 60 seconds before it ends and leaves its folder, giving every call to the database a limit of 5 seconds | Changed | contract: R12 (SYL-177), with recovery during work, after an outage immediately after acceptance, and by settling after the final retry budget; unit: `Saving`, `SavingTheEnd` and `Limit` in `trunks/trivial/tests/` of `arbol-trunk`, including moved-clock retries and the unscaled call limit; installed drill remains SYL-187 |
| RCT-2.12 | on `settle`, refuse while the record's process lives, save what the folder holds by the rules of RCT-3 with the state of the end or else as interrupted, write no row in `messages` or `tool_calls` of an Exchange that was replaced, change no Exchange that has ended, remove the folder, a folder that another settle removed first being no error, and print the state the Exchange then has in the database | Changed | contract: R2 and R10 (SYL-174), R13 (SYL-175), R11 and R12 (SYL-177); unit: `OpeningSettlement`, `ReplayingSettlement`, `SavingSettlement` and `RejectedRows` in `trunks/trivial/tests/` of `arbol-trunk` |
| RCT-2.13 | on `run --replaces`, accept only for the last Exchange of its chat that has ended and was not replaced, refuse a message or another chat with `invalid_input`, copy its message to the new Exchange, and in the transaction of the acceptance mark it replaced by the new one and delete its rows in `messages` and `tool_calls`, keeping its row in `exchanges` and its rows in `runs` | New | contract: R3 (SYL-172) and R14 (SYL-175), including races with a next message and with another replacement; unit: `ReplacingAMessage` in `trunks/trivial/tests/test_replacement.py` of `arbol-trunk` |

The Root manager:

| ID | The Root manager must… | Origin | Verified by |
|---|---|---|---|
| RCT-2.14 | start a Trunk only through `arbol-trunk`, naming the Root, the code name, the Build and the Exchange, with `--detach`, detached from the terminal in a process group of its own, and with the folder of the launchers first on the search path | New | unit |
| RCT-2.15 | stop an Exchange by a stop line in `control`, count it as ended once its record has `end` or its folder is gone, end processes by force only by signals to the process group of a record whose process lives, and save an Exchange whose process is gone only through `settle` | Changed | drill |

Arbol:

| ID | Arbol must… | Origin | Verified by |
|---|---|---|---|
| RCT-2.16 | show an Exchange as stopped only once its end says `cancelled`, and show a failed Exchange with its cause, a code it does not know with its message | Changed | journey |

## Interface

```ts
// arbol-trunk … run … : one line on standard output, the acceptance line
interface Accepted {
  accepted: true;
  duplicate: boolean;                  // true: this Exchange was accepted before, and nothing was started now
  root: string;                        // "sapling"
  exchange: string;                    // a UUID
  chat: string;                        // a UUID
  number: number;                      // the Exchange's place in its chat, 2
  trunk: string;                       // "trivial"
  build: string;                       // "4"
}

// a refusal of run or settle: one line on standard output, exit status 1
interface TrunkRefusal {
  error: { code: "invalid_input" | "unknown_root" | "contract_mismatch" | "database_unavailable" | "chat_busy"
              | "in_progress" | "not_found" | "cannot_replace" | "trunk_failed" | "alive";
           message: string };
}

// arbol-trunk … settle <exchange> : one line on standard output, exit status 0
interface Settled {
  exchange: string;
  settled: "completed" | "failed" | "cancelled" | "interrupted" | "none";   // the state in the database afterwards; none: no folder, or never accepted
}

type ExchangeError =                   // the cause of a failed Exchange, in its end and in the exchanges table
  | DriverError                        // as the Driver wrote it (DRV-4)
  | { code: "driver_failed" | "driver_silent" | "events_file_full" | "database_unavailable"
          | "folder_unwritable" | "trunk_failed";
      message: string };
```

| Command | Exit status |
|---|---|
| `run --detach` | As in the foreground once the Exchange has ended; 1 for a refusal. The starter decides from the acceptance line alone and never waits for it |
| `run` in the foreground | 0 completed, 1 failed or refused, 2 cancelled |
| `settle` | 0 settled or nothing to settle; 1 refused |

## Known problems in version 1

Not to be reproduced.

- **An end that could be refused.** A separate service checked each end against a closed list, and a real failure could not be recorded.
- **No end after an early failure.** A failure before the agent started could leave the Exchange without any end.
- **"Stopped" before anything stopped.** The window showed the stop before the coordinator confirmed it, and a reopened stopped Exchange looked completed.
- **Identifiers before the repeat check.** A repeat got fresh identifiers and was refused, and a repeat after the end started the agent again.
- **A cancelled Exchange stored no usage,** although its tokens were spent.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:product-spec/12-lessons-and-invariants.md#2. One turn's failure could block everything else`
- `arbol_deprecated_v1:product-spec/03-turn-execution.md#Known problems and lessons`
- `arbol_deprecated_v1:product-spec/03-turn-execution.md#Usage accounting`
- `arbol_deprecated_v1:grafts/arbol-trunks/m8-installed-acceptance.md#2026-09-01 continuation — duplicate edit redrive diagnosis and fix`

## Decisions

| ID | Question | Proposed default | Outcome |
|---|---|---|---|
| RCT-D14 | How does a Trunk started in the background tell its starter that it accepted the message? | With `--detach`: one JSON line on standard output, then it points its standard input, output and error at `/dev/null`, ignores SIGHUP and goes on alone. Without `--detach` it stays in the foreground and also prints the lines of `events.jsonl`, for use by hand. The alternative: always go into the background, which leaves a person at the terminal with one line and nothing to watch | Decided 15 October 2026, a technical choice: the proposed default |
| RCT-D15 | What does sending a message again do? | It discards the previous answer and its tool calls and runs everything again from the same message. Changes the agent already made in the working folder are not undone. Only the chat's last Exchange can be sent again, once it has ended; the Trunk copies its message into a new Exchange and marks the old one replaced; the new Exchange runs with the route it is handed | Decided 15 October 2026 by the owner as to the principle: sending a message again discards the previous answer and its tool calls and runs everything again, and changes already made in the working folder are not undone. The details are the proposed default, a technical choice made the same day |
| RCT-D16 | When you send a message again, the old answer disappears from the chat either way. Should it also be deleted from the database? | No. It stays, marked "replaced", is never shown and never sent to the agent, and the tokens it spent still count. If yes: its messages and tool calls are deleted, and its runs stay, because a chat keeps the token usage of each run (RCT-D21) | Decided 15 October 2026 by the owner: "Delete old answer. User rarely needs one". In the transaction that accepts the message sent again, the Trunk deletes the replaced Exchange's rows in `messages` and `tool_calls`. Its row in `exchanges`, marked with `replaced_by`, and its rows in `runs` stay, because a chat keeps the token usage of each run (RCT-D21), so its tokens still count |
| RCT-D17 | A start whose answer was lost is made again with the same Exchange identifier. Is that a second Exchange, or sending again? | Neither. The Trunk answers with the Exchange it accepted first and starts nothing, also after that Exchange has ended, as version 1 did for a repeated submission. It looks before it makes any identifier, so neither of version 1's two defects can occur. When two starts of one identifier meet, the folder lets only one in; a folder whose start died before the database had the Exchange is removed and the start goes on. The alternative: a second Exchange, which runs the agent twice for one message | Decided 15 October 2026, a technical choice: the proposed default |
| RCT-D18 | What does a Trunk do on the termination signal? | It ends its process group in two steps, SIGTERM and 5 seconds later SIGKILL, so that each Driver can first end the vendor tool and the agent's commands, and writes no end, so settling saves the Exchange as interrupted. ARC-3 says that an Exchange ended by force is interrupted and not cancelled, because the Trunk never confirmed the stop (ARC-D60). SIGINT, and SIGHUP in the foreground, are treated alike. The alternative: SIGKILL to the group at once, which leaves the vendor tool and the agent's commands running until the next call of that Driver | Decided 15 October 2026, a technical consequence of ARC-D60 drawn the same day |
| RCT-D19 | How long does a Trunk wait for the database? | At acceptance, all its work with the database ends within 10 seconds of the start, or it refuses; before a run that needs to read, 60 seconds, then the Exchange fails; while the agent works, it tries every 5 seconds; at the end, 60 seconds, then it ends and leaves its folder. Each call has a limit of 5 seconds, so the heartbeat goes on. ARC-2 says only "for a while". The alternative: wait without limit, which leaves a Leather Bag with neither an answer nor a refusal | Decided 15 October 2026, a technical choice: the proposed default |
| RCT-D20 | Which Trunk Build can settle an Exchange? | Every Build that implements the Root's version, because this contract fixes how the lines of `events.jsonl` become rows. An Exchange whose own Build was removed can then still be settled. Which Build the Root manager starts is in its specification. The alternative: only the Build that ran the Exchange, which then must never be removed while an Exchange of it is unsettled | Decided 15 October 2026, a technical choice: the proposed default |
| RCT-D30 | A command the agent leaves running in the background, such as a development server, gives no event, perhaps for hours, and ARC-3 forbids ending an Exchange that waits for a command. How does a Trunk tell that from a Driver that has hung? | It does not try: it counts the silence of ARC-3 only while the run has started no `shell` call. A Driver that hangs after a command is then caught only by the Leather Bag, who sees for how long the agent has been at work and stops it (ARC-D61). The alternative: the Driver Contract gains an event that says work goes on in the background, sent at least every few minutes, and the rule stays on throughout | Decided 15 October 2026 by the owner: "For trivial trunk we shouldn't consider long running commands. It will be implemented in future trunks". The Root Contract prescribes no way to tell them apart: a Trunk counts the silence while no tool call and no question is open and, where its own specification says how it tells one, no command left in the background. The Trivial Trunk tells none, so with it a command left running in the background that gives no event for 16 minutes ends the run as failed with `driver_silent` (TRV-0) |
