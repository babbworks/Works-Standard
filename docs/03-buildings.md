# Works-Standard — 03: Buildings

**Document:** 03-buildings  
**Version:** 0.1-draft  
**Status:** Draft  
**Date:** 2026-05-04  
**Depends on:** 00-concepts.md, 01-chunk-format.md, 02-stream-protocol.md

---

## 1. Scope

This document specifies the role, contract, Stream obligations, and required
behaviors of each of the nine core Works buildings. It defines what each building
must do and must not do. It does not specify how buildings are implemented.

Conveyance rules between buildings — which sends to which, under what conditions —
are in `05-conveyance.md`. The Actor capability system that governs what Actors
may invoke is in `04-actors.md`. Buildings that have sufficient complexity to
warrant dedicated documents are noted with a forward reference; this document
gives their binding contract, the dedicated document gives full treatment.

Two invariants apply to every building without exception:

1. **Stream obligation**: every building MUST emit to Stream for every significant
   action. A building action that is not on Stream has not occurred from the
   perspective of the Works record.

2. **Pacioli obligation**: any internal state a building maintains MUST be
   recoverable from Stream replay. A building MUST NOT maintain state that
   cannot be reconstructed from the Log.

---

## 2. Building Contract Template

Each building section below follows this structure:

- **Role** — what this building is responsible for
- **Receives from** — which buildings or Actor types may send to it
- **Sends to** — which buildings it may send to
- **Stream events** — the op codes and actions it emits, and when
- **MUST** — absolute requirements; a non-conforming implementation if violated
- **MUST NOT** — absolute prohibitions
- **SHOULD** — strong recommendations
- **Notes** — design clarifications and edge cases

---

## 3. Intake

**Role**: the sole entry point for all data arriving from outside the compound.
Intake verifies, classifies, normalizes, and routes every incoming item before
anything enters Works state.

**Receives from**: External Actors, WW shell adapters, BitPads sources, API
sources, Agent Actors submitting external data.

**Sends to**: Store (primary routing destination), Mill (direct routing when
a Mill is waiting for a specific input type).

**Stream events**:

| Event | When |
|-------|------|
| `I received` | Every item arrives at Intake, whether it will be accepted or rejected |
| `I verified` | Item passes integrity check |
| `I rejected` | Item fails verification or classification — MUST include reason in ctx |
| `I routed`   | Item dispatched to a Store or Mill — MUST include destination in ctx |

Every item MUST produce at least one `I received` and either `I routed` or
`I rejected`. An item that disappears at Intake without a Stream record is not
conforming.

**MUST**:
- Verify the integrity of every incoming item before routing. For BitPads records,
  CRC-15 verification. For plain JSON, schema validation against chunk format.
  For other formats, format-specific validation appropriate to the adapter.
- Classify every verified item by type, assigning the chunk's `type` field.
- Normalize every verified item to the canonical chunk format before routing.
- Emit `I rejected` with a reason before discarding any item that fails
  verification or classification.
- Never route unverified data into the compound.
- Assign `ts_created`, `src`, and an initial provenance entry (creation entry)
  to every chunk it creates.

**MUST NOT**:
- Route chunks directly to Dispatch or Vault. Intake→Dispatch and
  Intake→Vault are not conforming paths except for Manifest parent
  records written to Vault at decomposition time (see `01-chunk-format §7.4`).
- Modify the content of a received item during normalization. Normalization
  adds Works fields (id, provenance, address); it does not alter the content.

**SHOULD**:
- Support Manifest routing: receive a BitPads Rich Record (category `0110`)
  and route it intact or decompose it per installation configuration.
- Support adapter subtypes for common sources: task (TaskWarrior export),
  timew (TimeWarrior export), jrnl (journal export), hledger (ledger export),
  BitPads Intake, API Intake, Agent Intake.
- Use content-hash id generation for programmatic ingestion to enable
  idempotent re-ingestion without duplicate chunks.

**Notes**:

Idempotency: repeated ingestion of the same source data MUST NOT create
duplicate chunks when content-hash ids are in use. If an incoming item's
computed id already exists in the compound, Intake MUST emit `I rejected`
with reason `duplicate` and discard the item without routing.

Intake is the only building that creates chunks from raw external data. All
other buildings operate on chunks that already exist in Works state.

---

## 4. Store

**Role**: named, addressed chunk repository. The persistent memory of the compound.
Stores hold chunks at any lifecycle stage and are the primary resting place for
all Works content between active operations.

**Receives from**: Intake (newly created chunks), Gate (rerouted chunks), Mill
egress (when Gate reroutes a chunk back to a Store), Bench (Actor commits).

**Sends to**: Mill ingress (Barrel-invoked or Actor-invoked), Bench (Actor pull).

**Stream events**:

| Event | When |
|-------|------|
| `S store-write` | A chunk is written to the Store for the first time at a given address |
| `S stage-transition` | A chunk's Level changes within the Store |
| `S fill` | A fill instruction populates a Bin with N copies or sets source mode |
| `S bin-mode` | A Bin's mode changes (queue → source, etc.) |

**MUST**:
- Enforce Row × Level × Bin addressing. Every chunk in a Store has a fully
  qualified address.
- Never delete a chunk after it is written. Chunks may be archived to Vault
  (one-way transition) but not deleted.
- Never overwrite a chunk's `content` field. Content revisions create new
  chunks with provenance linking to the prior version.
- Enforce Level transition rules: Raw → Staged requires Actor or Mill
  authorization; Staged → Integrated requires a G `approve` event from Gate.
- Append a provenance entry to a chunk's provenance chain on every write
  and every stage transition.
- Support `put`, `get`, `list`, and `history` operations at minimum.
- Honor Bin modes: `queue` (pull removes), `source` (pull copies), `filled`
  (N copies, then depletes like queue).
- Emit `S fill` when a fill instruction is executed, recording source chunk
  id, target address, count, and authorizing Actor.

**MUST NOT**:
- Accept write requests that do not carry a valid Actor identity.
- Permit a chunk's Level to move backward (Integrated → Staged, Staged → Raw)
  without a G `reject` or G `reroute` event authorizing it.
- Allow chunks to be pulled from a `queue`-mode Bin without removing them,
  or from a `source`-mode Bin without generating a copy.

**SHOULD**:
- Support a filesystem-backed layout where the address maps directly to a
  directory path: `{ww_base}/stores/{store}/{row}/{level}/{bin}/`.
- Support `history(row)` — return all chunks ever written to a given Row
  across all Levels and Bins, ordered by `ts_created`.
- Support content-type filtering on `list` — return only chunks of a given
  `type` within a Store, Level, or Bin.

**Notes**:

A Works installation MAY have unlimited named Stores. Stores are independent —
a chunk's `address.store` field identifies which Store it lives in, and a Mill
or Barrel that reads from multiple Stores must be explicitly configured to do so.

Store names MUST be unique within an installation, contain only alphanumeric
characters, hyphens, and underscores, and be stable — renaming a Store breaks
all existing provenance chain address references.

---

## 5. Mill

**Role**: the transformation building. The only place in the compound where chunks
change form. A Mill takes chunks from Stores or Bench, applies a lens pipeline,
and produces output at egress.

**Receives from**: Store (Barrel or Actor invocation), Bench (Actor-supplied chunks).

**Sends to**: Bench (intermediate output, Actor inspection), Mill egress (output
chunks awaiting Gate — Content Mill only), Stream (views — Stream Mill only).

**Stream events**:

| Event | When |
|-------|------|
| `S mill-start` | Mill run begins — records which lens pipeline, which input |
| `S mill-complete` | Mill run completes successfully |
| `S mill-fail` | Mill run fails — records reason |
| typed event per lens | Each lens emits events appropriate to its op code during the run |

**MUST**:
- Distinguish Stream Mill (read-only view of Stream, no Gate required) from
  Content Mill (reads Stores, produces new chunks, Gate required for output).
- Never route Content Mill output directly to Dispatch. Content Mill output
  goes to Mill egress; Gate decides what happens next.
- Always run Tier 0 (Hollerith encoding enforcement, Pacioli guarantee check)
  regardless of which other lens tiers are configured.
- Emit `S mill-start` before processing begins and `S mill-complete` or
  `S mill-fail` when the run ends.

**MUST NOT**:
- Modify the `content` field of input chunks. Mill creates new output chunks;
  it does not alter its inputs.
- Emit chunks to Dispatch directly. Gate is always between Mill and Dispatch.

**SHOULD**:
- Support composable lens pipeline configuration — an operator should be able
  to specify which tiers and which lenses run, and in what order.
- Support partial tier runs — a Mill configured for document assembly need not
  run signal or field tiers.

**Lens tiers** (full specification in `11-stream-lenses.md`):

| Tier | Lenses | Concern |
|------|--------|---------|
| 0 — Substrate | Hollerith, Pacioli | Encoding and guarantee enforcement — always active |
| A — Event | Burroughs, Baldwin | Raw event log; state mutation diff |
| B — Structure | Frick, Bundy | Discrete transitions; interval accumulation |
| C — Signal | Grant, Felt, Dey | Metrics; density; continuous signal |
| D — Field | Cooper | Geometric projection |

**Notes**:

Stream Mill output is a view, not a new chunk. A Stream Mill that renders a
Burroughs chronological log or a Hollerith matrix is producing display output,
not Works state. Stream Mill output is not stored, not vaulted, not Gate-authorized.

Content Mill output is new Works state. Output chunks at egress are assigned new
ids, new provenance entries (linking back to input chunks), and wait at egress
for Gate. They are formally in Mill egress — not yet in any Store — until Gate acts.

---

## 6. Bench

**Role**: the session workspace. Ephemeral working memory for an Actor's current
session. Bench is where chunks are assembled, reviewed, and prepared before being
committed to Works state.

**Receives from**: Store (Actor pull), Mill (intermediate output delivered to Actor).

**Sends to**: Store (Actor commit), Mill ingress (Actor-supplied input to a Mill run).

**Stream events**:

| Event | When |
|-------|------|
| `S bench-commit` | A chunk moves from Bench to a Store |

Bench MUST NOT emit Stream events for internal operations — reads, edits, and
rearrangements on Bench before commitment are not recorded on Stream. Only the
moment of commitment (Bench → Store) is a Stream event.

**MUST**:
- Clear all contents when the Actor's session ends, regardless of how the
  session ends (explicit close, timeout, or crash).
- Assign null address to all chunks while they are on Bench.
- Emit `S bench-commit` when a chunk moves from Bench to a Store, recording
  the target address and Actor identity.

**MUST NOT**:
- Write Bench content directly to Dispatch or Vault. Bench → Store → (Mill →
  Gate →) Dispatch/Vault is the required path.
- Persist Bench contents across sessions. A chunk on Bench that has not been
  committed to a Store when the session ends is lost.

**SHOULD**:
- Support multi-chunk assembly — an Actor pulling chunks from multiple Stores
  and multiple Rows onto Bench to assemble a composite output.
- Surface the uncommitted chunk count to the Actor before session close, to
  prevent accidental loss of uncommitted work.

**Notes**:

Bench is the one place in Works where content is genuinely mutable before
commitment. An Actor on Bench can edit, annotate, rearrange, and discard freely.
None of this is on Stream. The moment the Actor commits to a Store, immutability
applies and the Pacioli guarantee begins.

---

## 7. Barrel

**Role**: stored-program instruction scheduler. A Barrel is a named sequence of
Works operations stored in a Store and executed by the Barrel scheduler. Barrels
make the compound programmable.

**Receives from**: Store (instruction fetch at execution time), Stream Bus
(event-triggered Barrels subscribe to the Bus and fire on matching events).

**Sends to**: Mill (invocation), Store (writes on behalf of its instructions),
Gate (submitting chunks for authorization), Stream (S events per step).

**Stream events**:

| Event | When |
|-------|------|
| `S barrel-start` | Barrel execution begins — records Barrel name, trigger, Actor |
| `S barrel-step`  | Each instruction in the Barrel sequence executes |
| `S barrel-complete` | Barrel run finishes successfully |
| `S barrel-fail`  | Barrel run fails — records step, reason |
| `S barrel-disable` | Barrel is disabled (not deleted) |

**MUST**:
- Be stored in a Store. A Barrel that exists only in memory is not conforming —
  the Pacioli guarantee requires that Barrel definitions are part of the permanent
  record.
- Emit `S barrel-start` before executing any instruction.
- Emit `S barrel-step` for each instruction before executing it — not after.
  If a step fails, the failure is attributed to the step that was last recorded
  on Stream.
- Never delete or overwrite a prior Barrel version in its Store. A new Barrel
  version is a new chunk with provenance linking to the prior version.
- Support CycleMode: `single` (run once), `bounded` (run on schedule between
  two timestamps), `perpetual` (run continuously at a fixed interval).
- Operate within its issued capability token. A Barrel MUST NOT invoke operations
  its token does not permit, and MUST NOT attempt to escalate its own permissions.

**MUST NOT**:
- Self-approve at Gate. If a Barrel's instructions require Gate authorization,
  it submits to Gate and waits. If the Gate decision requires a Human Actor, the
  Barrel holds and notifies.
- Delete chunks from Stores as part of its instruction set. Barrel instructions
  may archive (via Gate) but not delete.

**SHOULD**:
- Support conditional branching based on Stream state — a Barrel that checks
  the most recent G event for an object before deciding its next step.
- Support event-triggered execution — subscribe to the Stream Bus and fire when
  a matching event arrives.

**Notes**:

A Barrel runs as an Automated Actor. Its capability token is the narrowest default
in the Actor model. The token is issued by Office at Barrel registration time and
is stored alongside the Barrel definition in its Store chunk.

---

## 8. Gate

**Role**: the authorization boundary between Mill output and Dispatch. Gate is the
enforcement point for the Right to Assembly. Nothing leaves the compound through
Dispatch without a Gate `approve` decision. Nothing enters Vault from an `integrated`
Store chunk without a Gate `archive` decision.

**Receives from**: Mill egress (Content Mill output only).

**Sends to**: Dispatch (approved chunks), Store (rerouted chunks), Stream (G events).
Gate does not send rejected chunks anywhere — rejection leaves the chunk in Mill
egress with a hold flag cleared and a G `reject` event on Stream; Mill or Barrel
decides what to do next.

**Stream events**:

| Event | When |
|-------|------|
| `G approve`  | Gate authorizes a chunk for Dispatch |
| `G archive`  | Gate authorizes a chunk for direct Vault entry without Dispatch |
| `G reject`   | Gate denies; chunk remains in Mill egress |
| `G hold`     | Gate sets hold flag on chunk in Mill egress; awaiting further action |
| `G reroute`  | Gate redirects chunk to a Store or back to Mill instead of Dispatch |

**MUST**:
- Emit a G event to Stream before any chunk state changes. The G event is the
  authorization record; it precedes the action it authorizes.
- Be the only path through which chunks reach Dispatch. Any chunk arriving at
  Dispatch without a G `approve` event in its provenance chain is not conforming.
- Be the only path through which `integrated` chunks enter Vault directly.

**MUST NOT**:
- Maintain a hold queue or any persistent storage. Gate is a decision point only.
  A chunk that Gate has flagged as held remains physically in Mill egress. Gate
  maintains only a flag indicating the hold status; the chunk does not move.
- Self-approve. A Gate `approve` or `archive` decision MUST be attributed to a
  named Actor (Human, Agent with explicit capability, or authorized Automated).
- Pass a chunk to Dispatch before the G `approve` event is durably written to
  the Stream Log.

**SHOULD**:
- Implement the BitPads Task block (8-bit packed instruction) as the Gate
  instruction format for compact, wire-efficient authorization signals.
- Surface held chunks to the relevant Human Actor with sufficient context
  (chunk content summary, source Mill, reason for hold) to make an informed decision.
- Support timeout on holds: if a held chunk receives no decision within a
  configured duration, Gate SHOULD escalate to a Human Actor.

**Notes**:

Gate has no internal storage. When Gate issues a `hold`, the chunk does not move —
it stays in Mill egress with a hold flag. When the hold is resolved (by a subsequent
`approve`, `archive`, `reject`, or `reroute` decision), Gate emits the resolution
G event and the chunk moves accordingly. The hold flag exists only in Gate's
in-memory state for the duration of the hold; the Stream G `hold` event is the
durable record.

Office receives all G events from Stream. Office MAY veto a Gate approval by
emitting an S `veto` event referencing the G event id; a conforming Gate
implementation MUST treat an Office veto as a rejection and emit G `reject`.

---

## 9. Dispatch

**Role**: the output valve. Dispatch delivers Gate-authorized chunks to their
external destinations. It is a one-way output surface — nothing enters the
compound through Dispatch.

**Receives from**: Gate (approved chunks only — carrying a G `approve` event in
their provenance chain).

**Sends to**: external destinations (file, API, remote Works installation, BitPads
receiver, human-readable render, Agent handoff packet). After successful delivery,
Dispatch sends the chunk to Vault.

**Stream events**:

| Event | When |
|-------|------|
| `D dispatched` | Chunk successfully delivered to its external destination |
| `D failed`     | Delivery failed — chunk returned to Gate with error ctx |

**MUST**:
- Verify that every chunk it receives carries a G `approve` event in its
  provenance chain. A chunk without Gate authorization MUST be rejected by
  Dispatch and reported to Office.
- Emit `D dispatched` after successful delivery, before archiving to Vault.
- Emit `D failed` on delivery failure, including the error detail in ctx.
- Archive successfully dispatched chunks to Vault after `D dispatched` is
  durably written to Stream.
- Return failed chunks to Gate (not to a Store or Mill directly) — Gate
  decides what to do after a delivery failure.

**MUST NOT**:
- Accept chunks from any source other than Gate.
- Retain chunks after successful dispatch. Post-dispatch, the chunk lives in
  Vault. Dispatch holds nothing.

**SHOULD**:
- Support standard destination types: file write, HTTP POST, BitPads wire
  output, email, message queue publish, broadcast to subscribed Actors.
- Use BitPads compact encoding for Works-to-Works dispatch over a network.
- Support Agent handoff packets: a structured context bundle containing
  chunk content, provenance summary, and Stream event history formatted
  for consumption by an Agent Actor in a new session.

**Notes**:

Dispatch failure is not a Dispatch decision. Dispatch attempts delivery,
reports success or failure to Stream, and returns failed chunks to Gate.
Gate holds the policy decision about what happens next — retry, reroute,
or reject.

---

## 10. Office

**Role**: the policy authority and constitutional layer of the compound. Office
enforces the rules that all other buildings must follow. It receives all
policy-class events from Stream and maintains the compound audit log.

**Receives from**: Stream (all policy-class events — G, I rejected, S register,
S token-issued, D failed, and others designated policy-class in the Works
constitution).

**Sends to**: Stream (S register, S token-issued, S token-revoked, S policy-updated,
S veto), building registry, Actor capability token store.

**Stream events**:

| Event | When |
|-------|------|
| `S register`       | A new extension building is registered |
| `S token-issued`   | An Actor capability token is issued |
| `S token-revoked`  | An Actor capability token is revoked |
| `S policy-updated` | The Works constitution is amended |
| `S veto`           | Office vetoes a Gate approval |

**MUST**:
- Maintain the building registry — the authoritative list of registered buildings.
- Issue and revoke Actor capability tokens.
- Maintain the compound audit log — a separate append-only log, distinct from
  Stream, recording every policy decision. The audit log is itself subject to
  the Pacioli guarantee.
- Be the only building with write access to the policy layer.
- Receive and process all G events from Stream.

**MUST NOT**:
- Allow buildings to participate in Works conveyance without prior registration.
- Issue capability tokens that exceed the permissions defined in the Works
  constitution.

Full Office specification: `07-office.md`.

---

## 11. Vault

**Role**: the permanent archive. Vault is the compound's long memory — the final
resting place for completed work, authorized output, and Manifest parent records.

**Receives from**: Dispatch (post-dispatch archive, Path A), Gate (direct archive
decision, Path B — see `01-chunk-format §5.3`), Intake (Manifest parent records
at decomposition time).

**Sends to**: authorized Actors (read-only access only).

**Stream events**:

| Event | When |
|-------|------|
| `S archive` | A chunk is written to Vault |

**MUST**:
- Be append-only at all times without exception. No Vault entry is ever modified
  or deleted under any circumstances.
- Reject all write requests that are not authorized by Gate (approve or archive
  decisions) or by Intake (Manifest parent records).
- Reject all modification requests without exception. There is no override, no
  operator procedure, and no capability token that permits modification of a Vault
  entry. Modification requests MUST be logged to the Office audit log as violations.
- Support read access for Actors with the appropriate capability token.
- Emit `S archive` for every Vault write.

**MUST NOT**:
- Accept chunks from Store, Mill, Bench, or Barrel directly. Every path to
  Vault passes through Gate or Intake.
- Return chunks to any Store, Mill, or Bench. Vault is read-only and one-way.

**SHOULD**:
- Support structured query by `manifest_id`, chunk `type`, `ts_created` range,
  and Actor identity — enabling audit and compliance queries without replaying
  the full Stream Log.
- Organize entries by the same Row × Level × Bin addressing convention used in
  Stores, using `integrated` as the Level for all Vault entries.

**Notes**:

Vault is not a Store. The distinction is permanent: Stores are working repositories
where chunks move, change Level, and may be sent to Mill or Gate. Vault is the
archive where chunks are deposited and never move again. A Vault query returns
exactly what was deposited; there is no further lifecycle.

Full Vault specification: forthcoming dedicated document.

---

## 12. Conformance Summary

| Building | Required Stream events | Gate dependency | Pacioli obligation | Key prohibition |
|----------|----------------------|-----------------|-------------------|-----------------|
| Intake | I received, I verified/rejected, I routed | None | Provenance on every chunk created | No routing to Dispatch or Vault (except Manifest parent) |
| Store | S store-write, S stage-transition, S fill | Indirect (enforces Gate requirement for Staged→Integrated) | Never delete or overwrite chunks | No backward Level transition without G reject/reroute |
| Mill | S mill-start, S mill-complete/fail | Content Mill output waits at egress for Gate | Output chunks carry provenance links to inputs | No direct Dispatch from Mill |
| Bench | S bench-commit only | None | None (Bench is pre-commitment) | No Bench→Dispatch or Bench→Vault |
| Barrel | S barrel-start, S barrel-step, S barrel-complete/fail | Submits to Gate; cannot self-approve | Barrel definitions stored in Store; never deleted | No self-approval at Gate; no capability escalation |
| Gate | G approve/archive/reject/hold/reroute | Is the gate | G events are permanent authorization records | No hold queue; no storage; no self-approval |
| Dispatch | D dispatched, D failed | Requires G approve in chunk provenance | Archives to Vault post-dispatch | No input from anywhere except Gate |
| Office | S register, S token-issued/revoked, S policy-updated, S veto | Receives all G events; may veto | Audit log is append-only | Sole writer to policy layer |
| Vault | S archive | Gate authorize or Intake Manifest path only | Append-only always; no exceptions | No modification, no return, no deletion |

---

*Works-Standard 03-buildings — end of document*  
*Next: `docs/04-actors.md`*
