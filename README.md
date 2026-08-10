# Open Memory Specification (OMS)

**Version:** 1.6-draft (1.5 released) | **Status:** Standards Track — 1.6/CAL 1.3 open for comment | **License:** CC0 1.0 Universal (Public Domain)

Backed by [areev.ai](https://areev.ai).

OMS is an open standard for portable, auditable, and interoperable agent memory. It defines three layered specifications — the binary **memory format** (OMS), the **query and assembly language** (CAL), and the **LLM context markup** (SML) — covering the complete lifecycle from storing a memory grain to delivering it to an AI agent's context window.

| Layer | Spec | Role |
|-------|------|------|
| **Storage** | OMS — Open Memory Specification | Binary `.mg` container: how grains are encoded, hashed, signed, and stored |
| **Query** | CAL — Context Assembly Language | Language for recalling, assembling, and evolving agent context — append-only by construction; destruction and access control authorization-gated (1.3 draft) |
| **Output** | SML — Semantic Markup Language | Tag-based LLM context format produced by CAL `ASSEMBLE` |

---

## OMS — The Memory Format

OMS defines the **Memory Grain (`.mg`) container** — a binary format for immutable, content-addressed knowledge units called *grains*. A memory grain is the atomic unit of agent knowledge: a single immutable fact, event, observation, or decision record, identified by the SHA-256 hash of its canonical binary representation.

Think of the `.mg` container as what JSON is to APIs or `.git` objects are to version control — a universal, language-agnostic, self-describing interchange format for agent memory.

### Key Properties

| Property | Description |
|----------|-------------|
| **Deterministic serialization** | Identical content always produces identical bytes |
| **Content addressing** | SHA-256 hash for integrity, deduplication, and identity |
| **Compact binary encoding** | MessagePack (default) or CBOR (optional) |
| **Cryptographic verification** | COSE Sign1 envelopes (optional) |
| **Field-level privacy** | Selective disclosure without exposing full grain |
| **Compliance primitives** | GDPR, CCPA, HIPAA support baked in |
| **Multi-modal references** | Links to external images, video, audio, and embeddings |
| **Decentralized identity** | W3C DIDs — no certificate authority required |
| **Grain protection** | Invalidation policies restricting supersession rights |

### Grain Types

| Type | Byte | Description |
|------|------|-------------|
| Fact | `0x01` | Declarative knowledge: subject–relation–object triple |
| Event | `0x02` | Timestamped occurrence: message, interaction, or utterance |
| State | `0x03` | Agent state snapshot at a point in time |
| Workflow | `0x04` | Directed graph of procedural steps |
| Tool | `0x05` | Tool invocation or code execution |
| Observation | `0x06` | Sensor or cognitive input |
| Goal | `0x07` | Intent, objective, or desired outcome |
| Reasoning | `0x08` | Inference chain and thought audit trail |
| Consensus | `0x09` | Multi-agent agreement record |
| Consent | `0x0A` | DID-scoped permission grant or withdrawal |
| Skill | `0x0B` | Packaged, reusable agent capability: definition plus optional learned proficiency |
| Recommendation | `0x0C` | Governed, auditable proposal to change memory or agent config: reviewed, applied, rollback-able |
| `0xF0–0xFF` | — | Application-defined domain profile types |

### Blob Layout

```
 0       1       2       3   4   5       6       7       8       9      10 ...
+-------+-------+-------+---+---+-------+-------+-------+-------+-------+---
| Ver   | Flags | Type  |  NS hash  |        created_at (u32)   | MsgPack
| 0x01  | uint8 | uint8 |  uint16   |       (epoch seconds)     | payload
+-------+-------+-------+---+---+-------+-------+-------+-------+-------+---
 Fixed header (9 bytes)                                          Variable
```

The fixed 9-byte header enables O(1) field extraction without deserializing the payload — type, namespace, and timestamp are always at known byte offsets.

### Design Principles

1. **References, not blobs** — Multi-modal content is referenced by URI, never embedded
2. **Additive evolution** — New fields never break old parsers
3. **Minimal required fields** — Only essential fields per memory type
4. **Semantic triples** — Subject–relation–object model for knowledge graph compatibility
5. **Compliance by design** — Provenance and identity in every grain
6. **No AI in the format** — Deterministic serialization; LLMs belong in the engine layer
7. **Index without deserialize** — Fixed headers enable O(1) field extraction
8. **Sign without PKI** — DIDs enable verification without certificate authorities
9. **Share without exposure** — Selective disclosure for privacy-preserving interchange
10. **One file, full memory** — A `.mg` container is a portable, complete knowledge export

---

## CAL — Context Assembly Language

CAL is a **non-destructive, deterministic, LLM-native language** for assembling agent context from OMS memory stores. It answers a single question: *what should be in the agent's context window right now?*

CAL's core safety guarantee — **it cannot destroy data** — is enforced at the grammar level, not by convention. The lexer has no `DELETE`, `DROP`, `FORGET`, or `ERASE` tokens. Every write creates a new grain; old grains survive forever.

### Core Statements

| Statement | Tier | What it does |
|-----------|------|--------------|
| `RECALL` | Read | Retrieve grains matching filters |
| `ASSEMBLE` | Read | Compose a context block from multiple RECALL sources with a token budget |
| `EXISTS` | Read | Check whether a grain with a given hash is present |
| `HISTORY` | Read | Retrieve the supersession chain for a grain |
| `EXPLAIN` | Read | Return the execution plan without running the query |
| `BATCH` | Read | Run multiple queries in a single round trip |
| `ADD` | Evolve | Write a new grain (append-only) |
| `SUPERSEDE` | Evolve | Replace a grain's logical content; original grain survives |
| `REVERT` | Evolve | Undo a supersession; three grains exist afterward — original, supersession, revert |

### Real-World Example: Customer Support Agent

A support agent handles an inbound ticket: *"My invoice shows a charge I don't recognise."* Before the LLM generates a reply it needs to know who the customer is, their account history, prior tickets, relevant policies, and what tools have already run this session. CAL assembles all of that in one statement.

#### Step 1 — Assemble context at ticket open

```sql
CAL/1 ASSEMBLE support_context
  FOR "resolving billing dispute for customer:priya"
  FROM
    profile:   (RECALL facts  ABOUT "customer:priya"
                WHERE relation IS KNOWLEDGE
                LIMIT 10),
    history:   (RECALL events
                WHERE user_id = "customer:priya"
                SINCE "last 90 days"
                LIMIT 20),
    tickets:   (RECALL workflows
                WHERE subject = "customer:priya"
                  AND tags INCLUDE ["support:billing"]
                RECENT 5),
    policy:    (RECALL facts
                WHERE tags INCLUDE ["policy:billing"]
                LIMIT 5),
    session:   (RECALL tools
                WHERE session_id = "sess-20260303-priya"
                LIMIT 10)
  BUDGET 4000 tokens
  PRIORITY profile > history > tickets > policy > session
  FORMAT sml
```

The executor runs all five `RECALL` queries in parallel, applies the token budget and priority ordering, then emits SML (see below).

#### Step 2 — Record what the agent decided (Tier 1 write)

```sql
CAL/1 ADD reasoning
  SET subject     = "customer:priya"
  SET relation    = "dispute_analysis"
  SET object      = "charge-2026-02-28"
  SET content     = "Charge matches annual plan renewal on 2026-02-28. Customer last contacted re: plan in Jan. Likely unrecognised due to annual cycle."
  SET confidence  = 0.91
  SET tags        = ["support:billing", "resolution:explain"]
  REASON "agent inferred cause from renewal date and contact history"
```

#### Step 3 — Recall only open disputes across all customers (agent dashboard)

```sql
CAL/1 RECALL workflows
  WHERE tags    INCLUDE ["support:billing"]
    AND goal_state = "open"
  ORDER BY time DESC
  LIMIT 50
  FORMAT markdown
```

---

## SML — Semantic Markup Language

SML is the output format produced by CAL `ASSEMBLE FORMAT sml`. It is a **flat, tag-based markup format** designed for direct LLM consumption. Tag names are OMS grain types (`<fact>`, `<event>`, `<reasoning>`, …). The tag tells the LLM the epistemic status of the content; the attributes carry decision metadata; the element text is natural-language prose.

SML is **not XML**. It requires no parser, no schema, no escape sequences. An LLM reads it the same way a person reads a well-structured document.

### Structural Rules

1. **Tag names are grain types.** `<fact>`, `<goal>`, `<event>`, `<tool>`, `<observation>`, `<reasoning>`, `<state>`, `<workflow>`, `<consensus>`, `<consent>`, `<skill>`, `<recommendation>` — no others.
2. **Flat only.** No nesting beyond the `<context>` envelope.
3. **No storage internals.** No hashes, namespaces, or OMS metadata in the output.
4. **Natural language content.** Element text is prose, not decomposed triples.
5. **One envelope.** `<context intent="…">` is the sole container element.

### Real-World Example: Support Agent Context Window

This is the SML block injected into the LLM system prompt for the billing dispute above:

```sml
<context intent="resolving billing dispute for customer:priya">

  <fact subject="customer:priya" confidence="0.97">account tier is Professional, annual billing cycle</fact>
  <fact subject="customer:priya" confidence="0.93">primary contact email is priya@example.com</fact>
  <fact subject="customer:priya" confidence="0.89">enrolled in auto-renewal since 2024-03-01</fact>

  <event role="user"  time="2m ago">My invoice shows a charge I don't recognise — $299 on 28 Feb.</event>
  <event role="agent" time="2m ago">Looking into that now, Priya. Retrieving your billing history.</event>
  <event role="user"  time="5d ago">Can I switch to monthly billing?</event>
  <event role="agent" time="5d ago">Monthly billing is available — I've sent a link to make that change.</event>

  <workflow trigger="billing_dispute_opened" state="open">verify_charge -> check_renewal_date -> explain_or_escalate -> offer_billing_cycle_change -> close_ticket</workflow>

  <tool tool="get_invoice"    phase="completed">retrieved invoice INV-2026-02-28: $299 annual Professional plan renewal</tool>
  <tool tool="get_plan_history" phase="completed">plan enrolled 2024-03-01, renewed annually; last renewal 2026-02-28</tool>

  <observation observer="billing-system">renewal processed automatically on 2026-02-28 at 00:01 UTC; no failed payment</observation>
  <observation observer="system">customer last viewed billing page 2026-01-15</observation>

  <reasoning type="deductive">charge is the annual plan renewal; customer enrolled in auto-renewal; charge is valid</reasoning>
  <reasoning type="abductive">customer may be unaware of annual cycle because last billing interaction was January — explain renewal cadence before offering monthly switch</reasoning>

  <fact subject="policy:billing" confidence="1.0">customers may switch billing cycle within 30 days of renewal with pro-rated refund</fact>

  <consent action="granted" grantor="customer:priya" grantee="support-agent">access billing records and invoice history for dispute resolution</consent>

</context>
```

The LLM now knows: who the customer is, the exact charge, the full support workflow, what tools have already run, the agent's inferred cause, and the applicable refund policy — all tagged by epistemic type, all within a 4 000-token budget.

### Progressive Disclosure

SML metadata density is controlled by disclosure level — the element shape never changes, only the number of attributes:

| Level | Example |
|-------|---------|
| `summary` | `<fact subject="customer:priya">enrolled in auto-renewal</fact>` |
| `standard` | `<fact subject="customer:priya" confidence="0.89">enrolled in auto-renewal since 2024-03-01</fact>` |
| `full` | `<fact subject="customer:priya" confidence="0.89" source="crm" observed="14d ago">enrolled in auto-renewal since 2024-03-01</fact>` |

---

## How the Three Layers Work Together

```
┌─────────────────────────────────────────────────────────┐
│               AI Agent / Orchestrator                   │
│  1. Issues a CAL ASSEMBLE query                         │
│  2. Receives SML context block                          │
│  3. Generates response                                  │
│  4. Issues CAL ADD / SUPERSEDE to persist new grains    │
└──────────────────────────┬──────────────────────────────┘
                           │ CAL queries
┌──────────────────────────▼──────────────────────────────┐
│                   CAL Executor                          │
│  • Runs RECALL queries in parallel                      │
│  • Applies token budget + priority ordering             │
│  • Emits SML (or markdown / JSON / TOON)                │
│  • Enforces namespace, policy, and rate limits          │
└──────────────────────────┬──────────────────────────────┘
                           │ OMS store protocol
┌──────────────────────────▼──────────────────────────────┐
│                   OMS Memory Store                      │
│  • .mg containers on disk / S3 / IPFS / database        │
│  • SHA-256 content addressing + hexastore index         │
│  • COSE Sign1 signatures (optional)                     │
│  • Append-only; no grain is ever overwritten            │
└─────────────────────────────────────────────────────────┘
```

---

## Specification

| Document | Contents |
|----------|----------|
| [`SPECIFICATION.md`](./SPECIFICATION.md) | Full OMS wire format, grain types, signing, selective disclosure, conformance, domain profiles |
| [`CONTEXT-ASSEMBLY-LANGUAGE-CAL-SPECIFICATION.md`](./CONTEXT-ASSEMBLY-LANGUAGE-CAL-SPECIFICATION.md) | CAL grammar (EBNF), all statements, FORMAT system, streaming, policy integration, error codes |
| [`SEMANTIC-MARKUP-LANGUAGE-SML-SPECIFICATION.md`](./SEMANTIC-MARKUP-LANGUAGE-SML-SPECIFICATION.md) | SML format definition, structural rules, comprehensive example, progressive disclosure |

**SPECIFICATION.md table of contents:**
- Blob Layout and Structure
- Canonical Serialization and Content Addressing
- Field Compaction
- Multi-Modal Content References
- Grain Types and Field Specifications
- Cryptographic Signing
- Selective Disclosure
- File Format (`.mg` files)
- Identity and Authorization
- Sensitivity Classification
- Cross-Links and Provenance
- Temporal Modeling
- Encoding Options
- Conformance Levels
- Error Handling and Security Considerations
- Test Vectors
- Grain Protection and Invalidation Policy
- Observer Type, Observation Mode, and Scope Registries
- Query Conventions
- Store Protocol Convention
- Domain Profile Registry (Healthcare, Legal, Finance, Robotics, Science, Consumer)

---

## Conformance Levels

| Level | Name | Description |
|-------|------|-------------|
| Level 1 | Minimal Reader | Deserialize, verify SHA-256 content addresses, field compaction |
| Level 2 | Full Implementation | Level 1 + serialization, canonical encoding, store protocol, invalidation policy enforcement |
| Level 3 | Production Store | Level 2 + persistent backend, encryption, per-user keys, hexastore index, audit trail |

---

## Scope

**In scope:**
- Binary serialization format for individual grains
- `.mg` file container format for grain collections
- Deterministic encoding and hashing
- Cryptographic signing and selective disclosure
- Content reference and embedding reference schemas
- Identity and authorization models
- Sensitivity classification
- Cross-link and provenance tracking
- CAL query and assembly language
- SML LLM context output format

**Out of scope:**
- Storage layer implementation (filesystem, S3, database, IPFS)
- Transport protocols (HTTP, MQTT, Kafka, MCP)
- Encryption at rest
- Agent-to-agent communication protocol

---

## Contributing

Contributions are welcome. Please read [CONTRIBUTING.md](./CONTRIBUTING.md) before submitting changes.

## License

This specification is released into the public domain under [CC0 1.0 Universal](./LICENSE). See also the [Open Web Foundation Final Specification Agreement (OWFa 1.0)](https://www.openwebfoundation.org/the-agreements/the-owf-1-0-agreements-granted-claims/owfa-1-0).

No copyright — use it freely.
