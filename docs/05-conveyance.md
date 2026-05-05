# Works-Standard — 05: Conveyance

**Document:** 05-conveyance  
**Version:** 0.1-draft  
**Status:** Draft  
**Date:** 2026-05-04  
**Depends on:** 00-concepts.md, 01-chunk-format.md, 03-buildings.md, 04-actors.md

---

## 1. Scope

This document specifies conveyance: the rules governing how chunks move between
buildings, what is permitted and what is prohibited, and how every movement is
recorded. It defines the canonical conveyance path, the permitted variations, and
the edge cases that each building section in `03-buildings.md` references.

Conveyance is the operational contract of the compound. The buildings define what
each role does; conveyance defines how they connect.

---

## 2. The Conveyance Principle

Every movement of a chunk between buildings is a conveyance event. Every conveyance
event MUST be recorded on Stream before the chunk moves. The Stream event is the
authorization for the move; without it, the move has not occurred from the
perspective of the Works record.

This applies without exception. A chunk that moves between buildings without a
Stream event — even within the same process, even on the same machine — is not
conforming. The Pacioli guarantee requires that every state change in the compound
is recoverable from Stream replay. Unrecorded movement breaks replay.

A conveyance event also MUST append a provenance entry to the moving chunk,
cross-referencing the Stream event that authorized the move (via `stream_event_id`).
The chunk and the Stream are two sides of the same record — neither is complete
without the other.

---

## 3. The Canonical Conveyance Path

The full lifecycle of a chunk through the compound follows this path:

```
External Actor
     │
     ▼ (data arrives)
  INTAKE
  verify → classify → normalize
     │
     ├──────────────────────┐
     ▼                      ▼
  STORE                   MILL (direct routing — rare)
  (raw level)               │
     │                      │
     ├─── Actor pull ──► BENCH ◄─── Mill intermediate output
     │                      │
     │                      └─── Actor commit ──────────┐
     │                                                  │
     └──── Barrel/Actor invocation ──► MILL             │
                                        │               │
                                   lens pipeline        │
                                        │               │
                                   Mill egress          │
                                        │  ◄────────────┘
                                        ▼
                                      GATE
                                  decision point
                                 ╱    │    ╲    ╲
                          approve  archive reject reroute
                            │        │       │       │
                            ▼        ▼       │       ▼
                        DISPATCH   VAULT  (stays   STORE or
                            │               in     MILL
                            ▼            egress)
                          VAULT
                       (post-dispatch)
```

Every arrow in this diagram is a conveyance event on Stream. There are no shortcuts.

---

## 4. Permitted Conveyance Paths

The following paths are the complete set of conforming chunk movements. Any chunk
movement not listed here is not permitted.

### 4.1 Intake → Store

The primary entry path. Intake receives, verifies, normalizes, and routes a chunk
to a named Store at Level `raw`.

Stream event: `I routed` with `dest` in ctx specifying the Store address.  
Chunk state: `address` set to `{store}/{row}/raw/{bin}`, provenance creation
entry appended.

### 4.2 Intake → Mill

Direct routing from Intake to a waiting Mill ingress, bypassing Store. Used when
a Mill is configured to process incoming data immediately without persistence —
for example, a real-time classification Mill that processes Agent outputs as they
arrive.

Stream event: `I routed` with `dest` specifying the Mill name.  
Chunk state: `address` null (chunk is in Mill ingress, not a Store).  
Note: chunks routed Intake→Mill that produce Content Mill output must still pass
Gate before Dispatch. The bypass is on the input side only.

### 4.3 Intake → Vault (Manifest parent only)

The narrow exception path for Manifest parent records. When Intake decomposes a
Manifest, it writes the original compound record to Vault as a permanent archive
before routing the constituent chunks. This is the only path where Intake writes
directly to Vault.

Stream event: `S archive` with `reason: manifest-parent` in ctx.  
Chunk state: Vault address assigned, provenance entry appended.

### 4.4 Store → Bench

An Actor pulls a chunk from a Store to their Bench for inspection, editing, or
assembly. The chunk's Store address is retained in its provenance but its active
`address` becomes null (Bench has no address).

Stream event: none required for the pull itself — Bench pulls are not Stream
events. The pull is recorded only when the chunk is committed back to a Store
(§4.6). This is the one permitted unrecorded movement: Bench is pre-commitment
working memory, not Works state.  
Chunk state: `address` set to null, `actor` updated.  
Bin mode: if the source Bin is `queue` mode, the chunk is removed from the Bin.
If `source` mode, a copy is created with a new `id` and provenance linking to
the source; the original remains.

### 4.5 Store → Mill

A Barrel or Actor invokes a Mill with chunks from a Store as input. The chunks are
presented at Mill ingress. They are not removed from the Store — Mill reads are
non-destructive. The originals remain at their Store address throughout the Mill run.

Stream event: `S mill-start` records the input Store addresses and Actor.  
Chunk state: originals unchanged in Store; Mill creates new output chunks.

### 4.6 Bench → Store

An Actor commits a chunk from Bench to a Store. This is the moment a chunk enters
Works state — immutability begins here. The chunk receives a Store address for the
first time (if it was created on Bench) or returns to a Store with an updated
address (if it was pulled, modified on Bench, and committed as a revision).

Stream event: `S bench-commit` with source Actor and target address.  
Chunk state: `address` set to target Store address, `ts_modified` updated,
provenance entry appended.  
Note: a chunk committed from Bench to a Store as a revision creates a new chunk
with a new `id`. The prior version is not overwritten — the new chunk carries a
provenance entry with `action: revised-from` and the prior chunk's `id`.

### 4.7 Bench → Mill

An Actor submits chunks from Bench directly to a Mill for processing. Used when
an Actor has assembled content on Bench and wants to run it through a transformation
before committing to a Store.

Stream event: `S mill-start` records Bench source and Actor.  
Chunk state: Bench chunks remain on Bench until the Actor commits; Mill creates
new output chunks at egress.

### 4.8 Mill → Bench

A Mill delivers intermediate output to an Actor's Bench rather than to Gate egress.
Used for iterative assembly — the Actor reviews Mill output, modifies on Bench, and
either commits to Store or sends back through Mill.

Stream event: none — intermediate Mill→Bench delivery is internal to the Mill run
and covered by `S mill-complete`.  
Chunk state: output chunks have null address, provenance linking to input chunks.

### 4.9 Mill → Gate (egress)

Content Mill output accumulates at Mill egress and awaits Gate decision. The chunks
are not in any Store — they are in egress, which is logically part of the Mill.
Gate receives notification that output is waiting.

Stream event: `S mill-complete` indicates output is at egress.  
Chunk state: null address, provenance linking to inputs, `actor` set to the Mill's
identity (e.g. `automated:document-assembly-mill`).

### 4.10 Gate → Dispatch (approve)

Gate issues a `G approve` decision. The chunk moves from Mill egress to Dispatch.

Stream event: `G approve` emitted by Gate before the chunk moves.  
Chunk state: provenance entry appended with Gate decision and approving Actor.

### 4.11 Gate → Vault (archive)

Gate issues a `G archive` decision. The chunk moves from Mill egress directly to
Vault without passing through Dispatch. Used for productive work that completes
inside the compound with no external delivery required.

Stream event: `G archive` emitted by Gate, followed by `S archive` when the chunk
is written to Vault.  
Chunk state: Vault address assigned, provenance entry appended.

### 4.12 Gate → Store (reroute)

Gate issues a `G reroute` decision. The chunk is redirected from Mill egress to a
named Store rather than Dispatch or Vault. Used when Gate determines the chunk
needs further work — more processing, additional content, or a different Mill.

Stream event: `G reroute` with destination Store address in ctx.  
Chunk state: `address` set to the reroute Store address at Level `raw`,
provenance entry appended recording the reroute decision and reason.

### 4.13 Gate → Mill (reject back to egress)

Gate issues a `G reject` decision. The chunk remains in Mill egress with a cleared
hold flag. Gate does not move the chunk — it stays where it was. The Barrel or
Actor that submitted to Gate is notified of the rejection and decides what to do:
reprocess, reroute manually, or discard.

Stream event: `G reject` with reason in ctx.  
Chunk state: unchanged address (still in Mill egress), provenance entry appended.

### 4.14 Dispatch → Vault

After Dispatch successfully delivers a chunk to its external destination, the chunk
is archived to Vault automatically. This is the post-dispatch archive (Path A from
`01-chunk-format §5.3`).

Stream event: `D dispatched` followed by `S archive`.  
Chunk state: Vault address assigned, provenance entry appended.

### 4.15 Store → Vault (via Gate archive)

An `integrated` chunk in a Store may be archived to Vault via a Gate `archive`
decision without passing through Dispatch. A Barrel submits the chunk to Gate;
Gate issues `G archive`; the chunk moves to Vault.

Stream event: `G archive` then `S archive`.  
Chunk state: Vault address assigned. The Store address in the provenance chain
remains as historical record — it is not cleared.

---

## 5. Prohibited Paths

The following movements are explicitly prohibited in a conforming Works installation.
An implementation that permits any of these paths is not conforming.

| From | To | Prohibition |
|------|----|-------------|
| Any building | Dispatch (without Gate approve) | Right to Assembly — Gate is mandatory |
| Any building | Vault (except via Gate, Dispatch, or Intake Manifest path) | Vault entry is controlled |
| Bench | Dispatch | Bench content must pass Store and Gate |
| Bench | Vault | Bench content must pass Store and Gate |
| Vault | Any building | Vault is read-only and permanent |
| External Actor | Store (directly) | External Actors enter only through Intake |
| External Actor | Mill (directly) | External Actors enter only through Intake |
| External Actor | Vault (directly) | External Actors enter only through Intake |
| Mill | Dispatch (directly) | Gate is mandatory between Mill and Dispatch |
| Intake | Dispatch | No data enters and immediately leaves without Works processing |
| Store | Dispatch (directly) | Store output must pass Mill and Gate |
| integrated chunk | staged or raw (without G reject/reroute) | Level demotion requires Gate authorization |

---

## 6. The Fill Conveyance

Fill is a special conveyance operation that populates a Bin with copies of a source
chunk. It does not move a chunk to a new building — it multiplies a chunk within
a Store. Fill is invoked by a Barrel instruction or directly by an Actor with fill
permission on the target Store.

```
fill {store}/{row}/{bin} with {chunk_id}, count=N|∞
```

**`count=N`** (positive integer): N new chunks are created in the target Bin,
each with a new `id`, the same `content` as the source, and a provenance entry
with `action: filled-from` linking to the source chunk id. The Bin mode is set to
`filled`. Stream event: `S fill` with source id, target address, count, and Actor.

**`count=∞`**: the target Bin is set to `source` mode. No copies are created at
fill time — copies are generated on demand at pull time. Stream event: `S fill`
with `count: infinite` and Actor.

Fill MUST NOT be executed on a Vault entry. Fill operates only on Store Bins.

The source chunk MUST be in the same installation. Cross-installation fill (copying
a chunk from one Works installation into a Bin of another) is performed via Dispatch
→ Intake, not via fill.

---

## 7. Multi-Store Conveyance

A Mill run MAY draw from multiple Stores simultaneously. A Barrel MAY write to
multiple Stores in sequence. These are conforming multi-Store operations:

**Mill with multi-Store ingress**: the Mill is configured with multiple input
Store addresses. `S mill-start` records all input addresses. The Mill combines
chunks from multiple Stores in its lens pipeline. Output is a new chunk at egress
that carries provenance linking to all input chunks from all Stores.

**Barrel with multi-Store writes**: a Barrel instruction sequence may write to
different Stores at different steps. Each write emits its own `S barrel-step` and
`S store-write` events. The writes are independent — failure of one step does not
automatically roll back prior writes, but the Barrel MUST emit `S barrel-fail`
and the compound audit record shows exactly which steps succeeded.

**Cross-installation conveyance**: chunks moving from one Works installation to
another travel via Dispatch (outbound) and Intake (inbound) of the receiving
installation. The receiving Intake treats the incoming chunk as an External Actor
submission — it verifies, classifies, and routes. The chunk's original provenance
chain is preserved in the `content` of the newly created chunk in the receiving
installation, but the receiving installation creates a new chunk id and starts
a new provenance chain. Cross-installation provenance linking is via the
`manifest_id` field if the transfer is compound, or via `src` field and content
if the transfer is a single chunk.

---

## 8. Conveyance and Stream Ordering

Stream records conveyance events in append order. When two conveyance events occur
simultaneously (e.g., a Barrel writing to two Stores in rapid succession), their
relative Stream order is determined by write order to the Log. Replay processors
MUST process events in Log order and MUST NOT assume that events within the same
timestamp are causally ordered relative to each other (see `02-stream-protocol §2.4`).

A conveyance event MUST be written to the Stream Log before the chunk moves.
If the Log write fails, the conveyance MUST NOT proceed — the chunk stays where it
was. A conforming implementation treats a failed Log write as a failed conveyance,
not as a conveyance that happened to not be recorded.

---

## 9. Conveyance Failures

Conveyance can fail at any step. Failure handling follows a consistent pattern:

**Before the Stream event is written**: the conveyance has not begun. No state
changes. The chunk stays at its current address. The building that attempted the
conveyance reports the failure to the Actor or Barrel that invoked it.

**After the Stream event is written but before the chunk moves**: the Stream record
exists but the chunk did not move. This is a split-brain state. A conforming
implementation MUST detect this on next startup via Stream replay (the Stream event
exists but the chunk is not at the recorded destination) and MUST resolve it by
completing the move. The move MUST be completed, not rolled back — the Stream event
is the authority.

**After the chunk moves but before provenance is updated**: the chunk is at its
destination but its provenance chain does not reflect the move. The implementation
MUST complete the provenance update on next access to the chunk. The Stream event
is the authoritative record; the provenance is the derived record.

**Dispatch failure**: Dispatch emits `D failed`, returns the chunk to Gate, and Gate
decides. Dispatch failure does not undo the `G approve` event — the authorization
stands, but the delivery did not complete.

---

## 10. Conformance Requirements

| Requirement | Section | Level |
|-------------|---------|-------|
| Every chunk movement recorded on Stream before the move | 2 | MUST |
| Every conveyance appends a provenance entry with stream_event_id | 2 | MUST |
| No chunk reaches Dispatch without G approve in provenance | 4.10 | MUST |
| No chunk enters Vault except via Gate, Dispatch, or Intake Manifest path | 4.3, 4.11, 4.14, 4.15 | MUST |
| External Actors enter only through Intake | 5 | MUST |
| Bench pulls from source-mode Bins generate copies, not removals | 4.4 | MUST |
| Store → Mill reads are non-destructive; originals unchanged | 4.5 | MUST |
| Bench → Store commit creates new chunk if content revised | 4.6 | MUST |
| Gate reject leaves chunk in Mill egress; does not move it | 4.13 | MUST |
| Failed Log write prevents conveyance from proceeding | 8 | MUST |
| Split-brain state resolved by completing the move on recovery | 9 | MUST |
| Fill operates only on Store Bins, not Vault | 6 | MUST |
| S fill emitted for every fill instruction | 6 | MUST |
| Multi-Store Mill run records all input addresses in S mill-start | 7 | MUST |
| Vault entries are never returned to any building | 5 | MUST |

---

*Works-Standard 05-conveyance — end of document*  
*Next: `docs/06-gate-protocol.md`*
