# Works-Standard — 06: Gate Protocol

**Document:** 06-gate-protocol  
**Version:** 0.1-draft  
**Status:** Draft  
**Date:** 2026-05-04  
**Depends on:** 00-concepts.md, 01-chunk-format.md, 03-buildings.md, 04-actors.md, 05-conveyance.md

---

## 1. Scope

This document specifies the Gate protocol in full: the authorization instruction
format, the five Gate decisions and their precise semantics, the hold flag
mechanism, escalation, the BitPads Task block as the preferred instruction encoding,
and the Office veto. It extends the Gate contract in `03-buildings.md` with the
detail required to implement a conforming Gate.

Gate is the most structurally significant building in Works after Stream. Everything
that leaves the compound — and everything that enters permanent Vault storage directly
— passes through Gate. The Gate protocol is the enforcement mechanism for the Right
to Assembly.

---

## 2. Gate Fundamentals

### 2.1 What Gate Is

Gate is a decision point, not a building with storage. It sits between Mill egress
and Dispatch (or Vault for direct archive). It holds no chunks — chunks at Gate
are physically in Mill egress throughout the Gate interaction. Gate maintains only
one piece of state per chunk: a hold flag.

When Gate issues a decision, the decision is recorded on Stream before any chunk
moves. The G event is the authorization. Without a G event, no chunk moves.

### 2.2 What Triggers Gate

Gate is triggered when:

1. A Content Mill completes a run and output chunks accumulate at Mill egress.
   `S mill-complete` signals Gate that output is ready for review.
2. A Barrel instruction explicitly submits a chunk at Mill egress to Gate.
3. A Barrel or Actor instruction submits a Store chunk at `integrated` Level
   directly to Gate for archival consideration (§4.15 of `05-conveyance.md`).

Gate is not triggered by Stream Mill output. Stream Mills produce views, not chunks.
Gate is not triggered by Bench operations. Bench content is not Works state until
committed to a Store.

### 2.3 Who May Act at Gate

Gate decisions MUST be made by a named Actor with gate authority in their capability
token. The three authority levels (from `04-actors.md §5.2`):

| Decision | Human (default) | Agent (default) | Automated (default) | Explicit token grant required |
|----------|-----------------|-----------------|---------------------|-------------------------------|
| approve  | yes | no | no | Agent or Automated |
| archive  | yes | no | no | Agent or Automated |
| reject   | yes | no | no | Agent or Automated |
| hold     | yes | yes | yes | — (all may hold) |
| reroute  | yes | no | no | Agent or Automated |

Any Actor type MAY place a hold. Only Actors with explicit approve/archive/reject/
reroute authority in their token may make terminal decisions.

Office MAY veto any Gate approval after it is issued (§7). The veto converts the
approval to a rejection retroactively.

---

## 3. Gate Decisions

### 3.1 Approve

**Semantics**: the chunk is authorized for delivery to Dispatch.

**Who**: any Actor with `gate.approve: true` in their capability token.

**Stream event**:
```
<ts> G approve <chunk_id> {"actor":"human:mp","reason":"<optional>","dest":"<dispatch_dest>"}
```

**Sequence**:
1. Actor with approve authority reviews the chunk at Mill egress.
2. Actor issues approve decision to Gate.
3. Gate emits `G approve` to Stream. The event MUST be durably written to the
   Stream Log before any other action.
4. Gate appends a provenance entry to the chunk: `action: gate-approved`,
   `actor` set to the approving Actor, `stream_event_id` set to the G approve event.
5. Chunk moves from Mill egress to Dispatch.
6. Dispatch delivers and then archives to Vault (Path A).

**After approval**: the G approve event is permanent. If Dispatch fails, the
authorization stands — the chunk returns to Gate for a new decision on delivery,
but the original G approve event is not withdrawn. A new G approve is issued if
Dispatch retries successfully.

### 3.2 Archive

**Semantics**: the chunk is authorized for direct Vault entry. No external delivery.

**Who**: any Actor with `gate.archive: true` in their capability token.

**Stream event**:
```
<ts> G archive <chunk_id> {"actor":"human:mp","reason":"<optional>"}
```

**Sequence**:
1. Actor with archive authority reviews the chunk at Mill egress.
2. Actor issues archive decision to Gate.
3. Gate emits `G archive` to Stream, durably written before any action.
4. Gate appends provenance entry to the chunk: `action: gate-archived`.
5. Chunk moves from Mill egress directly to Vault.
6. Vault emits `S archive`.

**Use cases**: a completed project bundle that has no external recipient; a
finalized internal document that must be permanently preserved but not dispatched;
a regulatory record that must be kept but not transmitted.

### 3.3 Reject

**Semantics**: the chunk is not authorized. It remains in Mill egress. The hold
flag, if set, is cleared. The submitting Barrel or Actor is notified.

**Who**: any Actor with `gate.reject: true` in their capability token.

**Stream event**:
```
<ts> G reject <chunk_id> {"actor":"human:mp","reason":"missing line items"}
```

`reason` SHOULD always be present on reject. A reject without a reason is
technically conforming but SHOULD be flagged by Office as an audit gap.

**Sequence**:
1. Actor issues reject decision to Gate.
2. Gate emits `G reject` to Stream.
3. Gate clears the hold flag if set.
4. Gate appends provenance entry to the chunk: `action: gate-rejected`, with reason.
5. Chunk remains in Mill egress, physically unchanged.
6. Gate notifies the submitting Barrel or Actor of the rejection.

**After rejection**: the chunk is in Mill egress, unblocked. The Barrel or Actor
that submitted it must decide what to do: resubmit after modification (Bench→Mill→
Gate), reroute to a Store for later work, or abandon. Nothing in the Gate protocol
forces a next step — that is the submitter's responsibility.

A rejected chunk that sits in Mill egress indefinitely SHOULD be surfaced by Office
during audit runs as an unresolved Gate rejection.

### 3.4 Hold

**Semantics**: a flag is set on the chunk in Mill egress. The chunk does not move.
No terminal decision has been made. The hold indicates that a decision is pending.

**Who**: any Actor with a valid capability token and an active session. All Actor
types may hold by default.

**Stream event**:
```
<ts> G hold <chunk_id> {"actor":"automated:nightly-export","reason":"requires human review","notify":"human:mp"}
```

**Sequence**:
1. Any Actor (most commonly an Automated Actor that cannot self-approve) issues
   a hold to Gate.
2. Gate emits `G hold` to Stream.
3. Gate sets the hold flag on the chunk in Mill egress.
4. Gate notifies the Actor named in `notify` (if present).
5. The chunk does not move.

**Hold state**: the chunk is in Mill egress with the hold flag set. Gate tracks
the hold flag in memory only — it is not stored. The G hold event on Stream is the
durable record. On restart, Gate MUST reconstruct hold state by replaying Stream
events: any chunk with a `G hold` event and no subsequent terminal G event
(approve, archive, reject, reroute) is in held state.

**Multiple holds**: a chunk MAY have multiple sequential holds — a hold, then a
review, then another hold pending a second reviewer. Each hold is a separate G hold
event on Stream. The hold flag is re-set after each hold until a terminal decision.

**Hold timeout** (SHOULD): if a held chunk receives no terminal decision within
a configured duration, Gate SHOULD escalate automatically — placing a new G hold
with `notify` set to the designated escalation Actor (typically a senior Human
Actor). The escalation itself is a new G hold event, not a separate event type.

### 3.5 Reroute

**Semantics**: the chunk is redirected from Mill egress to a named Store or back
to a named Mill rather than proceeding to Dispatch or Vault.

**Who**: any Actor with `gate.reroute: true` in their capability token.

**Stream event**:
```
<ts> G reroute <chunk_id> {"actor":"human:mp","dest":"S-work/project-alpha/raw/drafts","reason":"needs additional sections"}
```

**Sequence**:
1. Actor issues reroute with a destination address.
2. Gate emits `G reroute` to Stream.
3. Gate appends provenance entry to the chunk: `action: gate-rerouted`, with
   destination and reason.
4. Chunk moves from Mill egress to the specified destination:
   - If destination is a Store address: chunk is written to that Store at the
     specified Row/Level/Bin. Level MUST be `raw` or `staged` — reroute cannot
     place a chunk at `integrated`.
   - If destination is a Mill name: chunk is placed at that Mill's ingress for
     a new run.
5. Hold flag cleared if set.

**Use cases**: a document that needs additional content before dispatch is rerouted
to a Store at `raw` for further editing; a chunk that went through the wrong Mill
is rerouted to the correct Mill; a chunk needs human additions on Bench before
being resubmitted.

---

## 4. The Hold Flag

The hold flag is Gate's only persistent in-session state. It is a boolean per
chunk-in-egress: held or not held.

**Setting**: `G hold` event sets the flag.  
**Clearing**: any terminal decision (approve, archive, reject, reroute) clears the flag.  
**Persistence**: the flag is in-memory only. It MUST be reconstructed from Stream
replay on startup (see §3.4).  
**Scope**: the hold flag is per chunk, not per Gate. If multiple chunks are at Mill
egress simultaneously, each has its own independent hold flag.

A chunk with the hold flag set MUST NOT be moved by any building other than Gate.
A Barrel that attempts to pull a held chunk from Mill egress MUST be denied.
The hold is Gate's lock on that chunk until it makes a terminal decision.

---

## 5. Gate Instruction Format

### 5.1 Plain Instruction

In its simplest conforming form, a Gate instruction is a structured request from
an Actor to Gate containing:

```json
{
  "decision":  "approve | archive | reject | hold | reroute",
  "chunk_id":  "<chunk id>",
  "actor":     "<actor identity string>",
  "session_id": "<session id>",
  "reason":    "<optional human-readable reason>",
  "dest":      "<optional destination for reroute>",
  "notify":    "<optional actor identity for hold escalation>"
}
```

This is the baseline. Any implementation that can parse and act on this structure
is conforming at the Gate protocol level.

### 5.2 Preferred: BitPads Task Block

The preferred Gate instruction encoding is the BitPads Task block — an 8-bit
packed instruction designed for compact, wire-efficient authorization signals.

The BitPads Task block encodes a Gate instruction in a single byte plus a variable
payload:

```
Byte 0 (instruction byte):
  Bits 7–5: Actor type   (000=Human, 001=Agent, 010=Automated, 011=External)
  Bits 4–3: Decision     (00=approve, 01=archive, 10=reject, 11=hold/reroute)
  Bits 2–0: Flags        (bit 2=has_reason, bit 1=has_dest, bit 0=has_notify)

Payload (variable, present only when corresponding flag bits are set):
  [chunk_id: 16 bytes UUID or 12 bytes hash]
  [reason:   length-prefixed UTF-8 string, present if bit 2 set]
  [dest:     length-prefixed UTF-8 string, present if bit 1 set]
  [notify:   length-prefixed UTF-8 string, present if bit 0 set]
```

**Decision encoding** (bits 4–3):
- `00` = approve
- `01` = archive  
- `10` = reject
- `11` = hold when `has_dest` flag is 0; reroute when `has_dest` flag is 1

**Minimum BitPads Gate instruction**: 1 byte instruction + 12 bytes chunk hash
= 13 bytes for a no-reason approve or reject. Compare to the plain JSON instruction
(~200 bytes minimum).

**When to use BitPads Task block**:
- Gate instructions transmitted over a network between a remote Actor interface
  and the compound
- High-frequency automated Gate decisions (Automated Actors processing a queue
  of chunks)
- Works-to-Works Gate forwarding (one installation forwarding Gate decisions to
  another)

**When plain instruction is sufficient**:
- Local CLI Gate interaction (human at a terminal)
- Development and testing environments
- Single-machine installations where wire efficiency is not a concern

A conforming Gate implementation MUST accept plain JSON instructions. It SHOULD
also accept BitPads Task block instructions. It MUST NOT accept only BitPads and
refuse plain JSON — the plain format is the baseline.

Full BitPads specification: `12-bitpads-binding.md`.

---

## 6. Gate and the Stream

Every Gate decision is recorded on Stream before the chunk moves. This is absolute.
The consequences:

**The G event is the authority**: if a G approve event is on Stream and the chunk
did not reach Dispatch (due to a crash between the G event write and the chunk
movement), the implementation MUST complete the move on recovery. The Stream record
is not withdrawn — it is the truth. State catches up to the record, not the
other way around.

**G events are auditable**: every Gate decision — every approve, archive, reject,
hold, reroute — is permanently on Stream with the Actor's identity and timestamp.
There is no Gate decision that is not on Stream. There is no Gate decision that
can be erased.

**Office receives all G events**: Office subscribes to the Stream Bus with a filter
for op=G. Every G event is received by Office in real time. Office MAY veto (§7).
Office records all G events in its audit log.

**Temporal ordering**: the G event timestamp is the moment of decision, not the
moment of chunk movement. The chunk may move milliseconds after the G event is
written. In the Stream record, the decision precedes the consequence — this is
the correct causal order.

---

## 7. Office Veto

Office may veto a Gate approval after it is issued. The veto is not a reversal of
history — the G approve event remains permanently on Stream. The veto is an
additional event that overrides the effect of the approval.

**Veto trigger**: Office receives a `G approve` event from Stream and determines
that the approval violates a policy rule — for example, the approving Actor lacked
the required capability token, or the chunk contains content prohibited by the
Works constitution.

**Veto stream event**:
```
<ts> S veto <chunk_id> {"vetoed_event":"<G approve stream_event_id>","reason":"actor lacked approve authority","policy":"capability-token-enforcement"}
```

**Veto sequence**:
1. Office emits `S veto` referencing the G approve `stream_event_id`.
2. Office notifies Gate of the veto, referencing the chunk id.
3. Gate treats the chunk as if the G approve had been a G reject.
4. Gate emits `G reject` with `reason: office-veto` and the veto event id in ctx.
5. If the chunk has already moved to Dispatch (the veto arrived after the chunk
   moved but before delivery), Dispatch MUST halt delivery, return the chunk to
   Gate, and Gate emits `G reject`.
6. If the chunk has already been delivered by Dispatch, the veto is recorded but
   cannot reverse the physical delivery. Office records a compliance violation in
   its audit log.

**Veto is exceptional**: the veto mechanism exists for policy enforcement, not for
routine Gate operation. A high rate of Office vetoes indicates a misconfigured
capability token system or a Gate implementation that is not validating tokens
before accepting decisions. Both SHOULD be addressed at the source, not managed
through veto.

**Veto window**: Office SHOULD apply vetoes within a configured window after the
G approve event — the practical limit before Dispatch may complete delivery.
A veto issued after Dispatch confirms delivery is a post-delivery compliance event
and MUST be handled as described in step 6 above.

---

## 8. Compound Gate Decisions

A single Gate interaction may cover multiple chunks simultaneously — for example,
a Barrel that submits a batch of documents for approval, or a Mill that produces
multiple output chunks in one run.

**Batch approve**: a single Actor decision covers N chunks. One G approve event is
emitted per chunk. The events share the same timestamp and Actor but have distinct
`chunk_id` values. Each chunk's provenance is updated independently.

**Batch reject**: same pattern — one G reject per chunk.

**Atomic batch (all-or-nothing)**: a Barrel or Actor MAY require that a batch of
chunks be approved or rejected as a unit — if any one fails, all fail. This is
declared in the Gate submission:

```json
{"batch_id": "<uuid>", "atomic": true, "chunks": ["<id1>", "<id2>", "<id3>"]}
```

For atomic batches:
- Gate emits one G event per chunk on approval, all sharing a `batch_id` in ctx.
- If any chunk in an atomic batch is rejected or vetoed by Office, Gate MUST
  reject all remaining chunks in the batch, emitting G reject for each.
- The batch_id links the events in Stream for audit reconstruction.

The BitPads Task block MAY encode a batch by repeating the instruction byte and
payload for each chunk in sequence, with a compound mode marker `1111` preceding
the batch to indicate atomic treatment.

---

## 9. Gate Recovery

Gate has no persistent storage. On restart, Gate reconstructs its state entirely
from Stream replay.

**Recovery procedure**:
1. Gate subscribes to Stream Replay from the timestamp of the last known startup.
2. For each G event in the replay:
   - `G hold`: set hold flag on the referenced chunk (if the chunk is still in
     Mill egress — confirmed by checking Mill state or asking Mill).
   - `G approve / archive / reroute`: the chunk should have moved. If it did not
     (split-brain), Gate initiates the move.
   - `G reject`: hold flag cleared. No action required — chunk is in egress,
     unblocked.
3. Gate subscribes to `S veto` events and rebuilds any pending veto state.
4. Gate signals recovery complete by emitting `S session-open` with
   `actor: automated:gate` and `trigger: recovery`.

After recovery, Gate resumes normal operation. Any chunks that were in held state
before the restart are still held — their hold flags are reconstructed from Stream.
Human Actors who were waiting on held chunks are notified via the Bus.

---

## 10. Conformance Requirements

| Requirement | Section | Level |
|-------------|---------|-------|
| G event written to Stream Log before chunk moves | 2.1, 6 | MUST |
| G event durably on Log before any provenance update or chunk movement | 3.1 | MUST |
| G approve requires actor with gate.approve token permission | 2.3 | MUST |
| G archive requires actor with gate.archive token permission | 2.3 | MUST |
| G reject requires actor with gate.reject token permission | 2.3 | MUST |
| Any Actor may hold; terminal decisions require explicit token authority | 2.3 | MUST |
| G reject leaves chunk in Mill egress unchanged | 3.3 | MUST |
| reason SHOULD be present on G reject | 3.3 | SHOULD |
| Hold flag reconstructed from Stream on Gate restart | 4, 9 | MUST |
| Held chunk MUST NOT be moved by any building other than Gate | 4 | MUST |
| Gate MUST accept plain JSON instructions | 5.2 | MUST |
| Gate SHOULD accept BitPads Task block instructions | 5.2 | SHOULD |
| Office veto handled: G reject emitted, Dispatch halted if in-flight | 7 | MUST |
| Batch atomic: all-or-nothing on reject or veto | 8 | MUST |
| Gate recovery via Stream replay on every restart | 9 | MUST |

---

*Works-Standard 06-gate-protocol — end of document*  
*Next: `docs/09-extension.md`*
