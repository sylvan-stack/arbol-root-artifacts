---
role: authored
spec: RCT-3
status: accepted
---
# Root Contract · 03 The chat tables

The *chat tables* are the tables of a Root's database that hold its chats: every message, every tool call with its result, the token usage of every run, and each chat's title. The Trunks write them, the Arbol backend reads them with SQL, and they look the same whichever Trunk wrote them. For example, the question "What does build.sh do?" and its answer are one row in `exchanges`, a row in `messages` for the question and one for each block of the answer, a row in `tool_calls` for the `cat build.sh` the agent ran, and a row in `runs` with the tokens it spent. This specification states the layout of contract version 1 as one file of SQL, what each table holds and which program writes which column when, how the lines of `events.jsonl` become rows, how a reader puts a chat together, what is a column and what is kept whole as `jsonb`, and why contract version 1 has no additions. It matters because version 1 kept no row for an Exchange, left the Exchange's identifier empty on the messages of an older writer so that old chats showed nothing, keyed tool calls by an identifier that is unique only within one run, and stored usage without saying which Exchange spent it.

It follows the conventions of the [Driver Contract](../../arbol/driver-contract/README.md#conventions) and the additions in this chapter's [README](README.md#conventions). The owner decided on 15 October 2026 what a chat keeps ([RCT-D21](#decisions)) and that the layout is a plain file of SQL ([RCT-D22](#decisions)). The owner answered the open questions on 15 October 2026, among them that the answer a message sent again replaces is deleted ([RCT-D16](02-starting-stopping-and-settling.md#decisions)), and the specification was accepted the same day. Contract version 1 holds only what the first thin path through every part needs ([ARC-1](../../arbol/architecture/01-roots.md#what-is-built-first), ARC-D39): one message sent, its answer streamed and saved, the chat opened again, and the chat migrated into a second Root.

## Scope

Covered: the layout file and how programs talk to the database, the five tables, who writes what and when, how lines become rows, how a chat is read, columns and `jsonb`, additions.

Not covered: Arbol's own tables, such as Drafts, and the record of where a migrated chat came from, which the Root keeps in its own layout beside this one ([ARC-1](../../arbol/architecture/01-roots.md), the [Root manager's specifications](../specs/README.md)); the migration script of a Root (ARC-1, [RTM-1](../specs/01-builds-layouts-and-migration.md#the-migration-script)); branches and edited messages, which come later (RCT-D21); the history a Trunk sends its agent, which each Trunk's own specification states, except that it leaves out replaced Exchanges.

## Behaviour

### The layout is one file of SQL

The layout of contract version 1 is the file `contract/1/layout.sql` in the `arbol-root` repository: `CREATE TABLE` statements with their keys and checks, and a comment on every table and column. It is not a set of models in a library: each program writes its own plain SQL against these tables. The Root's `arbol-root` Builds carry the file, and `arbol-roots` runs it with `psql` (ARC-1):

```sh
psql --no-psqlrc --set ON_ERROR_STOP=1 --single-transaction --file layout.sql "postgresql://arbol@127.0.0.1:5432/arbol_sapling"
```

Every statement in it can run again (`CREATE TABLE IF NOT EXISTS`), so running the file on a database that has everything changes nothing. The whole file is in the [Interface](#interface) below.

A Build that talks to the database from Python uses psycopg 3, at a version pinned in that Build. Every call has a time limit of 5 seconds ([RCT-2](02-starting-stopping-and-settling.md#when-the-database-cannot-be-reached)). No Trunk and no backend creates, changes or removes a table, a column, an index or a check (ARC-1).

### The tables

**`chats`: one row for each chat.** The Trunk that accepts a chat's first message writes `id` and `created_at`. `title` is NULL while the chat has none; no placeholder is ever stored as a title.

```text
id                                    title                 created_at
c3d4e5f6-0718-4293-a4b5-c6d7e8f90a1b  What build.sh does    2026-10-15 09:30:00.412+00
```

**`exchanges`: one row for each accepted Exchange.** The Trunk writes `id`, `chat_id`, `number`, `state` = `open`, `trunk`, `build`, `route`, `small_call_route`, `working_folder` and `accepted_at` when it accepts the message, and `state`, `ended_at` and `error` when the Exchange ends. A chat has at most one open Exchange: a unique index on the open ones, and the unique number of each Exchange in its chat, turn a second message to a busy chat, and two messages to one chat at the same moment, into `chat_busy` ([RCT-2](02-starting-stopping-and-settling.md#accepting-a-message)). `replaced_by` is written once, by the Trunk that accepts the message sent again, which in the same transaction deletes the replaced Exchange's rows in `messages` and `tool_calls` and keeps its rows in `runs` ([RCT-2](02-starting-stopping-and-settling.md#sending-a-message-again), RCT-D16).

```text
id        chat_id   number  state      error                         trunk    build  accepted_at  ended_at  replaced_by
7f3a2c1e  c3d4e5f6  1       completed                                trivial  4      09:30:00     09:30:11
2d4f6a8c  c3d4e5f6  2       failed     {"code": "rate_limited", …}   trivial  4      09:41:12     09:41:15  9b3a2c1e
9b3a2c1e  c3d4e5f6  3       completed                                trivial  4      10:12:40     10:12:58
```

**`messages`: one row for each block of text.** Position 0 is the Leather Bag's message, written at acceptance. Each block of the agent's answer is written as its `text` line is written to `events.jsonl`, with the text exactly as the Driver's `text` event carried it, because a Driver recognises a conversation by that text (DRV-2); only two characters that PostgreSQL cannot store are replaced ([below](#how-lines-become-rows)).

**`tool_calls`: one row for each tool call, with its result.** Written when the call has finished. A call that the end of its Exchange cut off is written with `ok` false and `interrupted` true. A question the agent asks the Leather Bag is a tool call too (DRV-3), and its row also keeps the question and the Leather Bag's answer.

**`runs`: one row for each Driver run of the Exchange, with its token usage.** The agent's runs are of the kind `agent`, and the runs of small calls, such as a title, of the kind `small`. Written when the run has ended, for a failed or cancelled run too, because its tokens were spent. A classification through `arbol-classifier` is no Driver run and has no row.

| Table | Written by | When | Read by |
|---|---|---|---|
| `chats` | The Trunk that accepts the chat's first message; `title` also by Arbol | At acceptance; the title at any time | Arbol, the Trunks |
| `exchanges` | The Trunk of the Exchange, or the Trunk Build that settles it; `replaced_by` by the Trunk that accepts the message sent again | At acceptance and at the end | Arbol, the Trunks |
| `messages`, `tool_calls`, `runs` | The Trunk of the Exchange, or the Trunk Build that settles it; the rows in `messages` and `tool_calls` of a replaced Exchange are deleted by the Trunk that accepts the message sent again | As each block, call and run is finished; the deletion at the acceptance of the message sent again | Arbol, the Trunks |

One writer for each table is the rule (ARC-D10), and the Trunks are that one writer, whichever Trunk it is: each writes only the rows of the Exchange it runs or settles, and, when it accepts a message sent again, marks the Exchange that message replaces and deletes that Exchange's messages and tool calls. Two exceptions remain, as ARC-0 states them. The title: Arbol writes it when the Leather Bag changes it, and a Trunk that makes titles writes it under the rule of its own specification, which keeps it from replacing a title the Leather Bag wrote. And the migration script of a new Root, which fills every table once before the Root is used, writes an Exchange that was open at the time of the migration as `interrupted`, and keeps the record of where each chat came from in the table `migrated_chats` of the Root's own layout (RTM-1).

<!-- sources:
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/database.py#PostgresConnection.accept
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/database.py#PostgresConnection.end
-->

### How lines become rows

A Trunk writes a row from the line of `events.jsonl` that carries it, once the line is in the file ([RCT-1](01-the-exchange-folder.md#the-answer-as-it-arrives-eventsjsonl)), and settling writes the same rows from the same lines ([RCT-2](02-starting-stopping-and-settling.md#settling)). The rule is the same for every Trunk:

| Line of `events.jsonl` | Row | Position | Written |
|---|---|---|---|
| A `text` event without `parentCallId` | `messages`: role `assistant`, `text` as the event carried it | The line's `n` | With the line |
| A `tool_started` event | `tool_calls`: `run_id`, `call_id`, `parent_call_id`, `name`, `input` from it; `ok`, `result`, `error`, `truncated`, `interrupted` from the `tool_finished` with the same `runId` and `callId`. For a question, `ask` from the `ask_opened` whose `ask` names the call, and `answer` from the last `answer` line for that `askId` that no `control_refused` for the same `askId` follows | The `n` of the `tool_started` line | With the `tool_finished` line. At the end, a call without one: `ok` false, `interrupted` true, `error` "The Exchange ended before the tool call finished." |
| A `finished` event | `runs`: kind `agent`; `outcome`, `error`, `ended_at` from it; `usage` from it, or else from the run's last `usage` event, or else `{}`; `started` from the run's `started` event | — | With the line. At the end, a run without one: `outcome` `interrupted`, its last usage, and the time of its last line |
| A `smallRun` line | `runs`: kind `small`; `outcome`, `error`, `ended_at` and `usage` from the `finished` event it carries | — | With the line |
| The end line | `exchanges`: `state`, `ended_at`, `error` | — | At the end ([RCT-2](02-starting-stopping-and-settling.md#the-end)) |
| Any other line: fragments, `thinking`, `notice`, `ask_closed`, a sub-agent's text | None. They are shown while the answer arrives and are not kept | | |

Each row is written with an insert that does nothing when the row is already there (`INSERT … ON CONFLICT DO NOTHING`), so saving the same line twice changes nothing. Positions keep the true order of the answer, because a Driver writes a block of text before the tool call that follows it (DRV-2). For the lines of the example in RCT-1, the rows are:

```text
messages    exchange 7f3a…  position 0  user       What does build.sh do?
messages    exchange 7f3a…  position 3  assistant  Let me read it.
tool_calls  exchange 7f3a…  position 4  shell      {"command": "cat build.sh"}  ok
messages    exchange 7f3a…  position 6  assistant  It compiles the three packages and copies them to dist/.
runs        exchange 7f3a…  run 7f3a….1  agent  completed  {"model": "sonnet", "inputTokens": 5120, "outputTokens": 812}
runs        exchange 7f3a…  run 7f3a….t  small  completed  {"model": "haiku", "inputTokens": 310, "outputTokens": 9}
```

PostgreSQL can store neither the character U+0000 nor one half of a surrogate pair that stands alone, and a Driver can send both: a NUL in the output of `cat` on a binary file, and an emoji split across two `text` blocks (DRV-6). Before it writes a value, a Trunk therefore replaces each of them with U+FFFD (�), in the text of a message and in every string inside a `jsonb` value, and changes nothing else ([RCT-D31](#decisions)). The split emoji is then saved as two replacement characters; the file keeps it as the Driver sent it.

A row of the answer that the database refuses for what it holds, with an error of the SQLSTATE class 22 (a value it cannot take) or 23 (a key or a check), is not tried again, because it would be refused for ever and hold up every row after it. The Trunk, and settling, leave that row out. An Exchange whose end is not yet written then ends as failed with the cause `trunk_failed`, whose message names the line `n`; an end already written stands. Either way the chat takes a new message once the end is saved.

### How a chat is read

A reader puts a chat together without knowing which Trunk wrote it. It leaves out every Exchange that was replaced, and orders by the Exchange's number and the position within it:

```sql
SELECT e.number, x.position, x.kind, x.body
FROM exchanges e
JOIN (SELECT exchange_id, position, role AS kind, text AS body FROM messages
      UNION ALL
      SELECT exchange_id, position, 'tool ' || name, input::text FROM tool_calls WHERE parent_call_id IS NULL) x
  ON x.exchange_id = e.id
WHERE e.chat_id = 'c3d4e5f6-0718-4293-a4b5-c6d7e8f90a1b' AND e.replaced_by IS NULL
ORDER BY e.number, x.position;
```

While the folder of an Exchange exists, its answer is read from its `events.jsonl`, which holds all of it so far ([RCT-1](01-the-exchange-folder.md)); only the Leather Bag's message comes from the table. A reader reads the Exchange's row first, and while it is open, the folder; when the folder is gone by then, it reads the row again, because the end is in the database before the folder goes.

A replaced Exchange is never shown and never sent to an agent: a Trunk leaves every replaced Exchange out of the history it sends its agent, so a message sent again runs as if the first answer had never been.

<!-- sources:
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/database.py#PostgresConnection.accept
arbol-trunk:trunks/trivial/src/arbol_trunk_trivial/history.py#make_history
arbol-trunk:trunks/trivial/tests/test_replacement.py#ReplacementHistory
arbol-root:contract/conformance/rct_conformance/scenarios/r14.py
-->

### Columns and `jsonb`

| Kept as | What | Why |
|---|---|---|
| A column of its own | Identifiers, numbers and positions, states, times, the code name and the build number, a tool's name, the kind of a run, the text of a message, the flags of a tool call | A reader filters, sorts or joins on them, and the database checks them |
| `jsonb`, whole, in the form of the Driver Contract | A tool call's input and result (DRV-2), a question and its answer (DRV-3), a run's usage (`RunUsage`) and its `started` event (DRV-2), a Driver's error (DRV-4) | No field is lost or renamed, and a field a newer Driver adds arrives without a change of the tables |
| `jsonb`, whole, in the form of ARC-0 | A route (`Route`) | It is recorded as it was handed |

For example, a newer Driver that adds a field `cacheWriteTokens` to a run's usage needs no change of the tables: the field arrives inside `usage`.

The identifiers of chats and Exchanges are `uuid` and never NULL. Times are `timestamptz`. A row is never deleted along with another (`ON DELETE RESTRICT`), and no table holds a second copy of the transcript ([RCT-D23](#decisions)).

### Additions

Contract version 1 has no additions: its layout is the whole file below. An addition to the chat tables, such as a new column, would reach every Root of the version, while a Trunk Build that writes it must not be chosen in a Root whose database lacks it; how both are done is decided with the first addition ([RCT-D24](#decisions)). Any other change, such as a column renamed or a check made stricter, needs a new version of this contract and a new Root (ARC-1).

## Requirements

A Trunk:

| ID | A Trunk must… | Origin | Verified by |
|---|---|---|---|
| RCT-3.1 | write rows only into `chats`, `exchanges`, `messages`, `tool_calls` and `runs`, only for the Exchange it runs or settles and, when it accepts a message sent again, the Exchange that message replaces (RCT-3.4), with SQL of its own, and create, change or remove no table, column, index or check | Changed | contract: R16 after every scenario, R1 and R19 (SYL-171) |
| RCT-3.2 | write each row by the rules of *How lines become rows* as soon as the line it comes from is in `events.jsonl`, with an insert that does nothing when the row is already there | Changed | contract: R1 (SYL-171), and R8 and R9 for the row of a question (SYL-173); also the Trivial Trunk's unit tests in `trunks/trivial/tests/` of `arbol-trunk`: `Rows`, `Saving` (SYL-171), `Questions` (SYL-173) |
| RCT-3.3 | write the Leather Bag's message at position 0 exactly as it was sent, and each block of the answer exactly as the Driver's `text` event carried it, apart from the replacements of RCT-3.10 | Kept | contract: R1 (SYL-171) |
| RCT-3.10 | replace the character U+0000 and every half of a surrogate pair that stands alone with U+FFFD in every value it writes, and change nothing else | New | contract |
| RCT-3.11 | try no row of the answer again that the database refused with an error of the SQLSTATE class 22 or 23, leave it out, and end an Exchange whose end is not yet written as failed with `trunk_failed`, naming the line | New | unit |
| RCT-3.4 | change no row of an Exchange that has ended, except to set its `replaced_by` once, and delete no row, except the rows in `messages` and `tool_calls` of the Exchange that a message sent again replaces, deleted by the Trunk that accepts that message | New | contract: R13 and R14 (SYL-175); unit: `ReplacingAMessage` and `SavingSettlement` in `trunks/trivial/tests/` of `arbol-trunk` |
| RCT-3.9 | leave every replaced Exchange out of the history it sends an agent | New | contract: R14 (SYL-175); unit: `History` and `ReplacementHistory` in `trunks/trivial/tests/` of `arbol-trunk` |

The layout:

| ID | The layout must… | Origin | Verified by |
|---|---|---|---|
| RCT-3.6 | be one file, `contract/<version>/layout.sql` in the `arbol-root` repository, of statements that can run again, with a comment on every table and column, run with `psql` on a Root's database by `arbol-roots` alone, and otherwise only on a database server of a test's or a check's own (RCT-4, RTM-1, TRV-0) | New | unit: `Layout.test_the_file_is_the_interface_of_rct_3`, `Layout.test_runs_twice_and_the_second_run_changes_nothing`, `Layout.test_every_table_and_column_has_a_comment`, `Layout.test_a_second_open_exchange_in_a_chat_is_refused`, `Layout.test_a_second_exchange_with_the_same_number_is_refused`, `Layout.test_a_failed_exchange_without_a_cause_is_refused`, `Layout.test_a_run_of_an_unknown_kind_is_refused`, `Layout.test_the_deletion_of_a_chat_that_has_exchanges_is_refused`, `Layout.test_nul_and_a_lone_half_of_a_surrogate_pair_are_refused_and_u_fffd_is_stored`, in `contract/conformance/tests/test_layout.py`, on a server of the test's own (SYL-170); that `arbol-roots` alone runs it on a Root's database: SYL-185 |

Arbol:

| ID | Arbol must… | Origin | Verified by |
|---|---|---|---|
| RCT-3.8 | read the chat tables directly, write in them only `chats.title` when the Leather Bag changes it, and leave out every Exchange that was replaced | Changed | unit |

RCT-3.5 and RCT-3.7, on tables and columns added inside a Root, were taken out in review on 15 October 2026: contract version 1 has no additions (RCT-D24).

## Interface

The file `contract/1/layout.sql` of the `arbol-root` repository, whole:

```sql
-- Root Contract, version 1: the chat tables (RCT-3).
-- Kept in the arbol-root repository as contract/1/layout.sql.
-- arbol-roots runs it with: psql --no-psqlrc --set ON_ERROR_STOP=1 --single-transaction --file layout.sql
-- Every statement can run again: on a database that already has everything, the file changes nothing.
-- No program creates or changes a table any other way. Each program writes its rows with SQL of its own.

CREATE TABLE IF NOT EXISTS chats (
  id          uuid        PRIMARY KEY,
  title       text        CHECK (title IS NULL OR char_length(title) BETWEEN 1 AND 240),
  created_at  timestamptz NOT NULL
);
COMMENT ON TABLE chats IS 'One row for each chat. Written by the Trunk that accepts its first message; the title also by Arbol.';
COMMENT ON COLUMN chats.id IS 'The chat''s identifier, a UUID chosen by the caller or by the Trunk, for example c3d4e5f6-0718-4293-a4b5-c6d7e8f90a1b.';
COMMENT ON COLUMN chats.title IS 'The chat''s one title, 1 to 240 characters; NULL while it has none. Arbol writes it when the Leather Bag changes it; a Trunk that makes titles writes it under the rule of its own specification.';
COMMENT ON COLUMN chats.created_at IS 'When the Trunk accepted the chat''s first message.';

CREATE TABLE IF NOT EXISTS exchanges (
  id                uuid        PRIMARY KEY,
  chat_id           uuid        NOT NULL REFERENCES chats (id) ON DELETE RESTRICT,
  number            integer     NOT NULL CHECK (number >= 1),
  state             text        NOT NULL CHECK (state IN ('open', 'completed', 'failed', 'cancelled', 'interrupted')),
  error             jsonb,
  trunk             text        NOT NULL CHECK (trunk ~ '^[a-z][a-z0-9]{0,31}$'),
  build             text        NOT NULL CHECK (build ~ '^(0|[1-9a-f][0-9a-f]*)$'),
  route             jsonb       NOT NULL CHECK (jsonb_typeof(route) = 'object'),
  small_call_route  jsonb       CHECK (small_call_route IS NULL OR jsonb_typeof(small_call_route) = 'object'),
  working_folder    text        NOT NULL CHECK (working_folder LIKE '/%'),
  accepted_at       timestamptz NOT NULL,
  ended_at          timestamptz,
  replaced_by       uuid        REFERENCES exchanges (id) ON DELETE RESTRICT,
  UNIQUE (chat_id, number),
  CHECK ((state = 'open') = (ended_at IS NULL)),
  CHECK ((state = 'failed') = (error IS NOT NULL)),
  CHECK (replaced_by IS NULL OR (replaced_by <> id AND state <> 'open'))
);
CREATE UNIQUE INDEX IF NOT EXISTS exchanges_one_open_per_chat ON exchanges (chat_id) WHERE state = 'open';
COMMENT ON TABLE exchanges IS 'One row for each accepted Chat Exchange. Written by the Trunk that accepted it, or by the Trunk Build that settles it.';
COMMENT ON INDEX exchanges_one_open_per_chat IS 'At most one open Exchange in a chat: a second message to a busy chat is refused with chat_busy.';
COMMENT ON COLUMN exchanges.id IS 'The Exchange''s identifier, a UUID, the same as the name of its folder, for example 7f3a2c1e-4b5d-4e6f-8a9b-0c1d2e3f4a5b.';
COMMENT ON COLUMN exchanges.chat_id IS 'The chat the Exchange belongs to.';
COMMENT ON COLUMN exchanges.number IS 'The Exchange''s place in its chat: 1 for the first, then 2, 3 and so on, including replaced Exchanges. Two messages to one chat at the same moment cannot both take the same number, so one of them is refused with chat_busy.';
COMMENT ON COLUMN exchanges.state IS 'open from acceptance until the end; then completed, failed, cancelled, or interrupted when its process died before it ended. An ended Exchange never changes state again.';
COMMENT ON COLUMN exchanges.error IS 'Why it failed, present exactly when state is failed: the Driver''s error object as the Driver wrote it (DRV-4), for example {"code": "rate_limited", "message": "…", "retryable": true}, or {"code", "message"} with a code of the Root Contract, such as events_file_full.';
COMMENT ON COLUMN exchanges.trunk IS 'The code name of the Trunk that accepted it, for example trivial. Never changes.';
COMMENT ON COLUMN exchanges.build IS 'The build number of the Trunk Build that accepted it, for example 3; 0 for a program outside a Build, as in development (BLD-0). Never changes.';
COMMENT ON COLUMN exchanges.route IS 'The route the Trunk was handed (ARC-0), for example {"driver": "claude", "subscription": "anthropic", "model": "sonnet", "effort": "high"}; only what was handed, so a Trunk started by hand may have fewer fields. What each run used is in runs.started.';
COMMENT ON COLUMN exchanges.small_call_route IS 'The small-call route the Trunk was handed, in the same form as route; NULL when none was handed, and then the Trunk made no small text call.';
COMMENT ON COLUMN exchanges.working_folder IS 'The absolute folder the agent worked in, for example /Users/sam/repos/shop.';
COMMENT ON COLUMN exchanges.accepted_at IS 'When the Trunk accepted the message.';
COMMENT ON COLUMN exchanges.ended_at IS 'When the Exchange ended; NULL exactly while it is open. For an interrupted Exchange, the time of the last line of its events.jsonl that carries one (the at of an event or a smallRun), or accepted_at when the file had none; for an Exchange that a migration wrote as interrupted, the time of the migration snapshot (RTM-1).';
COMMENT ON COLUMN exchanges.replaced_by IS 'When the Leather Bag sent the message again: the Exchange that runs it again. A replaced Exchange is never shown in the chat and never sent to an agent. Written once, by the Trunk that accepts the new Exchange, which in the same transaction deletes the replaced Exchange''s messages and tool calls; its runs stay, so its tokens still count.';

CREATE TABLE IF NOT EXISTS messages (
  exchange_id  uuid    NOT NULL REFERENCES exchanges (id) ON DELETE RESTRICT,
  position     integer NOT NULL CHECK (position >= 0),
  role         text    NOT NULL CHECK (role IN ('user', 'assistant')),
  text         text    NOT NULL,
  PRIMARY KEY (exchange_id, position),
  CHECK ((position = 0) = (role = 'user'))
);
COMMENT ON TABLE messages IS 'The Leather Bag''s message and the agent''s answer, one row for each block of text. Written by the Trunk, or by the Trunk Build that settles the Exchange.';
COMMENT ON COLUMN messages.exchange_id IS 'The Exchange the message belongs to.';
COMMENT ON COLUMN messages.position IS 'Where the message stands in its Exchange: 0 for the Leather Bag''s message; for a block of the answer, the number n of the line of events.jsonl that carried it, for example 17.';
COMMENT ON COLUMN messages.role IS 'user for the Leather Bag''s message, assistant for a block of the answer.';
COMMENT ON COLUMN messages.text IS 'The text exactly as it was sent, or exactly as the Driver''s text event carried it (DRV-2), except that the character U+0000 and every half of a surrogate pair that stands alone are replaced with U+FFFD (RCT-3).';

CREATE TABLE IF NOT EXISTS tool_calls (
  exchange_id     uuid    NOT NULL REFERENCES exchanges (id) ON DELETE RESTRICT,
  position        integer NOT NULL CHECK (position >= 1),
  run_id          text    NOT NULL,
  call_id         text    NOT NULL,
  parent_call_id  text,
  name            text    NOT NULL,
  input           jsonb   NOT NULL,
  ok              boolean NOT NULL,
  result          jsonb,
  error           text,
  truncated       boolean NOT NULL DEFAULT false,
  interrupted     boolean NOT NULL DEFAULT false,
  ask             jsonb,
  answer          jsonb,
  PRIMARY KEY (exchange_id, position)
);
COMMENT ON TABLE tool_calls IS 'One row for each tool call the agent made, with its result; a question to the Leather Bag is a tool call too. Written by the Trunk when the call has finished, or by the Trunk Build that settles the Exchange.';
COMMENT ON COLUMN tool_calls.exchange_id IS 'The Exchange the call belongs to.';
COMMENT ON COLUMN tool_calls.position IS 'Where the call stands in its Exchange: the number n of the line of events.jsonl that carried its tool_started event, for example 18.';
COMMENT ON COLUMN tool_calls.run_id IS 'The Driver run that made the call (DRV-2 runId).';
COMMENT ON COLUMN tool_calls.call_id IS 'The call''s identifier within its run, as the Driver gave it (DRV-2 callId).';
COMMENT ON COLUMN tool_calls.parent_call_id IS 'For a call a sub-agent made: the call that started the sub-agent (DRV-3 parentCallId); NULL for the agent''s own calls.';
COMMENT ON COLUMN tool_calls.name IS 'The tool''s name, for example shell or Read (DRV-3).';
COMMENT ON COLUMN tool_calls.input IS 'The input as the Driver''s tool_started event carried it, for example {"command": "ls"}.';
COMMENT ON COLUMN tool_calls.ok IS 'As tool_finished said: true when the call succeeded. false for a call that the end of its Exchange cut off before its tool_finished.';
COMMENT ON COLUMN tool_calls.result IS 'The result as the Driver''s tool_finished event carried it, for example {"output": "build.sh\n", "exitCode": 0}; NULL when the event had none.';
COMMENT ON COLUMN tool_calls.error IS 'The error text of a failed call, as tool_finished carried it; NULL when it had none.';
COMMENT ON COLUMN tool_calls.truncated IS 'true when the Driver cut the result to its size limit (DRV-2).';
COMMENT ON COLUMN tool_calls.interrupted IS 'true when the call was cut off: as tool_finished said after a stop, or because the Exchange ended before the call finished.';
COMMENT ON COLUMN tool_calls.ask IS 'For a question the agent asked the Leather Bag: the question as the ask_opened event carried it (DRV-3), for example {"callId": "call_7", "questions": [{"id": "branch", "question": "Which branch should I push to?", "multiple": false, "options": [{"label": "main"}, {"label": "dev"}]}]}; NULL for any other call.';
COMMENT ON COLUMN tool_calls.answer IS 'For a question: the Leather Bag''s answer as the answer line carried it (DRV-3), for example {"type": "answers", "answers": {"branch": ["main"]}} or {"type": "decline"}; NULL for any other call, and for a question that was withdrawn before it was answered.';

CREATE TABLE IF NOT EXISTS runs (
  exchange_id  uuid        NOT NULL REFERENCES exchanges (id) ON DELETE RESTRICT,
  run_id       text        NOT NULL,
  kind         text        NOT NULL DEFAULT 'agent' CHECK (kind IN ('agent', 'small')),
  outcome      text        NOT NULL CHECK (outcome IN ('completed', 'failed', 'cancelled', 'interrupted')),
  usage        jsonb       NOT NULL DEFAULT '{}'::jsonb CHECK (jsonb_typeof(usage) = 'object'),
  error        jsonb,
  started      jsonb,
  ended_at     timestamptz NOT NULL,
  PRIMARY KEY (exchange_id, run_id),
  CHECK ((outcome = 'failed') = (error IS NOT NULL))
);
COMMENT ON TABLE runs IS 'One row for each Driver run of the Exchange, the agent''s runs and the runs of its small calls, with its token usage. Written by the Trunk when the run has ended, or by the Trunk Build that settles the Exchange.';
COMMENT ON COLUMN runs.exchange_id IS 'The Exchange the run belongs to.';
COMMENT ON COLUMN runs.run_id IS 'The run''s identifier, as the Trunk gave it to the Driver (DRV-2 runId).';
COMMENT ON COLUMN runs.kind IS 'agent for a run of the agent''s work; small for a small call, such as a title, made on the small-call route (ARC-0).';
COMMENT ON COLUMN runs.outcome IS 'completed, failed or cancelled, as the Driver''s finished event said; interrupted when no finished event reached the Trunk.';
COMMENT ON COLUMN runs.usage IS 'The run''s token and cost totals in the Driver''s form (DRV-2 RunUsage), for example {"model": "sonnet", "inputTokens": 5120, "outputTokens": 812}; {} when the Driver reported none.';
COMMENT ON COLUMN runs.error IS 'The Driver''s error object of a failed run (DRV-4); present exactly when outcome is failed.';
COMMENT ON COLUMN runs.started IS 'The run''s started event as the Driver wrote it, which names the Subscription, the model, the effort and the Driver Build the run used (DRV-2); NULL for a small call, and when no started event reached the Trunk.';
COMMENT ON COLUMN runs.ended_at IS 'The time of the run''s finished event, or of its last event when it had none.';
```

The file was run twice on PostgreSQL 17.10 on 15 October 2026, in a throwaway server with no network: the second run changed nothing, and every table and column had its comment. On the same server a second open Exchange in one chat was refused by `exchanges_one_open_per_chat`, a second Exchange with the same number in its chat by the key on `(chat_id, number)`, and a failed Exchange without a cause, a run of an unknown kind and the deletion of a chat that has Exchanges were refused. A U+0000 and a lone half of a surrogate pair were refused with errors of the SQLSTATE class 22, and U+FFFD in their place was stored. The reader's query left out the replaced Exchange and the sub-agent's call. The installed server of ARC-0 was not used.

## Known problems in version 1

Not to be reproduced.

- **No row for an Exchange.** An Exchange was only an identifier on message rows and events in a log. "Running" was a column of the chat, set and cleared by events, and stayed set for ever when the end could not be recorded.
- **Empty identifiers.** The Exchange's identifier on a message could be NULL, and the older writer never filled it; after a change of how chats were drawn, every older chat showed nothing until a migration filled it in.
- **Tool calls keyed by the vendor's identifier.** It is unique only within one run, so saving the result of one call could change another chat's row, until a migration changed the key and emptied the table.
- **Usage without its Exchange.** Usage rows named no Exchange, none was written for a cancelled Exchange, and none without a bound Subscription.
- **JSON in text columns, and four copies of the transcript.** Mixed keys in text columns broke later reads, and each copy of the transcript needed its own repair.
- **Deletes that cascade.** Deleting a chat deleted its messages, folders and tool rows with it.
- **A placeholder as the title.** A window stored `Elma · <repo>` as the title, and the title job ran only for chats without one, so no title was ever made for them.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:daemons/core/arbol_core/projections/chat_session_list.py#_STATUS`
- `arbol_deprecated_v1:daemons/shared/arbol_shared/pg.py#_MIGRATION_V72_MESSAGE_TURN_BACKFILL`
- `arbol_deprecated_v1:daemons/shared/arbol_shared/pg.py#_MIGRATION_V103_NATIVE_TOOL_SCOPE`
- `arbol_deprecated_v1:daemons/shared/arbol_shared/pg.py#_BASELINE_V1`
- `arbol_deprecated_v1:product-spec/03-turn-execution.md#Usage accounting`
- `arbol_deprecated_v1:product-spec/02-chat-sessions.md#Known problems and lessons`
- `arbol_deprecated_v1:product-spec/12-lessons-and-invariants.md#3. Durable bookkeeping on the streaming path`

## Decisions

| ID | Question | Proposed default | Outcome |
|---|---|---|---|
| RCT-D21 | What does a chat keep? | Every message, every tool call with its result, and the token usage of each run. Branches and edited messages come later, not now | Decided 15 October 2026 by the owner: the proposed default. That "each run" includes the runs of small calls, such as a title, marked as such, that a question keeps the Leather Bag's answer beside it, and the five tables and their columns, are a technical consequence drawn the same day |
| RCT-D22 | In what form does the Root Contract give the table layout? | As a plain file, `layout.sql`, of `CREATE TABLE` statements, kept in the `arbol-root` repository, not as models of a library. Each program writes its own plain SQL. A Build that talks to the database from Python uses psycopg 3, pinned in that Build, and `arbol-roots` runs `layout.sql` and the migration scripts with `psql` | Decided 15 October 2026 by the owner: a plain `layout.sql` in the `arbol-root` repository, not ORM models, and each program writes its own SQL. Psycopg 3 and `psql` had been proposed before and not objected to: a technical choice made the same day. The place `contract/<version>/layout.sql` and statements that can run again are a technical choice made the same day too |
| RCT-D23 | Which values are columns of their own, and which are kept whole? | A column for whatever a reader filters, sorts or joins on, and the text of a message. What the Driver Contract defines, a tool call's input and result, a question and its answer, a run's usage and its `started` event, and an error, is kept whole as `jsonb`, and so is a route. Rows are keyed by the Exchange and their position in `events.jsonl`, so saving a line twice changes nothing. The alternative: a column for every field of the Driver's events, which every field a newer Driver adds would change | Decided 15 October 2026, a technical choice: the proposed default |
| RCT-D24 | How is a table or a column added to the chat tables of a version? | Not in contract version 1, which needs none. It is decided with the first addition, because an addition reaches every Root of the version, and a Trunk Build that writes it must not be chosen in a Root that lacks it. The alternative: the means now, such as dated additions that `arbol-roots` records in each Root and each Trunk Build lists, which version 1 would never use | Decided 15 October 2026, a technical choice: the proposed default |
| RCT-D25 | Must the chat tables say where a migrated chat came from? | No. The first thin path migrates a chat into a second Root (ARC-D39), and ARC-1 has each migrated chat record its Root, its chat and how far the original had got, but no Trunk writes or reads that record. It is kept in the Root's own layout, beside Arbol's own tables, so that a change to it never needs a new version of this contract. The alternative: a table in this layout, which would tie every Trunk to the bookkeeping of migrations | Decided 15 October 2026, a technical choice: the proposed default |
| RCT-D31 | PostgreSQL cannot store the character U+0000 or one half of a surrogate pair that stands alone, and a Driver can send both. What is saved? | U+FFFD (�) in their place, in message text and inside every `jsonb` value, and nothing else changed; a row the database still refuses is left out, and the Exchange fails with `trunk_failed`, so a chat is never held busy by one row. The alternative: keep the Driver's values in columns of the type `json`, which store both as sent, and the text of a message twice, which every reader would have to choose between | Decided 15 October 2026, a technical choice: the proposed default |
