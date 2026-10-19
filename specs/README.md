---
role: authored
---
# Root manager specifications

What `arbol-root` does: it manages the [Arbol Trunks](../../arbol/GLOSSARY.md#arbol-trunk) of one [Root](../../arbol/GLOSSARY.md#arbol-root). It starts a Trunk for a new [Chat Exchange](../../arbol/GLOSSARY.md#chat-exchange), lists the Exchanges in progress, stops them, and has a Trunk [Build](../../arbol/GLOSSARY.md#build) save what is left of an Exchange whose process is gone. Its Builds also carry the Root's table layouts and migration script, which `arbol-roots` runs. For example, `arbol-root --root sapling list` prints every Exchange in progress in First Sapling, each with its heartbeat. The architecture says where the component stands ([ARC-0](../../arbol/architecture/00-components-and-boundaries.md#arbol-root), "Arbol Root"; [ARC-1](../../arbol/architecture/01-roots.md)). The [Root Contract](../contract/README.md) says what the Root manager shares with every Trunk. The specifications here say what its commands do and print, and what its Builds carry. The owner answered the open questions on 15 October 2026, and the specifications were accepted the same day.

Conventions (shape of a specification, requirement identifiers, origin, checks, status, decisions) are defined once in the [Driver Contract's README](../../arbol/driver-contract/README.md#conventions), with the checks *journey* and *drill* of the [Architecture](../../arbol/architecture/README.md#conventions). Identifiers here start with `RTM`: `RTM-0.4` is requirement 4 of specification 0. Decisions are numbered once for the whole chapter, `RTM-D1` onwards.

**Verified by.** *unit* means a check of the kind `unit` of an `arbol-root` Build. A Build has three: `unit`, which runs the program's tests in its own process and its commands against a stand-in `arbol-trunk`; and `layout` and `migration`, which run the Build's SQL files on a database server of their own ([RTM-1](01-builds-layouts-and-migration.md#the-checks-of-a-build)). A row that needs one of the last two names it, as in `unit: layout`.

| Spec | Title | Status |
|---|---|---|
| [RTM-0](00-the-commands.md) | The commands | accepted |
| [RTM-1](01-builds-layouts-and-migration.md) | Builds, layouts and migration | accepted |

## Where the architecture's requirements of the Root manager are carried

Where the Arbol backend or a window carries a part, the row names the requirements of the [Arbol backend chapter](../../arbol/backend/README.md) (`BKD`), and for a part of Arbol not yet specified, its task.

| Requirement | Carried by |
|---|---|
| ARC-0.15 | RTM-0.2, RTM-0.6 |
| ARC-0.16 | RTM-0.4; RCT-0.7, RCT-0.9 |
| ARC-0.17 | RTM-0.2; RTM-1.1 (the Build carries SQL that only `arbol-roots` runs) |
| ARC-0.27 | RTM-0.3; RTM-0.6, RTM-0.9 and RTM-0.13 work from the records, which are alike for every Trunk |
| ARC-0.28 | RTM-0.1, and `rootBuild` and `rootPackageVersion` in the listing (RTM-0.6) |
| ARC-1.2 | RTM-1.5 for the Root's own tables; RCT-D24 for the chat tables; the backend's own tables and settings: BKD-0.7; the Bridge: BKD-1.9 |
| ARC-1.15 | RTM-1.7 |
| ARC-1.16 | RTM-1.6 and RTM-1.8 for the whole database; one chat: not in the first thin path ([RTM-D12](01-builds-layouts-and-migration.md#decisions)), left for a later task, in the migration script of RTM-1 |
| ARC-1.17 | RTM-1.9, RTM-1.10; converting the rows of a table that changed waits for the first Root that changes one, because the second, trivial Root changes none (RTM-1) |
| ARC-1.18 | RTM-1.7, RTM-1.8, and `arbol-roots migrate` only while the Root is `preparing` (RTM-1); a Root that is `preparing` in the header of its window: BKD-2.1; keeping it `preparing` and reporting the failure in the tray menu: the Arbol Swift App (SYL-192) |
| ARC-1.19 | Not in the first thin path (RTM-D12), left for a later task: the migration script of RTM-1 and a mark in the Root's own layout; RTM-1.11 keeps what the check compares; when the backend runs the check: SYL-195 |
| ARC-1.20 | Not in the first thin path (RTM-D12), left for a later task, in the migration script of RTM-1 |
| ARC-1.24 | RTM-0.3; RCT-0.7, RCT-0.10 |
| ARC-1.25 | RTM-0.6, RTM-0.7 give a listing that can be combined; asking every Root and reporting a failing one: BKD-0.22 |
| ARC-1.26 | RTM-0.1; RCT-1.2; restarts of the backend: BKD-0.21, BKD-0.22; of the App: SYL-192 |
| ARC-1.27 | RTM-1.7, RTM-1.10 |
| ARC-1.28 | RTM-1.1, RTM-1.4, RTM-1.5; RCT-3.1, RCT-3.6; running the files: `arbol-roots` (RTM-1); the backend creates or changes no table: BKD-0.6 |
| ARC-1.29 | RTM-1.1, RTM-1.6; RTM-0.2 (the program runs none of them) |
| ARC-2.7 | RTM-0.6, RTM-0.13; RCT-2.12; showing the Exchange: BKD-0.18, BKD-2.16 |
| ARC-2.12 | RTM-1.10, for the migration: it starts nothing and leaves the original Exchange with its Root; RCT-1.2, RCT-3.4 for the rest |
| ARC-3.5 | RTM-0.6 |
| ARC-3.6 | RTM-0.9, RTM-0.10, RTM-0.12; RCT-2.15 |
| ARC-3.7 | RTM-0.10, RTM-0.12; RCT-2.8, RCT-2.15 |
| ARC-3.8 | RTM-0.11 |
| ARC-3.10 | RTM-0.12; RCT-1.4, RCT-2.21; the looks of ARC-3.9: BKD-0.19 |

Tests of a Root's own Builds (ARC-1.22) are carried by RTM-0.15 and RTM-1.3.
