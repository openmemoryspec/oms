# Open Memory Specification (OMS)

**Version:** 1.0 | **Status:** Standards Track | **License:** CC0 1.0 Universal (Public Domain)

OMS is an open standard for portable, auditable, and interoperable agent memory across autonomous systems, AI agents, and distributed knowledge networks.

## What Is OMS?

OMS defines the **Memory Grain (`.mg`) container** — a binary representation for immutable, content-addressed knowledge units called *grains*. A memory grain is the atomic unit of agent knowledge: a single immutable fact, episode, observation, or decision record, identified by the SHA-256 hash of its canonical binary representation.

Think of the `.mg` container as what JSON is to APIs or `.git` objects are to version control — a universal, language-agnostic, self-describing interchange format for agent memory.

## Key Properties

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

## Memory Types

| Type | Byte | Description |
|------|------|-------------|
| Fact | `0x01` | Atomic assertion: subject–relation–object triple |
| Episode | `0x02` | Temporal experience with context and actors |
| Checkpoint | `0x03` | Agent state snapshot at a point in time |
| Workflow | `0x04` | Multi-step procedural record |
| ToolCall | `0x05` | External tool invocation and result |
| Observation | `0x06` | Sensor or environment reading |
| Goal | `0x07` | Intent, objective, or desired outcome |
| `0xF0–0xFF` | — | Application-defined types |

## Blob Layout

```
 0       1       2       3   4   5       6       7       8       9      10 ...
+-------+-------+-------+---+---+-------+-------+-------+-------+-------+---
| Ver   | Flags | Type  |  NS hash  |        created_at (u32)   | MsgPack
| 0x01  | uint8 | uint8 |  uint16   |       (epoch seconds)     | payload
+-------+-------+-------+---+---+-------+-------+-------+-------+-------+---
 Fixed header (9 bytes)                                          Variable
```

## Design Principles

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

## Specification

The full specification is in [`oms-specification.md`](./oms-specification.md).

**Table of Contents:**
- Blob Layout and Structure
- Canonical Serialization
- Content Addressing
- Field Compaction
- Multi-Modal Content References
- Memory Types
- Cryptographic Signing
- Selective Disclosure
- File Format (`.mg` files)
- Identity and Authorization
- Sensitivity Classification
- Cross-Links and Provenance
- Temporal Modeling
- Encoding Options
- Conformance Levels
- Device Profiles
- Error Handling
- Security Considerations
- Test Vectors
- Implementation Notes
- Grain Protection and Invalidation Policy

## Conformance Levels

| Level | Name | Description |
|-------|------|-------------|
| Level 1 | Minimal Reader | Deserialize, verify SHA-256 content addresses, field compaction |
| Level 2 | Full Implementation | Level 1 + serialization, canonical encoding, store protocol, invalidation policy enforcement |
| Level 3 | Production Store | Level 2 + persistent backend, encryption, per-user keys, hexastore index, audit trail |

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

**Out of scope:**
- Storage layer implementation (filesystem, S3, database, IPFS)
- Index layer queries and optimization
- Transport protocols (HTTP, MQTT, Kafka)
- Encryption at rest
- Agent-to-agent communication protocol

## Contributing

Contributions are welcome. Please read [CONTRIBUTING.md](./CONTRIBUTING.md) before submitting changes.

## License

This specification is released into the public domain under [CC0 1.0 Universal](./LICENSE). See also the [Open Web Foundation Final Specification Agreement (OWFa 1.0)](https://www.openwebfoundation.org/the-agreements/the-owf-1-0-agreements-granted-claims/owfa-1-0).

No copyright — use it freely.
