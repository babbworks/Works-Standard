# Works-Standard — 00: Concepts and Vocabulary

**Document:** 00-concepts  
**Version:** 0.1-draft  
**Status:** Draft  
**Date:** 2026-05-04  
**Depends on:** (none — this document is the vocabulary root)

---

## Purpose

This document defines every term used across the Works-Standard specification set.
All other documents in the `docs/` series reference this vocabulary. A term that
appears capitalized in any Works document (Store, Mill, Gate, Actor, Chunk, etc.)
has its authoritative definition here.

When two readings of a specification document conflict, the reading consistent with
the definitions in this document takes precedence.

---

## 1. The Works Compound

**Works** (noun, proper) — the compound standard defined by this specification set.
Works governs the structure, movement, transformation, and authorized output of
business information. Works is a protocol specification, not software. Software is
built to Works; Works outlasts any particular implementation.

**Works installation** — a specific deployed instance of Works. A Works installation
has at least one Stream, at least one Store, and at least one Office. All other
buildings are optional at installation time but must be available through the
extension protocol.

**Compound** — the totality of a Works installation: all buildings, all Actors, all
Stores, all Stream state. The word is used in the sense of a Victorian industrial
compound — a bounded site within which defined processes occur. What enters the
compound is known. What leaves the compound is authorized. What happens inside is
recorded.

**Works constitution** — the set of policy rules published by the Office of a specific
Works installation. The Works constitution is not part of this specification — it is
the installation-specific policy that the Office enforces. This specification defines
the mechanism (Office, policy layer, Actor capability tokens); the constitution is
what each installation configures within that mechanism.

---

## 2. Buildings

**Building** — a role in the Works protocol. Buildings are not software components —
they are defined roles that a conforming implementation must provide. How a building
is implemented (as a process, a library, a remote service, a shell script) is outside
this specification.

A building has:
- A defined **role** — what it is responsible for within the compound
- A defined **contract** — the behaviors it must exhibit (MUST), should exhibit
  (SHOULD), and may optionally exhibit (MAY) per RFC 2119
- A defined set of **Op codes** it emits to Stream (see §5, §8)
- A defined set of **conveyance interfaces** — which buildings it receives chunks
  from and sends chunks to (see §6)

The nine core buildings are: **Intake**, **Store**, **Mill**, **Bench**, **Barrel**,
**Gate**, **Dispatch**, **Office**, **Vault**. These are defined in `docs/03-buildings.md`.

**Extension building** — a building type not defined in the core nine that has been
registered with Office through the extension protocol. Extension buildings are
first-class participants in Works conveyance once registered. See `docs/09-extension.md`.

**Building registration** — the act of an extension building declaring itself to
Office and receiving a registration token. An unregistered building that claims to
participate in Works conveyance is not conforming.

---

## 3. Core Buildings — Definitions

**Intake** — the entry point for all external data entering the compound. Intake
receives raw materials, verifies their integrity, classifies their type, normalizes
them to chunk format, and routes them to their initial destination (a named Store
or directly to a Mill). Intake MUST emit an I event to Stream for every item received,
whether accepted or rejected. Intake MUST NOT route unverified data into the compound.

**Store** — a named chunk repository within the compound. A Works installation has
one or more Stores, each with a human-assigned or serial name. Stores hold chunks
at any lifecycle stage. Stores are append-only: chunks are never deleted or modified
after being written. A Store's address space is three-dimensional: Row × Level × Bin.
See §7 (Chunk Address).

**Mill** — the transformation building. A Mill takes chunks from Stores or from Bench,
applies a defined sequence of operations (a lens pipeline), and produces output chunks
at its egress. Output chunks at Mill egress wait for Gate authorization before
proceeding. A Mill that produces output for external consumption is a Content Mill.
A Mill that produces only views of Stream data (read-only) is a Stream Mill. Stream
Mills do not require Gate authorization for their output.

**Bench** — the session workspace. Bench holds active chunks that an Actor is currently
working with. Bench is ephemeral: its contents are not Works state and are cleared when
the Actor's session ends. Moving a chunk from Bench to a Store is a deliberate
commitment act that MUST be recorded on Stream.

**Barrel** — the instruction scheduler and stored-program building. A Barrel is a named
sequence of operations — stored in a Store — that is fetched and executed by the
Barrel scheduler. Barrels make the compound programmable: recurring processes, triggered
pipelines, and automated handoffs are all Barrels. Barrels are versioned in their Store
and MUST NOT be deleted (Pacioli guarantee). A Barrel may be disabled but not erased.

**Gate** — the authorization boundary between Mill output and Dispatch. Gate is the
enforcement point for the Right to Assembly (see §10). Nothing leaves the compound
through Dispatch without passing Gate. Gate MUST emit a G event to Stream for every
authorization decision (approve, reject, hold, reroute). Gate MAY require human
Actor confirmation before approving.

**Dispatch** — the output building. Dispatch receives Gate-authorized chunks and
delivers them to their external destination (file write, API call, publication,
broadcast, another Works installation). Dispatch MUST NOT receive chunks from any
source other than Gate. Dispatch MUST emit a D event to Stream for every completed
delivery and every failed delivery attempt.

**Office** — the policy authority for the compound. Office receives all policy-class
events from Stream and has authority to veto, escalate, or record. Office is the
only building with write access to the policy layer. Office maintains the building
registry, issues and revokes Actor capability tokens, publishes the Works constitution,
and maintains the compound audit log. The Office audit log is separate from Stream
and is itself append-only.

**Vault** — the permanent archive. Completed Mill outputs, authorized Dispatch records,
committed Barrel runs, and closed project bundles are archived to Vault. Vault is
append-only: nothing in Vault is ever modified or deleted. A chunk in Vault cannot
be retrieved for modification — it can only be read. Moving a chunk from a Store
to Vault is a one-way transition that MUST be authorized by Gate and recorded on Stream.

---

## 4. Actors

**Actor** — a named entity that interacts with the Works compound. All Actor types
are session-scoped: every action an Actor takes is recorded on Stream with the Actor's
identity. There are no anonymous actions in a conforming Works installation.

Works defines four Actor types:

**Human Actor** `{name, session_id, capability_token}` — a person interacting with
the compound. Humans have the highest default authority within their capability bound.
Human sessions are finite; Bench is cleared on session end.

**Agent Actor** `{name, model, session_id, capability_token, context_limit}` — a named
AI agent (LLM-based or otherwise) interacting with the compound within a session.
An Agent Actor MUST be named and identified; anonymous agent actions are not permitted.
Agent Actors MUST NOT self-approve at Gate without an explicit capability grant from
Office. Agent Actors have a context limit — the bounded amount of Stream history or
Store content they can hold in working memory during a session.

**Automated Actor** `{barrel_name, trigger, capability_token}` — a Barrel running
autonomously (cron-triggered or event-triggered) with no human in the loop for that
execution. Automated Actors have the narrowest default capability tokens. If an
Automated Actor requires Gate approval to proceed, it MUST hold and notify a Human
Actor rather than self-approving.

**External Actor** `{system_id, protocol, capability_token}` — an external system
(another Works installation, a BitLedger node, an API, a legacy tool) interacting
with the compound. External Actors MUST interact only through Intake (entering data)
or through Dispatch (receiving authorized output). External Actors MUST NOT directly
access Stores, Mills, or Barrels.

**Session** — the bounded period during which an Actor is active in the compound.
A session begins when an Actor authenticates and receives a session token. A session
ends when the Actor explicitly closes it or when it times out. All Bench contents
associated with the session are cleared on session end.

**Capability token** — a scoped authorization record issued by Office to an Actor,
specifying exactly what the Actor may do within the compound: which Stores it may
read, which Stores it may write, which Mills it may invoke, whether it may approve
at Gate, whether it may register extension buildings, etc. An Actor without a
capability token for a given operation MUST be denied that operation.

---

## 5. Stream

**Stream** — the constitutional layer of the Works compound. Stream is the medium
through which all buildings communicate and the mechanism that makes Works guarantees
structural rather than policy-dependent. Stream is not optional. A Works installation
without Stream is not a conforming Works installation.

Stream has three operational sub-layers:

**Log layer** — the append-only event file. Every building action that is recorded
on Stream is permanently stored. Log entries are never modified or deleted. The Log
layer enforces the Pacioli guarantee (see §9) at the event level. State is always
reconstructible by replaying the Log from the beginning.

**Bus layer** — the real-time event delivery mechanism. Buildings and Actors subscribe
to Stream events in real time. Subscriptions MAY be filtered by op code, object,
profile, or project. The Bus layer does not store events; that is the Log layer's
responsibility.

**Replay layer** — the catch-up and recovery mechanism. Any subscriber MAY request
all events from a given timestamp forward, catching up to current state after a
disconnect or cold start. If any building's internal state is lost, it MUST be
recoverable through Stream Replay.

**Stream event** — a single record in the Stream Log. Every Stream event MUST conform
to the Hollerith encoding format (see `docs/02-stream-protocol.md`). A Stream event
has exactly five positional fields: timestamp, op code, action, object, and context.

**Op code** — a single uppercase letter that classifies the type of a Stream event.
The Works-Standard core op codes are:

| Code | Name        | Emitted by   | Meaning |
|------|-------------|--------------|---------|
| `T`  | Task        | Intake, Barrel | Task lifecycle event (add, modify, done, delete) |
| `F`  | Frick       | Mill, Barrel | State transition (start, stop, context switch) |
| `B`  | Bundy       | Mill, Barrel | Interval boundary (clock-in, clock-out) |
| `D`  | Dey         | Mill         | Continuous signal sample (intensity, stability, fragmentation) |
| `H`  | Hollerith   | Stream       | Encoding marker (schema version, field map — Log header) |
| `S`  | System      | Any building | System or meta event (sync, reset, agent handoff, disable) |
| `A`  | Annotation  | Intake, Bench | Annotation or journal entry attached to an object |
| `I`  | Intake      | Intake       | Arrival event (received, verified, routed, rejected) |
| `G`  | Gate        | Gate         | Authorization event (approve, reject, hold, reroute) |

Extension buildings MAY register new op codes through the extension protocol. A
registered extension op code is a single uppercase letter not in the core set above,
assigned by Office at registration time.

**Hollerith encoding** — the positional line encoding used for all Stream Log entries.
Named for Herman Hollerith's punched-card encoding principle: fixed positional fields,
compact, machine-readable, sortable by the first field alone. See `docs/02-stream-protocol.md`.

**Pacioli guarantee** (at Stream level) — the guarantee that no Stream Log entry is
ever modified or deleted after being written. Named for Luca Pacioli's double-entry
accounting principle. A conforming implementation MUST enforce the Pacioli guarantee
on the Stream Log. See §9 (Guarantees).

---

## 6. Conveyance

**Conveyance** — the act of moving a chunk from one building to another. All conveyance
MUST cross Stream: a conveyance event MUST be emitted to Stream for every chunk
movement between buildings. Conveyance that is not on Stream has not occurred from
the perspective of the Works record.

**Conveyance path** — the ordered sequence of buildings a chunk passes through from
arrival at Intake to archival in Vault. The canonical conveyance path is:

```
Intake → Store → (Bench ←→ Store) → Mill → Gate → Dispatch → Vault
```

Chunks MUST NOT skip Intake on entry. Chunks MUST NOT reach Dispatch without
passing Gate. Chunks that fail at Gate MAY be returned to Store or Mill. A chunk
that has entered Vault MUST NOT return to any other building.

**Routing** — the decision made by Intake (or by a Barrel) about which Store or Mill
a chunk is directed to after arrival. Routing decisions MUST be recorded on Stream.

---

## 7. Chunks

**Chunk** — the atomic unit of content in Works. Every piece of information that
enters, moves through, or leaves the compound is a chunk. A chunk is indivisible:
if a chunk must be split into parts for separate processing, each part becomes a new
chunk with provenance linking back to the original.

**Chunk anatomy** — the required fields of every chunk in a conforming implementation:

| Field         | Type              | Required | Meaning |
|---------------|-------------------|----------|---------|
| `id`          | UUID or hash      | MUST     | Unique chunk identifier |
| `src`         | string            | MUST     | Source building or adapter name |
| `type`        | Op code char      | MUST     | Op-code classification of chunk content |
| `content`     | any               | MUST     | The actual data |
| `address`     | StoreAddress      | if in Store | Current Store address (null if on Bench) |
| `provenance`  | array of records  | MUST     | Ordered history of prior addresses and actions |
| `ts_created`  | unix timestamp    | MUST     | When the chunk first entered the compound |
| `ts_modified` | unix timestamp    | MUST     | When the chunk's stage last changed |
| `actor`       | Actor identity    | MUST     | Actor who last acted on this chunk |
| `manifest_id` | UUID              | if applicable | Parent Manifest id, if chunk was decomposed from one |

**Chunk address** — the three-dimensional location of a chunk within a named Store.
Format: `{store: string, row: identifier, level: Level, bin: OpCode}`.

**Row** — the first dimension of a chunk address. Row identifies what the chunk is
*about* — the entity, task, project, or object this chunk describes. Multiple chunks
about the same object share a Row. Row values are UUIDs or human-assigned names.

**Level** — the second dimension of a chunk address. Level is the chunk's lifecycle
stage within the Store:
- `Raw` — arrived from Intake; unprocessed
- `Staged` — assembled by Mill; awaiting Gate authorization
- `Integrated` — Gate-authorized; complete

Stage transitions MUST be recorded on Stream. A chunk moves from Raw → Staged when
a Mill or Actor promotes it. A chunk moves from Staged → Integrated when Gate approves
it for Dispatch or archival.

**Bin** — the third dimension of a chunk address. Bin is the Op-code-level type of
the chunk's content, drawn from the Op codes table in §5. A chunk in a Task Bin
(T) contains task lifecycle data. A chunk in an Annotation Bin (A) contains a
journal entry or note.

**Provenance chain** — the ordered list of every address and action this chunk has
had since it entered the compound. The provenance chain is append-only: entries are
added on every stage transition and conveyance event. The provenance chain is the
chunk-level expression of the Pacioli guarantee.

**Manifest** — a compound chunk containing multiple chunk types in a single verified
unit. A Manifest is the Works representation of a BitPads Rich Record (category 0110:
Value + Time + Task + Note). Intake MAY route a Manifest intact — preserving its
compound provenance — or decompose it into constituent chunks, each with a
`manifest_id` linking back to the original. See `docs/08-manifest.md`.

---

## 8. Mill and Lenses

**Lens** — a named transformation operation that a Mill applies to chunks or Stream
events. Lenses are composable: the output of one lens is the input to the next.
The ordered set of lenses applied in a Mill run is the lens pipeline.

**Lens pipeline** — the sequence of lenses a Mill applies during a run. The Works
reference lens pipeline has four tiers:

| Tier | Name       | Lenses                        | Concern |
|------|------------|-------------------------------|---------|
| 0    | Substrate  | Hollerith, Pacioli            | Encoding and guarantee enforcement |
| A    | Event      | Burroughs, Baldwin            | Raw event log and state mutation diff |
| B    | Structure  | Frick, Bundy                  | Discrete transitions and intervals |
| C    | Signal     | Grant, Felt, Dey              | Metrics, density, continuous signal |
| D    | Field      | Cooper                        | Geometric field projection |

A Mill is not required to run all tiers. A Mill is configured with the subset of
tiers and lenses appropriate to its purpose. Tier 0 (Hollerith, Pacioli) is always
active — it is the substrate, not a configurable option.

**Stream Mill** — a Mill that reads from the Stream Log and produces views. A Stream
Mill's output is a derived view, not a new chunk. Stream Mills do not require Gate
authorization for their output.

**Content Mill** — a Mill that reads from Stores and produces new chunks. Content Mill
output chunks are placed at Mill egress and MUST pass Gate before proceeding to
Dispatch or Store.

**Ingress** — the Mill's input interface. Chunks or Stream events arrive at ingress
before the lens pipeline begins.

**Egress** — the Mill's output interface. Transformed chunks accumulate at egress
after the lens pipeline completes. For Content Mills, chunks at egress wait for Gate.

**Difference Engine cascade** — the computational method used in Dey and Felt lenses
for continuous signal computation. Named for Babbage's Difference Engine No. 2.
A difference table of order N maintains N registers. Each advance step applies a
pure addition cascade: for i from N down to 1, `reg[i-1] += reg[i]`; the result is
`reg[0]`. This computes polynomial approximations of event-density signals through
addition alone, without multiplication.

---

## 9. Guarantees

**Pacioli guarantee** — the fundamental non-erasure invariant of the Works compound.
Named for Luca Pacioli's double-entry accounting principle (1494): the present state
of a system is a derivation of its recorded history, not a stored snapshot.

The Pacioli guarantee applies at four levels simultaneously in a conforming Works installation:

1. **Chunk level**: chunks in Stores are never deleted or overwritten. Stage transitions
   are new entries, not modifications of existing entries.
2. **Stream level**: Stream Log entries are never deleted or modified after being written.
3. **Office level**: the Office audit log is never deleted or modified.
4. **Vault level**: Vault entries are never deleted or modified under any circumstances.

A conforming implementation MUST enforce the Pacioli guarantee at all four levels.
An implementation that permits deletion, truncation, or modification of any of these
four logs — except through a documented, audited, operator-level override procedure
that itself creates a Stream record — is not conforming.

**Right to Assembly** — the constitutional guarantee that no chunk may be dispatched
from the compound without passing Gate. Gate is not a feature that can be bypassed
in a conforming Works installation. The Right to Assembly ensures that human oversight
is structurally present in the output pathway — not as a policy option, but as a
protocol requirement.

**Identity guarantee** — the guarantee that every action in a conforming Works
installation is attributed to a named Actor. Anonymous actions are not permitted.
An action taken without a valid session and Actor identity MUST be rejected.

**Historical coherence** — the guarantee that a Stream Log written under Works-Standard
v0.1 can be correctly replayed by a conforming implementation of any later
Works-Standard version. The Hollerith encoding is positionally defined and
version-tagged (H events). Extension op codes are registered and enumerated.
No migration is required to read an old log — only the schema version header
is consulted.

---

## 10. Rights and Principles

**Right to Assembly** — see §9. No output leaves the compound without Gate authorization.

**Right of Access** — any Actor with a valid capability token MAY pull any chunk
from any Store onto their Bench for inspection. Office MAY restrict this right
through the capability token system, but the default is access — not restriction.
The Right of Access ensures that the compound does not become opaque to its operators.

**Accountability principle** — every action is attributed; every attribution is on
Stream; Stream is permanent. There is no action in a conforming Works installation
that cannot be traced to a named Actor and a specific timestamp.

**Extension principle** — the core standard is fixed; capabilities expand through
registration. The nine core buildings are defined once. New capabilities emerge as
registered extensions. The standard does not change to accommodate new use cases —
new use cases register as extensions within the existing standard.

**Bottleneck principle** — Stream is the intentional architectural bottleneck.
All building communications cross Stream. This is a performance cost accepted in
exchange for three structural properties: universal auditability, universal
recoverability, and universal policy enforceability. A conforming implementation
MUST NOT route building-to-building communication outside Stream for policy-class
events.

---

## 11. External Standards

**BitPads** — the compact binary record encoding standard published by babbworks
(`babbworks/bitpads-standard`). Works uses BitPads at Intake, Stream Bus layer,
and Dispatch for wire-efficient data exchange. BitPads is not a dependency of
Works-Standard — a conforming implementation MAY use alternative encodings at
wire boundaries. BitPads is the preferred encoding.

**BitLedger** — the financial integrity standard published by babbworks
(`babbworks/bitledger-standard`). Works uses BitLedger conservation invariants in
Office for verification of financial chunks entering through Intake. BitLedger
defines 16 Universal Domain archetypes that map to Works conveyance vocabulary.
BitLedger is not a dependency of Works-Standard.

**WW-Standard** — the Workwarrior Technical Standard published by babbworks
(`babbworks/ww-standard`). WW-Standard governs the profile-based shell productivity
system that serves as the reference implementation of Works Stream, Intake, and
Barrel for the shell-based operational context. WW-Standard is a Works binding —
it implements a conforming subset of Works-Standard in shell and Python.

---

## 12. Historical Name Sources

Works names its buildings and lenses after historical figures whose work directly
contributed to the architectural principle each building or lens embodies:

| Name       | Person / Source | Contribution |
|------------|-----------------|--------------|
| Pacioli    | Luca Pacioli (1494) | Double-entry accounting; append-only ledger as truth |
| Hollerith  | Herman Hollerith (1890) | Punched-card symbolic encoding; positional fixed fields |
| Burroughs  | William S. Burroughs (1888) | Adding machine; raw event accumulation |
| Baldwin    | Frank S. Baldwin (1870s) | Reversible calculation; state mutation diff |
| Frick      | Frederick W. Frick (late 19th C) | Discrete state transition recorder |
| Bundy      | Willard L. Bundy (1888) | Start/stop time clock; interval accumulation |
| Grant      | George B. Grant (1870s) | Mechanical differential analyzer; derived metrics |
| Felt       | Dorr E. Felt (1886) | Comptometer; compression scoring, density |
| Dey        | Alexander Dey (1888) | Dial time recorder; continuous analog signal |
| Cooper     | (instrument maker tradition) | Geometric field projection; spatial manifold |
| Barrel     | Charles Babbage (1830s–1871) | Barrel drum stored program; instruction sequencing |
| Mill       | Charles Babbage (1830s–1871) | Analytical Engine Mill; ingress/operation/egress |
| Store      | Charles Babbage (1830s–1871) | Analytical Engine Store; Row × Level × Bin memory |

These names are not decoration. Each name is a claim about what the component does
and how it is grounded in historical precedent. A developer reading these names
should be able to trace the architectural lineage.

---

## 13. RFC 2119 Key Words

All Works-Standard specification documents use the key words defined in RFC 2119:

- **MUST** / **MUST NOT**: absolute requirement or prohibition; a conforming
  implementation has no discretion
- **SHOULD** / **SHOULD NOT**: strong recommendation; deviation requires documented
  justification
- **MAY**: permitted behavior; neither required nor prohibited
- **SHALL** / **SHALL NOT**: used interchangeably with MUST / MUST NOT in this standard

When a MUST requirement conflicts with a practical constraint in a specific deployment,
the constraint MUST be documented in the Works constitution for that installation, and
the deviation MUST be recorded in the Office audit log.

---

*Works-Standard 00-concepts — end of document*  
*Next: `docs/02-stream-protocol.md`*
