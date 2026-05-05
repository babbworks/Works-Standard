# Works-Standard — 10: Compliance

**Document:** 10-compliance  
**Version:** 0.1-draft  
**Status:** Draft  
**Date:** 2026-05-05  
**Depends on:** 00-concepts.md through 09-extension.md

---

## 1. Scope

This document specifies what it means for a Works installation to be conforming.
It consolidates the conformance requirements distributed across all prior documents
into a single reference and adds implementation-level requirements that apply across
documents.

A Works-compliant installation is one that satisfies all MUST requirements in this
document and in the referenced section documents. Satisfying SHOULD requirements is
recommended but not required for the conformance claim. MAY requirements are
optional enhancements.

This document does not describe how to build Works — that is outside the standard's
scope. It describes what a built Works installation must be capable of, and what it
must never do.

---

## 2. Conformance Levels

This standard uses three conformance levels per RFC 2119:

**MUST** / **MUST NOT** — unconditional. A Works installation that violates a MUST
or MUST NOT requirement is not conforming, regardless of context or configuration.

**SHOULD** / **SHOULD NOT** — strong recommendation. Departures are permitted only
when the installation explicitly documents the reason and what equivalent guarantee
is provided in its place.

**MAY** — optional capability. Absence does not affect conformance.

An installation that satisfies all MUST/MUST NOT requirements is a **conforming
Works installation** and may claim compliance with Works-Standard v0.1.

An installation that also satisfies all SHOULD/SHOULD NOT requirements is a
**fully conforming Works installation**.

---

## 3. Core Invariants

The following requirements are not derived from any single building or protocol —
they are properties of the installation as a whole. They take precedence over all
other requirements when there is apparent conflict.

### 3.1 Pacioli Invariant

**MUST**: every state change in the compound is recorded on the Stream Log before
it takes effect. No state change is final until its Stream event exists.

**MUST**: the Stream Log is append-only. No line in the Stream Log is modified or
deleted during normal operation. `stream reset` is the only permitted destructive
operation and it MUST emit an `S reset` event before truncating.

**MUST**: the full operational state of the compound is reconstructible by replaying
the Stream Log from ts=0 to present, applying each event in order. No auxiliary
state store is required for recovery — the Log alone is sufficient.

**MUST NOT**: any building maintain state that cannot be recovered from Stream Log
replay. If a building holds state that Stream cannot reconstruct, it is not
conforming.

### 3.2 Right of Assembly (Gate Invariant)

**MUST**: every chunk that leaves the compound (reaches Dispatch) carries a `G approve`
event in its provenance chain. No chunk exits without Gate authorization.

**MUST NOT**: any conveyance path exist that allows a chunk to reach Dispatch without
passing Gate. This prohibition applies at the implementation level — a conforming
implementation MUST make this path structurally impossible, not merely policy-blocked.

### 3.3 Identity Invariant

**MUST**: every action in the compound is attributed to a named Actor. Anonymous
actions are not permitted. A request without a valid Actor identity MUST be rejected.

**MUST**: every Stream event carries the identity of the Actor that caused it in its
`ctx` field. A Stream event without an Actor attribution is not conforming.

### 3.4 Immutability Invariant

**MUST**: once a chunk is committed to a Store (receives a Store address), its `id`
and `content` fields are immutable. Modification creates a new chunk with a new `id`
and provenance linking to the prior version.

**MUST**: Vault entries are immutable and permanent. No building may modify or remove
a Vault entry. Vault entries are read-only once written.

---

## 4. Stream Compliance

| Requirement | Source | Level |
|-------------|--------|-------|
| Stream Log is append-only during normal operation | 02 §2.2 | MUST |
| Every log line is a valid Hollerith record: `ts op a b c` | 02 §3 | MUST |
| `ts` is a Unix timestamp integer; strictly non-decreasing | 02 §3 | MUST |
| `op` is a single uppercase ASCII letter | 02 §3 | MUST |
| First line of every log is an H header at ts=0 | 02 §3.2 | MUST |
| Log is sortable by `sort -n -k1` without modification | 02 §3.3 | MUST |
| Stream Bus provides real-time event delivery to subscribers | 02 §5 | MUST |
| Bus supports at least plain-text Hollerith encoding | 02 §5.3 | MUST |
| Bus SHOULD support BitPads encoding | 02 §5.3 | SHOULD |
| Plain-text Bus over non-local network MUST use TLS | 02 §5.3 | MUST |
| Replay delivers all events in ts order from requested ts | 02 §6 | MUST |
| Replay emits synthetic `replay-complete` S event at end | 02 §6.4 | MUST |
| `S reset` emitted before log truncation | 02 §2.4 | MUST |
| stream_event_id is deterministic from log line content | 02 §4 | MUST |
| Core op codes not repurposed by extension buildings | 09 §5 | MUST |

---

## 5. Chunk Compliance

| Requirement | Source | Level |
|-------------|--------|-------|
| Chunk has all required fields: id, type, content, content_enc, src, address, ts_created, ts_modified, actor, provenance | 01 §2 | MUST |
| `id` is globally unique within an installation | 01 §2 | MUST |
| `content_enc` is one of: text, json, binary, base64 | 01 §2 | MUST |
| Address format is `{store}/{row}/{level}/{bin}` when assigned | 01 §3 | MUST |
| Level is one of: raw, staged, integrated | 01 §3 | MUST |
| Bin is any non-empty string without `/`; human-defined | 01 §4.4 | MUST |
| Bin name changes recorded on Stream | 01 §4.4 | MUST |
| Bin mode changes recorded on Stream | 01 §4.5 | MUST |
| `address` is null when chunk is on Bench or in Mill egress | 01 §3 | MUST |
| Provenance chain: first entry always `action: created` | 01 §5 | MUST |
| Every provenance entry references `stream_event_id` | 01 §5 | MUST |
| Revisions create new chunk with new `id`; prior version preserved | 01 §5.2, 05 §4.6 | MUST |
| Source-mode Bin pull generates copy with new `id`; original stays | 01 §4.5, 05 §4.4 | MUST |
| `S fill` emitted for every fill instruction | 01 §4.5 | MUST |
| Fill MUST NOT operate on Vault entries | 01 §4.5, 05 §6 | MUST |
| Manifest parent written to Vault at Intake decomposition | 01 §6, 05 §4.3 | MUST |
| Vault path A: post-Dispatch archive with `S archive` | 01 §5.3 | MUST |
| Vault path B: Gate `G archive` direct, followed by `S archive` | 01 §5.3, 05 §4.11 | MUST |

---

## 6. Building Compliance

### 6.1 All Buildings

| Requirement | Source | Level |
|-------------|--------|-------|
| Every building validates Actor token before executing any operation | 04 §5.3 | MUST |
| Revoked tokens denied; active sessions closed | 04 §5.3 | MUST |
| Expired tokens treated as revoked | 04 §5.3 | MUST |
| Buildings do not accept requests from unidentified Actors | 04 §2 | MUST |

### 6.2 Intake

| Requirement | Source | Level |
|-------------|--------|-------|
| Every incoming chunk receives a unique `id` at Intake | 03 §3.1 | MUST |
| External Actor identity recorded in `src` and first provenance entry | 03 §3.1 | MUST |
| `I received` emitted before any routing decision | 03 §3.1 | MUST |
| `I routed` emitted before chunk moves to Store or Mill | 03 §3.1 | MUST |
| Intake MUST NOT route directly to Dispatch | 05 §5 | MUST |
| Intake MUST NOT route directly to Vault (except Manifest parent path) | 05 §5 | MUST |

### 6.3 Store

| Requirement | Source | Level |
|-------------|--------|-------|
| Store assigns and records address on every write | 03 §3.2 | MUST |
| Level demotion (integrated→staged, staged→raw) requires Gate authorization | 05 §5 | MUST |
| Vault entries MUST NOT be returned to any building | 05 §5 | MUST |
| Bin mode changes emit Stream event | 01 §4.5 | MUST |

### 6.4 Mill

| Requirement | Source | Level |
|-------------|--------|-------|
| `S mill-start` emitted with input addresses before processing begins | 03 §3.3, 05 §4.5 | MUST |
| `S mill-complete` emitted when output is at egress | 03 §3.3, 05 §4.9 | MUST |
| Mill output waits at egress for Gate; MUST NOT proceed to Dispatch directly | 03 §3.3, 05 §5 | MUST |
| Store inputs at Mill ingress are non-destructive; originals unchanged | 05 §4.5 | MUST |
| Mill→Bench intermediate delivery covered by `S mill-complete`; no separate event | 05 §4.8 | MUST |

### 6.5 Gate

| Requirement | Source | Level |
|-------------|--------|-------|
| Gate has no internal storage; hold flag is memory-only on chunk in egress | 03 §3.5, 06 §3.2 | MUST |
| Gate hold flags fully reconstructible from Stream replay on startup | 06 §3.2, 06 §7 | MUST |
| Gate emits Stream event BEFORE chunk moves for every decision | 06 §2 | MUST |
| `G approve` emitted before chunk moves to Dispatch | 05 §4.10 | MUST |
| `G archive` emitted before chunk moves to Vault | 05 §4.11 | MUST |
| `G reroute` emitted before chunk moves to Store | 05 §4.12 | MUST |
| `G reject` leaves chunk in Mill egress; MUST NOT move it | 05 §4.13 | MUST |
| `G hold` places hold flag on chunk; MUST NOT move it | 06 §3.2 | MUST |
| Office veto takes effect on any G approve immediately | 06 §6 | MUST |
| Batch decisions are all-or-nothing | 06 §5 | MUST |

### 6.6 Dispatch

| Requirement | Source | Level |
|-------------|--------|-------|
| Dispatch MUST NOT accept chunk without G approve in provenance | 03 §3.7 | MUST |
| `D dispatched` emitted on successful delivery | 03 §3.7 | MUST |
| `D failed` emitted on delivery failure; chunk returned to Gate | 03 §3.7, 05 §9 | MUST |
| Post-dispatch archive to Vault (`S archive`) on success | 03 §3.7, 05 §4.14 | MUST |

### 6.7 Vault

| Requirement | Source | Level |
|-------------|--------|-------|
| Vault entries are immutable once written | 03 §3.9 | MUST |
| `S archive` emitted when chunk is written to Vault | 03 §3.9 | MUST |
| Vault MUST NOT allow any building to modify or delete an entry | 03 §3.9 | MUST |
| Vault MUST NOT return chunks to any other building | 05 §5 | MUST |

### 6.8 Office

| Requirement | Source | Level |
|-------------|--------|-------|
| Office maintains building registry with full registration history | 09 §6 | MUST |
| `S register` emitted on successful extension registration | 09 §6 | MUST |
| `S register` MUST NOT be removed from Stream after registration | 09 §7.4 | MUST |
| `S deregister` emitted on deregistration; `S register` preserved | 09 §7.4 | MUST |
| `S token-issued` emitted when capability token is created | 04 §5.3 | MUST |
| `S token-revoked` emitted when token is revoked | 04 §5.3 | MUST |
| Office provides specific failure reasons on declaration rejection | 09 §6 | MUST |
| Office veto references G approve stream_event_id | 06 §6 | MUST |

---

## 7. Actor Compliance

| Requirement | Source | Level |
|-------------|--------|-------|
| Every action attributed to a named Actor | 04 §2 | MUST |
| Anonymous actions rejected | 04 §2 | MUST |
| Every Actor operates within a session | 04 §2 | MUST |
| Bench cleared on session end regardless of cause | 04 §4.1 | MUST |
| `S session-open` emitted at session start | 04 §4.2 | MUST |
| `S session-close` or `S session-interrupt` emitted at session end | 04 §4.2 | MUST |
| Agent Actor carries name, model, context_limit | 04 §3.2 | MUST |
| Agent Actor identity includes model in Stream ctx | 04 §3.2 | MUST |
| Agent Actor MUST NOT self-approve at Gate without explicit token grant | 04 §3.2 | MUST |
| Automated Actor MUST NOT self-approve at Gate | 04 §3.3 | MUST |
| Automated Actor MUST NOT escalate capabilities at runtime | 04 §3.3 | MUST |
| Automated Actor on Gate hold MUST notify Human and suspend | 04 §3.3 | MUST |
| External Actor interacts only through Intake and Dispatch | 04 §3.4 | MUST |
| External Actor MUST NOT access Stores, Mills, Barrels, or Bench directly | 04 §3.4 | MUST |
| Token permissions are additive — not granted means not permitted | 04 §5.2 | MUST |

---

## 8. Conveyance Compliance

| Requirement | Source | Level |
|-------------|--------|-------|
| Every chunk movement recorded on Stream before the move | 05 §2 | MUST |
| Every conveyance appends a provenance entry with stream_event_id | 05 §2 | MUST |
| No chunk reaches Dispatch without G approve in provenance | 05 §4.10 | MUST |
| No chunk enters Vault except via Gate, Dispatch, or Intake Manifest path | 05 §4.3, 4.11, 4.14, 4.15 | MUST |
| External Actors enter only through Intake | 05 §5 | MUST |
| Bench pulls from source-mode Bins generate copies, not removals | 05 §4.4 | MUST |
| Store → Mill reads are non-destructive; originals unchanged | 05 §4.5 | MUST |
| Bench → Store commit creates new chunk if content revised | 05 §4.6 | MUST |
| Gate reject leaves chunk in Mill egress; does not move it | 05 §4.13 | MUST |
| Failed Log write prevents conveyance from proceeding | 05 §8 | MUST |
| Split-brain state resolved by completing the move on recovery | 05 §9 | MUST |
| Multi-Store Mill run records all input addresses in `S mill-start` | 05 §7 | MUST |
| Vault entries are never returned to any building | 05 §5 | MUST |

---

## 9. Extension Building Compliance

| Requirement | Source | Level |
|-------------|--------|-------|
| Unregistered buildings participate only as External Actors | 09 §4.5 | MUST |
| Every extension building declares role, op_codes, stream_events | 09 §4.1 | MUST |
| Extension building emits only declared op codes | 09 §8 | MUST |
| Office provides specific failure reasons on declaration rejection | 09 §6 | MUST |
| `S register` emitted on successful registration | 09 §6 | MUST |
| `S register` never removed from Stream after registration | 09 §7.4 | MUST |
| `S deregister` emitted on deregistration; `S register` preserved | 09 §7.4 | MUST |
| Op code assignment is installation-local in v0.1 | 09 §5 | MUST |
| Core op codes MUST NOT be used for new semantics by extensions | 09 §5 | MUST |
| output-role extensions declare gate_dependency: true | 09 §4.1 | MUST |
| Breaking changes require new registration, not version update | 09 §7.2 | MUST |
| Registered building continuously satisfies registration requirements | 09 §8 | MUST |

---

## 10. Prohibited Implementations

The following implementation patterns are incompatible with Works conformance.
Any installation that permits any of these is not conforming, regardless of other
capabilities.

| Prohibited pattern | Reason |
|--------------------|--------|
| Mutable Stream Log (edit or delete any log line) | Breaks Pacioli invariant and replay guarantee |
| Chunk moves without prior Stream event | Breaks Pacioli invariant; moves unrecoverable |
| Gate bypass (any path from Mill to Dispatch not via Gate) | Violates Right of Assembly |
| Anonymous operations (any action without named Actor) | Violates Identity invariant |
| Vault modification or deletion | Violates immutability invariant |
| External Actor writing directly to Store, Mill, or Barrel | Violates compound perimeter |
| Agent or Automated Actor self-approving at Gate (without explicit token grant) | Violates Actor role boundary |
| Automated Actor acquiring capabilities at runtime beyond its token | Violates capability bound |
| Extension building using an undeclared op code | Violates extension registration contract |
| Stream Log in binary format (not Hollerith text) | Breaks human-readability and sort invariant |
| Level demotion without Gate authorization | Violates chunk immutability in level progression |
| Session without S session-open / session-close or session-interrupt | Breaks Actor attribution on Stream |
| Building that does not validate Actor token before executing operations | Violates capability token contract |

---

## 11. Conformance Claim

An installation that satisfies all MUST requirements in §3–§9 and avoids all
prohibited patterns in §10 MAY claim:

> "This installation conforms to Works-Standard v0.1"

The claim covers the core nine buildings, four Actor types, Stream protocol,
conveyance rules, and extension protocol. It does not cover any specific
application domain, legal jurisdiction, or security certification — Works-Standard
v0.1 is an architectural protocol standard, not a security or compliance standard.

An installation MAY claim partial conformance for a subset of the standard if the
scope is explicitly stated:

> "This installation conforms to Works-Standard v0.1: Stream and Chunk requirements."

Partial conformance claims MUST specify which building and protocol sections are
satisfied. A claim that does not specify scope implies full conformance.

---

## 12. Relationship to Referenced Standards

Works-Standard v0.1 depends on the following companion standards for specific
protocol definitions:

| Standard | Dependency | Where used |
|----------|-----------|------------|
| BitPads-Standard | Wire encoding for Stream Bus and Gate decisions | 02 §5.3, 06 §4 |
| BitLedger-Standard | Conservation invariants in Office audit operations | 07 (forthcoming) |
| WW-Standard | Reference implementation binding | 14 (forthcoming) |

Compliance with Works-Standard v0.1 does not require compliance with BitPads-Standard,
BitLedger-Standard, or WW-Standard. BitPads encoding is SHOULD in the Stream Bus
(plain-text is sufficient for conformance). BitLedger and WW bindings are defined in
forthcoming binding documents and are not required for v0.1 conformance.

---

## 13. Non-Normative Notes

### 13.1 Test vectors

Works-Standard does not specify a test suite in this version. A conforming test suite
for v0.1 SHOULD verify at minimum:

- Log append-only invariant under all operations including failure paths
- Gate cannot be bypassed by any sequence of conforming API calls
- Stream replay from ts=0 reconstructs identical building state to current runtime
- Revoked tokens are denied immediately, including for in-flight operations
- Vault entries cannot be modified or deleted through any API path
- External Actors cannot reach Store, Mill, or Barrel through any conforming API

### 13.2 Partial implementations

An implementation that omits some buildings is permitted if the omitted buildings
are not invoked. An installation with no Barrel building is conforming if it never
routes a chunk through a Barrel path and never claims Barrel capability. The
conformance claim MUST specify which buildings are present.

### 13.3 Single-process installations

Works-Standard does not require buildings to be separate processes or machines.
A single-process installation where all buildings are in-memory modules is
conforming if it satisfies all MUST requirements. The Stream Log MUST be persisted
to durable storage regardless of the single-process architecture — in-memory-only
logs do not satisfy the Pacioli invariant.

### 13.4 Ordering of events within the same timestamp

If two events occur at the same Unix timestamp (second resolution), their relative
order in the Log is determined by write order. Replay processors MUST process
events in Log write order and MUST NOT assume causal ordering among same-timestamp
events. Implementations SHOULD use sub-second timestamps (millisecond precision
appended to the ts field, e.g., `1746403200.394`) when available to reduce
ambiguity. Sub-second precision is not required for v0.1 conformance.

---

*Works-Standard 10-compliance — end of document*  
*Next: `docs/07-office.md`*
