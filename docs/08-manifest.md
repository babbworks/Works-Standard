# Works-Standard — 08: Manifest

**Document:** 08-manifest  
**Version:** 0.1-draft  
**Status:** Draft  
**Date:** 2026-05-05  
**Depends on:** 00-concepts.md, 01-chunk-format.md, 03-buildings.md, 05-conveyance.md, 06-gate-protocol.md

---

## 1. Scope

This document specifies the Manifest: the Works mechanism for representing compound
records — structured collections of content that travel together as a unit. A
Manifest is how Works handles documents with attachments, multi-part submissions,
batch deliveries, compound responses, and any other case where multiple content
chunks belong together and must be tracked, routed, and delivered as a coherent whole.

The Manifest concept maps to the BitPads Rich Record (bit pattern `0110`) and
provides Works-level semantics for Rich Record handling.

This document covers:
- What a Manifest is structurally
- How Intake handles Manifest decomposition
- How the constituent chunks carry Manifest membership
- How Gate and Dispatch handle Manifest delivery
- The relationship between Manifest routing and ordinary chunk routing

---

## 2. What a Manifest Is

A Manifest is a chunk whose `content` field is a structured index of constituent
chunks that belong together. The Manifest chunk itself is a navigational record —
it points to its constituents but does not contain their content. The constituents
are full chunks in their own right.

Manifest chunk structure (using the standard chunk fields):

```json
{
  "id":          "uuid",
  "type":        "I",
  "content":     {
    "manifest_version": "0.1",
    "label":            "Q2 Financial Report Package",
    "routing":          "intact | decompose",
    "members": [
      {
        "id":      "<chunk_id>",
        "role":    "primary | attachment | appendix | cover | signature",
        "seq":     1,
        "label":   "Executive Summary"
      },
      {
        "id":      "<chunk_id>",
        "role":    "attachment",
        "seq":     2,
        "label":   "Spreadsheet Export"
      }
    ],
    "delivery_constraint": "all | any | primary-only"
  },
  "content_enc": "json",
  "src":         "external:client-erp",
  "address":     null,
  "ts_created":  1746403200,
  "ts_modified":  1746403200,
  "actor":       "intake",
  "provenance":  [...]
}
```

**`routing`**: `intact` means the Manifest and its members travel together as a
unit; `decompose` means Intake breaks the Manifest into individual constituent chunks
that are routed independently with Manifest membership recorded in provenance.

**`delivery_constraint`**: governs what Gate and Dispatch require before delivery
is complete:
- `all` — every member chunk must be approved and delivered; no partial delivery.
- `any` — any approved member may be delivered independently.
- `primary-only` — only the primary-role member must be approved; attachments
  follow the primary's Gate decision automatically.

**`members[].role`**: the relationship of this constituent to the Manifest:
- `primary` — the main content record; exactly one member MUST carry this role.
- `attachment` — supplementary material accompanying the primary.
- `appendix` — reference material; may be delivered separately if constraint allows.
- `cover` — a routing or transmission header record.
- `signature` — an authorization or signature record (e.g., a BitPads-signed block).

**`members[].seq`**: ordering sequence for display and delivery. MUST be unique
within a Manifest. Does not imply dependency — all members are peers unless
`delivery_constraint` restricts.

---

## 3. BitPads Rich Record Mapping

A BitPads Rich Record (bit pattern `0110` in the BitPads record type nibble) is
the wire-level representation that Works maps to a Manifest when received at Intake.

On reception:
1. Intake recognizes the BitPads record type `0110`.
2. Intake reads the Rich Record header: record count, total payload size, per-record
   lengths, per-record type bytes.
3. Intake constructs a Manifest chunk using the Rich Record header as the `content`,
   mapping each sub-record to a `members` entry.
4. Intake creates individual chunks for each sub-record payload.
5. The routing mode (`intact` or `decompose`) is determined by Intake configuration
   and the Rich Record's own routing flag (if present in the BitPads header).

Works MUST NOT require BitPads encoding for Manifest handling. A Manifest may arrive
as a JSON submission at Intake, as a multipart HTTP request, or as any other
structured compound representation. BitPads is the preferred encoding (see
`12-bitpads-binding.md`) but the Manifest concept is encoding-independent.

---

## 4. Intake Handling of Manifests

### 4.1 Detection

Intake detects a Manifest on arrival by:
- BitPads record type `0110` in the wire encoding.
- `Content-Type: application/works-manifest` in HTTP submissions.
- A top-level `manifest_version` field in JSON submissions.
- An explicit `type: "I"` field in a submitted chunk (Intake op code, indicating
  a compound record).

### 4.2 Intact Routing

When `routing: intact`, Intake treats the Manifest and all its members as a single
unit. The Manifest chunk is routed to a Store; all member chunks are routed to the
same Store (SHOULD be the same Bin, MUST be the same Row) and their `address` values
are linked via the Manifest chunk's `manifest_id` field (see §4.5).

Stream events:
```
<ts> I received  <manifest_id> {"type":"manifest","routing":"intact","member_count":3}
<ts> I routed    <manifest_id> {"dest":"S-legal/q2-report/raw/inbox","routing":"intact"}
```

Member chunks receive their own `I routed` events:
```
<ts> I routed    <member_chunk_id_1> {"dest":"S-legal/q2-report/raw/inbox","manifest_id":"<manifest_id>"}
<ts> I routed    <member_chunk_id_2> {"dest":"S-legal/q2-report/raw/inbox","manifest_id":"<manifest_id>"}
```

### 4.3 Decompose Routing

When `routing: decompose`, Intake separates the compound record into its constituent
chunks. Each member chunk is routed independently, potentially to different Stores
or Bins based on Intake's classification logic.

The original compound record (the unsplit Manifest) is written to Vault as a
permanent reference before decomposition:
```
<ts> S archive <manifest_id> {"reason":"manifest-parent","routing":"decompose"}
```

This is the Intake → Vault path specified in `05-conveyance §4.3`. The manifest
parent in Vault is the authoritative record of the original compound structure.

Member chunks carry `manifest_id` in their provenance to indicate that they
originated from a decomposed Manifest:
```json
{
  "action":    "created",
  "ts":        1746403200,
  "actor":     "intake",
  "manifest_id": "<manifest_id>",
  "manifest_role": "primary",
  "stream_event_id": "..."
}
```

### 4.4 Classification During Decomposition

When decomposing, Intake classifies each member chunk independently. The primary
member's classification determines the row and Store routing by default. Attachment
members MAY be routed differently if Intake's classification rules distinguish by
`role`. For example, an Intake configured for a legal compound might route all
`signature` role members to a dedicated signature Store regardless of the primary's
routing.

Classification MUST be applied individually to each member, not only to the
Manifest header. A member chunk's content may change its routing from what the
Manifest header suggests.

### 4.5 The `manifest_id` Field

Every chunk that is part of a Manifest — including the Manifest chunk itself —
SHOULD carry a `manifest_id` field in its provenance chain's creation entry.
The `manifest_id` is the `id` of the Manifest chunk (not the manifest parent in
Vault, which may have a different id).

`manifest_id` enables:
- Queries that retrieve all chunks belonging to a Manifest.
- Gate batch decisions linked by Manifest membership.
- Dispatch to assemble member chunks for delivery.
- Audit reconstruction of the original compound structure.

For intact-routed Manifests, the `manifest_id` is the id of the Manifest chunk
itself. For decompose-routed Manifests, the `manifest_id` is the id of the Vault
parent record.

---

## 5. Mill Handling of Manifests

A Mill MAY receive a Manifest as input:
- All member chunks at Mill ingress simultaneously (intact routing).
- Individual member chunks without awareness of Manifest membership (decompose routing,
  where each member was routed independently to a Store from which Mill reads).

When a Mill receives an intact Manifest, its lens pipeline MUST preserve Manifest
membership in its output: any output chunk produced from a Manifest-level transformation
MUST carry the `manifest_id` of the input Manifest in its provenance.

A Mill that is configured to process member chunks individually (without Manifest
awareness) does not need to handle Manifest metadata — it operates on chunks as
ordinary chunks.

A Mill configured for Manifest-aware processing (for example, a document assembly
Mill that combines a primary with its attachments into a single rendered output)
MUST:
1. Receive all member chunks at ingress before beginning the pipeline.
2. Emit `S mill-start` recording all member chunk ids and the `manifest_id`.
3. Produce output chunk(s) that carry the `manifest_id` in provenance.
4. Emit `S mill-complete` when output is at egress.

---

## 6. Gate Handling of Manifests

Gate decisions on Manifests follow the `delivery_constraint` declared in the
Manifest chunk.

### 6.1 `delivery_constraint: all`

All member chunks must be at Gate egress and approved before any member is delivered.
Gate MUST enforce this as an atomic batch (see `06-gate-protocol §5`):

1. All member chunks accumulate at Gate egress.
2. Gate treats them as a batch keyed on `manifest_id`.
3. A `G approve` on the batch approves all members simultaneously.
4. If any member is rejected, the entire batch is rejected.
5. A hold on any member holds the entire batch.

Stream event:
```
<ts> G approve <manifest_id> {"batch":"manifest","member_count":3,"manifest_id":"<id>","actor":"human:mp"}
```

Each member chunk also receives an individual provenance entry for the approval.

### 6.2 `delivery_constraint: any`

Each member chunk is processed by Gate independently. A member may be approved,
rejected, or held without affecting other members. Delivery is per-member: an
approved member may be dispatched even if sibling members are still pending.

### 6.3 `delivery_constraint: primary-only`

Gate evaluates only the primary-role member. On G approve for the primary:
- All other members receive an automatic `G approve` event attributed to the
  Manifest approval:
  ```
  <ts> G approve <attachment_chunk_id> {"reason":"manifest-primary-approval","primary_id":"<primary_chunk_id>","manifest_id":"<id>"}
  ```
- All members proceed to Dispatch together.

If the primary is rejected, all members are rejected. If the primary is held, all
members hold.

---

## 7. Dispatch Handling of Manifests

Dispatch assembles approved Manifest members for delivery based on the Manifest's
`routing` mode and `delivery_constraint`.

### 7.1 Intact Delivery

For intact-routed Manifests with `delivery_constraint: all`, Dispatch MUST assemble
all member chunks into a compound delivery. The delivery format is determined by
the destination External Actor's protocol:
- BitPads Rich Record (`0110`) for BitPads endpoints.
- Multipart envelope for HTTP/REST endpoints.
- Works-native compound format for Works-to-Works delivery.

Dispatch emits a single `D dispatched` event for the Manifest delivery, referencing
the `manifest_id` and listing all member chunk ids:
```
<ts> D dispatched <manifest_id> {"member_count":3,"members":["<id1>","<id2>","<id3>"],"dest":"external:client-erp"}
```

After dispatch, all member chunks and the Manifest chunk are archived to Vault
as a batch.

### 7.2 Independent Member Delivery

For decompose-routed Manifests or Manifests with `delivery_constraint: any`, each
approved member chunk is dispatched independently. Each dispatch emits its own
`D dispatched` event carrying the `manifest_id` for cross-reference.

The Manifest chunk itself may be delivered as a delivery receipt or index if the
destination protocol supports it, or it may be retained in the compound for
internal reference only.

---

## 8. Manifest Lifecycle Summary

```
External Actor
     │
     │ submits compound record
     ▼
  INTAKE
  detect manifest
     ├── routing: intact ──────────────────────────────────────┐
     │   Manifest + members → same Store Bin                   │
     │   member chunks carry manifest_id                       │
     └── routing: decompose ──────────────────────────────────►│
         Manifest parent → Vault (permanent reference)         │
         member chunks → individual routing paths              │
                                                               │
              [members processed through Store → Mill]         │
                                                               │
                                                               ▼
                                                            GATE
                                              delivery_constraint evaluation
                                              ┌── all:          batch approve
                                              ├── any:          per-member
                                              └── primary-only: cascade
                                                               │
                                                               ▼
                                                          DISPATCH
                                              ┌── intact delivery: compound assembly
                                              └── decompose delivery: per-member
                                                               │
                                                               ▼
                                                            VAULT
                                                  Manifest + all members archived
```

---

## 9. Manifest and Provenance

The provenance chain of every Manifest member carries enough information to
reconstruct the original compound structure:

- `manifest_id` in the creation provenance entry identifies which Manifest this
  chunk belongs to.
- `manifest_role` in the creation provenance entry identifies the member's role
  within the Manifest.
- For decompose-routed Manifests, the Vault parent record (archival Manifest chunk)
  is the authoritative source of the original compound structure.
- The `manifest_id` field on every Gate approval event enables audit reconstruction
  of which members were approved together.

A Works installation that receives a single member chunk with a `manifest_id` in
its provenance SHOULD be able to retrieve all sibling chunks by querying the Store
and Vault for chunks sharing the same `manifest_id`. This query capability is a
SHOULD for conforming implementations.

---

## 10. Conformance Requirements

| Requirement | Section | Level |
|-------------|---------|-------|
| Manifest chunk carries `manifest_version` and `members` array | 2 | MUST |
| Every member has exactly one `primary` role | 2 | MUST |
| `delivery_constraint` declared on every Manifest | 2 | MUST |
| Intake archives manifest parent to Vault before decomposing | 4.3, 05 §4.3 | MUST |
| Every member chunk carries `manifest_id` in creation provenance | 4.5 | SHOULD |
| `delivery_constraint: all` processed as atomic batch at Gate | 6.1 | MUST |
| `delivery_constraint: primary-only` cascades approval to all members | 6.3 | MUST |
| `D dispatched` for intact delivery references `manifest_id` and all member ids | 7.1 | MUST |
| All member chunks archived to Vault after intact dispatch | 7.1 | MUST |
| Works installations SHOULD support manifest_id query across Store and Vault | 9 | SHOULD |
| Manifest handling does not require BitPads encoding | 3 | MUST |

---

*Works-Standard 08-manifest — end of document*  
*Next: `docs/11-stream-lenses.md`*
