# WORKS — Full Overview

> A compound standard for the deliberate storage, transformation, and authorized output
> of business information. Works is not software. It is a protocol — a specification of
> buildings, conveyance rules, actor types, and constitutional guarantees that any
> conforming implementation must honor. Software is built to the standard; the standard
> outlasts any particular implementation.

---

## 1. What Works Is

Works is an industrial compound. The metaphor is intentional and load-bearing: a works
(in the Victorian sense — a textile works, an ironworks, a gasworks) is a complete
operational site where raw materials arrive, move through specialized processes, and
leave as finished goods. The physical metaphor carries structural weight:

- **Raw materials arrive** at a defined intake point, are verified and classified
- **Materials are held** in named stores with addressed locations
- **Transformation happens** in mills, where inputs become outputs through defined operations
- **Finished goods are authorized** for departure through a gate
- **The whole site runs** under a set of operational rules enforced by an office
- **A ledger records** everything that entered, moved, and left — permanently

Works applies this model to business information: tasks, time records, journal entries,
financial transactions, documents, agent outputs, API responses, and any other chunk of
operational content that a business produces or consumes.

**The core claim**: the same structural guarantees that made Victorian industrial compounds
reliable — physical containment, specialized roles, gated departure, permanent ledger —
apply to information processing. Works enforces them as a protocol, not a suggestion.

---

## 2. Architectural Lineage

Works is grounded in three historical architectural models, each contributing a distinct
structural layer:

### Von Neumann (1945) — Processing Architecture

Works maps directly to the von Neumann stored-program computer (see `von-neumann-works.md`
for the full diagram). The correspondence:

```
Von Neumann          Works
─────────────────    ──────────────────────────────────────────
Memory Unit      →   Stores  (named, addressed chunk repositories)
Stored Program   →   Barrels (instruction sequences stored in Stores)
Control Unit     →   Barrel  (fetch, sequence, cycle, policy-check)
ALU              →   Mill    (transform, combine, run lens pipeline)
Registers        →   Bench   (session workspace, staging area)
Input Unit       →   Intake  (receive, verify, classify, route)
Output Unit      →   Dispatch (publish, write, broadcast)
System Bus       →   Stream  (append-only log + real-time bus + replay)
```

Works extends von Neumann in three structural directions that VN had no need for:
**Gate** (output authorization — nothing leaves without approval), **Office** (policy
authority and audit log), and **Actors** (first-class operators — Human, Agent,
Automated, External — with session scope and bounded capabilities).

The von Neumann bus bottleneck — all data must cross a single shared bus — is accepted
and made constitutional in Works. Stream is the bottleneck by design. Every building
action crosses Stream. This costs performance and buys auditability, replay, and
policy enforcement as structural properties rather than optional features.

### Babbage's Analytical Engine (1830s–1871) — Computation Model

Charles Babbage's Analytical Engine introduced the stored-program concept, conditional
branching, and a separation between the Store (memory) and the Mill (computation) that
von Neumann later formalized. Works borrows directly:

- **Store as Row × Level × Bin address space**: each Store is a three-dimensional
  coordinate system. Row is a unique object identifier. Level is the lifecycle stage
  (Raw → Staged → Integrated). Bin is the field classifier (T=Task, F=Time, B=Interval,
  A=Annotation, etc.). This gives chunks a precise, human-memorable address.

- **Mill as ingress / operation / egress**: Babbage's Mill had explicit input registers,
  an operational core (addition, subtraction, multiplication, division), and output
  registers. Works' Mill follows this: chunks arrive at ingress, transformations run
  in operation, outputs collect at egress and await Gate authorization.

- **Barrel as stored program**: Babbage's Barrel drums carried punched instruction
  sequences that controlled the Mill's operations. Works' Barrel is the same: a named
  instruction sequence stored in a Store, fetched by the Barrel scheduler, executed
  against Mill. Programs live in memory. This is the stored-program concept, first.

- **Difference Engine finite-difference cascade**: Babbage's Difference Engine No. 2
  computes polynomial approximations through pure addition cascades — no multiplication
  required. Works uses this as the computational substrate for continuous signal lenses
  (Dey, Felt): `advance() → for i in rev(1..n) { reg[i-1] += reg[i] }; reg[0]`.
  A pure addition cascade computing arbitrarily smooth time-series from raw event data.

### Ada Lovelace's Notes (1843) — Symbolic and Cyclic Operations

Lovelace's annotations on the Analytical Engine, particularly Notes A through G,
establish two principles Works inherits:

- **Symbolic generality**: the engine operates on symbols, not just numbers. Lovelace
  demonstrated that a computational engine can manipulate any symbolic domain. Works'
  Op enum (T, F, B, D, H, S, A) encodes operational semantics symbolically —
  Hollerith encoding, not schema-dependent parsing.

- **Cyclic execution (Note G — the first program)**: Lovelace's program for computing
  Bernoulli numbers introduced the concept of a loop — bounded repetition with a
  counter. Works' Barrel CycleMode inherits this directly:
  `Single | Bounded{from, to} | Perpetual{interval_ms}`.
  A Barrel can run once, run a bounded number of times, or run perpetually on a schedule.

---

## 3. The Buildings

Works defines nine core buildings. These are not software components — they are roles
in a protocol. Any conforming implementation must provide these roles; how they are
implemented is outside the standard's scope.

### Intake
The entry point for all external data. Intake receives raw materials and is responsible
for verification, classification, and routing before anything enters the Works proper.

Intake must:
- Verify integrity of incoming data (CRC-15 for BitPads records; format validation for others)
- Classify data by type (Task, Time, Journal, Ledger, Document, Agent output, etc.)
- Normalize to Works' internal chunk format before routing
- Route to named Store(s) or directly to a waiting Mill
- Emit an I event to Stream for every received item, whether accepted or rejected
- Never route unverified data to a Store or Mill

Intake supports the **Manifest**: a BitPads Rich Record containing a compound package
of Task + Time + Journal + Ledger data in a single verified unit. Intake can route the
Manifest intact (preserving compound provenance) or decompose it into constituent chunks
for separate routing. The decision is configuration, not protocol — both are valid.

Works defines specialized Intake subtypes for common sources: task adapter, timew
adapter, jrnl adapter, hledger adapter, BitPads Intake, API Intake, Agent Intake.
New Intake subtypes may be registered via the extension protocol (see §6).

### Store
Named chunk repositories. A Works installation has multiple Stores, each with a short
serial or human-arbitrary name (S-01, S-legal, S-api, S-finance, main, scratch, etc.).

Each Store is addressed as **Row × Level × Bin**:
- **Row**: a unique object identifier (UUID or human-assigned name) — identifies what the chunk is *about*
- **Level**: lifecycle stage — Raw (as received), Staged (assembled, awaiting authorization), Integrated (authorized, complete)
- **Bin**: field classifier — the Op-code-level type of the chunk's content

Chunks in a Store are immutable once written. Stage transitions (Raw → Staged,
Staged → Integrated) are recorded as new rows, not overwrites — the Pacioli guarantee
at the chunk level. A Store can never delete a chunk; it can only add new versions.

Store operations: `put(row, level, bin, chunk)`, `get(row, level, bin)`,
`list(level)`, `history(row)`, `address(row)` → Store address.

### Mill
The transformation site. A Mill takes chunks from one or more Stores (or from Bench),
applies a defined sequence of operations, and produces output chunks at egress.
Output chunks wait at egress until Gate authorizes their next destination.

Mill operations are organized as a **lens pipeline**, borrowed from WWSS:

```
Tier 0  Substrate:  Hollerith (encoding), Pacioli (guarantee)
Tier A  Event:      Burroughs (raw log), Baldwin (state diff)
Tier B  Structure:  Frick (transitions), Bundy (intervals)
Tier C  Signal:     Grant (metrics), Felt (density), Dey (continuous)
Tier D  Field:      Cooper (geometric projection)
```

A Mill does not need to run all tiers. A Mill configured for document assembly might
use only ingress + Burroughs + egress. A Mill configured for time analytics runs
through Bundy → Grant → Dey → Cooper.

Two classes of Mill:
- **Stream Mill**: reads from Stream log; read-only; no Gate needed for output (output is a view, not a new artifact)
- **Content Mill**: reads from Stores; produces new chunks; Gate authorization required before output leaves egress

### Bench
The session workspace. Bench holds active chunks that an Actor (Human or Agent) is
currently working with — drafts, intermediates, fragments pulled from multiple Stores
for comparison or assembly.

Bench is ephemeral: contents are cleared when the session ends. Bench is not a Store;
chunks on Bench have not been committed to Works state. An Actor moving a chunk from
Bench to a Store is a deliberate commitment act, recorded on Stream.

Bench supports the **Right of Access** — any authorized Actor can pull any chunk from
any Store onto their Bench for inspection or use, subject to Office capability bounds.

### Barrel
The instruction scheduler and control sequence. A Barrel is a named list of operations —
put this in this Store, run this Mill, send this to Gate, emit this event — stored in a
Store and fetched for execution by the Barrel scheduler.

Barrels make Works programmable. A recurring payroll run, a nightly export, a triggered
document assembly pipeline, an agent handoff sequence — all are Barrels. The Barrel
scheduler is the Control Unit (von Neumann). The Barrel itself is the stored program
(Babbage).

Barrel CycleModes:
- **Single**: run once, then done
- **Bounded{from, to}**: run on a schedule between two timestamps
- **Perpetual{interval_ms}**: run continuously on a fixed interval

Barrels are versioned in the Store. An old Barrel version is never deleted (Pacioli
guarantee). A Barrel can be disabled (emitting an S disable event) but not erased.

### Gate
The authorization boundary between Mill output and Dispatch. **Nothing leaves Works
without passing Gate.** This is the Right to Assembly: only a human Actor or an
authorized Barrel can approve the transition from Mill egress to Dispatch.

Gate operations:
- **Approve**: output proceeds to Dispatch; G approve event emitted to Stream
- **Reject**: output returned to Mill egress with reason; G reject event emitted
- **Hold**: output held at Gate pending further action; G hold event emitted
- **Reroute**: output redirected to a different Store or Mill instead of Dispatch
- **Require confirmation**: Gate holds and sends notification to a human Actor

Gate instructions use the BitPads Task block protocol (8-bit packed instruction) —
compact, wire-efficient, structurally auditable. Every Gate decision is on Stream.
Office receives all G events; can veto a Gate approval or escalate a Gate hold.

Gate is the primary enforcement point for the Pacioli guarantee at the output level:
because every output crosses Gate, and Gate emits to Stream, and Stream is
append-only, the complete output history of a Works installation is permanently
recoverable from Stream replay.

### Dispatch
The output building. Dispatch receives Gate-authorized chunks and sends them to their
destination:

- File write (local filesystem, network share)
- API call (POST to external service, webhook)
- Publish (message queue, event bus, email)
- Broadcast (all subscribed Actors notified)
- BitPads output (compact wire record to external Works or BitLedger installation)
- Human-readable render (formatted document, report, dashboard)
- Agent handoff packet (structured context bundle for AI agent session)

Dispatch cannot receive input from anywhere except Gate. It has no Store, no Mill,
no direct Actor access. It is a one-way output valve.

Dispatch emits D events to Stream for every completed output. If Dispatch fails
(network error, API rejection), it emits a D fail event and returns the chunk to
Gate with an error context. Gate decides what to do next.

### Office
The constitutional authority for the Works compound. Office enforces the rules that
all other buildings must follow. Office receives every policy-class event from Stream
and has the authority to veto, escalate, or record.

Office responsibilities:
- **Building registry**: new building types must register with Office before they can
  participate in Works conveyance (the extension protocol, §6)
- **Actor capability tokens**: Office issues and revokes capability tokens that determine
  what each Actor can do (read a Store, put to a Store, trigger a Mill, approve at Gate)
- **Policy publication**: Office publishes the Works constitution — the set of rules
  that define what is permitted in this installation
- **Audit log**: Office maintains its own append-only log, separate from Stream,
  recording every policy decision. This log is the final authority for compliance queries
- **Conservation reconciliation**: Office reconciles BitLedger conservation invariants
  against Store state — ensuring that financial data entering Works sums correctly

Office is the only building with write access to the policy layer. All other buildings
are policy-takers, not policy-setters.

### Vault
The permanent archive. Committed bundles, completed Mill outputs, authorized Dispatch
records, and closed Barrel runs are archived to Vault. Vault is append-only at the
compound level — the Pacioli guarantee extended to the entire Works output history.

Vault is not a Store. Chunks in Vault cannot be retrieved for modification; they can
only be read. A chunk moving from a Store to Vault is a one-way transition, recorded
on Stream, authorized by Gate.

Vault supports compliance, audit, and recovery. If Stream is replayed from the beginning,
Vault state can be fully reconstructed — this is the Pacioli guarantee in its strongest
form: truth is the cumulative record.

---

## 4. The Stream — Constitutional Layer

Stream is not a feature of Works. Stream is the Works constitutional layer — the medium
through which all buildings communicate and the mechanism that makes Works guarantees
structural rather than policy-dependent.

Stream has three operational layers:

**Log layer** (Pacioli guarantee):
Append-only event file. Every building action that crosses Stream is permanently
recorded. Lines are never modified or deleted. State is reconstructed by replaying
the log from the beginning. The log is the truth; all other state is derived.

Event format (Hollerith encoding — positionally fixed):
```
<unix_ts> <op> <action> <object> <ctx_json>
col 1     col2  col3     col4     col5 (optional minified JSON)
```

Op codes:
```
T  Task       task lifecycle (add, modify, done, delete)
F  Frick      state transition (start, stop, context switch)
B  Bundy      interval boundary (clock-in, clock-out)
D  Dey        signal sample (quality, energy, score)
H  Hollerith  encoding marker (schema version, field map)
S  System     system/meta event (sync, reset, agent handoff)
A  Annotation journal entry or note attached to object
I  Intake     arrival event (received, verified, routed, rejected)
G  Gate       authorization event (approve, reject, hold, reroute)
```

**Bus layer** (real-time delivery):
WebSocket or Unix socket pub/sub. Any building or Actor can subscribe to Stream
events in real time. Subscriptions can be filtered by op code, object, profile,
or project. The bus layer does not store events — that is the Log layer's job.

**Replay layer** (recovery):
Any subscriber can request events from a given timestamp forward, catching up to
current state after a disconnect or cold start. Replay is the recovery mechanism
for the entire Works installation: if any building's internal state is lost, it
replays Stream from the point of last known state.

**Stream as the von Neumann bottleneck, reframed**:
Von Neumann's single shared bus limits throughput. Works keeps this constraint
and makes it constitutional. The cost — all events cross Stream — purchases three
guarantees: every event is auditable (Office), every state is recoverable (Replay),
and every policy rule is enforceable (Bus filter). These are not features that can
be turned off. They are consequences of the architecture.

---

## 5. Actors

Works defines four Actor types, each with distinct structural properties:

**Human** `{name, session_id, capability_token}`
A person interacting with Works. Humans have the highest default authority — they
can approve at Gate, pull any chunk to Bench, trigger any Barrel, and override
automated decisions within their capability bound. Human sessions are finite;
Bench is cleared on session end.

**Agent** `{name, model, session_id, capability_token, context_limit}`
An AI agent (LLM-based or otherwise) interacting with Works within a session.
Agents have a context limit — the amount of Stream history or Store content they
can hold in working memory. Agents can read Stores, put chunks to Stores, trigger
Mills and Barrels, but cannot self-approve at Gate (by default — Office can grant
this capability explicitly). An Agent is always a named, identified entity;
anonymous agent actions are not permitted.

**Automated** `{barrel_name, trigger, capability_token}`
A Barrel running autonomously — cron-triggered or event-triggered — with no human
in the loop for that execution. Automated Actors have the narrowest capability tokens
by default. They can run their defined Barrel operations but cannot escalate their
own permissions. If an Automated Actor needs Gate approval, it holds and notifies
a Human Actor.

**External** `{system_id, protocol, capability_token}`
An external system — another Works installation, a BitLedger node, an API, a legacy
tool — interacting with Works through Intake or Dispatch. External Actors can only
enter through Intake (verified and classified) or receive through Dispatch (Gate-
authorized). They cannot directly access Stores, Mills, or Barrels.

All Actor types are session-scoped. Every action an Actor takes is recorded on Stream
with the Actor's identity. There are no anonymous actions in Works.

---

## 6. Chunks and Conveyance

A **chunk** is the atomic unit of content in Works. Every piece of information that
enters, moves through, or leaves Works is a chunk.

Chunk anatomy:
```
{
  id:        unique identifier (UUID or hash)
  src:       source adapter or building name
  type:      op-code classification (T, F, B, A, D, ...)
  content:   the actual data (text, JSON, binary, structured record)
  address:   Store address {store, row, level, bin} — null if on Bench
  provenance: [list of prior addresses and actions — full history]
  ts_created: unix timestamp
  ts_modified: unix timestamp of last stage transition
  actor:     Actor who last touched this chunk
  manifest_id: if this chunk is part of a Manifest, its compound id
}
```

**Conveyance** is the act of moving a chunk between buildings. All conveyance crosses
Stream — an event is emitted for every chunk movement. Conveyance rules:

- Chunks move from Intake → Store or Intake → Mill (never Intake → Dispatch)
- Chunks move from Store → Bench (Actor pull) or Store → Mill (Barrel invocation)
- Chunks move from Mill → Bench (intermediate output) or Mill → Gate egress
- Chunks move from Gate → Dispatch (approved) or Gate → Store (rerouted) or Gate → Mill (returned for modification)
- Chunks move from Dispatch → Vault (after successful output)
- Chunks never move backward in the authorized path without a G reject or G reroute event

**The Manifest** is a compound chunk — a BitPads Rich Record containing multiple
chunk types in a single verified unit. A Manifest has a category marker (BitPads
0110 = Rich Log Entry: Value + Time + Task + Note) and a compound mode marker
(BitPads 1111 = atomic — all fields present and verified). Intake receives a
Manifest as a single unit; routing it intact preserves compound provenance.

---

## 7. The Extension Protocol

Works is a living standard. New building types — Stores, Mills, Intakes, and novel
roles not yet imagined — can emerge and register with Office without changing the
core standard. The extension protocol defines how this happens.

A new building type must:

1. **Declare** its type name, role (input/storage/transformation/output/control/audit),
   and the Op code(s) it emits to Stream
2. **Register** with Office, providing its declaration and a conformance statement
3. **Emit** to Stream for every significant action — no building type may operate
   silently (this is the constitutional requirement, not a suggestion)
4. **Honor** the Gate boundary — if the building produces output for external consumption,
   that output must pass Gate
5. **Honor** the Pacioli guarantee — if the building has internal state, that state
   must be recoverable from Stream replay
6. **Receive** a registration token from Office — only registered buildings may
   participate in Works conveyance

Buildings that do not register are External Actors. They can interact with Works
through Intake and Dispatch, but they cannot directly access Stores, Mills, or Barrels.

The extension protocol is the mechanism by which Works evolves without breaking
conforming implementations. Core buildings are defined once and frozen. New capabilities
emerge as registered extensions. A Works installation twenty years from now will be
able to replay a stream from today's installation without ambiguity — because the
encoding is positional (Hollerith), the log is append-only (Pacioli), and the Op
codes are fixed-width and extensible by registration.

---

## 8. BitPads and BitLedger Integration

Works is designed to interoperate with the BitPads and BitLedger protocol families.
These are not dependencies — Works can operate without them — but they are the
preferred wire protocols for Works' external interfaces.

**BitPads as Works' wire protocol**:
BitPads defines compact binary records from 1 byte (heartbeat) to 29 bytes (full
Rich Record). Works uses BitPads at three points:
- Intake: BitPads Intake receives Rich Records and decomposes or routes them as Manifests
- Stream: BitPads wire format for Stream bus layer (compact, structurally verified)
- Dispatch: BitPads output for external Works or BitLedger installation consumption

BitPads category 0110 (Rich Log Entry: Value + Time + Task + Note) is the Works
Manifest format. When an external system sends a BitPads 0110 record to Intake,
Works recognizes it as a Manifest and applies compound routing logic.

**BitLedger as Works' financial integrity protocol**:
BitLedger defines conservation invariants for financial data — debits equal credits,
conservation holds across compound records. Works' Office uses BitLedger invariants
to reconcile financial chunks: when ledger data enters through Intake, Office verifies
that the conservation invariant holds before the chunk is routed to a Store.

BitLedger's 16 Universal Domain archetypes (Asset, Liability, Equity, Revenue, Expense,
and 11 others) map to Works' conveyance vocabulary — they define what kind of financial
chunk is moving and where it is permitted to go.

**What Works does not adopt from BitPads/BitLedger**:
- BitPads' binary packing is used at wire boundaries only. Inside Works, chunks are
  structured JSON — human-readable, inspectable, debuggable. Binary packing at the
  internal level would sacrifice inspectability for compression.
- BitLedger's node topology (distributed ledger consensus) is not adopted. Works is
  not a distributed system by default. It is a compound standard for a single
  installation. Federation between Works installations is an extension protocol concern.

---

## 9. Works and Workwarrior (WWSS)

The Workwarrior Stream Service (WWSS) in `services/stream/` is the reference
implementation of Works' Stream layer for the Workwarrior context. It predates the
formal Works specification and embodies its principles in shell and Python:

- `stream.log` is Stream's Log layer (Pacioli guarantee implemented)
- `adapters.sh` is a set of Intake adapters for task/timew/jrnl/hledger
- `lenses/` is a set of Stream Mill configurations (Burroughs through Cooper)
- `codecs.sh` is Dispatch format handling (text/JSON/ASCII)
- The `ww stream hooks` system is real-time emission from shell wrappers (Barrel-level automation)

WWSS is a Stream Mill installation — it reads from the Stream log and produces views.
It does not yet have a Gate, a full Barrel scheduler, or a multi-Store address space.
These are the gaps that the full Works specification closes.

As Works develops as a formal standard and implementation, WWSS becomes the `works-stream`
library — the Stream constitutional layer that any Works installation must implement.
The lens pipeline (Hollerith/Pacioli through Cooper) becomes the standard Mill lens set.
The Hollerith event format becomes the canonical Stream wire format.

---

## 10. What Works Is Not

**Works is not a database.**
Works does not optimize for query performance, indexing, or throughput. Works optimizes
for provenance, auditability, and authorized transformation. A database holds data;
Works processes it through a defined pathway with constitutional guarantees at each step.

**Works is not a workflow engine.**
Works does not define business logic. Barrel defines sequences of operations; Mill
defines transformation lenses. But what those operations mean — what constitutes a
completed invoice, a finished proposal, an authorized payroll run — is outside the
standard. Works provides the structure within which business logic runs.

**Works is not a message queue.**
Works' Stream bus layer resembles a message queue, but Works makes a stronger guarantee:
the Log layer means no message is ever lost, and Replay means any subscriber can
reconstruct the full event history. A message queue delivers and forgets; Works delivers
and remembers.

**Works is not AI-specific.**
Works is designed so that AI agents (Agent Actors) participate alongside Humans on
equal structural footing — bounded by the same capability token system, audited by the
same Stream, subject to the same Gate authorization. Works does not privilege or
disadvantage AI agents. It makes their participation visible, bounded, and auditable.

**Works is not finished.**
The standard defines nine core buildings, four Actor types, one wire protocol family,
and an extension protocol. Many buildings that will eventually register as Works
extensions do not yet exist — including buildings for specific industries, specific
AI modalities, specific data types. The extension protocol is the mechanism by which
Works grows without breaking. The standard itself is versioned. This is version 0.

---

## 11. The Summary Statement

Works is the answer to the question: **what does a principled information compound
look like when humans and AI agents work together at production volume?**

The answer has nine buildings, four Actor types, one constitutional layer (Stream),
one extension protocol, and three historical debts: to von Neumann for the processing
architecture, to Babbage for the computation model and the stored-program concept,
and to Lovelace for the symbolic generality and cyclic execution that make a general
information engine possible.

Works does not tell you what your business does. It tells you how the information
your business produces moves, transforms, and leaves — with a permanent record of
every step, an authorization boundary on every exit, and a policy authority that
can answer any compliance question from first principles.

The goal is an information compound that, decades from now, can replay its complete
operational history from the first Stream event to the last Vault entry, without
ambiguity, without loss, and without requiring the original implementation to still
be running.

That is what Works is for.

---

*Version: 0*
*Date: 2026-05-04*
*Repository: babb/repos/works*
*Related: von-neumann-works.md, babb/repos/ww/services/stream/, babb/repos/ww-standard/times-research/babbage-lovelace-framework.md*
