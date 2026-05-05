# Works-Standard — 01: Chunk Format

**Document:** 01-chunk-format  
**Version:** 0.1-draft  
**Status:** Draft  
**Date:** 2026-05-04  
**Depends on:** 00-concepts.md, 02-stream-protocol.md

---

## 1. Scope

This document specifies the canonical chunk format: the fields every chunk must
carry, the three-dimensional address system that locates chunks within Stores, the
provenance chain that records a chunk's complete history, and the Manifest compound
chunk format.

This document does not specify conveyance rules — which buildings send chunks to
which, and under what conditions. That is `05-conveyance.md`. It does not specify
building-level chunk handling (how a Store writes, how a Mill reads). That is
`03-buildings.md`. This document specifies what a chunk is and what it must contain,
independent of where it is or how it got there.

---

## 2. The Chunk

A chunk is the atomic unit of content in the Works compound. Every piece of
information that enters, moves through, or leaves the compound is a chunk.

**Atomic**: a chunk is indivisible for the purposes of Works conveyance. If content
must be split for separate processing, each part becomes a new independent chunk
with provenance entries linking back to the original. The original chunk is not
deleted — it acquires a provenance entry recording that it was the source of the
split.

**Chunk vs. Stream event**: these are distinct concepts that cross-reference each
other. A Stream event is a record in the Log layer — a Hollerith-encoded line
documenting that something happened. A chunk is a content unit — the thing that
something happened to. Most chunk actions generate a Stream event; that event's
identifier is recorded in the chunk's provenance chain. Stream events do not contain
chunk content; chunks carry a provenance reference to the Stream events that
document their history.

**Immutability**: a chunk's `content` field never changes after the chunk is first
written to a Store. What changes over a chunk's lifetime is its address (as it moves
between Levels and Stores) and its provenance chain (as new actions are recorded).
An implementation that modifies a chunk's content field in place is not conforming.
Content revisions create new chunks with provenance linking back to the prior version.

---

## 3. Chunk Fields

Every chunk in a conforming Works installation MUST carry the following fields.
Fields marked MUST are required at all times. Fields marked COND are required
when the stated condition is true.

| Field          | Type                 | Level    | Meaning |
|----------------|----------------------|----------|---------|
| `id`           | string               | MUST     | Unique chunk identifier within the compound |
| `src`          | string               | MUST     | Source building or adapter that created the chunk |
| `type`         | Op code char         | MUST     | Op-code classification of chunk content |
| `content`      | any                  | MUST     | The chunk's actual data |
| `content_enc`  | string               | MUST     | Content encoding: `"text"`, `"json"`, `"binary"`, `"base64"` |
| `address`      | StoreAddress         | COND     | Current Store address; null or absent when chunk is on Bench |
| `provenance`   | array of ProvenanceEntry | MUST | Ordered history of all actions on this chunk |
| `ts_created`   | integer              | MUST     | Unix timestamp when chunk first entered the compound |
| `ts_modified`  | integer              | MUST     | Unix timestamp when chunk's address or stage last changed |
| `actor`        | string               | MUST     | Actor identity string of the Actor who last acted on this chunk |
| `manifest_id`  | string               | COND     | Parent Manifest id; required when chunk was decomposed from a Manifest |

### 3.1 The `id` Field

A chunk's `id` MUST be unique within the Works installation for the lifetime of
the installation. Two id generation strategies are conforming:

**UUID v4**: a randomly generated 128-bit identifier in standard hyphenated string
form (`xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx`). Appropriate for chunks created
interactively or by human Actors.

**Content hash**: SHA-256 of the concatenation of `src + ts_created + content`,
represented as a lowercase hexadecimal string. Appropriate for chunks created by
automated ingestion where the same source content should always produce the same
chunk id (enabling idempotent ingestion).

A conforming implementation MUST NOT reuse a chunk id, even after a chunk has been
archived to Vault.

### 3.2 The `type` Field

The `type` field MUST be a single uppercase ASCII character matching a registered
Works op code (`T`, `F`, `B`, `D`, `H`, `S`, `A`, `I`, `G`, or a registered
extension op code). The `type` field ties the chunk to the Stream vocabulary:
a chunk of type `T` contains task lifecycle content; a chunk of type `A` contains
annotation content; and so on.

The `type` field determines which Store Bin the chunk occupies in its address
(see §4.3).

### 3.3 The `content` Field

The `content` field carries the chunk's actual data. Works places no constraint on
the data type or structure of `content` beyond what is implied by the `type` and
`content_enc` fields.

`content_enc` MUST declare how the content is encoded:
- `"text"`: UTF-8 plain text; no additional processing required to read
- `"json"`: a JSON value (object, array, string, number, or boolean)
- `"binary"`: raw binary data; implementation-defined handling
- `"base64"`: binary data encoded as Base64 (RFC 4648); decode before use

A conforming implementation MUST be able to round-trip content through
serialization and deserialization without data loss for all four encoding types.

### 3.4 The `actor` Field

The `actor` field MUST be a non-empty string identifying the Actor who last acted
on this chunk. Actor identity strings use a `type:name` format:

| Format             | Actor type | Example |
|--------------------|------------|---------|
| `human:<name>`     | Human      | `human:mp` |
| `agent:<name>`     | Agent      | `agent:claude-sonnet-4-6` |
| `automated:<name>` | Automated  | `automated:nightly-export` |
| `external:<id>`    | External   | `external:bitledger-node-01` |

The `actor` field MUST be updated on every stage transition and every Bench
operation. A chunk that has never been touched by an Actor (i.e., a chunk
programmatically generated and immediately written to a Store) MUST carry the
identity of the Automated Actor or Agent Actor responsible.

---

## 4. Chunk Address — Row × Level × Bin

A chunk's address is a three-dimensional coordinate that locates it within a named
Store. The address system is named for Babbage's Analytical Engine Store, which
used a physical row-and-column addressing scheme for its number columns. Works
extends this to three dimensions suited to information lifecycle management.

A chunk's full address has four components:

```
{store}  /  {row}  /  {level}  /  {bin}
```

As a human-readable path string: `S-finance/invoice-2026-001/staged/contracts`

### 4.1 Store

The Store component is the name of the containing Store — a short string assigned
when the Store is created (`S-01`, `S-legal`, `S-finance`, `main`, `scratch`, etc.).
Store names MUST be unique within an installation. Store names MUST contain only
alphanumeric characters, hyphens, and underscores.

### 4.2 Row

The Row identifies what the chunk is *about* — the entity, task, project, object,
or document that this chunk describes or is part of. Multiple chunks about the same
entity share a Row. A Row is not a single chunk; it is a coordinate that groups
related chunks across Levels and Bins.

Row values MUST be non-empty strings containing no whitespace, no path separators
(`/`), and no embedded newlines. Row values SHOULD be either:
- UUID v4 strings (for system-generated object identity)
- Human-assigned names following the Store name convention
  (`invoice-2026-001`, `project-alpha`, `matter-2026-04`)

A Row is created implicitly when the first chunk with that Row value is written
to a Store. A Row is not deleted when all its chunks are archived to Vault; the
Row coordinate remains valid for provenance cross-reference.

### 4.3 Level

Level is the chunk's lifecycle stage within its Store. Level is always one of
three values:

| Level        | Meaning |
|--------------|---------|
| `raw`        | Arrived from Intake or created by an Actor; unprocessed |
| `staged`     | Assembled or processed by Mill; awaiting Gate authorization |
| `integrated` | Gate-authorized; complete; ready for archival or dispatch |

Level transitions are one-directional under normal operation:
`raw` → `staged` → `integrated`. Reverse transitions are only permitted when Gate
issues a `reject` or `reroute` decision, in which case `staged` MAY return to `raw`
with a G event recorded on Stream.

Every Level transition MUST be recorded on Stream. A chunk that changes Level without
a corresponding Stream event is not conforming.

### 4.4 Bin

Bin is a user-defined organizational label — a human-meaningful name assigned by
the Actor or Barrel that places the chunk in the Store. Bin values are invented on
the fly; no registration or prior declaration is required. A Bin is created
implicitly when the first chunk is written to it.

Bin values MUST be non-empty strings containing no whitespace, no path separators
(`/`), and no embedded newlines. Bin values SHOULD be lowercase and hyphen-separated
for filesystem compatibility (`contracts`, `client-notes`, `q4-invoices`, `scratch`,
`forms`, `correspondence`).

Bin has no required relationship to the chunk's `type` field. A Bin named
`correspondence` MAY contain chunks of type `A` (a letter), `T` (a related task),
and `B` (a billed hour) simultaneously. Mixed-type bins are normal and expected —
the Bin is an organizational decision, not a classification constraint. The chunk's
`type` field carries Op-code classification independently.

A Row MAY contain multiple named Bins — for example, a matter Row in `S-legal`
might have a `correspondence` Bin, a `filings` Bin, and a `notes` Bin, each holding
whatever chunks the user associates with that label.

#### Bin Modes

Every Bin has a mode that governs how chunks are pulled from it. The mode is set
when the Bin is first written to and MAY be changed by an authorized Actor or Barrel
through a Store operation recorded on Stream.

**`queue`** (default) — pull removes the chunk from the Bin. The Bin depletes as
chunks are consumed. Appropriate for work queues, task lists, and single-use items.

**`source`** — pull generates a copy of the chunk (new `id`, provenance linking back
to the source chunk); the original remains in the Bin. The Bin never depletes.
Appropriate for templates, standing forms, reference documents, or any resource
that many Actors or Barrels should be able to use without consuming.

**`filled`** — the Bin is pre-populated with N discrete copies of a chunk via a fill
instruction. Each copy is a full independent chunk with its own `id` and provenance.
The Bin then depletes like a `queue` as copies are pulled. Appropriate for print
runs, form batches, or any scenario where a defined quantity of identical starting
chunks is required.

The fill instruction is a Barrel operation or direct Mill output:

```
fill {store}/{row}/{bin} with {chunk_id}, count=N
fill {store}/{row}/{bin} with {chunk_id}, count=∞
```

`count=∞` sets the Bin to `source` mode. `count=N` (a positive integer) sets the
Bin to `filled` mode with N copies created at instruction time. Both operations
MUST emit an S `fill` event to Stream recording the source chunk id, the target
address, the count, and the authorizing Actor.

### 4.5 The Null Address

A chunk that is on an Actor's Bench has no Store address. Its `address` field MUST
be null or absent. A chunk with a null address is not in Works state — it is in
an Actor's working memory. When the Actor commits the chunk to a Store, the
address is assigned and recorded in the provenance chain.

### 4.6 Address as Filesystem Path

The four-component address format is intentionally compatible with filesystem paths.
A conforming filesystem-backed Store implementation MAY use the address directly as
a directory path:

```
~/works/stores/S-finance/invoice-2026-001/staged/contracts/
~/works/stores/S-legal/matter-23/raw/correspondence/
~/works/stores/S-work/project-alpha/raw/notes/
```

This is a SHOULD-level recommendation, not a requirement. Store implementations
backed by databases or key-value stores MAY use any internal structure, provided
the address is faithfully represented in the chunk's `address` field.

---

## 5. Stage Transitions

Stage transitions are the lifecycle events of a chunk's journey through the Works
compound. Each transition MUST be authorized by the appropriate building and MUST
be recorded on Stream.

### 5.1 Raw → Staged

**Who**: any Actor with write capability on the destination Store, or a Mill
operating on the chunk as part of a run.

**Stream event required**: a T, F, B, A, or S event (matching the chunk type)
MUST be emitted to Stream at the moment of promotion. The event's `action` field
SHOULD be `stage` when there is no more specific action verb.

**Provenance entry**: a ProvenanceEntry (§6) MUST be appended to the chunk's
provenance chain recording the transition, the authorizing Actor, and the
Stream event id of the emitted event.

### 5.2 Staged → Integrated

**Who**: Gate only. No Actor may directly promote a chunk from `staged` to
`integrated` without Gate authorization. This is the Right to Assembly (00-concepts §10).

**Stream event required**: a G `approve` event MUST be emitted to Stream by Gate
at the moment of promotion. The G event's `object` field MUST be the chunk's `id`.

**Provenance entry**: a ProvenanceEntry MUST be appended recording the Gate decision,
the authorizing Actor at Gate, and the Stream event id of the G approve event.

### 5.3 Integrated → Vault

**Who**: Gate, via one of two authorized paths:

**Path A — post-Dispatch archive**: Gate approves a chunk for Dispatch; after
Dispatch confirms successful delivery, the chunk is automatically archived to Vault.
This is the path for content that leaves the compound and is then permanently
recorded.

**Path B — direct archive**: Gate issues an `archive` decision on a chunk at
`integrated` Level without routing it through Dispatch. This is the path for
productive work that completes inside the compound — a finished contract, a closed
project, a finalized report — that does not need to be dispatched externally but
MUST be permanently preserved. A Gate `archive` decision is structurally identical
to a Gate `approve` decision; the difference is destination (Vault vs Dispatch).

An authorized Barrel MAY trigger Gate to issue an `archive` decision as part of a
scheduled archival run (e.g., end-of-quarter project closure). The Barrel cannot
archive directly — it must go through Gate.

**Stream event required**: a G `archive` event MUST be emitted by Gate at the
moment the archive decision is made, followed by an S `archive` event when the
chunk is written to Vault. The chunk's permanent Vault address MUST be recorded
in the provenance chain.

**One-way**: a chunk that has entered Vault MUST NOT return to any Store or appear
on any Bench. Vault is read-only.

### 5.4 Prohibited Transitions

The following transitions MUST NOT occur in a conforming implementation without
the specified authorization:

| From         | To           | Prohibition |
|--------------|--------------|-------------|
| `integrated` | `staged`     | MUST NOT — Gate approval is final |
| `integrated` | `raw`        | MUST NOT — no demotion of authorized chunks |
| Vault        | any Store    | MUST NOT — Vault is permanent |
| Bench        | Vault        | MUST NOT — Bench content must pass through a Store first |

A G `reject` event demotes a `staged` chunk back to `raw` at Gate's direction.
This is the only conforming backward transition. The G reject event MUST be on
Stream before the chunk's Level is changed.

---

## 6. The Provenance Chain

The provenance chain is the ordered, append-only record of every significant action
on a chunk from its creation to its current state. The provenance chain is the
chunk-level expression of the Pacioli guarantee: the present state of a chunk is
derivable from its recorded history.

### 6.1 ProvenanceEntry Format

Each entry in the provenance chain is a ProvenanceEntry record:

| Field              | Type    | Required | Meaning |
|--------------------|---------|----------|---------|
| `ts`               | integer | MUST     | Unix timestamp of the action |
| `action`           | string  | MUST     | What happened (`created`, `staged`, `approved`, `rejected`, `routed`, `archived`, `split-from`, `merged-into`, etc.) |
| `from_address`     | address | COND     | Prior address; null for creation events |
| `to_address`       | address | COND     | New address; null for Bench operations |
| `actor`            | string  | MUST     | Actor identity string of the Actor who performed the action |
| `stream_event_id`  | string  | COND     | Cross-reference to the corresponding Stream Log event (see §6.2) |

The provenance chain MUST be ordered chronologically by `ts`. Entries MUST NOT be
modified or deleted after being appended.

### 6.2 Stream Event Cross-Reference — `stream_event_id`

The `stream_event_id` field links a provenance entry to its corresponding Stream
Log event, creating a bidirectional cross-reference between the chunk record and
the Stream record. This enables complete audit reconstruction from either direction:
given a Stream event, find the chunk; given a chunk provenance entry, find the
Stream event.

**Works-wide config option: event hashing**

When event hashing is enabled at the installation level, the Stream implementation
computes a 12-character event hash for each Log entry at write time:

```
event_hash = SHA-256( full log line text )[:12]
```

The hash is computed over the complete log line including all five fields and the
terminating newline. The hash is not stored in the Log file itself — the Log remains
pure Hollerith text. The hash is made available by the Stream implementation as a
computed property of each event and is used as the `stream_event_id` in provenance
records.

Event hashing is a Works-wide configuration option set in the installation's Office
configuration. When enabled, it applies to all buildings in the installation. When
disabled, the fallback cross-reference format MUST be used.

**Fallback cross-reference: composite key**

When event hashing is disabled, `stream_event_id` MUST use the composite key format:

```
<ts>:<op>:<object>
```

Example: `1746348000:T:9f3c2a1b-4d5e-6789-abcd-ef0123456789`

The composite key is not guaranteed unique when two events share the same timestamp
and op code for the same object. In practice this is rare, but implementations MUST
be aware of this limitation and SHOULD use event hashing in installations where
provenance audit integrity is a compliance requirement.

**Recommendation**: event hashing SHOULD be enabled in any installation that uses
Works for compliance, legal, or financial records. Event hashing MAY be disabled
in lightweight installations (developer environments, personal use) where the
overhead of hash computation is undesirable and strict provenance cross-referencing
is not required.

### 6.3 Creation Entry

Every chunk's provenance chain MUST begin with a creation entry. The creation entry
records the chunk's arrival in the compound:

```json
{
  "ts": 1746348000,
  "action": "created",
  "from_address": null,
  "to_address": null,
  "actor": "external:taskwarrior-hook",
  "stream_event_id": "a3f9c12d8e4b"
}
```

The `stream_event_id` in the creation entry MUST reference the I `received` event
emitted by Intake when the chunk arrived. A chunk whose provenance chain does not
begin with a creation entry linked to an Intake I event is non-conforming — its
origin cannot be traced.

### 6.4 Orphan Detection

A conforming Works installation MUST be able to detect orphaned chunks — chunks
whose provenance chain cannot be traced to an Intake I event via the
`stream_event_id` cross-references. Orphan detection is performed by the Office
building during audit runs.

An orphaned chunk MUST NOT be treated as conforming Works data. It MAY be quarantined
(moved to a designated Store for manual review) or rejected, depending on the
installation's Works constitution.

---

## 7. The Manifest

A Manifest is a compound chunk — a single verified unit containing multiple typed
chunks representing different facets of one operational record. The Manifest is
the Works representation of a BitPads Rich Record.

### 7.1 Manifest Identity

A Manifest is identified by a `manifest_id` — a UUID v4 or content hash generated
at the time the Manifest is created or received. Every chunk that is part of a
Manifest, whether routed intact or decomposed, MUST carry this `manifest_id` in
its `manifest_id` field.

### 7.2 BitPads Rich Record as Manifest Wire Format

The preferred wire format for a Manifest arriving at Intake is a BitPads Rich
Record with category marker `0110` (Rich Log Entry: Value + Time + Task + Note).
BitPads compact mode `1111` indicates that all four constituent fields are present
and CRC-15 verified.

The four constituent fields of a BitPads `0110` record map to Works chunk types:

| BitPads field | Works chunk type | Bin |
|---------------|-----------------|-----|
| Value         | Financial record | `T` (post) |
| Time          | Interval record  | `B` |
| Task          | Task record      | `T` (add/modify) |
| Note          | Annotation       | `A` |

Not all four fields need to be present in every Manifest. A Manifest MAY contain
any subset of these types. A BitPads record with compact mode `1111` guarantees
all four are present; other mode values indicate partial records.

See `12-bitpads-binding.md` for the complete BitPads encoding specification.

### 7.3 Intake Routing Options

When a Manifest arrives at Intake, the implementation has two conforming routing
options. The choice is installation configuration, not protocol — both produce
conforming Works state.

**Option A — Route intact**

The Manifest is stored as a single compound chunk in a Store, with `type` set to
`S` (system/compound) and `content` containing the full Manifest structure. The
`manifest_id` is the chunk's own `id`. Intake emits one I `routed` event referencing
the Manifest id.

Intact routing preserves the compound provenance of the original record. Any Actor
or Mill that reads this chunk receives the full Manifest and can decompose it
at will. Intact routing is preferred when the compound relationship between the
constituent records is operationally significant (e.g., a financial record that must
be read alongside its corresponding task and time records to be meaningful).

**Option B — Decompose on arrival**

Intake decomposes the Manifest into its constituent chunks, assigns each a `id`,
sets `manifest_id` on each to the parent Manifest's id, and routes each chunk
independently to its typed Store and Bin. Intake emits one I `received` event
for the Manifest and one I `routed` event per constituent chunk.

Decomposed routing is preferred when the constituent records will be processed
independently by different Mills or stored in different Stores. The `manifest_id`
field on each chunk preserves the compound relationship for later reassembly.

### 7.4 Manifest Provenance in Vault

When a Manifest is decomposed, the parent Manifest record MUST be preserved in Vault
as a compound archive entry, even if none of the constituent chunks are individually
archived to Vault. This ensures that the original compound record — the Manifest
as received — is always recoverable, regardless of what happens to the decomposed
parts.

The Vault entry for a decomposed Manifest is created by Intake at decomposition time.
It MUST carry the original Manifest content (the full BitPads record or its JSON
equivalent), the `manifest_id`, and a provenance chain entry recording the
decomposition event and the ids of all constituent chunks produced.

---

## 8. Chunk Serialization

### 8.1 Internal Format

Inside the Works compound, chunks MUST be serialized as JSON objects. JSON is the
internal format because it is human-readable, inspectable without tools, debuggable
with any text editor, and natively supported by every implementation language.

Binary packing MUST NOT be used for chunk storage inside the compound. The
inspectability requirement — that any authorized Actor can read any chunk's content
without specialized tooling — takes precedence over storage efficiency.

A conforming chunk serialization is a single JSON object that is self-contained:
all fields needed to understand the chunk are present in the object itself. A chunk
MUST NOT require external lookups to be parsed (no foreign key references that must
be resolved before the chunk is readable).

### 8.2 Wire Format at Boundaries

At wire boundaries — Intake receiving data, Dispatch sending data, Bus and Replay
delivery — the encoding is implementation-defined with two conforming options,
following the same pattern as Stream Bus encoding (02-stream-protocol §4.2):

**Preferred: BitPads compact encoding**

Chunks transmitted between Works installations or to external systems SHOULD be
encoded as BitPads records at the wire boundary. BitPads encoding reduces bandwidth
and provides CRC-15 integrity verification without a higher-level checksum.
The full BitPads chunk encoding specification is in `12-bitpads-binding.md`.

**Fallback: plain JSON**

A chunk MAY be transmitted as a plain JSON object (UTF-8 encoded, no compression)
at wire boundaries when the receiving system does not support BitPads, or for
development and debugging purposes. Plain JSON transmission over non-local network
interfaces MUST use TLS.

### 8.3 Minimum Viable Chunk

The smallest conforming chunk — the minimum that a Works implementation must be
able to create, store, and recover — contains:

```json
{
  "id": "9f3c2a1b-4d5e-6789-abcd-ef0123456789",
  "src": "manual",
  "type": "A",
  "content": "First chunk in this Works installation.",
  "content_enc": "text",
  "provenance": [
    {
      "ts": 1746348000,
      "action": "created",
      "from_address": null,
      "to_address": null,
      "actor": "human:mp",
      "stream_event_id": "a3f9c12d8e4b"
    }
  ],
  "ts_created": 1746348000,
  "ts_modified": 1746348000,
  "actor": "human:mp"
}
```

---

## 9. Conformance Requirements

| Requirement | Section | Level |
|-------------|---------|-------|
| Every chunk carries all MUST fields | 3 | MUST |
| Chunk id is unique within installation for lifetime | 3.1 | MUST |
| Chunk content is never modified after first Store write | 2 | MUST |
| `type` field matches a registered op code | 3.2 | MUST |
| `content_enc` declared and accurate | 3.3 | MUST |
| `actor` updated on every stage transition and Bench operation | 3.4 | MUST |
| Bin value contains no whitespace or path separators | 4.4 | MUST |
| Bin mode change recorded on Stream | 4.4 | MUST |
| Fill instruction emits S fill event with source id, count, actor | 4.4 | MUST |
| Address null when chunk is on Bench | 4.5 | MUST |
| Every Level transition emits a Stream event | 5 | MUST |
| Staged → Integrated requires Gate G approve event | 5.2 | MUST |
| Integrated → Vault is one-way and MUST NOT be reversed | 5.3 | MUST |
| Provenance chain is ordered and append-only | 6.1 | MUST |
| Creation entry links to Intake I event via stream_event_id | 6.3 | MUST |
| Orphan detection available to Office | 6.4 | MUST |
| Manifest provenance preserved in Vault on decomposition | 7.4 | MUST |
| Internal chunk storage uses JSON, not binary | 8.1 | MUST |
| Chunks self-contained: no external lookups required to parse | 8.1 | MUST |
| Plain JSON over non-local network requires TLS | 8.2 | MUST |
| Event hashing config applies installation-wide consistently | 6.2 | MUST |

---

*Works-Standard 01-chunk-format — end of document*  
*Next: `docs/03-buildings.md`*
