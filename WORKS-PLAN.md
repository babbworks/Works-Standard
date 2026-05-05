# WORKS — Strategic Plan
## A Historically Grounded, Globally Viable Standard for the Information Age

> *Version: 0.1-draft — 2026-05-04*
> *Basis: WORKS-OVERVIEW.md, von-neumann-works.md, babbworks/bitpads-standard,*
> *babbworks/bitledger-standard, babbworks/ww-standard*

---

## Preface: Why This Moment

The defining crisis of the current era is not a shortage of computation. It is a
shortage of principled structure around computation. Businesses generate information
at volumes and velocities that outpace every informal system humans have built to
manage it — spreadsheets, folders, email threads, project management tools, chat
archives. And now AI agents, themselves capable of producing vast quantities of
content, are being introduced into these already-overwhelmed systems with no protocol
defining how their work enters, moves through, or leaves an organization.

The result is predictable and universal: fragmented memory (task trackers ≠ time
records ≠ journal notes ≠ financial data); no temporal structure (agents don't
understand work rhythm); no grounding in real operational load; and no audit trail
when something goes wrong.

The solution is not a better application. The solution is a standard — a protocol
that any conforming application can implement, that outlasts any particular
implementation, and that provides structural guarantees rather than optional features.

Works is that standard.

It is not the first attempt at such a standard. What makes this one different is its
lineage: Works is grounded in five centuries of precedent — Pacioli's double-entry
accounting (1494), Babbage's Analytical Engine (1830s), Lovelace's symbolic operations
(1843), Hollerith's punched-card encoding (1890), the industrial timekeeping of Bundy,
Frick, and Dey (late 19th C), and von Neumann's stored-program architecture (1945).
Each contributed a structural principle that Works inherits. None are decoration.

The moment is right because the problem is acute, the precedents are proven, the
reference implementation exists (Workwarrior Stream Service, operational since early
2026), and the companion standards (BitPads, BitLedger) are already published under
the babbworks organization.

---

## Part I: The Standard Family

Works is not a standalone standard. It is the capstone of a family of four standards
published by the babbworks organization, each governing a different layer of the
information compound. Understanding the family is essential to understanding Works.

### The Four Standards

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   WORKS-STANDARD  (this document's subject)                                 │
│   Buildings, conveyance, actors, constitutional guarantees.                 │
│   The compound as a whole.                                                  │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  WW-STANDARD  (github.com/babbworks/ww-standard)                   │   │
│   │  Profile-based shell productivity system. Workwarrior Technical     │   │
│   │  Standard v0.1.0. The reference implementation surface for Works'   │   │
│   │  Stream, Intake (task/timew/jrnl/hledger adapters), and Barrel     │   │
│   │  (heuristic engine, shell wrappers).                                │   │
│   │                                                                     │   │
│   │  ┌─────────────────────────┐  ┌──────────────────────────────────┐ │   │
│   │  │ BITPADS-STANDARD        │  │ BITLEDGER-STANDARD               │ │   │
│   │  │ (babbworks/bitpads-std) │  │ (babbworks/bitledger-standard)   │ │   │
│   │  │ Wire protocol. Compact  │  │ Financial integrity. Conservation │ │   │
│   │  │ binary records 1–29B.   │  │ invariants. 16 universal domain  │ │   │
│   │  │ Works' Intake/Dispatch  │  │ archetypes. Works' Office        │ │   │
│   │  │ wire format.            │  │ reconciliation protocol.         │ │   │
│   │  └─────────────────────────┘  └──────────────────────────────────┘ │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**BitPads** governs how data is packed and transmitted at wire boundaries — 1-byte
heartbeats to 29-byte compound records with CRC-15 integrity. Works uses BitPads at
Intake (receiving Rich Records as Manifests), Stream (bus layer wire format), and
Dispatch (compact output to external systems).

**BitLedger** governs financial data integrity — conservation invariants, the
debit-credit balance, and the 16 Universal Domain archetypes (Asset, Liability,
Equity, Revenue, Expense, and 11 others) that classify financial chunks in conveyance.
Works' Office uses BitLedger invariants to verify that financial chunks entering through
Intake are internally consistent before they are routed to a Store.

**WW-Standard** (Workwarrior Technical Standard v0.1.0, publicly available) governs
the reference shell implementation: profile model, CLI dispatcher, service architecture,
heuristic engine, browser UI, GitHub sync, and the operational patterns for the five
open-source tools WW wraps (TaskWarrior, TimeWarrior, JRNL, Hledger, Bugwarrior). WW
is the first reference implementation of Works' Stream, Intake, and Barrel concepts.

**Works-Standard** (this document's subject) governs the compound as a whole:
buildings, conveyance rules, actor types, chunk anatomy, the constitutional Stream
layer, the Gate authorization boundary, and the extension protocol by which new
building types register and participate.

### Governance Principle

The four standards are governed as a family under babbworks. Each is versioned
independently. Breaking changes to one must be assessed for impact on all others.
The dependency order is bottom-up: BitPads and BitLedger make no assumptions about
Works or WW. WW references BitPads for wire format and BitLedger for financial integrity.
Works references all three.

A conforming Works implementation must implement the Works-Standard. It may optionally
implement WW-Standard (for shell-based actor interfaces), BitPads (for wire-efficient
Intake/Dispatch), and BitLedger (for Office financial reconciliation). None of the
companion standards are hard dependencies — Works can operate without them, using
plain JSON at boundaries and manual audit in place of BitLedger invariants.

---

## Part II: The Architecture in Full

The Works architecture is detailed in WORKS-OVERVIEW.md and the von Neumann
correspondence diagram in von-neumann-works.md. The summary sufficient for planning:

**Nine core buildings**: Intake (verify, classify, route), Store (named chunk
repositories, Row×Level×Bin address), Mill (lens pipeline transformation), Bench
(session workspace), Barrel (stored-program instruction scheduler), Gate (authorization
boundary — nothing leaves without approval), Dispatch (output to external destination),
Office (policy authority, building registry, Actor tokens, audit log), Vault
(permanent append-only archive).

**Four Actor types**: Human (highest default authority, session-scoped), Agent (named
AI agent, context-limited, cannot self-approve at Gate by default), Automated (Barrel
running cron or event-triggered, narrowest capability token), External (enters through
Intake, receives through Dispatch only).

**One constitutional layer**: Stream. Three sub-layers: Log (Pacioli guarantee —
append-only, never delete, replay = truth), Bus (real-time WebSocket/Unix socket
pub/sub), Replay (catch-up for any subscriber). Hollerith encoding: five positional
fields, single uppercase Op code, unix timestamp sort key.

**One extension protocol**: new building types declare, register with Office, emit
to Stream, honor Gate, honor Pacioli guarantee, receive registration token. The
extension protocol is the growth mechanism — the core standard freezes; capabilities
expand through registered extensions.

**The three novel Works additions beyond von Neumann**: Gate (output authorization —
VN had none), Office (policy authority — VN had none), Actors (first-class operators —
VN had none). These three additions are the answer to the question VN never had to ask:
*who is operating this machine, are they authorized, and can we prove it?*

---

## Part III: The Gap This Fills

Before building, we must be precise about what problem Works solves that existing
systems do not.

### What exists

**Event streaming platforms** (Kafka, Pulsar, NATS): high-throughput message buses with
replay. No concept of authorized output. No building registry. No chunk provenance.
No Pacioli guarantee on the event log itself (Kafka topics are retention-bounded,
not append-forever). No Actor model. No Gate.

**Workflow engines** (Temporal, Airflow, Prefect): define business logic sequences with
retry and observability. No persistent chunk store with address-space. No append-only
memory guarantee. No authorized output boundary. Heavy infrastructure.

**Document management systems** (SharePoint, Notion, Confluence): hold documents, not
chunks. No Stream. No Gate. No Actor capability tokens. No extension protocol. Opaque
internals.

**AI agent frameworks** (LangChain, AutoGen, CrewAI): orchestrate LLM agent calls.
No persistent store. No Pacioli guarantee. No Gate between agent output and publication.
No audit trail. No Actor identity model. Agent actions are largely unaudited.

**Double-entry accounting systems** (hledger, GnuCash, QuickBooks): excellent financial
integrity. No general chunk concept. No Stream. No extension protocol. Not designed
for the information-compound model.

### What Works provides that none of these do

1. **Structural audit**: every building action crosses Stream. Audit is not a feature —
   it is a consequence of the architecture. You cannot produce unaudited output.

2. **Authorized output boundary**: Gate between computation and dispatch is a hard
   structural wall. Not a permission check in code — a protocol boundary.

3. **Append-only compound memory**: Pacioli guarantee applies at chunk, Stream, Vault,
   and Office levels simultaneously. State is always reconstructible from replay.

4. **First-class Actor identity**: Human, Agent, Automated, External — all named,
   session-scoped, capability-bounded, and recorded on Stream with every action.
   No anonymous operations.

5. **Extension protocol**: new building types register and participate without changing
   the core standard. Works grows without the standard changing.

6. **Historical coherence**: a Works installation from 2026 can communicate with one
   from 2046. The Hollerith encoding is positional and versioned. The Op codes are
   a fixed-width, extension-registered set. No schema migration.

### The target user

Works is for any organization — small business, professional partnership, research
group, agency, enterprise team — that produces information operationally and needs
principled structure around how that information moves, transforms, and leaves. The
target is particularly acute for organizations beginning to use AI agents operationally,
where the question of *who did what, was it authorized, and can we reconstruct it* has
no current answer.

---

## Part IV: Specification Documents — v0.1 Scope

The following is the complete set of specification documents that constitute
Works-Standard v0.1. These are the artifacts that must be written, reviewed, and
published before v0.1 is declared complete. Each document is a protocol specification —
it defines behavior, not implementation.

### Document Set

```
docs/
  00-concepts.md          Vocabulary, metaphors, definitions of all terms
  01-chunk-format.md      Canonical chunk anatomy, address format, provenance chain
  02-stream-protocol.md   Hollerith encoding, Op codes, Log/Bus/Replay layers
  03-buildings.md         All nine core buildings — role, contract, required behaviors
  04-actors.md            Four Actor types, capability token format, session model
  05-conveyance.md        Conveyance rules, chunk lifecycle, stage transitions
  06-gate-protocol.md     Gate operations, BitPads Task block, approval/reject/hold
  07-office.md            Policy layer, building registry, Actor issuance, audit log
  08-manifest.md          BitPads Rich Record as Works Manifest, compound routing
  09-extension.md         Extension protocol — how new buildings register
  10-compliance.md        Conformance requirements — what a compliant implementation must do
  11-stream-lenses.md     The lens pipeline — 10 inventors, Tiers 0–D, composition rules
  12-bitpads-binding.md   How Works uses BitPads at Intake, Stream bus, Dispatch
  13-bitledger-binding.md How Office uses BitLedger conservation invariants
  14-ww-binding.md        How WW-Standard implements Works Stream, Intake, Barrel
  15-versioning.md        Version scheme, compatibility guarantees, migration path
```

### Writing Sequence

Documents must be written in dependency order:

**Phase 1 — Substrate (write first)**
- `00-concepts.md` — all other docs reference this vocabulary
- `02-stream-protocol.md` — Stream is the constitutional layer everything else depends on
- `01-chunk-format.md` — chunks are the atomic unit of all conveyance

**Phase 2 — Core buildings**
- `03-buildings.md` — defines the nine buildings and their contracts
- `04-actors.md` — defines the four Actor types
- `05-conveyance.md` — defines how chunks move between buildings

**Phase 3 — Authorization and policy**
- `06-gate-protocol.md` — Gate is the authorization boundary
- `07-office.md` — Office governs everything
- `09-extension.md` — the growth mechanism

**Phase 4 — Integration**
- `08-manifest.md` — BitPads Manifest routing
- `12-bitpads-binding.md` — BitPads integration specification
- `13-bitledger-binding.md` — BitLedger integration specification
- `14-ww-binding.md` — WW as Works reference implementation
- `11-stream-lenses.md` — lens pipeline specification

**Phase 5 — Compliance and versioning**
- `10-compliance.md` — conformance requirements
- `15-versioning.md` — version and compatibility policy

### Specification Quality Bar

Each document must:
- Define every term it uses (cross-reference `00-concepts.md` for shared vocabulary)
- Distinguish MUST / SHOULD / MAY behaviors using RFC 2119 language
- Include at least one worked example per major concept
- Be reviewable by someone who has not read any other Works document (self-contained)
- Have no implementation-specific language (Shell, Python, Rust are not in the spec)

---

## Part V: Reference Implementation

The specification defines behavior. The reference implementation proves it is
implementable and provides working code that conforming implementations can test against.

### Existing Implementation (WWSS)

The Workwarrior Stream Service (`services/stream/` in the ww repo) is the direct
predecessor to the Works reference implementation. It implements:

- Stream Log layer (Pacioli guarantee, `stream.log`)
- Hollerith encoding (`<ts> <op> <action> <object> <ctx_json>`)
- Intake adapters for task, timew, jrnl, hledger (`lib/adapters.sh`)
- Lens pipeline: Burroughs, Bundy, Hollerith, Pacioli (`lenses/`)
- Replay engine (`lib/replay.sh`)
- Codec layer — text, JSON, ASCII (`lib/codecs.sh`)
- Hook-based real-time emission (TaskWarrior `on-add`/`on-modify` hooks)
- Stream bus layer: `ww stream ingest`, `ww stream view`, `ww stream emit`

WWSS is a Stream Mill installation — it reads from Stream and produces views. It does
not yet have a Gate, a multi-Store address space, a full Barrel scheduler, an Office,
or a Vault. These are the gaps the reference implementation closes.

### Implementation Language: Rust

Rust is the reference implementation language for the core Works runtime kernel,
grounded in the Babbage-Lovelace framework specification
(`ww-standard/times-research/babbage-lovelace-framework.md`).

Rationale:
- **Typed Op enum**: `enum Op { T, F, B, D, H, S, A, I, G }` — exhaustive matching,
  no stringly-typed dispatch
- **serde**: canonical JSON serialization/deserialization without a runtime
- **Concurrent writer safety**: multiple Actors writing to Stream simultaneously requires
  correct append semantics — Rust's ownership model enforces this structurally
- **ratatui**: TUI for human Actor interfaces (Bench, Stream view, Gate approval)
- **No runtime**: Works must run on minimal infrastructure — a VPS, a Raspberry Pi,
  a developer laptop. Zero-cost abstractions, no GC pauses

The shell implementation (WWSS in bash/Python) remains the WW-Standard reference and
is not replaced. The Rust kernel is the Works-Standard reference. They are companions.

### Rust Module Structure (`works-rs`)

```
works-rs/
  works-core/         Event types, Op enum, chunk format, Hollerith encoding
  works-schema/       Chunk validation, address format, provenance chain
  works-actor/        Actor types, session model, capability token
  works-policy/       Office policy rules engine
  works-stream/       Log layer, Bus layer, Replay layer
  works-registry/     Building registry, extension protocol
  works-office/       Office building implementation
  works-bench/        Bench session workspace
  works-conveyance/   Conveyance rules, stage transition engine
  works-store/        Store implementation (filesystem, SQLite backends)
  works-mill/         Mill + lens pipeline (all 10 lenses)
  works-gate/         Gate building, BitPads Task block instruction
  works-intake/       Intake building + adapter framework
  works-dispatch/     Dispatch building + output adapters
  works-vault/        Vault append-only archive
  works-bundle/       Manifest (BitPads Rich Record) handling
  works-query/        Cross-store chunk query engine
  works-cli/          `works` CLI — all subcommands
```

Each `works-*` crate is publishable to crates.io independently. A minimal installation
compiles only `works-core` + `works-stream` + `works-cli`. A full installation compiles
all 18.

### Implementation Sequence

**Phase A — Stream kernel (weeks 1–3)**
`works-core` → `works-stream` → `works-cli stream` subcommands
Goal: `works stream emit`, `works stream ingest`, `works stream view --lens burroughs`
working end-to-end. Prove the substrate.

**Phase B — Store and Bench (weeks 4–5)**
`works-schema` → `works-store` → `works-bench`
Goal: `works store put <name> <row> <content>`, `works bench pull <store> <row>` working.
Row×Level×Bin addressing functional.

**Phase C — Mill and lens pipeline (weeks 6–8)**
`works-mill` → remaining lenses (frick, baldwin, grant, felt, dey, cooper)
Goal: all 10 lens tiers implemented, composable, testable independently.
Difference Engine finite-difference cascade for Dey and Felt.

**Phase D — Gate and Office (weeks 9–11)**
`works-actor` → `works-policy` → `works-gate` → `works-office`
Goal: `works gate approve <chunk_id>`, `works gate reject <chunk_id> --reason "..."` working.
Office receiving all G events from Stream. Actor capability tokens issued.

**Phase E — Intake, Dispatch, Vault (weeks 12–14)**
`works-intake` → `works-dispatch` → `works-vault` → `works-bundle`
Goal: full end-to-end flow: data enters Intake, routes to Store, runs through Mill,
Gate approves, Dispatch outputs, Vault archives.

**Phase F — Registry and extension protocol (weeks 15–16)**
`works-registry` → extension protocol conformance tests
Goal: a new building type can register with Office and participate in conveyance
using only the published extension protocol, with no changes to core libraries.

**Phase G — Query and bundle (weeks 17–18)**
`works-query` → `works-bundle` (Manifest/BitPads integration)
Goal: `works query --store S-01 --level staged --type T` returns matching chunks.
BitPads Rich Record received at Intake, routed intact as Manifest.

---

## Part VI: Repository Structure

### `github.com/babbworks/works-standard`

The specification repository. No implementation code. Pure documents.

```
works-standard/
  README.md              One-paragraph statement: what Works is, why it exists
  WORKS-OVERVIEW.md      Full prose overview (current doc in babb/repos/works/)
  von-neumann-works.md   VN correspondence diagram
  WORKS-PLAN.md          This document
  docs/                  All 15 specification documents (Phase IV above)
  CHANGELOG.md           Version history — every change to every spec document
  CONFORMANCE.md         How to test that an implementation conforms
  LICENSE                Apache 2.0 (maximally permissive for standards adoption)
```

### `github.com/babbworks/works-rs`

The Rust reference implementation.

```
works-rs/
  README.md              Quick start, build instructions, link to works-standard
  crates/
    works-core/
    works-stream/
    ... (18 crates)
  examples/
    minimal/             Minimal Stream installation — stream.log only
    full/                Full Works installation — all buildings
    ww-integration/      Works + Workwarrior integration example
  tests/
    conformance/         Tests that verify works-rs conforms to works-standard
    integration/         End-to-end flows
  CHANGELOG.md
  Cargo.toml             Workspace manifest
```

### `github.com/babbworks/works-py`

Python reference implementation (secondary). Simpler for contributors and integrators.
Implements the core protocol — works-core, works-stream — not the full 18-crate suite.
Designed for scripting, data analysis, and the existing WW Python layer (browser, adapters).

### `github.com/babbworks/works-examples`

A gallery of Works installations — minimal configurations, industry-specific setups,
integration patterns. Each is a standalone directory with a README showing exactly
what buildings are active, how they are configured, and what the data flow looks like.

---

## Part VII: Adoption Strategy

A standard that does not spread is not a standard — it is a private convention.
Works is designed to spread. The adoption strategy has four vectors, in priority order.

### Vector 1: The Workwarrior Bridge (immediate)

Workwarrior has operational users today. WWSS is running in the ww repo. The immediate
adoption path is: every Workwarrior user who installs `ww stream` becomes a Works user,
whether or not they know the Works name. Works is backwards-compatible with WW — the
WW-Standard is a Works binding.

Action items:
- `ww stream` help text references Works-Standard by URL
- WW-Standard README prominently links Works-Standard
- WWSS is documented as "implements Works Stream and Intake (Workwarrior binding)"
- The lens pipeline is documented as the Works Mill reference implementation

### Vector 2: The Open Standard (months 1–6)

Works-Standard v0.1 published to `babbworks/works-standard` before any implementation
code is merged to `works-rs`. Standards-first means the standard can be implemented
by anyone, in any language, without the Rust codebase.

Action items:
- Publish all 15 spec documents (Phase IV above) — quality bar: RFC-level precision
- Submit Works-Standard to relevant standards discussions (IETF informal, W3C community groups)
- Write a public announcement post: "A standard for the information compound — Works v0.1"
- Encourage any implementor — in any language, any platform — to build against the spec
  and list themselves as a conforming implementation

### Vector 3: Industry Vertical Entry Points (months 3–12)

Works is general but enters through specific industry pain points. The five most
acute entry points:

**1. Small professional services** (legal, accounting, consulting):
These firms have the most acute need for audit trails, authorized output, and
financial integrity. A Works installation for a legal firm has: Intake (email, document),
Store (matter files), Mill (document assembly), Gate (partner approval before filing),
Dispatch (court submission), Vault (permanent record). Every feature is a compliance requirement.

**2. AI agent deployments**:
Any organization deploying AI agents for content creation, research, or analysis needs
exactly what Works provides: named Agent Actors, Gate between agent output and publication,
Stream audit of every agent action, Office capability bounds. The absence of this
structure is the current crisis.

**3. Freelance and independent professionals**:
Workwarrior's existing user base. Works extends WW with the compound model — multiple
Stores for multiple clients, Gate for deliverable approval, Vault for completed project
archive. No new tools required; the WW binding covers this.

**4. Research groups**:
Academic and applied research needs chunk provenance, reproducible Mill pipelines,
and Vault for data archiving. The Pacioli guarantee is the research world's data
lineage requirement.

**5. Small manufacturing and operations**:
The industrial metaphor is not accidental. A Works installation for a small manufacturer
maps directly: Intake (orders, materials), Store (production state), Mill (assembly
pipeline), Gate (quality sign-off), Dispatch (shipping), Vault (batch records).

### Vector 4: The Extension Ecosystem (months 6–24)

The extension protocol is the mechanism by which Works grows beyond what any single
team can build. An extension building type that registers with Office and emits to
Stream is a first-class Works participant.

Target extension classes:
- **Industry-specific Stores**: medical records, legal matters, financial instruments
- **AI-native Mills**: LLM-powered document assembly, classification, summarization
- **Specialized Intakes**: email, GitHub issues, Slack messages, IoT sensor data
- **Compliance Dispatches**: regulatory filing, e-signature, certified delivery
- **Analytics Offices**: Works installations that aggregate Stream data from multiple compounds

Each extension class is a potential open-source project or commercial product.
Works does not need to build them — it needs to provide the protocol that makes them possible.
The babbworks organization publishes the standard and the registry. The ecosystem builds on top.

---

## Part VIII: Governance Model

Works is an open standard. Governance must be structured for long-term credibility —
not corporate-controlled, not anarchic.

### Phase 0: Founding (current)

Single maintainer (Babb). All specification decisions and reference implementation
commits go through the founding maintainer. This is appropriate while the standard is
being defined — multiple voices too early produces incoherence.

### Phase 1: Technical Committee (months 6–12)

When at least two independent conforming implementations exist (in different languages,
by different teams), a technical committee is established. The committee governs
specification changes — PRs to any spec document require TC approval.

TC composition: founding maintainer (permanent seat) + elected members from conforming
implementors. Initial size: 3 members. Maximum size: 7.

### Phase 2: Extension Registry (months 12+)

When the extension ecosystem produces its first external registered building type,
a separate extension registry process is established. New building type registrations
are reviewed by the TC for protocol conformance, not for usefulness — Works does not
judge what buildings are for, only that they follow the protocol.

### Intellectual Property

All Works-Standard documents: Apache 2.0 license (compatible with FOSS and commercial
use; maximally permissive for standards adoption).

All babbworks reference implementations (works-rs, works-py): Apache 2.0.

The name "Works" and "Works-Standard": trademark registered to Babb / the founding
organization. Conforming implementations may use "Works-compatible" or
"implements Works-Standard" but not "Works" alone as a product name.

### Versioning Policy

Works-Standard version numbers follow semantic versioning:
- Major version: breaking change to Stream encoding, chunk format, or building contracts
- Minor version: new buildings, new Op codes, new extension protocol features
- Patch version: clarifications, examples, wording corrections

Major version changes require a 12-month deprecation period during which the previous
major version continues to be supported. Stream replay must remain valid across major
versions — a v1 stream log must be replayable by a v2 implementation.

---

## Part IX: The Historical Argument

Works is not trying to be novel. It is trying to be right. The historical grounding
is not decoration — it is the argument for correctness.

### Pacioli's Principle (1494)

Double-entry accounting introduced the idea that the present state of a system is a
derivation of recorded history, not a stored snapshot. The ledger is never corrected —
only appended to. Pacioli's ledger has been the global financial infrastructure for
530 years because this principle is correct: truth is the cumulative record.

Works applies this principle to all business information, not just financial. Every
Store is a Pacioli ledger. Every Stream is a Pacioli ledger. The Vault is the final
Pacioli ledger. A Works installation from 2026 can answer any question about its
complete operational history — not because it stored snapshots, but because it never
deleted anything.

### Babbage's Architecture (1830s–1871)

Babbage's separation of Store (memory) from Mill (computation) — and the Barrel drum
as stored program — is the first correct architecture for a general-purpose information
machine. Works uses these names because they are right: Stores hold, Mills transform,
Barrels program. The naming honors the priority of invention and signals that Works
is not reinventing architecture — it is applying it.

The Difference Engine's pure-addition cascade is a genuine computational method, not
a metaphor. Works uses it for Dey and Felt lens computation because it works: it
computes polynomial approximations from event streams using only addition, with no
need for floating-point multiplication at the innermost loop.

### Lovelace's Generality (1843)

Note G — the first program — proves that a computation engine operates on symbols,
not just numbers. This is the license for Works' Op enum: T, F, B, D, H, S, A, I, G
are symbols, not integers. The Hollerith encoding is symbolic encoding, not binary
packing. The extension protocol extends the symbol set through registration, not
through redefinition.

Lovelace's CycleMode (Single | Bounded | Perpetual) is Note G's loop concept applied
to Barrel scheduling. The first program was a loop. All automation is a loop.
Works makes this explicit.

### Von Neumann's Bottleneck (1945)

The von Neumann bus bottleneck — all data must cross a single shared bus — is usually
discussed as a limitation to be overcome. Works accepts it and makes it constitutional.
Stream is the bottleneck. All events cross Stream. This costs performance and purchases
three guarantees: auditability (every event is on record), recoverability (any state
can be replayed), and enforceability (every policy rule has one enforcement point).

The three things Works adds beyond von Neumann — Gate, Office, and Actors — are the
response to a question von Neumann never had to ask: who is operating this machine,
are they authorized, and can we prove it years from now? VN was designing for human
operators in a controlled environment. Works is designed for human and AI operators
in an open, adversarial, compliance-governed world.

---

## Part X: What Must Be True for Works to Spread Worldwide

Works spreads if and only if the following conditions are met:

**1. The specification is unambiguous.**
A developer in any country, reading the spec in their second language, must be able
to build a conforming implementation without asking a question. This is the RFC
standard. It requires precise language, worked examples, and a compliance test suite.

**2. The reference implementation is excellent.**
works-rs must be fast, well-documented, and demonstrably correct. It must run on
a Raspberry Pi. It must install in one command. It must produce output that is
immediately intelligible to a non-expert. If the reference implementation is hard to
use, the standard will not be used.

**3. The first use case is undeniable.**
Small professional firms (legal, accounting) and individual AI agent deployments are
the first use cases. These are organizations that have acute pain, understand the
value of audit trails, and will pay for (or contribute to) a solution. The first ten
real Works installations must be documented, published, and compelling.

**4. The extension ecosystem starts early.**
The first non-babbworks extension building type — whether it is an email Intake, an
LLM-powered Mill, or a regulatory compliance Dispatch — validates the extension
protocol and proves that Works grows beyond its founders. This should happen before
Works-Standard v0.2.

**5. The standard family is coherent.**
BitPads, BitLedger, WW-Standard, and Works-Standard must tell a consistent story.
A developer who reads all four must find that they fit together without contradiction.
The binding documents (12, 13, 14) in the spec are the mechanism — they must be
written with the same care as the core spec documents.

**6. The Pacioli guarantee is non-negotiable.**
The single most important property of Works is that it cannot lie about its history.
Any implementation that allows Store deletion, Stream log modification, or Gate bypass
is not conforming. The compliance document must make this absolutely clear. The test
suite must verify it. A "Works-compatible" implementation that does not honor Pacioli
is not Works-compatible — it is something else with a borrowed name.

---

## Part XI: Immediate Next Actions

The following actions are sequenced and ready to begin now. Each is a concrete,
bounded task.

### Specification (works-standard repo)

1. **Initialize `babbworks/works-standard` repository** — README, LICENSE (Apache 2.0),
   CHANGELOG, `docs/` directory skeleton
2. **Write `docs/00-concepts.md`** — vocabulary for all 15 documents
3. **Write `docs/02-stream-protocol.md`** — Hollerith encoding, Op codes, three layers
4. **Write `docs/01-chunk-format.md`** — chunk anatomy, Row×Level×Bin, provenance chain
5. **Write `docs/03-buildings.md`** — all nine buildings, role and contract
6. **Write `docs/04-actors.md`** — four Actor types, capability token format
7. **Write remaining 9 spec documents** — in Phase 2–5 order above
8. **Write `CONFORMANCE.md`** — test cases that implementations must pass

### Reference Implementation (works-rs repo)

9. **Initialize `babbworks/works-rs`** — Cargo workspace, 18 crate stubs, CI
10. **Implement `works-core`** — Event, Op, Chunk, StoreAddress, Ctx types
11. **Implement `works-stream`** — append to log, filter, replay, Bus pub/sub
12. **Implement `works-cli stream`** — emit, ingest, view, status subcommands
   (port WWSS logic from bash/Python to Rust)
13. Continue Phase B through G per Part V above

### WW Integration

14. **Update WWSS (`services/stream/`)** — add `ww stream` reference to Works-Standard URL
15. **Write `docs/14-ww-binding.md`** — how WW-Standard implements Works-Standard
16. **Update WW-Standard README** — Works-Standard as the upstream specification for WWSS

### Governance

17. **Register `works-standard` trademark** — protect the name while keeping the spec open
18. **Publish first announcement** — "Works v0.1 — an open standard for the information compound"

---

## Closing Statement

Works is the right standard at the right moment. It has 530 years of precedent (Pacioli),
190 years of architectural grounding (Babbage), 183 years of symbolic generality
(Lovelace), 136 years of encoding discipline (Hollerith), 81 years of processing
architecture (von Neumann), and a working reference implementation in shell and Python
that has been running in production since early 2026.

The problem it solves — how do humans and AI agents work together on information at
production volume, with audit trails, authorized output, and permanent memory — is the
defining infrastructure problem of the next decade.

The babbworks family of standards (BitPads, BitLedger, WW-Standard, Works-Standard)
provides a coherent, layered answer from wire format to compound protocol.

The work ahead is specification, implementation, and adoption — in that order.
The specification defines behavior. The implementation proves it. The adoption spreads it.

The goal is an information compound that, decades from now, can replay its complete
operational history from the first Stream event to the last Vault entry — without
ambiguity, without loss, and without requiring the original implementation to still
be running.

That is the standard Works sets for itself.

---

*Works-Standard v0.1-draft*
*2026-05-04*
*Maintainer: Babb (ww@babb.tel)*
*GitHub: babbworks/works-standard (to be initialized)*
*License: Apache 2.0*
*Related standards: babbworks/bitpads-standard · babbworks/bitledger-standard · babbworks/ww-standard*
