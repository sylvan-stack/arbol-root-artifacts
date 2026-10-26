---
role: authored
---
# Root Contract

What every [Arbol Trunk](../../arbol/GLOSSARY.md#arbol-trunk) of a [Root](../../arbol/GLOSSARY.md#arbol-root) has to do, and what the Root gives every Trunk, written once. It is the contract a Root shares with all its Trunks: the command `arbol-trunk`, the folder of each [Chat Exchange](../../arbol/GLOSSARY.md#chat-exchange) while it runs, the chat tables in the Root's database, and a conformance suite that every Trunk Build passes. With it the Root manager `arbol-root` can start any Trunk and list, stop and settle any Exchange, and the Arbol backend can follow and show any Exchange, without knowing which Trunk ran it. For example, `arbol-trunk run "What does build.sh do?"` and `arbol-trunk advanced run "What does build.sh do?"` leave the same kind of folder and the same kind of rows behind, although two different Trunks answered.

The owner named the contract on 15 October 2026 and decided its points the same day (SYL-168, RCT-D1 to RCT-D6, RCT-D15, RCT-D21, RCT-D22). The owner answered the open questions on 15 October 2026 (RCT-D16, RCT-D28, RCT-D29 and RCT-D30), and the specifications were accepted the same day. It takes the place of the *Trunk Contract* that the architecture named until 15 October 2026 ([ARC-0](../../arbol/architecture/00-components-and-boundaries.md#what-any-trunk-must-offer), [ARC-2](../../arbol/architecture/02-chat-exchange-across-components.md), [ARC-3](../../arbol/architecture/03-stuck-trunks.md)). Two things lie outside it: how the Root manager works, in the [Root manager's specifications](../specs/README.md) (`RTM`), and what a Trunk does inside an Exchange, in each Trunk's own specifications, the first of which is the [Trivial Trunk's](../../arbol-trunk/trunks/trivial/specs/README.md) (`TRV`).

## Specifications

| Spec | Title | Status |
|---|---|---|
| [RCT-0](00-overview.md) | Overview | accepted |
| [RCT-1](01-the-exchange-folder.md) | The Exchange folder | accepted |
| [RCT-2](02-starting-stopping-and-settling.md) | Starting, stopping and settling | accepted |
| [RCT-3](03-the-chat-tables.md) | The chat tables | accepted |
| [RCT-4](04-conformance.md) | Conformance | implemented |

The table layout of contract version 1 is also a file of its own, `contract/1/layout.sql` in the `arbol-root` repository ([RCT-3](03-the-chat-tables.md#interface)).

## Where the architecture's requirements of a Trunk are carried

*Each Trunk's own specification* below means, for the first Trunk, the [Trivial Trunk's](../../arbol-trunk/trunks/trivial/specs/README.md) (`TRV`). Where the Arbol backend or a window carries a part, the row names the requirements of the [Arbol backend chapter](../../arbol/backend/README.md) (`BKD`). What the Root manager carries is in the [Root manager's README](../specs/README.md#where-the-architectures-requirements-of-the-root-manager-are-carried).

| Requirement | Carried by |
|---|---|
| ARC-0.18 | RCT-2.4, RCT-2.6; the route it was handed: each Trunk's own specification, for the Trivial Trunk TRV-0.1 and TRV-0.2 |
| ARC-0.19 | RCT-1.1, RCT-1.5, RCT-1.7 |
| ARC-0.20 | RCT-3.1, RCT-3.2 |
| ARC-0.21 | RCT-2.4, RCT-2.11 |
| ARC-0.22 | RCT-0.1 |
| ARC-0.24 | RCT-2.22, RCT-1.11; what a Trunk does with a title: each Trunk's own specification, for the Trivial Trunk TRV-0.8 |
| ARC-1.22 | For a Trunk Build: RCT-4.1, RCT-4.2 ([RCT-D29](04-conformance.md#decisions)); for a Root's own Builds: the [Root manager's README](../specs/README.md) |
| ARC-2.1 | RCT-2.1, RCT-2.2; keeping the Draft: BKD-0.10, BKD-0.12 |
| ARC-2.2 | RCT-1.5, RCT-1.6, RCT-1.12, RCT-3.2 |
| ARC-2.3 | RCT-3.2 |
| ARC-2.5 | RCT-1.7, RCT-1.8, RCT-2.7, RCT-2.16 |
| ARC-2.6 | RCT-2.9, RCT-2.20 |
| ARC-2.8 | RCT-2.17 and the index `exchanges_one_open_per_chat` of RCT-3; other chats go on because each Exchange runs in a process of its own (RCT-0), and the backend and the windows serve them meanwhile: BKD-0.9, BKD-0.14 |
| ARC-2.9 | RCT-2.2 (`database_unavailable`), RCT-2.11 |
| ARC-2.10 | RCT-2.10; for the Trivial Trunk TRV-0.2 |
| ARC-3.1 | RCT-1.3 |
| ARC-3.2 | RCT-1.3 |
| ARC-3.3 | RCT-1.4 |
| ARC-3.4 | RCT-2.21 (RCT-D30); for the Trivial Trunk TRV-0.10, TRV-0.11 |
| ARC-3.10 | RCT-1.4, RCT-2.21, both counted with `CLOCK_UPTIME_RAW`; for the Trivial Trunk TRV-0.10, TRV-0.11; the 20 and 5 seconds of a stop: RTM-0.12; the looks of ARC-3.9: BKD-0.19 |

## Conventions

The chapter follows the conventions of the [Driver Contract](../../arbol/driver-contract/README.md#conventions) and the additions of the [Architecture](../../arbol/architecture/README.md#conventions) (the checks *journey* and *drill*). Identifiers here start with `RCT`: `RCT-1.4` is requirement 4 of specification 1. Decisions are numbered once for the whole chapter, `RCT-D1` onwards.

**Verified by.** *contract* means here the Root Contract's own conformance suite ([RCT-4](04-conformance.md)). It runs against each Trunk Build with a stand-in Driver and a scratch database made from `layout.sql`, on a database server of its own, and needs no network, no account and no quota.

**Version.** This chapter states version 1 of the contract, called *contract version 1* where it could be mistaken for version 1 of Arbol. Only the Root Contract has version numbers; a Trunk has a code name, and its Builds are numbered by `arbol-builds` ([RCT-0](00-overview.md#versions-what-fits-together)).
