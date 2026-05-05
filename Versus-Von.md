# Works Architecture — Von Neumann Correspondence

> Works is a compound standard for the storage, transformation, and authorized output of
> business information. This diagram shows its structural correspondence to the von Neumann
> architecture — and the places where Works deliberately extends beyond it.

---

## The Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                             │
│   ACTORS  (novel — no VN equivalent)                                                        │
│   Human · Agent{name,model,session} · Automated{barrel} · External{system_id}              │
│   → interact with every building; session-scoped; capability-bounded by Office              │
│                                                                                             │
└──────────┬─────────────────────────────────────────────────────────────┬────────────────────┘
           │                                                             │
           ▼                                                             ▼
┌──────────────────────────────┐           ┌──────────────────────────────────────────────────┐
│                              │           │                                                  │
│   INPUT                      │           │   OUTPUT                                         │
│   ══════                     │           │   ══════                                         │
│   INTAKE                     │           │   DISPATCH                                       │
│                              │           │                                                  │
│   • task / timew / jrnl /    │           │   • file write                                   │
│     hledger adapters         │           │   • API call                                     │
│   • BitPads Intake           │           │   • publish / broadcast                          │
│     (Rich Record decompose   │           │   • BitPads output                               │
│      or route intact as      │           │     (compact wire record)                        │
│      Manifest)               │           │   • human-readable render                        │
│   • CRC-15 verification      │           │   • agent handoff packet                         │
│   • format normalization     │           │                                                  │
│   • routing to Store or      │           │   [only reachable through GATE]                  │
│     direct to Mill           │           │                                                  │
└──────────┬───────────────────┘           └──────────────────┬───────────────────────────────┘
           │                                                  ▲
           │                                                  │
           ▼                                                  │
╔══════════════════════════════════════════════════════════════════════════════════════════════╗
║                                                                                            ║
║   THE BUS — STREAM                                                                         ║
║   ═══════════════════════════════════════════════════════════════════════════════════════  ║
║                                                                                            ║
║   Three layers:                                                                            ║
║     Log    append-only event log (Pacioli guarantee — never delete, replay = truth)        ║
║     Bus    real-time event delivery (WebSocket / Unix socket; emit → subscribe)            ║
║     Replay catch-up and recovery for any subscriber that missed events                     ║
║                                                                                            ║
║   Wire protocol: BitPads (1 byte heartbeat → 29 byte full record)                         ║
║   Audit: every building action crosses Stream; Office receives all policy-class events     ║
║   Hollerith encoding: <ts> <op> <action> <object> <ctx_json> — sortable by col 1          ║
║   Op codes: T=Task  F=Frick  B=Bundy  D=Dey  H=Hollerith  S=System  A=Annotation         ║
║                                                                                            ║
║   [Stream is the Works bottleneck — like VN's bus — and its constitutional guarantee]      ║
║                                                                                            ║
╚════════════╦═════════════╦══════════════╦══════════════╦════════════════════════════════════╝
             ║             ║              ║              ║
             ▼             ▼              ▼              ▼
┌────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                            │
│   MEMORY — STORES                                                                          │
│   ══════════════════════════════════════════════════════════════════════════════════════   │
│                                                                                            │
│   Named stores — short serial or human arbitrary names — each a Babbage Row×Level×Bin     │
│   address space. Multiple stores coexist; Stores hold chunks at any lifecycle stage.       │
│                                                                                            │
│   ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────────────┐ │
│   │  S-01      │  │  S-legal   │  │  S-api     │  │  S-finance │  │  S-…  (unlimited)  │ │
│   │            │  │            │  │            │  │            │  │                    │ │
│   │ Row 0      │  │ Row 0      │  │ Row 0      │  │ Row 0      │  │  any named store   │ │
│   │  Lv: Raw   │  │  Lv: Raw   │  │  Lv: Raw   │  │  Lv: Raw   │  │  can emerge and   │ │
│   │  Bin: T/F/B│  │  Bin: A    │  │  Bin: T    │  │  Bin: T/B  │  │  register per the │ │
│   │ Row 1      │  │ Row 1      │  │ Row 1      │  │ Row 1      │  │  extension proto  │ │
│   │  Lv: Staged│  │  Lv: Staged│  │  Lv: Staged│  │  Lv: Staged│  │                   │ │
│   │ Row 2      │  │ Row 2      │  │ Row 2      │  │ Row 2      │  │                   │ │
│   │  Lv: Integr│  │  Lv: Integr│  │  Lv: Integr│  │  Lv: Integr│  │                   │ │
│   └────────────┘  └────────────┘  └────────────┘  └────────────┘  └────────────────────┘ │
│                                                                                            │
│   VAULT (append-only archive — Pacioli compound guarantee; nothing deleted)                │
│   ┌──────────────────────────────────────────────────────────────────────────────────┐    │
│   │  committed bundles, completed Mill outputs, authorized Dispatch records          │    │
│   └──────────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                            │
│   BARRELS (stored programs — Babbage's stored-program concept)                             │
│   ┌──────────────────────────────────────────────────────────────────────────────────┐    │
│   │  named instruction sequences; fetched by Control (Barrel scheduler) and          │    │
│   │  executed by Mill — exact von Neumann stored-program parallel                    │    │
│   └──────────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                            │
└────────────────────────────────────────────────────────────────────────────────────────────┘
             ║             ║              ║              ║
             ▼             ▼              ▼              ▼
┌────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                            │
│   CENTRAL PROCESSING UNIT                                                                  │
│   ══════════════════════════════════════════════════════════════════════════════════════   │
│                                                                                            │
│  ┌──────────────────────────┐  ┌──────────────────────────┐  ┌────────────────────────┐  │
│  │                          │  │                          │  │                        │  │
│  │   CONTROL UNIT           │  │   ALU                    │  │   REGISTERS            │  │
│  │   ─────────────          │  │   ───                    │  │   ─────────            │  │
│  │   BARREL                 │  │   MILL                   │  │   BENCH                │  │
│  │                          │  │                          │  │                        │  │
│  │  • fetch instruction     │  │  • combine chunks        │  │  • active workspace    │  │
│  │    sequences from Store  │  │  • transform content     │  │  • session chunks      │  │
│  │  • schedule: cron /      │  │  • run lens pipeline     │  │  • staging area for    │  │
│  │    event-triggered /     │  │    (Burroughs→Baldwin→   │  │    assembly            │  │
│  │    actor-invoked         │  │     Frick→Bundy→Grant→   │  │  • ephemeral; cleared  │  │
│  │  • enforce CycleMode:    │  │     Felt→Dey→Cooper)     │  │    on session end      │  │
│  │    Single | Bounded |    │  │  • Difference Engine     │  │  • direct Actor-Mill   │  │
│  │    Perpetual             │  │    finite-diff signals   │  │    interaction surface │  │
│  │  • policy check via      │  │  • Babbage Mill ingress/ │  │                        │  │
│  │    Stream before each    │  │    operation/egress      │  │  (like CPU registers:  │  │
│  │    op                    │  │  • Stream Mill (read-    │  │   fast, bounded, local)│  │
│  │  • emit S events to      │  │    only, no Gate) vs     │  │                        │  │
│  │    Stream on each step   │  │    Content Mill (Gate    │  │                        │  │
│  │                          │  │    required for output)  │  │                        │  │
│  └──────────────────────────┘  └──────────────────────────┘  └────────────────────────┘  │
│                                                                                            │
└────────────────────────────────────────────────────────────────────────────────────────────┘
             ║             ║              ║              ║
             ▼             ▼              ▼              ▼
┌────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                            │
│   GATE  (novel — no VN equivalent)                                                         │
│   ══════════════════════════════════════════════════════════════════════════════════════   │
│                                                                                            │
│   Authorization boundary between Mill output and Dispatch. Nothing leaves Works            │
│   without passing Gate. Gate holds the Right to Assembly — only human or authorized        │
│   Barrel can promote Staged → Integrated, or authorize Integrated → Dispatch.              │
│                                                                                            │
│   • Gate instruction = BitPads Task block (8-bit packed instruction protocol)              │
│   • Gate emits G events to Stream (every authorization decision is auditable)              │
│   • Gate can: approve / reject / hold / reroute / require human confirmation               │
│   • Office receives all G events; can veto or escalate                                     │
│   • Compound Mode 1111 marker = atomic Mill output guarantee (BitPads)                     │
│                                                                                            │
└────────────────────────────────────────────────────────────────────────────────────────────┘
             ║             ║              ║              ║
             ▼             ▼              ▼              ▼
┌────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                            │
│   OFFICE  (novel — no VN equivalent)                                                       │
│   ══════════════════════════════════════════════════════════════════════════════════════   │
│                                                                                            │
│   The policy and audit authority for the whole Works compound. Receives all               │
│   Stream events classified as policy-class. Enforces constraints on Actor                  │
│   capabilities, building registrations, Gate authorizations, and extension                 │
│   protocol compliance. Publishes the Works constitution.                                   │
│                                                                                            │
│   • Office log is itself append-only (Pacioli guarantee at compound level)                 │
│   • Registers new building types (extension protocol)                                      │
│   • Issues Actor capability tokens                                                         │
│   • Reconciles BitLedger conservation invariants with Store state                          │
│   • Human-readable audit trail; queryable by authorized Actors                             │
│                                                                                            │
└────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Von Neumann Correspondence Table

| Von Neumann Component | Works Equivalent   | Notes |
|-----------------------|--------------------|-------|
| Memory Unit           | Stores + Vault     | Row×Level×Bin addressing (Babbage); Vault = non-erasable archive |
| Stored Program        | Barrels in Stores  | Exact VN parallel — programs live in memory, fetched by control |
| Control Unit          | Barrel             | Fetches from Store, sequences ops, enforces CycleMode |
| ALU                   | Mill               | Lens pipeline + Difference Engine signal computation |
| Registers             | Bench              | Fast ephemeral workspace; session-scoped |
| Input Unit            | Intake             | BitPads decompose/route; CRC-15 verification |
| Output Unit           | Dispatch           | Only reachable through Gate |
| System Bus            | Stream             | Append-only log + real-time bus + replay; BitPads wire; Hollerith encoding |
| —                     | Gate               | **Novel**: authorization boundary; Right to Assembly; no VN equivalent |
| —                     | Office             | **Novel**: policy authority; audit log; building registry; no VN equivalent |
| —                     | Actors             | **Novel**: first-class operators (Human/Agent/Automated/External); no VN equivalent |
| —                     | Pacioli guarantee  | **Novel**: non-erasable memory at every level; VN assumed mutable memory |

---

## What Works Adds Beyond Von Neumann

**1. Authorization layer (Gate)**
Von Neumann's model has no concept of authorization between computation and output.
Works inserts Gate as a hard boundary — computation (Mill) and output (Dispatch) are
structurally separated. This is the Right to Assembly: nothing leaves without approval.

**2. Audit as constitutional structure (Office + Stream)**
VN's bus is a transport mechanism. Works' Stream is simultaneously transport AND audit log AND
policy enforcement. Every building action crosses Stream; Office receives policy-class events.
The architecture cannot produce unaudited output — this is structural, not a feature.

**3. Non-erasable memory (Pacioli guarantee)**
VN's memory is mutable — the stored-program concept depends on being able to overwrite.
Works' Stores are append-only at the chunk level. Stage transitions are recorded, not overwritten.
Vault is permanently non-erasable. History is a first-class structural property, not a logging afterthought.

**4. First-class Actors**
VN has no model of the entities that operate the machine. Works defines Human, Agent, Automated,
and External as distinct Actor types with session scope and capability bounds. The architecture
knows who is operating it at all times — necessary when the operators include AI agents.

**5. The Manifest (BitPads Rich Record as Works input atom)**
VN's input is raw data. Works' Intake can receive a BitPads Rich Record — a compound packet
containing Task + Time + Journal + Ledger data in a single verified unit with CRC-15 integrity.
The Manifest can be routed intact or decomposed; either way the Works knows its provenance.

---

## The Von Neumann Bottleneck — Reframed

Von Neumann's bus is a bottleneck — all data must pass through it, limiting throughput.
Works accepts this constraint and makes it constitutional. Stream is the bottleneck by design:

- **Performance cost**: every event crosses Stream
- **Audit gain**: therefore every event is auditable
- **Recovery gain**: therefore any building can replay from Stream to reconstruct state
- **Policy gain**: therefore Stream is the enforcement point for Office rules

This is the same trade Von Neumann made — a single shared bus — reframed not as a limitation
to be optimized away, but as the structural guarantee that makes Works trustworthy.

---

## Babbage's Contributions (beyond VN)

The stored-program concept is often attributed to von Neumann (1945), but Babbage's
Analytical Engine design (1830s–1871) anticipated it: Barrels (programs) were stored
on punched cards, fetched sequentially, and could conditionally branch. Works honors
this by naming the control component Barrel, not Program Counter.

The **Difference Engine No. 2** contributes the DifferenceTable signal computation:
```
advance() → for i in (1..n).rev() { reg[i-1] += reg[i] }; return reg[0]
```
Pure addition cascades computing arbitrary polynomial approximations — no multiplication required.
This is the computational substrate for Works' Dey (continuous signal) and Felt (density) lenses.

Lovelace's **Note G** — the first program — demonstrates that a general computational engine
can be programmed to produce any sequence, not just numbers. Works' Barrel inherits this:
a Barrel can sequence arbitrary building operations, not just arithmetic transformations.

---

*Document created: 2026-05-04*
*Repository: babb/repos/works*
