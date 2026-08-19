# Open Memory Specification (OMS)
## Memory Grain (.mg) Container Definition

**Version:** 1.6-draft
**Status:** Draft for Comment (OMS 1.6 RFC) — changed sections are normative only upon release
**Category:** Data Formats
**Date:** August 2026
**Copyright:** Public Domain (CC0 1.0 Universal)
**License:** This specification is offered under the Open Web Foundation Final Specification Agreement (OWFa 1.0)

---

## Table of Contents

- [Abstract](#abstract)
1. [Introduction](#introduction)
2. [Conventions and Terminology](#conventions-and-terminology)
3. [Blob Layout and Structure](#blob-layout-and-structure)
4. [Canonical Serialization](#canonical-serialization)
5. [Content Addressing](#content-addressing)
6. [Field Compaction](#field-compaction)
7. [Multi-Modal Content References](#multi-modal-content-references)
8. [Grain Types](#grain-types)
9. [Cryptographic Signing](#cryptographic-signing)
10. [Selective Disclosure](#selective-disclosure)
11. [File Format (.mg files)](#file-format-mg-files)
12. [Identity and Authorization](#identity-and-authorization)
13. [Sensitivity Classification](#sensitivity-classification)
14. [Cross-Links and Provenance](#cross-links-and-provenance)
15. [Temporal Modeling](#temporal-modeling)
16. [Encoding Options](#encoding-options)
17. [Conformance Levels](#conformance-levels)
18. [Device Profiles](#device-profiles)
19. [Error Handling](#error-handling)
20. [Security Considerations](#security-considerations)
21. [Test Vectors](#test-vectors)
22. [Implementation Notes](#implementation-notes)
23. [Grain Protection and Invalidation Policy](#grain-protection-and-invalidation-policy)
24. [Observer Type Registry](#observer-type-registry)
25. [Observation Mode Registry](#observation-mode-registry)
26. [Observation Scope Registry](#observation-scope-registry)
27. [Grain Type Field Specifications](#grain-type-field-specifications)
28. [Query Conventions](#query-conventions)
- [Appendix A: Domain Profile Registry](#appendix-a-domain-profile-registry)
- [Appendix B: ABNF Grammar](#appendix-b-abnf-grammar)
- [Appendix C: Field Mapping Table](#appendix-c-field-mapping-table-compact-reference)
- [Appendix D: Compliance Mapping](#appendix-d-compliance-mapping)
- [Appendix E: Version History](#appendix-e-version-history)
- [Appendix F: Glossary](#appendix-f-glossary)
- [Appendix G: Complete Example Grain](#appendix-g-complete-example-grain)

---

## Abstract

The **Open Memory Specification (OMS)** is an open standard for portable, auditable, and interoperable agent memory across autonomous systems, AI agents, and distributed knowledge networks. **OMS** defines the Memory Grain (.mg) container — a standard binary representation for immutable, content-addressed knowledge units (grains). This document specifies the wire format, serialization rules, cryptographic integrity mechanisms, and compliance features necessary for secure and portable interchange of agent memory across platforms, languages, and deployment models.
A memory grain is the atomic unit of agent knowledge—a single immutable fact, episode, observation, or decision record—identified by the SHA-256 hash of its canonical binary representation. The .mg container provides:

- **Deterministic serialization** ensuring identical content always produces identical bytes
- **Content addressing** via SHA-256 for integrity, deduplication, and identity
- **Compact binary encoding** using MessagePack (default) or CBOR (optional)
- **Cryptographic verification** via COSE Sign1 envelopes (optional)
- **Field-level privacy** through selective disclosure
- **Compliance primitives** for GDPR, CCPA, HIPAA, and other regulations
- **Multi-modal references** to external content (images, video, embeddings)
- **Decentralized identity** via W3C DIDs
- **Grain protection** via invalidation policies that restrict who may supersede or contradict a grain

The .mg container format is to autonomous systems what JSON is to APIs and .git objects are to version control: a universal, language-agnostic, self-describing interchange format. It is the foundational wire format of OMS.

**CAL (Context Assembly Language)** ([CONTEXT-ASSEMBLY-LANGUAGE-CAL-SPECIFICATION.md](./CONTEXT-ASSEMBLY-LANGUAGE-CAL-SPECIFICATION.md)) and **SML (Semantic Markup Language)** ([SEMANTIC-MARKUP-LANGUAGE-SML-SPECIFICATION.md](./SEMANTIC-MARKUP-LANGUAGE-SML-SPECIFICATION.md)) are part of OMS v1.5. CAL defines the query and context-assembly layer that operates on OMS stores; SML is CAL's default output format for LLM context consumption. See §1.5 for details.

---

## 1. Introduction

### 1.1 Purpose

Autonomous systems and AI agents require persistent memory to function effectively over time. Unlike transient conversation context (which lives in an LLM's context window), persistent memory must be:

- **Portable** – transferable between agents, systems, and organizations
- **Verifiable** – integrity can be cryptographically proven
- **Immutable** – once created, never modified (supersession creates new records)
- **Auditable** – full provenance chain recorded
- **Compliant** – designed for regulatory requirements (GDPR, HIPAA, etc.)
- **Interoperable** – works across programming languages and platforms
- **Efficient** – minimal storage with content deduplication
- **Secure** – encryption, signing, and selective disclosure support

OMS addresses this gap by defining a universal standard for knowledge interchange, with the .mg container as the foundational wire format.

### 1.2 Design Principles

1. **References, not blobs** — Multi-modal content (images, audio, video, embeddings) is referenced by URI, never embedded in grains
2. **Additive evolution** — New fields never break old implementations; parsers ignore unknowns
3. **Minimal required fields** — Each memory type defines only essential fields
4. **Semantic triples** — Subject-relation-object model for natural knowledge graph mapping
5. **Compliance by design** — Provenance, timestamps, user identity, and namespace baked into every grain
6. **No AI in the format** — Deterministic serialization; LLMs belong in the engine layer, not the wire protocol
7. **Index without deserialize** — Fixed headers enable O(1) field extraction for efficient scanning
8. **Sign without PKI** — Decentralized identity (DIDs) enable verification without certificate authorities
9. **Share without exposure** — Selective disclosure reveals some fields while hiding others
10. **One file, full memory** — A .mg container file is the portable unit for full knowledge export

### 1.3 Terminology

| Term | Definition |
|------|-----------|
| **Memory grain** | Atomic, indivisible unit of knowledge — one .mg blob (fact, episode, observation, etc.) |
| **Blob** | Complete .mg binary — version byte + optional header + canonical payload |
| **Content address** | Lowercase hex SHA-256 hash of complete blob bytes — the grain's unique identifier |
| **Canonical serialization** | MessagePack or CBOR encoding with deterministic key ordering, string normalization, null omission |
| **Field compaction** | Mapping human-readable field names to short keys for storage efficiency |
| **Grain container** | .mg file — portable unit containing indexed set of grains with checksum |
| **Modality** | Type of content: text, image, audio, video, point cloud, 3D mesh, embedding, binary |
| **DID** | Decentralized identifier — W3C standard for cryptographic identity without central registry |
| **COSE** | CBOR Object Signing and Encryption — RFC 9052 standard for signing binary payloads |

### 1.4 Scope and Limitations

**In scope:**
- Binary serialization format for individual grains
- .mg file container format for grain collections
- Deterministic encoding and hashing
- Cryptographic signing and selective disclosure
- Content reference and embedding reference schemas
- Identity and authorization models
- Sensitivity classification
- Cross-link and provenance tracking

**Out of scope:**
- Storage layer implementation (filesystem, S3, database, IPFS)
- Index layer queries and optimization — see CAL (§1.5)
- Policy engines and compliance rule evaluation
- Transport protocols (HTTP, MQTT, Kafka)
- Encryption at rest (applications of per-grain encryption are external to this spec)
- Agent-to-agent communication protocol (which uses .mg format)

### 1.5 Companion Specifications

OMS defines the wire format and grain semantics. Two companion specifications are part of the OMS v1.5 release and are included in this repository:

**CAL — Context Assembly Language** ([CONTEXT-ASSEMBLY-LANGUAGE-CAL-SPECIFICATION.md](./CONTEXT-ASSEMBLY-LANGUAGE-CAL-SPECIFICATION.md))

CAL is a non-destructive, deterministic, LLM-native language for assembling agent context from OMS memory stores. It answers the question: *"what should be in the agent's context window right now?"* Key properties:

- Operates on all 12 OMS grain types (Fact, Event, State, Workflow, Tool, Observation, Goal, Reasoning, Consensus, Consent, Skill, Recommendation)
- Extends the OMS Store Protocol (§28.4) with a formal, structured query syntax
- `ASSEMBLE` statements compose context from multiple grain sources within a token budget
- Append-only: CAL writes create new grains via `put`; the language cannot delete or modify existing grains — this is enforced at the grammar level
- Dual wire format: human-readable `text/cal` and machine-readable `application/json+cal` are bijectively equivalent

**SML — Semantic Markup Language** ([SEMANTIC-MARKUP-LANGUAGE-SML-SPECIFICATION.md](./SEMANTIC-MARKUP-LANGUAGE-SML-SPECIFICATION.md))

SML is a flat, tag-based markup format optimized for LLM context consumption. It is not XML. Tag names are OMS grain types (`<fact>`, `<goal>`, `<event>`, …); attributes carry lightweight decision metadata; text content is natural language. SML is the default output format for CAL `ASSEMBLE` statements and is designed to be consumed directly by an LLM without an XML processor.

---

## 2. Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

Hexadecimal values are lowercase. Byte sequences are represented in hex with spaces between bytes for clarity (e.g., `01 89 a2`).

---

## 3. Blob Layout and Structure

### 3.1 Blob Format (byte `0x01`)

```
 0       1       2       3   4   5       6       7       8       9      10 ...
+-------+-------+-------+---+---+-------+-------+-------+-------+-------+---
| Ver   | Flags | Type  |  NS hash  |        created_at (u32)   | MsgPack
| 0x01  | uint8 | uint8 |  uint16   |       (epoch seconds)     | payload
+-------+-------+-------+---+---+-------+-------+-------+-------+-------+---
 Fixed header (9 bytes)                                          Variable
```

#### 3.1.1 Header Bytes

**Byte 0 — Version:** `0x01` — any other value is rejected with `ERR_VERSION`

**Byte 1 — Flags (bit field):**

| Bit | Flag | Meaning |
|-----|------|---------|
| 0 | `signed` | COSE Sign1 envelope wraps this grain |
| 1 | `encrypted` | Payload is encrypted (AES-256-GCM) |
| 2 | `compressed` | Payload is zstd-compressed before encryption |
| 3 | `has_content_refs` | Grain references external multi-modal content |
| 4 | `has_embedding_refs` | Grain references external vector embeddings |
| 5 | `cbor_encoding` | Payload is CBOR instead of MessagePack |
| 6-7 | `sensitivity` | Classification: 00=public, 01=internal, 10=pii, 11=phi |

**Byte 2 — Type (cognitive grain type):**

| Value | Type | Description |
|-------|------|-------------|
| 0x01 | **Fact** | Structured fact — (subject, relation, object) triple with confidence and source |
| 0x02 | **Event** | Timestamped occurrence — message, interaction, or behavioral event |
| 0x03 | **State** | Agent state snapshot — portable save point |
| 0x04 | **Workflow** | Directed graph of procedural steps — plans, pipelines, processes |
| 0x05 | **Tool** | Tool invocation or code execution |
| 0x06 | **Observation** | Raw sensory or cognitive input |
| 0x07 | **Goal** | Objective with lifecycle semantics |
| 0x08 | **Reasoning** | Inference chain and thought audit trail |
| 0x09 | **Consensus** | Multi-agent agreement record |
| 0x0A | **Consent** | Permission grant or withdrawal — DID-scoped, purpose-bounded |
| 0x0B | **Skill** | Packaged, reusable agent capability — definition (how-to, tools, resources) plus optional learned proficiency |
| 0x0C | **Recommendation** | Governed, auditable proposal to change memory or agent configuration — reviewed, applied, and rollback-able |
| 0x0D | **Trigger** | Standing rule that starts a Workflow — an interval, a schedule, a source to poll, a condition to watch |
| 0x0E–0xEF | Reserved | Future standard types |
| 0xF0–0xFF | Domain profile types | Application-defined per Appendix A domain profiles |

**Bytes 3-4 — Namespace Hash:** First two bytes of SHA-256(namespace), encoded as `uint16` big-endian. Provides 65,536 routing buckets without deserialization. Full namespace string remains authoritative in payload. This field is a routing hint only and MUST NOT be used for security decisions (see §13.3, §20).

**Bytes 5-8 — Created-at:** `uint32` epoch seconds (1970-01-01 onwards). Range: 1970 to 2106. The `created_at` header field is a **coarse routing hint only** — for TTL and time-range indexing. It MUST NOT be used as the authoritative event timestamp. Authoritative timestamps belong in the payload (`timestamp_ms` field). Full millisecond precision available via `timestamp_ms` (§6.1).

### 3.2 Byte Order

All multi-byte values follow big-endian (network) byte order. MessagePack and CBOR specifications handle encoding details.

### 3.3 Minimum and Maximum Sizes

- **Minimum blob:** 10 bytes (9-byte header + 1-byte empty MessagePack map `0x80`)
- **Maximum blob:** 4 GB (uint32 in standard MessagePack, larger via extension)
- **Recommended maximum:** 1 MB for extended profile, 32 KB for standard profile, 512 bytes for lightweight profile

---

## 4. Canonical Serialization

To ensure deterministic hashing and cross-implementation compatibility, all serialization MUST follow these canonical rules:

### 4.1 Key Ordering

Map keys MUST be sorted lexicographically by their UTF-8 byte representation. This applies recursively to all nested maps. Ordering is case-sensitive and treats bytes as unsigned integers.

```
CORRECT ordering:   {"adid": ..., "c": ..., "ca": ..., "ns": ..., "o": ..., "r": ..., "s": ..., "st": ..., "t": ...}
WRONG ordering:     {"s": ..., "c": ..., "ca": ..., "adid": ..., ...}
```

Lexicographic comparison: byte 0 vs byte 0, if equal advance to byte 1, etc.

Map keys MUST be unique within a map. Duplicate keys MUST be rejected with `ERR_CORRUPT`.

### 4.2 Integer Encoding

Integers MUST use the smallest MessagePack/CBOR representation:

| Range | MessagePack Encoding |
|-------|----------------------|
| 0 to 127 | positive fixint (1 byte) |
| -32 to -1 | negative fixint (1 byte) |
| 128 to 255 | uint8 (2 bytes) |
| 256 to 65,535 | uint16 (3 bytes) |
| -128 to -33 | int8 (2 bytes) |
| -32,768 to -129 | int16 (3 bytes) |

For CBOR, follow RFC 8949 Section 4.2.1 (Preferred Encoding).

### 4.3 Float Encoding

Floating-point numbers MUST be encoded as IEEE 754 double precision (float64, 8 bytes) in MessagePack format. Single-precision (float32) MUST NOT be used. In CBOR, use major type 7 with 27 (64-bit IEEE 754).

Float64 values MUST NOT be NaN or Infinity. Serializers MUST reject non-finite values with `ERR_FLOAT_INVALID`. IEEE 754 permits multiple NaN bit patterns (varying sign, exponent, and mantissa bits), which produce different byte sequences and therefore different content addresses across runtimes. Rejecting all non-finite values eliminates this ambiguity and ensures cross-implementation hash stability.

### 4.4 String Encoding

All strings (keys and values) MUST be UTF-8 encoded and MUST be NFC-normalized (Unicode Normalization Form Canonical Composition per [UAX #15](https://unicode.org/reports/tr15/)) before encoding. Strings MUST NOT contain a byte-order mark (BOM, bytes `EF BB BF`). Parsers MUST reject strings beginning with a BOM with `ERR_CORRUPT`.

**Example:** Combining character `e` + `\u0301` (combining acute) → precomposed character `\u00e9` (é)

### 4.5 Null Omission

Map entries with null/None/nil values MUST be omitted entirely from the serialized form. Absent fields default to:
- Strings: None or empty
- Numbers: 0 or 0.0
- Booleans: false
- Arrays: empty list
- Maps: None

**Semantic distinction:** Absent fields are semantically distinct from fields explicitly set to a default value. Consumers MUST NOT treat an absent field as equivalent to a field present with its default value. Serializers MUST NOT auto-insert default values during round-trip serialization; doing so changes the blob bytes and produces a different content address.

**Rationale:** Forward compatibility (new optional fields don't change existing hashes), determinism (no ambiguity between absent and null), compactness.

### 4.6 Array Ordering

Array elements MUST preserve insertion order. Arrays are NOT sorted.

### 4.7 Nested Compaction

Three fields use nested field compaction:
- `content_refs` — use CONTENT_REF_FIELD_MAP (Section 7.1)
- `embedding_refs` — use EMBEDDING_REF_FIELD_MAP (Section 7.2)
- `related_to` — use RELATED_TO_FIELD_MAP (Section 14.2)

Other array-of-maps fields (`provenance_chain`, `context`, `history`) are NOT compacted recursively.

### 4.8 Datetime Conversion

All datetime fields (`valid_from`, `valid_to`, `created_at`, `system_valid_from`, `system_valid_to`) are converted to Unix epoch milliseconds (int64) before serialization:

```
epoch_ms = floor(datetime.timestamp() * 1000)
```

Example: `2026-01-15T10:00:00.000Z` → `1768471200000`

### 4.9 Serialization Algorithm

1. **Validate required fields** per memory type schema. Reject if missing.
2. **Compact field names** via FIELD_MAP (Section 5).
3. **Compact nested maps** in `content_refs` and `embedding_refs` only.
4. **Convert datetimes** to epoch milliseconds.
5. **NFC-normalize all strings** (recursive).
6. **Omit null/None values** (recursive).
7. **Sort map keys** lexicographically (recursive).
8. **Encode as MessagePack/CBOR** using rules above.
9. **Prepend version byte and header** — build the 9-byte header: `[0x01, flags, type, ns_hash_hi, ns_hash_lo, created_at_sec_b3, created_at_sec_b2, created_at_sec_b1, created_at_sec_b0]` where `ns_hash_hi:ns_hash_lo = SHA-256(namespace)[0:2]` as uint16 big-endian, and prepend to payload.
10. **Compute SHA-256** over complete blob bytes.

### 4.10 Nesting Depth Limit

Implementations SHOULD enforce a maximum nesting depth to prevent stack overflow vulnerabilities from adversarially or accidentally deeply nested payloads. Recommended limits by profile:

| Profile | Maximum Nesting Depth |
|---------|-----------------------|
| Extended | 32 levels |
| Standard | 16 levels |
| Lightweight | 8 levels |

Parsers MAY reject payloads exceeding their profile limit with `ERR_CORRUPT`.

---

## 5. Content Addressing

The content address of a .mg blob is computed as:

```
content_address = lowercase_hex(SHA-256(complete_blob_bytes))
```

Where `complete_blob_bytes` is the complete 9-byte fixed header followed by the canonical MessagePack/CBOR payload:
- Bytes 0–8: Fixed header (version, flags, type, ns_hash[2], created_at_sec[4])
- Bytes 9+: Canonical MessagePack/CBOR payload

The hash MUST be represented as a 64-character lowercase hexadecimal string. Uppercase hexadecimal MUST be rejected.

### 5.1 Content Address Format (ABNF)

```abnf
content-address = 64 HEXDIG
HEXDIG          = DIGIT / "a" / "b" / "c" / "d" / "e" / "f"
DIGIT           = %x30-39
```

### 5.2 Hash Function

SHA-256 is defined in [FIPS 180-4](https://csrc.nist.gov/publications/detail/fips/180/4/final). No alternative hash functions are permitted in v1.0.

### 5.3 Collision Resistance

SHA-256 provides 128-bit collision resistance (in practical terms). At 2^128 hashes, collision probability becomes significant. Current estimates suggest SHA-256 remains secure for the foreseeable future.

### 5.4 Content Address as Identity

The content address serves as:
- **Unique identifier** — filename in content-addressed stores
- **Integrity check** — any byte change produces different hash
- **Deduplication key** — byte-identical content maps to same address
- **Provenance link** — derived grains reference source hashes
- **Access key** — retrieve grain from store by address

### 5.5 Temporal Uniqueness of Content Addresses

The content address includes `created_at_sec` from the fixed header (bytes 5–8), which is part of the hashed bytes. Two grains with identical semantic payload but different creation timestamps produce different content addresses — creation time is part of grain identity.

**Rationale:** Binding the content address to the creation time ensures each write event is a unique, non-replayable grain. An adversary cannot substitute a grain with an older timestamp without producing a different hash, preserving audit chain integrity.

**Implication for deduplication:** Content-address deduplication applies only to byte-identical blobs (same payload encoded at the same creation second). For semantic deduplication — the same fact written at different times — use `superseded_by` to mark the older grain as replaced, or `derived_from` to express provenance. The phrase "identical content maps to same address" (§5.4) means byte-identical, including the creation timestamp.

### 5.6 Immutability Boundary

A grain has two distinct layers with different mutability guarantees:

| Layer | Contents | Mutability | Covered by content address | Covered by COSE signature |
|-------|----------|------------|---------------------------|--------------------------|
| **Blob** | 9-byte fixed header + MessagePack/CBOR payload | **Immutable** — once written, never modified | Yes | Yes |
| **Index** | Status and access-tracking fields (§28.3) | **Mutable** — updated by the store/index layer | No | No |

**A grain's content is the immutable blob identified by its content address. A grain's status is maintained in the index layer.** Index-layer fields — `superseded_by`, `system_valid_to`, `verification_status`, `rec_status`, `access_count`, `last_accessed_at` — are NOT part of the hashed blob bytes and are NOT covered by COSE signatures. They are managed exclusively by the store after initial write (see §28.3 for update rules).

This separation is fundamental to the OMS architecture:

1. **Integrity** — the content address guarantees the blob is unchanged. Index-layer mutations cannot alter a grain's identity or tamper with signed content.
2. **Lifecycle** — grains can be superseded, retracted, or verified without rewriting the original blob or invalidating its signature.
3. **Access tracking** — read counters and timestamps can be updated without breaking content addressing.

Implementations MUST store index-layer fields outside the .mg blob — in a database index, sidecar metadata, or equivalent external structure. Writers MUST NOT embed index-layer fields in the blob payload; stores MUST NOT recompute content addresses when index-layer fields change.

**Portability:** When grains are exported as .mg files, index-layer state is carried in the optional **index manifest** (§11.7). This preserves the "one file, full memory" principle — a .mg file contains both the immutable grain blobs and their current lifecycle state.

---

## 6. Field Compaction

To minimize blob size, human-readable field names are mapped to short keys before serialization. The mapping is bijective (one-to-one).

### 6.1 Core Fields

| Full Name | Short Key | Type | Description |
|-----------|-----------|------|-------------|
| `type` | `t` | string | Memory type: "fact", "episode", etc. |
| `subject` | `s` | string | Entity being described (RDF subject) |
| `relation` | `r` | string | Semantic relationship (RDF predicate) |
| `object` | `o` | string | Value or target (RDF object) |
| `confidence` | `c` | float64 | Credibility score [0.0, 1.0] |
| `source_type` | `st` | string | Provenance origin (open enum). Common values: `"user_explicit"`, `"consolidated"`, `"llm_generated"`, `"sensor"`, `"imported"`, `"agent_inferred"`, `"system"`. See note below. |
| `created_at` | `ca` | int64 | Creation timestamp (epoch ms) |
| `temporal_type` | `tt` | string | "state" or "observation" |
| `valid_from` | `vf` | int64 | Temporal validity start (epoch ms) |
| `valid_to` | `vt` | int64 | Temporal validity end (epoch ms) |
| `system_valid_from` | `svf` | int64 | When grain became active in system |
| `system_valid_to` | `svt` | int64 | When grain was superseded in system |
| `context` | `ctx` | map | Contextual metadata (string→string) |
| `superseded_by` | `sb` | string | Content address of superseding grain |
| `contradicted` | `ct` | bool | Whether this grain is contradicted |
| `importance` | `im` | float64 | Importance weighting [0.0, 1.0] |
| `author_did` | `adid` | string | DID of creating agent |
| `namespace` | `ns` | string | Memory partition/category |
| `user_id` | `user` | string | Associated data subject (GDPR) |
| `structural_tags` | `tags` | array[string] | Classification tags |
| `derived_from` | `df` | array[string] | Parent content addresses |
| `consolidation_level` | `cl` | int | 0=raw, 1=frequency, 2=pattern, 3=sequence |
| `success_count` | `sc` | int | Feedback: successful uses |
| `failure_count` | `fc` | int | Feedback: failed uses |
| `provenance_chain` | `pc` | array[map] | Full derivation trail |
| `origin_did` | `odid` | string | Original source agent DID |
| `origin_namespace` | `ons` | string | Original source namespace |
| `content_refs` | `cr` | array[map] | References to external content |
| `embedding_refs` | `er` | array[map] | References to vector embeddings |
| `related_to` | `rt` | array[map] | Cross-links to related grains |
| `_elided` | `_e` | map | Selective disclosure — elided field hashes |
| `_disclosure_of` | `_do` | string | Content address of original grain (if disclosed) |
| `invalidation_policy` | `ip` | map | Protection policy governing supersession and contradiction (see §23) |
| `supersession_justification` | `sj` | string | Required on superseding grain when original has `mode: "soft_locked"` |
| `supersession_auth` | `sa` | array | COSE signatures authorizing supersession for `mode: "quorum"` |
| `owner` | `own` | map | LegalEntity map (§12.5.1) — legal entity with rights and liabilities over the agent |
| `category` | `cat` | uint8 | Routing category within the grain type — see §27 Grain Type Field Specifications |
| `run_id` | `rid` | string | Session or run identifier — scopes grain to a specific agent execution. Distinct from `user_id` (data subject) and `namespace` (logical partition). |
| `role` | `role` | string | Message role for Event grains — open enum, standard values: `"user"`, `"assistant"`, `"system"`, `"tool"` |
| `access_count` | `ac` | int | Number of times this grain has been retrieved — updated by the store on reads, not by the writer. Enables recency/frequency scoring. |
| `last_accessed_at` | `laa` | int64 | Epoch ms of most recent retrieval — updated by the store on reads. Pair with `access_count` for importance decay models. |
| `timestamp_ms` | `tms` | int64 | High-precision payload timestamp (epoch ms). The authoritative event timestamp. The header's `created_at_sec` is a coarse routing hint only. |
| `observer_did` | `obsdid` | string | DID of the entity that observed or measured — distinct from `author_did` (who wrote the grain into the store). |
| `subject_did` | `sdid` | string | DID of the entity this grain is **about** — distinct from `user_id` (GDPR data subject) and `author_did` (writer). |
| `session_id` | `sid2` | string | Session scope — distinct from `run_id` (execution scope) and `user_id` (data subject). |
| `entity_id` | `eid` | string | External entity reference — product ID, patient MRN, vehicle chassis ID, instrument serial. Not a DID; opaque to the spec. |
| `epistemic_status` | `epstat` | string | Categorical certainty: `"certain"`, `"probable"`, `"uncertain"`, `"estimated"`, `"derived"`. Complements the continuous `confidence` float. Open enum. |
| `verification_status` | `vstatus` | string | Values: `"unverified"` (default), `"verified"`, `"contested"`, `"retracted"`. |
| `rec_status` | `rstat` | string | **Recommendation grains only (§8.12).** Review state: `"pending"` (default), `"approved"`, `"rejected"`, `"applied"`, `"rolled_back"`, `"expired"`. Rebuilt from the audit chain (§8.12.1), never author-written. |
| `requires_human_review` | `rhr` | bool | If `true`, this grain's content MUST NOT drive automated decisions until a human has reviewed and cleared it. Binding for Reasoning grains; advisory for others. |
| `processing_basis` | `pbasis` | string | Content address of the Consent grain that authorized this grain's creation. Used to compute erasure scope on consent revocation. |
| `identity_state` | `idst` | string | Identity resolution state: `"anonymous"`, `"pseudonymous"`, `"authenticated"`. Affects personalization logic and compliance scope. |
| `license` | `lic` | string | SPDX license identifier for the grain's content. Example: `"CC-BY-4.0"`, `"CC0-1.0"`, `"proprietary"`. |
| `trusted_timestamp` | `tts` | map | RFC 3161 timestamp token: `{tsp_response: bytes, tsa_uri: string}`. Legally defensible creation time from an accredited TSA, independent of self-reported `created_at`. |
| `invalidation_type` | `itype` | string | Semantic reason for supersession: `"superseded"`, `"retraction"`, `"erratum"`, `"corrigendum"`, `"retraction_with_replacement"`, `"expression_of_concern"`. Set by actor creating the superseding grain. |
| `invalidation_reason` | `ireason` | string | Human-readable rationale for `invalidation_type`. |
| `invalidation_initiator` | `iinit` | string | DID of the party initiating the invalidation. |
| `retention_policy` | `rpol` | map | Minimum retention requirements: `{minimum_retention_years: int, regulation: string, deletion_requires: string}`. Distinct from `invalidation_policy` (which controls supersession). |
| `recall_priority` | `rpri` | string | Retrieval priority hint: `"hot"`, `"warm"`, `"cold"`. Guides index layer storage tier selection. |
| `embedding_text` | `et` | string | Source text for vector embedding and full-text indexing. When present, implementations SHOULD use this value instead of the grain's default per-type text representation for embedding generation and search indexing. Complements `embedding_refs` (which records generated vectors) by specifying the input text. Implementations MAY impose a size limit (RECOMMENDED: 8192 bytes). |

> **Note — `source_type` for Observation grains:** Use `"sensor"` when `observer_type` is a physical instrument; `"agent_inferred"` when `observer_type` is a cognitive AI observer (`"llm"`, `"reflector"`, `"classifier"`, `"detector"`); `"user_explicit"` for human observers.

> **Index-layer fields (§5.6, §28.3):** The following fields in the table above are **not stored in the immutable .mg blob**. They are maintained by the store/index layer and are excluded from the content address and COSE signature: `superseded_by`, `system_valid_to`, `verification_status`, `rec_status`, `access_count`, `last_accessed_at`. Writers MUST NOT set these fields; see §28.3 for store update rules. `rec_status` is listed here — rather than only in §8.12 — so that this prohibition binds it; it is the one entry scoped to a single grain type, and it is meaningless on any grain that is not a Recommendation.

### 6.2 Event-Specific Fields

| Full Name | Short Key | Type | Notes |
|-----------|-----------|------|-------|
| `content` | `content` | string | Raw text of the event. MAY be omitted if `content_blocks` is present. |
| `consolidated` | `consolidated` | bool | Whether this event has been distilled into Fact grains |
| `content_blocks` | `cblocks` | array[map] | Typed content blocks for structured LLM messages. When present, takes precedence over flat `content` string. Each entry: `{type: "text"/"image"/"tool_use"/"tool_result"/"thinking", ...}`. See note below. |
| `model_id` | `mdl` | string | LLM model identifier that produced the response (e.g., `"claude-opus-4-6"`, `"gpt-4o"`). Absent for human-authored events. |
| `stop_reason` | `stopr` | string | Why LLM generation stopped: `"end_turn"`, `"max_tokens"`, `"stop_sequence"`, `"tool_use"`. Open enum. |
| `token_usage` | `toku` | map | Token consumption: `{input_tokens: int, output_tokens: int, cache_creation_tokens: int, cache_read_tokens: int}`. Enables cost tracking. |
| `parent_message_id` | `pmid` | string | Content address of the preceding message grain in the conversation thread. Enables linked-list message threading and conversation branching (two Event grains sharing the same `parent_message_id` represent a branch point). |

> **Note — `content_blocks` schema:** Each block in the array MUST contain a `type` field. Standard block types mirror the Anthropic Messages API: `"text"` (`{type, text}`), `"image"` (`{type, source}`), `"tool_use"` (`{type, id, name, input}`), `"tool_result"` (`{type, tool_use_id, content, is_error}`), `"thinking"` (`{type, thinking}`). Implementations MAY define additional block types. When `content_blocks` is present and `content` is also present, `content` serves as a plain-text fallback for readers that do not support structured blocks.

### 6.3 State-Specific Fields

| Full Name | Short Key | Type |
|-----------|-----------|------|
| `plan` | `plan` | array[string] |
| `history` | `history` | array[map] |

### 6.4 Workflow-Specific Fields

| Full Name | Short Key | Type | Notes |
|-----------|-----------|------|-------|
| `nodes` | `nodes` | array[string] | Graph node IDs/labels |
| `edges` | `edges` | array[map] | Directed edges (see §8.4 edge schema) |
| `bindings` | `bind` | map[string→string] | Node ID → Tool definition grain hash |
| `retries` | `retries` | map[string→int] | Node ID → max repeat count |

**Edge nested field compaction:**

| Full Name | Short Key | Type |
|-----------|-----------|------|
| `src` | `src` | string |
| `dst` | `dst` | string |
| `cond` | `cond` | string |
| `max_cycles` | `mxc` | int |

### 6.5 Tool-Specific Fields

| Full Name | Short Key | Type | Notes |
|-----------|-----------|------|-------|
| `tool_phase` | `aphase` | string | Discriminator: `"definition"` \| `"call"` \| `"result"` \| absent = complete |
| `tool_name` | `tn` | string | |
| `input` | `inp` | map | Canonical name for tool arguments (replaces `arguments`) |
| `content` | `cnt` | any | Canonical name for tool result (replaces `result`) |
| `is_error` | `iserr` | bool | Canonical error flag (replaces `success`) |
| `tool_call_id` | `tcid` | string | Anthropic/MCP correlation ID; links result phase to call phase |
| `call_batch_id` | `cbid` | string | Groups parallel calls issued in the same agent turn |
| `tool_type` | `ttype` | string | `"client"` \| `"server"` \| `"builtin"` |
| `tool_version` | `tver` | string | For versioned builtins, e.g. `"web_search_20250305"` |
| `execution_mode` | `emode` | string | `"function_call"` \| `"code_exec"` \| `"computer_use"` |
| `code` | `code` | string | Executable code for `execution_mode: "code_exec"` (CodeAct) |
| `stdout` | `out` | string | Standard output from code execution |
| `stderr` | `err2` | string | Standard error from code execution |
| `exit_code` | `xc` | int | Process exit code from code execution |
| `interpreter_id` | `iid` | string | Links Tool grains sharing a stateful interpreter session |
| `error` | `err` | string | Error message (use with `is_error: true`) |
| `error_type` | `etype` | string | Structured error classification: `"timeout"`, `"rate_limit"`, `"auth_failure"`, `"invalid_input"`, `"server_error"`, `"not_found"`, `"quota_exceeded"`. Open enum. Enables retry policy decisions without parsing free-text `error`. |
| `duration_ms` | `dur` | int | Execution time in milliseconds |
| `parent_task_id` | `ptid` | string | Content address of parent task grain |
| `tool_description` | `tdesc` | string | Human-readable description of the tool (definition phase) |
| `input_schema` | `isch` | map | JSON Schema for tool inputs; mirrors Anthropic `input_schema` / MCP `inputSchema` (definition phase) |
| `output_schema` | `osch` | map | JSON Schema (draft-07 compatible) describing the tool's return value (definition phase) |
| `strict` | `strict` | bool | If `true`, model guarantees strict JSON Schema conformance for `input` (definition phase) |

### 6.6 Observation-Specific Fields

| Full Name | Short Key | Type |
|-----------|-----------|------|
| `observer_id` | `oid` | string |
| `observer_type` | `otype` | string |
| `frame_id` | `fid` | string |
| `sync_group` | `sg` | string |
| `observation_mode` | `omode` | string |
| `observation_scope` | `oscope` | string |
| `observer_model` | `omdl` | string |
| `compression_ratio` | `ocmp` | float64 |

### 6.7 Goal-Specific Fields

| Full Name | Short Key | Type |
|-----------|-----------|------|
| `description` | `desc` | string |
| `goal_state` | `gs` | string |
| `criteria` | `crit` | array[string] |
| `criteria_structured` | `crs` | array[map] |
| `priority` | `pri` | int |
| `parent_goals` | `pgs` | array[string] |
| `state_reason` | `sr` | string |
| `satisfaction_evidence` | `se` | array[string] |
| `progress` | `prog` | float64 |
| `delegate_to` | `dto` | string |
| `delegate_from` | `dfo` | string |
| `expiry_policy` | `ep` | string |
| `recurrence` | `rec` | string |
| `evidence_required` | `evreq` | int |
| `rollback_on_failure` | `rof` | array[string] |
| `allowed_transitions` | `atr` | array[string] |
| `depends_on` | `depg` | array[string] |
| `assigned_agent` | `asgn` | string |
| `expected_output` | `expout` | string |
| `output_grain` | `outg` | string |
| `deadline` | `dline` | int64 |

### 6.8 Consent-Specific Fields

> **Note:** `subject_did` (short key `sdid`) is a common field (§6.1) used here as the consenting party. `grantee_did` is Consent-specific.

| Full Name | Short Key | Type |
|-----------|-----------|------|
| `grantee_did` | `gdid` | string |
| `scope` | `scope` | array[string] |
| `is_withdrawal` | `isw` | bool |
| `basis` | `basis` | string |
| `jurisdiction` | `jur` | string |
| `prior_consent` | `pcon` | string |
| `witness_dids` | `wdids` | array[string] |

### 6.9 Reasoning-Specific Fields

| Full Name | Short Key | Type |
|-----------|-----------|------|
| `premises` | `prem` | array[string] |
| `conclusion` | `conc` | string |
| `inference_method` | `imethod` | string |
| `alternatives_considered` | `altc` | array[map] |
| `thinking_content` | `think` | string |
| `thinking_redacted` | `tredact` | bool |
| `statistical_context` | `statctx` | map |
| `software_environment` | `swenv` | map |
| `parameter_set` | `params` | map |
| `random_seed` | `rseed` | int64 |

### 6.10 Consensus-Specific Fields

| Full Name | Short Key | Type |
|-----------|-----------|------|
| `participating_observers` | `pobs` | array[string] |
| `threshold` | `thold` | int |
| `agreement_count` | `agcnt` | int |
| `dissent_count` | `discnt` | int |
| `dissent_grains` | `disgrn` | array[string] |
| `agreed_content` | `agcon` | any |

### 6.11 Delegation-Specific Fields

When a Goal or Fact grain uses the `mg:delegates_to` relation, the following fields specify the scope and constraints of the delegation. Without these fields, a delegation is unbounded — the delegatee receives no machine-readable limits. Implementations SHOULD populate delegation scope fields for any inter-agent authority grant.

| Full Name | Short Key | Type | Notes |
|-----------|-----------|------|-------|
| `authorized_namespaces` | `ans` | array[string] | Namespaces the delegatee may read and write. `["*"]` = all namespaces (dangerous — SHOULD be avoided). |
| `authorized_types` | `atypes` | array[uint8] | Grain type bytes the delegatee may create. E.g., `[0x01, 0x02, 0x05]` for Fact, Event, Tool. |
| `authorized_tools` | `atools` | array[string] | Tool names the delegatee may invoke. Empty array = no tool restriction. |
| `delegation_depth` | `ddepth` | int | Maximum re-delegation depth. 0 = delegatee MUST NOT re-delegate. Absent = unlimited (NOT RECOMMENDED). |
| `delegation_expiry` | `dexp` | int64 | Epoch ms when delegation expires. After expiry, the delegatee's writes SHOULD be rejected by stores that enforce delegation scope. |
| `context_grains` | `cgrains` | array[string] | Content addresses of grains to transfer as context to the delegatee. Enables session handoff: the delegator selects which grains the delegatee needs to continue. |
| `return_to` | `retdid` | string | DID of the agent to return control to after the delegated task completes. |

### 6.12 Skill-Specific Fields

> **Note:** `description` reuses the shared `desc` key (also used by Goal, §6.7). `holder_did` (`hdid`) identifies the agent that possesses the skill, distinct from the common `author_did` (`adid`) which records the grain's writer. Bundled binary/file payloads use the common `content_refs` field (`cr`, §6.1); `license` uses the common `lic` key.

| Full Name | Short Key | Type | Group |
|-----------|-----------|------|-------|
| `name` | `skname` | string | definition |
| `instructions` | `instr` | string | definition |
| `when_to_use` | `wtu` | string | definition |
| `version` | `sver` | string | definition |
| `allowed_tools` | `atls` | array[string] | definition |
| `resources` | `res` | array[string] | definition |
| `dependencies` | `deps` | array[string] | definition |
| `input_modalities` | `imod` | array[string] | definition |
| `output_modalities` | `omod` | array[string] | definition |
| `domain` | `dom` | string | definition |
| `holder_did` | `hdid` | string | learned |
| `proficiency` | `prof` | float64 | learned |
| `practice_count` | `prcnt` | int | learned |
| `last_practiced_at` | `lpa` | int64 | learned |
| `strategies` | `strat` | array[map] | learned |
| `transferable` | `xfer` | bool | learned |

### 6.13 Recommendation-Specific Fields

> **Note:** A Recommendation reuses common fields for evidence and scoring — `derived_from` (`df`, evidence hashes), `confidence` (`c`), `importance` (`im`), and `valid_to` (`vt`, expiry). The review state (`rec_status`) is deliberately absent from the table below: it is an index-layer field, not a blob field, so it is compacted in §6.1 (short key `rstat`) alongside `verification_status` and is reconstructed from the audit chain (§8.12.1) rather than written by an author.

| Full Name | Short Key | Type |
|-----------|-----------|------|
| `target_ref` | `tref` | string |
| `analyzer` | `anlz` | map |
| `summary` | `summ` | map |
| `dedup_key` | `ddk` | string |
| `proposal_cal` | `pcal` | string |
| `proposal_edit` | `pedit` | map |
| `proposal_data` | `pdata` | map |
| `severity` | `sev` | string |
| `metric_snapshot` | `msnap` | map |
| `evidence_query` | `evq` | string |

### 6.14 Trigger-Specific Fields

| Full Name | Short Key | Type | Notes |
|-----------|-----------|------|-------|
| `kind` | `aknd` | string | Shared with Tool's `kind` — same field name, same short key; the type byte disambiguates the value space |
| `workflow` | `twf` | string | Content address of the Workflow this trigger starts |
| `connector` | `tcon` | string | |
| `scope` | `tscp` | string | |
| `enabled` | `tena` | bool | **Always emitted**, unlike most defaulted booleans — see below |
| `dedup_key` | `tdk` | array[string] | |
| `interval_secs` | `tint` | int | |
| `cron` | `tcron` | string | |
| `at_ms` | `tat` | int64 | |
| `predicate` | `tpred` | map | Inner keys are the expression's own and are NOT compacted |
| `members` | `tmem` | map[string→string] | Alias → trigger content address |
| `correlate` | `tcorr` | string | |
| `window_ms` | `twin` | int64 | |
| `concurrency` | `tconc` | string | |
| `catchup` | `tcatch` | string | |
| `config` | `tcfg` | map | `int:` keys compact per §A.7; connector-specific keys stay verbatim |

> **`enabled` is a deliberate exception to omit-when-default.** Omitting a
> defaulted value exists to keep *pre-existing* blobs byte-identical across a
> rollout, and a newly registered type has none. Omitting it would instead break
> the most natural query over triggers — selecting the live ones — because the
> field would be absent on every one of them. A type whose fields are declared
> so they can be queried must not then hide the field people query by.

### 6.14 Compaction Rules

- Serializers MUST replace full field names with short keys before encoding
- Deserializers MUST replace short keys with full field names after decoding
- Unknown keys (not in mapping) MUST be preserved as-is in both directions
- Field compaction mapping is normative and MUST NOT be modified by implementations

---

## 7. Multi-Modal Content References

Multi-modal content (images, audio, video, embeddings, sensor data) is referenced by URI, never embedded in grains.

### 7.1 Content Reference Schema

```json
{
  "uri": "cas://sha256:abc123...",
  "modality": "image",
  "mime_type": "image/jpeg",
  "size_bytes": 1048576,
  "checksum": "sha256:abc123...",
  "metadata": {"width": 1920, "height": 1080}
}
```

**Field compaction for content_refs entries:**

| Full Name | Short Key | Type | Required | Description |
|-----------|-----------|------|----------|-------------|
| `uri` | `u` | string | REQUIRED | Content URI |
| `modality` | `m` | string | REQUIRED | Content type: image, audio, video, point_cloud, 3d_mesh, document, binary, embedding |
| `mime_type` | `mt` | string | RECOMMENDED | Standard MIME type |
| `size_bytes` | `sz` | int | OPTIONAL | File size in bytes |
| `checksum` | `ck` | string | RECOMMENDED | SHA-256 hash for integrity |
| `metadata` | `md` | map | OPTIONAL | Modality-specific metadata |

### 7.2 Embedding Reference Schema

```json
{
  "vector_id": "vec-12345",
  "model": "text-embedding-3-large",
  "dimensions": 3072,
  "modality_source": "text",
  "distance_metric": "cosine"
}
```

**Field compaction for embedding_refs entries:**

| Full Name | Short Key | Type | Required | Description |
|-----------|-----------|------|----------|-------------|
| `vector_id` | `vi` | string | REQUIRED | ID in vector store |
| `model` | `mo` | string | REQUIRED | Embedding model name |
| `dimensions` | `dm` | int | REQUIRED | Vector dimensionality |
| `modality_source` | `ms` | string | OPTIONAL | Source modality: "text", "image", "audio", etc. |
| `distance_metric` | `di` | string | OPTIONAL | "cosine", "l2", "dot" |
| `chunk_index` | `ci` | int | OPTIONAL | Position of this chunk within the source grain (0-indexed). When a grain is embedded as a single unit, `chunk_index` = 0. |
| `chunk_text` | `ct` | string | OPTIONAL | The exact text that was embedded. Enables reconstruction from a vector search hit without re-reading and re-chunking the source grain. |
| `chunk_strategy` | `cs` | string | OPTIONAL | Chunking method: `"full"` (entire grain), `"sentence"`, `"paragraph"`, `"token_window"`, `"recursive"`, `"semantic"`. Open enum. |
| `chunk_overlap` | `co` | int | OPTIONAL | Overlap in tokens between adjacent chunks. Absent or 0 for non-overlapping strategies. |

> **Note — RAG round-trip:** When a vector search returns a hit, the `chunk_text` field enables immediate context assembly without a second read of the source grain. The `chunk_index` + `chunk_strategy` fields enable re-chunking validation. Implementations that generate embeddings internally MUST populate `embedding_refs` entries on the grain. Implementations that delegate to an external vector store SHOULD populate `chunk_text` to ensure retrieval provenance is self-contained.

### 7.3 Modality-Specific Metadata

**Image:**
```json
{"width": 1920, "height": 1080, "color_space": "sRGB"}
```

**Audio:**
```json
{"sample_rate_hz": 48000, "channels": 2, "duration_ms": 15000}
```

**Video:**
```json
{"width": 3840, "height": 2160, "fps": 30, "duration_ms": 120000, "codec": "h264"}
```

**Point Cloud:**
```json
{"point_count": 1234567, "format": "pcd_binary", "has_color": true}
```

---

## 8. Grain Types

The `type` byte (Byte 2 of the fixed header) encodes the **cognitive grain type** — the class of knowledge unit this grain represents. Twelve standard types are defined.

### Standard `mg:` Relation Vocabulary

The `mg:` namespace is reserved for standard semantic relations. Applications define custom relations freely outside this namespace.

| Relation | Typical grain type | Meaning |
|---|---|---|
| `mg:perceives` | Observation | Raw sensory or cognitive input |
| `mg:knows` | Fact | Derived or learned fact |
| `mg:said` | Event | Message or utterance |
| `mg:did` | Tool | Tool or action invocation |
| `mg:infers` | Reasoning | Derived conclusion from prior grains |
| `mg:agrees_with` | Consensus | Multi-agent threshold agreement |
| `mg:state_at` | State | Agent state snapshot |
| `mg:has_graph` | Workflow | Directed graph of procedural steps |
| `mg:intends` | Goal | Agent objective |
| `mg:permits` | Consent, Fact | User grants agent right to retain or act (Consent); as a Fact in `agent:authz`, an authorization grant (§12.6.2) |
| `mg:revokes` | Consent | User revokes prior consent (data-subject domain only — authorization revocation is retraction-by-supersession, §12.6.3) |
| `mg:prohibits` | Fact/Goal | Hard prohibition |
| `mg:requires` | Fact/Goal | Hard requirement |
| `mg:prefers` | Fact | Soft preference |
| `mg:avoids` | Fact | Soft avoidance preference |
| `mg:delegates_to` | Goal | Scoped authority grant (§6.11 delegation scope) |
| `mg:owned_by` | Fact | Legal entity ownership (§12.5) |
| `mg:has_capability` | Fact | Agent capability advertisement (§28.5 Agent Card) |
| `mg:handed_off_to` | Event | Session handoff event record (§28.7) |
| `mg:depends_on` | Goal | Task dependency (distinct from `parent_goals` hierarchy) |
| `mg:assigned_to` | Goal | Task assigned to agent for execution |
| `mg:capable_of` | Skill | Learned skill with proficiency and strategies (§8.11; legacy Fact convention retained for back-compat) |

### 8.1 Fact (type = 0x01)

A structured fact about the world — a (subject, relation, object) triple with confidence and source. The canonical unit of declarative knowledge.

**Required fields:**
- `type` = "fact" (payload string; header byte = `0x01`)
- `subject` (non-empty string)
- `relation` (non-empty string)
- `object` (string or map)
- `confidence` (float64, [0.0, 1.0])
- `created_at` (int64, epoch ms)

**Optional fields:** All common fields from §6.1. Type-specific: `temporal_type`, `success_count`, `failure_count`, bi-temporal fields (`valid_from`, `valid_to`, `system_valid_from`, `system_valid_to`).

**RDF mapping:** `<grain:subject> <grain:relation> "grain:object" .`

### 8.2 Event (type = 0x02)

A raw, timestamped record of something that happened — a message, interaction, utterance, or behavioral occurrence.

**Required fields:**
- `type` = "event"
- `content` (non-empty string) — raw text. MAY be omitted if `subject`/`relation`/`object` fully describe the event.
- `created_at` (int64, epoch ms)

**Optional fields:** `role` (`"user"`, `"assistant"`, `"system"`, `"tool"`), `content_blocks` (array[map] — structured multi-block content; takes precedence over flat `content`), `model_id` (string), `stop_reason` (string), `token_usage` (map), `parent_message_id` (string — content address of preceding message for conversation threading), `consolidated` (bool), `run_id` (string), `session_id` (string), all common fields.

### 8.3 State (type = 0x03)

An agent state snapshot — the portable save point at a moment in time.

**Required fields:**
- `type` = "state"
- `context` (map) — agent state snapshot. For Letta-compatible agents, SHOULD include `memory_blocks`, `system_prompt`, `tools`, `model`.
- `created_at` (int64, epoch ms)

**Optional fields:** `plan` (array[string]), `history` (array[map]), all common fields.

### 8.4 Workflow (type = 0x04)

Directed graph of procedural steps — plans, pipelines, and multi-path processes.

**Required fields:**
- `type` = "workflow"
- `nodes` (non-empty array[string]) — graph node identifiers. Each string serves as both ID and human-readable label. Must be unique within the grain.
- `created_at` (int64, epoch ms)

**Optional fields:**
- `edges` (array[map]) — directed edges between nodes (see edge schema below). When absent, nodes are unconnected.
- `bindings` (map[string→string]) — maps node IDs to Tool definition grain hashes (`tool_phase: "definition"`). Unbound nodes are resolved by convention (tool name match) or treated as abstract steps.
- `retries` (map[string→int]) — maps node IDs to maximum repeat count on failure. Absent means no retry.
- All common fields.

**Edge schema** (each element of `edges`):

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `src` | string | yes | ID of the source node (must exist in `nodes`) |
| `dst` | string | yes | ID of the target node (must exist in `nodes`) |
| `cond` | string | no | Opaque condition string; absent means unconditional |
| `max_cycles` | int | no | Maximum traversal count for back-edges; absent means unlimited |

**Structural semantics** (inferred from graph topology, not declared):
- A node with multiple outgoing unconditional edges implies **parallel fan-out** (fork).
- A node that is the target of multiple edges implies an **AND-join** (all predecessors must complete).
- A node with multiple outgoing conditional edges (`cond` present) implies a **decision point**.
- A node with no outgoing edges is a **terminal node**.
- The **entry point** is the first element of `nodes`.
- Back-edges (forming cycles) are permitted when bounded by `max_cycles` or `retries`.

**Validation constraints:**
- Every `src` and `dst` in `edges` MUST reference an element of `nodes`.
- Every key in `bindings` MUST reference an element of `nodes`.
- Every key in `retries` MUST reference an element of `nodes`.
- Node IDs MUST be unique within `nodes`.
- Unbounded cycles (back-edge without `max_cycles`, and no entry in `retries` for the target node) SHOULD be rejected by implementations.

**Node-to-Tool binding** (three-tier resolution):

| Tier | Condition | Resolution |
|------|-----------|------------|
| Bound | Node ID present in `bindings` | Fetch the Tool definition grain by hash; use its `tool_name`, `input_schema`, `output_schema` |
| Named | No binding, but a Tool definition grain exists with matching `tool_name` | Resolve by convention (implementation-defined lookup) |
| Abstract | No binding, no matching tool | Node label is a human-readable instruction; executor (LLM or agent) interprets it |

**Execution record relation:** When an agent executes a workflow node, it creates a Tool grain (phase: complete or call/result) with a relation of type `mg:step_action:<node_id>` targeting the Workflow grain hash. This links execution records to the plan without modifying the immutable Workflow grain.

**Example — linear pipeline:**

```json
{
  "type": "workflow",
  "nodes": ["build", "test", "deploy"],
  "edges": [
    {"src": "build", "dst": "test"},
    {"src": "test", "dst": "deploy"}
  ],
  "created_at": 1711324800000
}
```

**Example — parallel fork/join:**

```json
{
  "type": "workflow",
  "nodes": ["lint", "security", "compliance", "evaluate"],
  "edges": [
    {"src": "lint", "dst": "security"},
    {"src": "lint", "dst": "compliance"},
    {"src": "security", "dst": "evaluate"},
    {"src": "compliance", "dst": "evaluate"}
  ],
  "created_at": 1711324800000
}
```

**Example — conditional branching with retry and bindings:**

```json
{
  "type": "workflow",
  "nodes": ["build", "unit_test", "lint", "integration_test", "stage_deploy", "approval", "prod_deploy", "rollback", "notify"],
  "edges": [
    {"src": "build", "dst": "unit_test"},
    {"src": "build", "dst": "lint"},
    {"src": "unit_test", "dst": "integration_test"},
    {"src": "lint", "dst": "integration_test"},
    {"src": "integration_test", "dst": "stage_deploy"},
    {"src": "stage_deploy", "dst": "approval"},
    {"src": "approval", "dst": "prod_deploy", "cond": "approved"},
    {"src": "approval", "dst": "rollback", "cond": "rejected"},
    {"src": "prod_deploy", "dst": "notify"},
    {"src": "rollback", "dst": "notify"}
  ],
  "bindings": {
    "build": "sha256:def111...",
    "stage_deploy": "sha256:def333...",
    "prod_deploy": "sha256:def444..."
  },
  "retries": {
    "stage_deploy": 3
  },
  "created_at": 1711324800000
}
```

### 8.5 Tool (type = 0x05)

A record of a tool invocation, code execution, or computer-use action. See §27.1 for the full `tool_phase` discriminator and field tables.

**Required fields:**
- `type` = "tool"
- Phase-dependent required fields (see §27.1)
- `created_at` (int64, epoch ms)

### 8.6 Observation (type = 0x06)

Raw sensory or cognitive input — what an observer perceived at a moment in time.

**Required fields:**
- `type` = "observation"
- `observer_id` (non-empty string) — unique identifier of the observing entity
- `observer_type` (non-empty string) — open enum, see §24
- `created_at` (int64, epoch ms)

**Optional fields:** `observer_model`, `frame_id`, `sync_group`, `observation_mode`, `observation_scope`, `compression_ratio`, all common fields.

### 8.7 Goal (type = 0x07)

An explicit objective with lifecycle semantics. Goals transition through states via the supersession chain.

**Required fields:**
- `type` = "goal"
- `description` (non-empty string)
- `goal_state` (string enum) — `"active"`, `"satisfied"`, `"failed"`, `"suspended"`
- `created_at` (int64, epoch ms)

**Optional fields:** `criteria`, `criteria_structured`, `priority`, `parent_goals`, `depends_on` (array[string] — content addresses of prerequisite Goal grains that must complete before this one starts; distinct from `parent_goals` which implies decomposition, not dependency ordering), `assigned_agent` (string — DID of the agent assigned to execute this task), `expected_output` (string — description of expected output format), `output_grain` (string — content address of the grain containing the task's completed output), `deadline` (int64 — epoch ms hard deadline for task completion), `state_reason`, `satisfaction_evidence`, `progress`, `delegate_to`, `delegate_from`, `expiry_policy`, `recurrence`, `evidence_required`, `rollback_on_failure`, `allowed_transitions`, all common fields.

Constraints, policies, and delegations are expressed as Goal or Fact grains with `mg:prohibits`, `mg:prefers`, `mg:avoids`, or `mg:delegates_to` relations, combined with `invalidation_policy` (§23) for enforcement.

> **Note — plan-and-execute agents:** The `depends_on` field enables DAG-structured task dependency graphs for hierarchical task decomposition. Agents using plan-and-execute patterns (e.g., LangGraph StateGraph, CrewAI task dependencies) SHOULD express task ordering via `depends_on` and task hierarchy via `parent_goals`. A Goal grain with `depends_on` references MUST NOT transition to `goal_state: "active"` until all referenced Goal grains have `goal_state: "satisfied"`. The `assigned_agent` field enables multi-agent task routing: the orchestrator creates Goal grains with `assigned_agent` pointing to worker agent DIDs.

### 8.8 Reasoning (type = 0x08)

An inference step or thought chain — what the agent considered, concluded, and rejected. Enables audit trails for high-stakes decisions.

**Required fields:**
- `type` = "reasoning"
- `created_at` (int64, epoch ms)

**Optional fields:**
- `premises` (array[string]) — content addresses of grains that informed this reasoning
- `conclusion` (string) — the conclusion reached
- `inference_method` (string) — `"deductive"`, `"inductive"`, `"abductive"`, `"analogical"`
- `alternatives_considered` (array[map]) — rejected hypotheses, each: `{hypothesis: string, rejection_reason: string}`
- `thinking_content` (string) — raw thinking/reasoning trace from the LLM's extended thinking feature (e.g., Anthropic `thinking` blocks). Distinct from `conclusion` (the output) and `premises` (the inputs). This is the primary audit artifact.
- `thinking_redacted` (bool) — if `true`, the LLM's thinking was present but redacted before storage (e.g., for compliance or IP protection). The `thinking_content` field will be absent or contain a placeholder.
- `requires_human_review` (bool) — if `true`, MUST NOT drive automated decisions until cleared
- `statistical_context` (map) — `{p_value: float, confidence_interval: [float, float], effect_size: float, sample_size: int}`
- `software_environment` (map) — `{language: string, runtime_version: string, library_versions: map, os: string}`
- `parameter_set` (map) — model parameters or hyperparameters used
- `random_seed` (int64) — for reproducibility
- All common fields from §6.1

### 8.9 Consensus (type = 0x09)

A multi-agent agreement record — N observers voted on a shared claim, threshold was met (or not).

**Required fields:**
- `type` = "consensus"
- `participating_observers` (array[string]) — DIDs of agents that contributed votes
- `threshold` (int) — minimum agreement count required
- `agreement_count` (int) — actual agreement count
- `dissent_count` (int) — disagreement count
- `created_at` (int64, epoch ms)

**Optional fields:**
- `dissent_grains` (array[string]) — content addresses of minority-opinion grains
- `agreed_content` (string or map) — the consensus claim
- All common fields from §6.1

### 8.10 Consent (type = 0x0A)

A DID-scoped, purpose-bounded permission grant or withdrawal. Four of six industry review domains independently required a dedicated Consent type at the type-byte level — HIPAA patient consent, legal privilege and DPA, regulatory consent, and GDPR/CCPA at scale. The `Fact + mg:permits` pattern is semantically correct but impractical when consent queries are compliance-critical and frequent.

**Required fields:**
- `type` = "consent"
- `subject_did` (string) — DID of the consenting party
- `grantee_did` (string) — DID of the party receiving permission
- `scope` (array[string]) — operations consented to. Standard values: `"store"`, `"retrieve"`, `"share"`, `"process"`, `"infer"`, `"train"`, `"profile"`. Open enum.
- `is_withdrawal` (bool) — `true` if revoking a prior consent
- `created_at` (int64, epoch ms)

**Optional fields:**
- `valid_from`, `valid_to` (int64, epoch ms) — consent window
- `basis` (string) — `"explicit_consent"`, `"legitimate_interest"`, `"contract"`, `"legal_obligation"`. Open enum.
- `jurisdiction` (string) — `"eu"`, `"us_ccpa"`, `"us_hipaa"`, `"br_lgpd"`. Open enum.
- `prior_consent` (string) — content address of the Consent grain being superseded (REQUIRED when `is_withdrawal: true`)
- `witness_dids` (array[string]) — DIDs of witness agents

**Normative rules:**
1. A Consent grain with `is_withdrawal: true` MUST reference `prior_consent`.
2. Stores MUST honor consent withdrawal immediately. Consent grains MUST NOT be subject to automatic forgetting or retention decay.
3. A withdrawn Consent grain is NOT deleted — both grant and withdrawal are retained for audit.
4. Default `invalidation_policy.mode` for Consent grains is `"soft_locked"`.
5. The `processing_basis` common field (§6.1) on any grain carries the content address of the Consent grain that authorized its creation — enabling GDPR Art. 17 erasure cascade.

### 8.11 Skill (type = 0x0B)

A reusable, self-contained agent capability — the durable definition of *what a capability is and how to perform it*, plus optional records of *how well a given agent performs it*. A Skill grain is the unit an agent loads to acquire a capability: a name, a description of when to use it, the procedural instructions, the tools it may invoke, the resources it bundles, and the other skills it depends on. The same type also carries optional **learned-competence** fields (`proficiency`, `practice_count`, strategies) so an agent can record and improve its mastery of a skill over time.

This type promotes the §28.8 Skill Convention (formerly a `Fact + mg:capable_of` pattern) to a first-class grain, following the precedent set by Consent (§8.10). Skill grains became performance-critical for authoring, discovery, and transfer across multi-agent deployments, where the `object`-map convention was semantically correct but impractical to query at scale.

Where an Agent Capability Fact (§28.5, `mg:has_capability`) is a static identity card of what an agent *is built to do*, and a Workflow (§8.4) is a single fixed procedure, a Skill is a packaged, transferable capability — optionally adaptive and practiced.

**Required fields:**
- `type` = "skill"
- `name` (string) — machine-readable skill identifier (e.g., `"code_review"`)
- `description` (string) — human-readable summary of what the skill does and when to use it
- `created_at` (int64, epoch ms)

**Optional — definition fields** (the durable, shareable specification):
- `instructions` (string) — the procedural body: how to perform the skill (the prose/markdown "skill body" an agent loads when applying it)
- `when_to_use` (string) — natural-language activation cue describing the conditions under which the skill SHOULD be selected (a discovery and routing signal)
- `version` (string) — opaque, skill-defined version string (e.g., `"2.1.0"`). Supersession by system time remains authoritative for "latest" (§5).
- `allowed_tools` (array[string]) — content addresses or identifiers of Tool definition grains (§27.1) the skill is permitted to invoke
- `resources` (array[string]) — content addresses of bundled resource grains the skill ships with (scripts, templates, reference files). Binary or large file payloads are carried via the common `content_refs` field (§6.1).
- `dependencies` (array[string]) — content addresses of Skill grains that must be available before this skill is effective
- `input_modalities` (array[string]) — what the skill operates on: `"text"`, `"image"`, `"code"`, `"audio"`. Open enum.
- `output_modalities` (array[string]) — what the skill produces. Open enum.
- `domain` (string) — domain context: `"software"`, `"research"`, `"healthcare"`, `"finance"`. Open enum.

**Optional — learned-competence fields** (a particular agent's mastery; omit for a pure definition):
- `holder_did` (string) — DID of the agent that holds and practices this skill
- `proficiency` (float64, [0.0, 1.0]) — current mastery level
- `practice_count` (int) — number of successful applications (mirrors `success_count` in §6.1)
- `last_practiced_at` (int64, epoch ms) — most recent successful application
- `strategies` (array[map]) — context-dependent approaches. Each entry: `{condition: string, workflow: string, description?: string}`, where `workflow` is the content address of a Workflow grain (§8.4).
- `transferable` (bool) — if `true`, another agent MAY ingest this grain and its referenced grains to acquire the skill
- All common fields from §6.1 (e.g., `license`, `structural_tags`, `context` for arbitrary metadata)

**Normative rules:**
1. A Skill grain MAY be a pure **definition** (no `holder_did`/`proficiency`) or a held **instance** that adds learned-competence fields. `name` + `description` are sufficient to define a skill.
2. When present, `proficiency` SHOULD equal the grain's top-level `confidence`.
3. Learned-competence improvement is recorded by supersession: each change in `proficiency` writes a new Skill grain that supersedes the prior one and increments `practice_count`. The supersession chain is the complete learning history; Skill grains are never mutated in place. Degradation (e.g., when `last_practiced_at` exceeds an implementation threshold) is likewise realized by a superseding grain with reduced proficiency, not a mutation.
4. Each `strategies[].workflow` MUST be the content address of a Workflow grain (§8.4). Each `dependencies` entry MUST be the content address of a Skill grain. Each `allowed_tools` entry SHOULD reference a Tool definition grain (§27.1).
5. When an agent acquires a transferable skill, it writes a new Skill grain that sets `derived_from` to the source grain's content address, preserves the originating agent in `origin_did`, sets its own `holder_did` and `author_did`, and (for learned skills) resets `proficiency` and `practice_count` to the acquiring agent's initial values (typically `0`).
6. Skill grains SHOULD use the `"agent:skills"` namespace to enable efficient discovery queries (see §28.8).

### 8.12 Recommendation (type = 0x0C)

A governed, auditable **proposal to change memory or agent configuration** — the unit a self-improvement or curation layer emits, reviews, applies, and can roll back. A Recommendation grain names a *target* (a grain, entity, saved query, template, document, or host object), the *analyzer* that produced it, a deterministic *summary*, and exactly one *proposal* describing the change (a batch of CAL evolve-writes, a document edit, or an opaque host-config change). It never mutates memory by itself: it enters a review lifecycle (§8.12.1) whose transitions are recorded as immutable audit Observation grains, and only an approved-and-applied recommendation changes the store.

This type is realized from the OMS `0x0C–0xEF` reserved range, following the precedent set by Skill (§8.11, from `0x0B`) and Consent (§8.10). It became a dedicated type because governed self-improvement — propose → review → apply → measure → roll back — needs a first-class, portable, content-addressed record of *what an autonomous layer wants to change and why*, with a tamper-evident audit trail; the `Fact + mg:proposes` pattern is expressible but impractical when the proposal queue, its dedup identity, and its audit chain are trust- and compliance-critical.

Where a Reasoning grain (§8.8) records *why the agent believes something* and a Goal (§8.7) records *what the agent intends*, a Recommendation records *a specific, reviewable change an analysis layer proposes to the memory or the agent itself* — bounded by evidence and gated by review.

**Required fields:**
- `type` = "recommendation"
- `target_ref` (string) — the change target as `<scheme>:<opaque>`. Standard schemes: `grain:sha256:<hash>` (a specific grain), `entity:<namespace>/<subject>` (an entity), `query:<namespace>/<name>` (a saved query), `template:<namespace>/<name>` (a template), `doc:<host-id>` (a host document, e.g. `doc:claude.md`), `host:<opaque>` (an opaque host object). The scheme is the target-kind discriminator; the scheme set is open for host-defined targets.
- `analyzer` (map) — the producing logic: `{id: string, params?: map}`, where `id` is a versioned identifier `publisher.name/major` (e.g. `"waiser.duplicate_sweep/1"`) and `params` is an optional snapshot of the parameters it ran with. Full "why" provenance without a second grain.
- `summary` (map) — a deterministic, template-rendered description: `{template_id: string, args: map}`. An analyzer MUST NOT store free prose here; the human-readable string is produced by rendering `template_id` with `args`, so summaries stay reproducible and translatable.
- `dedup_key` (string) — the recommendation's stable proposal identity, computed (never author-chosen) per normative rule 5. Two recommendations with the same `dedup_key` are the same proposal; a supersession chain (evidence refresh, a growing cluster) shares one `dedup_key` across all of its content addresses.
- exactly one **proposal** field:
  - `proposal_cal` (string) — a CAL batch of Tier-1 *evolve* writes (`ADD` / `SUPERSEDE` / `REVERT`; CAL §2.2), the standard change form, and — since CAL 1.3, where the host authorizes destructive proposals — Tier-2 destruction statements (`FORGET`; CAL §8.14). A Tier-1-only `proposal_cal` is always non-destructive and always invertible. A `proposal_cal` carrying Tier-2 statements executes only if the applying principal holds the required `delete`/`erase` verbs (§12.6; CAL §8.16.1 two-key property), and its apply MUST be recorded **not rollbackable** (§8.12.1). CAL's Tier-3 governance statements (`APPROVE`/`APPLY`/…) are refused inside `proposal_cal` unconditionally — a recommendation must not approve recommendations (CAL §8.14.3). Predicate destruction remains grammar-excluded at every tier (CAL §2.4). A destructive change beyond CAL's Tier-2 shapes MUST be proposed as `proposal_data` and executed outside CAL.
  - `proposal_edit` (map) — `{format: string, base_digest: string, diff: string}` for `doc:` targets; `base_digest` lets an applier detect a stale base before editing.
  - `proposal_data` (map) — an opaque host-config change for `host:` targets.
- `created_at` (int64, epoch ms)

**Optional fields:**
- `severity` (string) — `"info"`, `"low"`, `"medium"`, or `"high"`. Advisory triage signal.
- `metric_snapshot` (map) — the measurable claim the recommendation rests on, for later outcome review: `{metric: string, baseline: number, unit?: string, n?: int, window?: string, query?: string, review_after?: int64}`, where `query` is a CAL expression that reproduces the measurement and `review_after` is the epoch-ms checkpoint to re-measure at.
- `evidence_query` (string) — a CAL query that regenerates the full evidence set when `derived_from` is truncated to a representative subset (see below).
- All common fields from §6.1. In particular a Recommendation reuses:
  - `derived_from` — the **evidence**: content addresses of the grains the recommendation is grounded in (bounded to a representative set, RECOMMENDED ≤ 64; `evidence_query` regenerates the full set). Provenance traversal (§23.6) works unchanged.
  - `confidence` — the reviewer/verifier's calibrated credence that the recommendation is correct and useful.
  - `importance` — triage weighting.
  - `valid_to` — when the recommendation expires; expiry is computed from `valid_to`, never by a background daemon.

**Lifecycle status (index-layer):** A recommendation's review state (`rec_status` ∈ `pending | approved | rejected | applied | rolled_back | expired`) is **not stored in the .mg blob**. Like `superseded_by` and `verification_status` it is a rebuildable index-layer cache derived from the authoritative audit chain (§8.12.1), and it is enumerated as an index-layer field in §5.6, §6.1 and §28.3 — so the general prohibition on writers setting index-layer fields (§6.1) binds it, and a plain `write` / `SUPERSEDE … SET` cannot forge a status. A recommendation's content address is therefore **stable across its lifecycle transitions**: propose, approve, apply and roll back never re-address the grain, so a host can hold one identifier through review. Content changes are the exception and are handled by supersession — the identifier that is durable across a supersession chain is `dedup_key`, not the content address (§8.12.1).

#### 8.12.1 Lifecycle and audit

- **Authoritative record.** Each lifecycle transition is one immutable **audit Observation grain** (§8.6) hash-chained per recommendation: its `derived_from` includes the recommendation's content address and the previous audit grain's address (plus any result addresses). The acting principal is recorded in the audit grain's existing `observer_id` (§8.6) — e.g. `"user:alice"`, `"agent:worker-3"`, `"policy:auto"` — and its class in `observer_type`, which MUST be either `"human"` (registered, §24.2) or one of the namespaced values `"rec:agent"`, `"rec:policy"`, `"rec:system"` (§24.3). No field beyond the Observation schema is introduced. The transition's mandatory human-readable reason is carried in the audit grain's `object` (§6.1) — the common field CAL projects as an Observation's text content (CAL §10.3.2), so the reason renders correctly in SML with no additional rule (RECOMMENDED ≤ 500 chars). An audit grain's `observer_id` MUST NOT be elided under §10.2: the acting principal is what makes the chain tamper-evident, and a selectively-disclosed chain that has dropped it is no longer an audit trail. The chain is portable, tamper-evident, and fork-mergeable.
- **State machine.** `pending → approved | rejected`; `approved → applied`; `applied → rolled_back`; and `rejected → pending`, which lets a recommendation re-enter review on refreshed evidence under its existing `dedup_key` rather than becoming a second proposal. `pending → applied` directly is permitted **only** when the transition's `observer_type` is `"rec:policy"` (auto-apply); `observer_type` — never `observer_id` — is the discriminator for this gate, since `observer_id` is a host-asserted label with no normative structure. `applied` and `rolled_back` are terminal. `expired` is computed from `valid_to` and is reachable only from `pending` or `approved`; an expired recommendation MUST NOT be applied, and expiry of an `approved` recommendation withdraws that approval.
- **Content vs. lifecycle.** A change in the recommendation's *content* (refreshed evidence, a larger cluster) is a **supersession** of the grain — a new content address with the **same `dedup_key`** — not a status change. Lifecycle ≠ content. A supersession MUST reset the superseding grain's `rec_status` to `pending`: approval is granted to specific content and never carries forward to content no reviewer has seen.
- **Reproducible undo.** The inverse of an applied proposal is derived at apply time and recorded on the applied audit grain as a store-operation plan (no new CAL syntax): a `SUPERSEDE` reinstates the prior content; an `ADD` is undone by superseding the added grain with `invalidation_type: "retraction"`, which closes its `system_valid_to` and removes it from the default recall path while the blob itself survives (§5.6, §28.3) — OMS defines no separate tombstone mechanism, and none is needed; a `REVERT` is undone by superseding back to the reverted head; a document edit supersedes back to the prior head. Because CAL's Tier-1 statements are structurally non-destructive, every Tier-1-only `proposal_cal` is rollbackable. A `proposal_cal` carrying CAL Tier-2 destruction (tombstone or erasure — one-way by definition, CAL §8.14.1) MUST be recorded as **not rollbackable** on the applied audit grain. A `proposal_data` change is opaque to OMS and MAY be irreversible: a host that cannot derive an inverse for one MUST record it as not rollbackable on the applied audit grain rather than offer a rollback it cannot perform.

**Normative rules:**
1. Exactly one of `proposal_cal`, `proposal_edit`, `proposal_data` MUST be present.
2. Recommendation grains MUST NOT be mutated in place; every content change is a supersession, every lifecycle transition an audit Observation grain (§8.6). The `rec_status` index-layer cache MUST be reconstructible from the audit chain alone.
3. Lifecycle-derived state MUST NOT be writer-settable: `rec_status` is an index-layer field (§5.6, §6.1, §28.3), so the general rule that writers MUST NOT set index-layer fields applies to it, and a `SUPERSEDE … SET` targeting it MUST be rejected. Transitions occur only through the review/apply path that emits audit grains.
4. `summary.template_id` MUST reference a deterministic template and `summary.args` MUST carry the values it interpolates; implementations MUST NOT store analyzer-authored free prose in `summary`.
5. `dedup_key` MUST be computed (never author-chosen) as the lowercase hex SHA-256 of three NUL-separated, UTF-8 encoded components — **analyzer-family**, **`target_ref`**, **action-kind** — in exactly this order:

   ```
   dedup_key = hex(sha256(nfc(casefold(analyzer_family)) || 0x00 ||
                          nfc(target_ref)                || 0x00 ||
                          action_kind))
   ```

   Normalization is NFC applied *after* case-folding. `target_ref` is NFC-normalized but deliberately **not** case-folded, because a `grain:sha256:` digest and a host-defined opaque segment may both be case-sensitive. **`analyzer-family`** is `analyzer.id` with the `/major` suffix removed, so bumping `.../1` → `.../2` does not re-propose the queue as novel while the exact version stays in `analyzer.id`. **action-kind** is the proposal variant — the literal `cal`, `edit` or `data`. `dedup_key` MUST NOT incorporate the proposal body or the evidence, so that a content refresh supersedes rather than re-proposes. Because `dedup_key` is the shared identity across hosts, this construction is normative: two implementations MUST derive the same key for the same proposal or dedup fails on any imported, federated or forked store.
6. Each `derived_from` entry SHOULD be the content address of a grain the recommendation is grounded in; when present, `evidence_query` MUST regenerate the full evidence set.
7. Recommendation grains SHOULD use a dedicated namespace (e.g. `"agent:recommendations"`) to keep the proposal queue efficiently discoverable and separable from user memory.
8. **Execution authority.** A stored `proposal_cal` is CAL authored by one principal and executed later by another, so it does not carry a capability of its own — CAL binds every query to a `CapabilityToken` at execution time (CAL §2.3). An applier MUST execute the batch under the **approving principal's** capability, never the analyzer's and never an ambient store authority; for an auto-applied recommendation it is the host-configured policy principal for the recommendation's namespace, not a principal derived from `observer_id` — that field is a host-asserted label with no normative structure and MUST NOT be resolved to a capability. An applier MUST reject a proposal whose writes resolve outside the recommendation's own namespace: a recommendation MUST NOT propose writes into a namespace it does not itself inhabit.
9. **Bounds and target validation.** `proposal_cal` MUST NOT exceed CAL's `MAX_QUERY_LENGTH` (8192 bytes), and applying it is subject to the same Tier-1 write quotas as any other CAL batch (CAL §2.3). Before applying a `proposal_edit`, a host MUST resolve the `doc:` target against an allowlist and MUST verify `base_digest` against the current document head, rejecting the proposal if the base has drifted. `proposal_data` is opaque to OMS: a host MUST validate it against its own schema before applying, and SHOULD require it to be signed (§9) when the recommendation crosses a trust boundary.

### 8.13 Trigger (type = 0x0D)

A standing rule that starts a Workflow — an interval, a calendar schedule, an
external source to poll, a condition over this memory, or a gate over other
triggers.

**Required fields:**
- `type` = "trigger"
- `kind` (string) — see the kind registry below
- `workflow` (string) — content address of the Workflow grain (§8.4) this
  trigger starts
- `created_at` (int64, epoch ms)

**Optional fields:**

| Field | Type | Description |
|---|---|---|
| `connector` | string | Name of the external system, e.g. `"gmail"`, `"stripe"`. Part of the firing identity (see §27.8) |
| `scope` | string | What is watched, in the connector's own vocabulary, e.g. `"mailbox:accounts@example.com"` |
| `enabled` | bool | Whether the trigger is live. Default `true` |
| `dedup_key` | array[string] | JSON Pointers (RFC 6901) into an item, joined in order, identifying the *occurrence* |
| `interval_secs` | int | Seconds between firings, for `interval` and `polling` |
| `cron` | string | Calendar expression, for `schedule` |
| `at_ms` | int64 | Absolute instant, for `once` |
| `predicate` | map | A filter expression: selects grains for `memory`, composes members for `composite` |
| `members` | map[string→string] | Alias → trigger content address, for `composite` |
| `correlate` | string | JSON Pointer whose value correlates member firings |
| `window_ms` | int64 | How long a partial `composite` match stays live |
| `concurrency` | string | `"forbid"` (default) \| `"allow"` \| `"replace"` |
| `catchup` | string | `"last"` (default) \| `"none"` \| `"all"` |
| `config` | map | Connector transport configuration, using the `int:` keys of the Integration profile (§A.7) |
| All common fields | | |

**`kind` registry** (closed):

| Value | Fires when |
|---|---|
| `"interval"` | `interval_secs` have elapsed since the last firing |
| `"schedule"` | the `cron` expression matches |
| `"once"` | `at_ms` passes; never again |
| `"polling"` | a connector reports new items |
| `"memory"` | grains matching `predicate` appear in this memory |
| `"webhook"` | the host delivers a payload |
| `"manual"` | an operator fires it |
| `"composite"` | `predicate` over `members` is satisfied |

**Normative rules:**

1. **Binding direction.** A Trigger names its Workflow; a Workflow MUST NOT
   carry a list of triggers. A Workflow is content-addressed, so a plan
   accumulating trigger references would change address whenever one was added
   or removed, invalidating every reference to it — including a run's record of
   which plan it executed.
2. **Coherence.** A declaration that cannot fire MUST be rejected at write
   time, not stored: an `interval` or `polling` trigger without a positive
   `interval_secs`, a `schedule` without `cron`, a `once` without `at_ms`, a
   `memory` without `predicate`, a `composite` with fewer than two `members` or
   no `predicate`, or any trigger whose `workflow` is empty. A trigger that can
   never fire has no symptom other than silence, which is indistinguishable from
   a source with nothing new.
3. **Member aliases.** For `composite`, `members` maps an alias to a content
   address and `predicate` references the **aliases**. A content address is not
   a legal identifier in an expression grammar, and an alias survives a member
   being re-declared at a new address.
4. **`config` carries transport, not identity.** The `int:` keys describe how to
   reach a connector. A field that determines *whether or when* a trigger fires
   is a first-class field above, so that it is queryable: implementations index
   declared fields, not the interior of a `context`-shaped map.
5. **Evaluation state is not part of the grain.** Next-due time, cursor,
   lease and failure count are host-local operational state. See §27.8.

**Example — polling an external source:**

```json
{
  "type": "trigger",
  "kind": "polling",
  "workflow": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
  "connector": "gmail",
  "scope": "mailbox:accounts@example.com",
  "enabled": true,
  "interval_secs": 120,
  "dedup_key": ["/message_id"],
  "config": {
    "int:cursor_field": "since",
    "int:cursor_type": "timestamp",
    "int:allowed_outbound_hosts": ["https://gmail.googleapis.com"]
  },
  "namespace": "accounting",
  "created_at": 1740700000000
}
```

**Example — a gate over two other triggers:**

```json
{
  "type": "trigger",
  "kind": "composite",
  "workflow": "b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90a1",
  "members": {
    "invoice": "c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2",
    "purchase_order": "d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3"
  },
  "predicate": {
    "kind": "and",
    "left": { "kind": "comparison", "field": "invoice", "comparator": "eq", "value": true },
    "right": { "kind": "comparison", "field": "purchase_order", "comparator": "eq", "value": true }
  },
  "correlate": "/thread_id",
  "window_ms": 600000,
  "created_at": 1740700000000
}
```

---

## 9. Cryptographic Signing

### 9.1 COSE Sign1 Envelope

For A2A sharing and audit compliance, grains MAY be wrapped in [COSE Sign1](https://www.rfc-editor.org/rfc/rfc9052) (RFC 9052) envelopes.

```
Signed Grain Structure:

COSE_Sign1 {
  protected: {
    1: -8,                              // alg: EdDSA (see note below)
    4: "did:key:z6MkhaXg..."           // kid: signer DID
    3: "application/vnd.mg+msgpack"    // content_type
  },
  unprotected: {
    "iat": 1737000000                   // timestamp: epoch seconds
  },
  payload: <.mg blob bytes>,
  signature: <Ed25519 signature, 64 bytes>
}
```

**Key points:**

1. Signature wraps the complete .mg blob (version byte + optional header + payload)
2. Content address is still the inner blob's SHA-256 hash (unchanged by signing)
3. EdDSA (Ed25519) is default algorithm; ES256 (ECDSA P-256) is alternative
4. Signing is optional; `signed` flag in header indicates presence
5. Signer identity is the DID in `kid` (Key ID) field

> **Note on EdDSA algorithm value:** This specification uses COSE algorithm value `-8` (EdDSA). The IANA COSE Algorithms registry has introduced more specific values: `-19` for Ed25519 and `-53` for Ed448. Implementations MAY use `-19` instead of `-8` when Ed25519 is the only supported curve. Verifiers MUST accept both `-8` and `-19` for Ed25519 signatures.

### 9.2 Signed Flag and Wrapper Consistency

The `signed` flag (byte 1, bit 0) is part of the **inner blob's** fixed header. The COSE_Sign1 wrapper is external to the content-addressed blob and is NOT included in the SHA-256 hash:

```
[Inner .mg blob]                     [Outer COSE_Sign1 — not content-addressed]
├─ Byte 1, bit 0: signed = 1         ├─ protected headers
├─ payload bytes                     ├─ unprotected headers
└─ content address = SHA-256(blob)   └─ signature over inner blob bytes
```

**Invariant:** The `signed` flag MUST match the presence of an outer COSE wrapper:
- If `signed` = 1, the grain MUST be delivered wrapped in COSE_Sign1
- If `signed` = 0, the grain MUST NOT be wrapped

Parsers MUST reject with `ERR_SIGNED_MISMATCH` if the flag is 1 but no wrapper is present, or the flag is 0 but a wrapper is present.

**Content address stability:** Signing does not change the inner blob bytes or its content address. An unsigned and a signed delivery of the same grain share the same content address.

### 9.3 Identity Verification

To verify a signed grain:

1. Parse COSE_Sign1 structure
2. Extract `kid` (signer DID) from protected headers
3. Resolve DID to public key (did:key self-contained, did:web via HTTPS)
4. Verify signature over the payload
5. Deserialize payload to verify content address matches

---

## 10. Selective Disclosure

Grains MAY use field-level selective disclosure (inspired by [SD-JWT RFC 9901](https://www.rfc-editor.org/rfc/rfc9901)) to hide sensitive fields while proving they exist.

### 10.1 Elision Model

When sharing a grain with restricted visibility:

1. **Full grain** (held by creator):
```json
{
  "type": "fact",
  "subject": "Alice",
  "relation": "works_at",
  "object": "ACME Corp",
  "user_id": "alice-123",
  "namespace": "hr",
  "created_at": 1737000000000
}
```

2. **Disclosed grain** (shared with receiver):
```json
{
  "type": "fact",
  "subject": "Alice",
  "relation": "works_at",
  "object": "ACME Corp",
  "created_at": 1737000000000,
  "_elided": {
    "user_id": "sha256:a1b2c3d4...",
    "namespace": "sha256:e5f6a7b8...",
  },
  "_disclosure_of": "sha256:original_grain_hash..."
}
```

### 10.1.1 Elision Hash Computation

The value stored in `_elided` for each elided field is the SHA-256 hash of the canonical MessagePack encoding of that field's value:

```
elision_hash = "sha256:" + lowercase_hex(SHA-256(canonical_msgpack_encode(field_value)))
```

The hash covers the **value bytes only** — the field name (key) is not included. The field value is serialized using the same canonical MessagePack rules as the full grain (Section 4): NFC-normalized strings, sorted map keys, omitted nulls, float64, etc.

**Examples:**
- `user_id = "alice-123"`: encode `"alice-123"` as MessagePack fixstr → SHA-256 the resulting bytes
- `confidence = 0.95`: encode `0.95` as float64 (9 bytes) → SHA-256 the resulting bytes
- `context = {"k": "v"}`: encode as canonical sorted map → SHA-256 the resulting bytes

**Verification:** A receiver holding the disclosed grain can verify that a declared-absent field was faithfully elided by encoding the revealed value and comparing its SHA-256 against the entry in `_elided`.

### 10.2 Field Elision Rules

| Field | Elidable | Reason |
|-------|----------|--------|
| `type` | No | Receiver must know grain type |
| `subject` | Yes | May contain PII |
| `relation` | No | Core knowledge structure |
| `object` | Yes | May contain PII |
| `confidence` | No | Essential for trust decisions |
| `user_id` | Yes | GDPR personal data |
| `namespace` | Yes | May reveal organizational structure |
| `created_at` | No | Essential for temporal queries |
| `provenance_chain` | Yes | May reveal system architecture |
| `context` | Yes | May contain sensitive details |
| `structural_tags` | Yes | May reveal classification system |
| `goal_state` | No | Essential for routing and trust decisions |
| `source_type` | No | Required for human-vs-agent trust decisions |
| `priority` | No | Required for cross-system scheduling |
| `description` | Yes | May reveal strategic intent |
| `criteria` | Yes | May reveal operational thresholds |
| `criteria_structured` | Yes | May reveal operational thresholds |
| `parent_goals` | Yes | May reveal goal hierarchy (system architecture) |
| `state_reason` | Yes | May reveal internal reasoning |
| `satisfaction_evidence` | Yes | May reveal system internals |
| `delegate_to` | Yes | May reveal agent architecture |
| `delegate_from` | Yes | May reveal agent architecture |
| `rollback_on_failure` | Yes | May reveal system control flow |
| `observer_id` | Yes | May reveal physical sensor topology or agent infrastructure identity |
| `observer_type` | No | Core routing and trust-domain field; receiver must know observer category to calibrate confidence |
| `observer_model` | Yes | May reveal internal AI stack or model versioning |
| `observation_mode` | No | Required for trust calibration; changes the interpretation of `confidence` |
| `observation_scope` | No | Required for temporal interpretation of `valid_from`/`valid_to` |
| `compression_ratio` | No | Required for confidence calibration; cannot assess fidelity without knowing compression factor |
| `frame_id` | Yes | May reveal spatial coordinate topology or internal contextual system architecture |
| `sync_group` | Yes | May reveal multi-sensor or multi-agent coordination topology |

### 10.3 Elision in .mg Format

**Field compaction:**

| Full Name | Short Key | Type |
|-----------|-----------|------|
| `_elided` | `_e` | map {string: string} |
| `_disclosure_of` | `_do` | string |

Disclosed grain has different content address than original (bytes changed). If COSE-signed, signature covers original grain; receiver can verify all non-elided fields are authentic.

### 10.4 Canonical Form and Disclosure

The **original (undisclosed) grain is the canonical form**. Selective disclosure produces a derived view with a different content address; it does not create a new canonical grain.

- **Original grain:** content address is the hash of the complete, unelided blob — this is the authoritative identity
- **Disclosed grain:** content address is the hash of the elided blob — different from the original's address; `_disclosure_of` links back to the original's content address
- **COSE signatures** wrap and cover the **original** blob. Receivers verify the signature against the original's content address, not the disclosed variant's

In distributed systems:
- Primary storage holds the original grain (canonical, fully populated)
- Disclosed variants are presentation artifacts generated on demand; they SHOULD NOT be stored as independent grains
- When `_disclosure_of` resolves to an address in the store, the authoritative content is the original grain at that address

**Rationale:** Treating the original as canonical preserves the immutability guarantee (original is a fixed point) while allowing dynamic, per-recipient selective disclosure without re-signing or rehashing.

---

### 10.5 Pseudonymized Egress (new in 1.6)

Selective disclosure (Sections 10.1-10.4) hides fields from a *receiving
store*. Pseudonymized egress addresses the other disclosure channel: the
**external model**. When context leaves for an LLM, sensitive values are
replaced with category-typed placeholder tokens (`[PERSON_1]`,
`[EMAIL_2]`); the placeholder-to-value mapping stays with the holder, and
the model's response is rehydrated by exact token replacement before the
caller sees it. This is **pseudonymization, not anonymization** -- an
implementation MUST NOT describe it otherwise: with the mapping retained,
the egress stream remains personal data, and rich content stays linkable
under auxiliary-knowledge attacks regardless of identifier removal.

#### 10.5.1 The `anon:<namespace>` policy (reserved store-metadata prefix)

A per-namespace JSON policy declaring mode (`off | egress | ingress | both
| audit`), category-to-action mappings (`pseudonym | mask | redact |
generalize:<bucket> | allow`), detector demands, pseudonym scope, and
placeholder template. Conformance requirements:

- **Replication is write-if-absent.** Like retention declarations, a sync
  MUST NOT silently swap a live local policy, and an applied row MUST take
  effect on the live handle rather than at the next open.
- **An unreadable policy row fails reads closed.** A policy the
  implementation cannot parse MUST NOT silently mean "no policy": reads of
  covered namespaces refuse until the row is repaired, while policy writes
  keep working so it can be.
- **Declaring a policy stamps the file's minimum-reader version**, so an
  older implementation is warned loudly at open that the file declares
  protection it cannot honor.
- Value-derived (cross-handle-stable) pseudonyms MUST be keyed from the
  file's encryption key; an implementation MUST refuse such declarations
  on an unencrypted file rather than degrade to unkeyed derivation.

#### 10.5.2 The `vault:` prefix (reserved, never replicated)

`vault:<namespace>:<placeholder>` rows persist the sealed
placeholder-to-value mapping for flows that must rehydrate across
processes. Conformance requirements are prohibitions:

- The prefix is **reserved**; implementations MUST NOT repurpose it.
- Vault rows **MUST NOT ride bundles or segments in either direction** --
  export omits them and import refuses them. A re-identification table
  travelling to a replica undoes the pseudonymization it serves.
- Values MUST be sealed under a key derived from the file's encryption key
  with its own domain separation, so destroying the file key destroys the
  vault -- crypto-erasure reaches it by construction.
- **Erasure reaches the vault**: subject erasure (CAL `FORGET SUBJECT`)
  MUST also remove every vault row and live mapping entry whose plaintext
  names the erased identity, and MUST treat a failure to do so as an
  error, never best-effort.
- Reverse lookup (reveal / CAL `REHYDRATE`) is a privileged act: gated on
  a grant and recorded as a Tier-2 audit Observation naming value
  *fingerprints*, never identities.

#### 10.5.3 The `anonymized` response report

When an egress policy is active, query responses carry an `anonymized`
object beside the result: covered namespaces, whether a host-level floor
forced coverage, and per-namespace mapping **ids**. The mappings
themselves MUST NOT ride any response payload -- rehydration custody stays
with the holding process. Consumers MUST treat the field's absence as
"values are as stored", never as "values are safe".

## 11. File Format (.mg files)

### 11.1 Purpose

The .mg file is the portable unit of memory. Individual grains live in blob storage by content hash; .mg files are what users see, copy, share, and archive.

**Mental model:**
```
.sqlite = database file (many rows)
.git = repository (many objects)
.mg = memory file (many grains)
```

### 11.2 Layout

```
.mg File Structure:

+----------+------------------+
| Header   | Magic: "MG\x01"  |  3 bytes
|          | Flags: uint8     |  1 byte
|          | Grain count: u32 |  4 bytes
|          | Field map ver: u8|  1 byte
|          | Compression: u8  |  1 byte
|          | Reserved: 6 bytes|  6 bytes
+----------+------------------+  = 16 bytes
| Index    | Grain offsets    |  4 bytes × grain_count (u32 each)
|          | (enables random access)
+----------+------------------+
| Grains   | grain 0 bytes    |  variable
|          | grain 1 bytes    |  variable
|          | ...              |
|          | grain N-1 bytes  |  variable
+----------+------------------+
| Manifest | Index manifest   |  variable (canonical MessagePack/CBOR)
| (opt.)   | (if flags bit 4) |  see §11.7
+----------+------------------+
| Footer   | SHA-256 checksum |  32 bytes (over header + index + grains + manifest)
+----------+------------------+
```

### 11.3 Header Fields

**Magic:** `0x4D 0x47 0x01` — "MG" + version 1

**Flags (uint8):**

| Bit | Meaning |
|-----|---------|
| 0 | `sorted` — grains are sorted by created_at (ascending) |
| 1 | `deduplicated` — no duplicate content addresses |
| 2 | `compressed` — grain region is zstd-compressed (single block) |
| 3 | `field_map_included` — file includes custom FIELD_MAP for app-defined fields |
| 4 | `has_index_manifest` — file includes an index manifest section (§11.7) |
| 5-7 | Reserved |

**Compression codec (uint8):**

| Value | Codec |
|-------|-------|
| 0x00 | None (uncompressed) |
| 0x01 | zstd (default, level 3) |
| 0x02 | lz4 (low-latency) |
| 0x03-0xFF | Reserved |

### 11.4 Random Access via Offsets

The offset index (4 bytes × grain count) enables fast random access:

```python
# Read grain #42 from a .mg file
header_size = 16
offset_start = header_size + (42 * 4)
offset = int.from_bytes(data[offset_start:offset_start+4], 'big')
next_offset = int.from_bytes(data[offset_start+4:offset_start+8], 'big')
grain_bytes = data[offset:next_offset]
```

For compressed files (flags bit 2 = 1), offsets point into the decompressed grain region. The entire grain region MUST be fully decompressed before any grain can be accessed by offset; implementations MUST NOT attempt to index into the compressed byte stream directly. This is a deliberate trade-off: compression reduces file size at the cost of requiring full decompression before random access.

### 11.5 Footer Checksum

SHA-256 over: `header (16 bytes) || index (grain_count*4 bytes) || grains (variable) || manifest (variable, if present)`

Enables integrity verification of entire file.

### 11.6 Wire Framing (Transport Layer)

For streaming scenarios (WebSocket, SSE, Kafka, TCP), use length-prefixed framing (NOT saved to disk):

```
+------+------------------+
| u32  | grain 0 bytes    |  length-prefixed frame
+------+------------------+
| u32  | grain 1 bytes    |  length-prefixed frame
+------+------------------+
| 0x00000000             |  zero-length sentinel = end of stream
+------+------------------+
```

### 11.7 Index Manifest (Portable Index-Layer State)

When flag bit 4 (`has_index_manifest`) is set, the .mg file includes an **index manifest** section between the grain region and the footer. The manifest carries index-layer field values (§5.6, §28.3) so that a single .mg file is a self-contained, portable unit of memory — including lifecycle state, not just immutable content.

**Format:** The manifest is a canonical MessagePack (or CBOR, matching the grains' encoding) map keyed by content address:

```
{
  "<content_address>": {
    "sb":      "<superseding content address>",
    "svt":     1737000000000,
    "vstatus": "verified"
  },
  "<content_address>": {
    "vstatus": "contested",
    "ac":      42,
    "laa":     1737500000000
  }
}
```

Field names use the compacted short keys from §6.1. Null/absent values are omitted per §4. Only grains with at least one non-default index-layer field need an entry.

**Field portability classes:**

| Class | Fields | Export | Import |
|-------|--------|--------|--------|
| **Portable** | `superseded_by`, `system_valid_to`, `verification_status` | MUST include | MUST merge into index |
| **Rebuildable** | `rec_status` | MAY include | MUST rebuild from the audit chain |
| **Local** | `access_count`, `last_accessed_at` | MAY include | MAY merge or reset to zero |

Portable fields carry semantic state (supersession chains, verification decisions) that is meaningful across systems. Local fields carry store-specific access statistics that may not be meaningful in a different deployment. `rec_status` is neither: unlike `verification_status` it is fully derivable from data that travels in the same file — the audit Observation grains of §8.12.1 — so an importer MUST reconstruct it from that chain and MUST NOT trust an imported value, which is what keeps a forged manifest from conferring an approval.

**Export rules:**
- Exporters MUST set flag bit 4 and include a manifest when any grain in the file has non-default portable index-layer fields.
- Exporters SHOULD include local fields as a convenience; omitting them is not an error.

**Import rules:**
- Importers MUST parse the manifest when flag bit 4 is set.
- Importers MUST apply portable fields to their index layer. If a grain already exists in the target store with conflicting index state, the conflict resolution strategy is implementation-defined (last-writer-wins, manual review, etc.).
- Importers MUST rebuild Rebuildable fields from the audit chain (§8.12.1) and MUST NOT apply an imported value, even when the manifest carries one.
- Importers MAY ignore local fields or reset them to defaults (e.g., `access_count: 0`).
- Importers MUST NOT inject manifest fields into the immutable blob. The manifest is index-layer metadata only.

**Integrity:** The manifest bytes are included in the footer checksum (§11.5) but are NOT part of any grain's content address. Tampering with the manifest is detectable via the footer checksum, but the immutable grain blobs remain independently verifiable by their own content addresses.

> **Implementation note:** For .mg files without flag bit 4, importers SHOULD initialize all index-layer fields to defaults (`verification_status: "unverified"`, `access_count: 0`, etc.). The absence of a manifest means either the exporter predates this feature or all grains had default index-layer state.

---

## 12. Identity and Authorization

### 12.1 DID-Based Identity (author_did)

Replaces the earlier `agent_id` string (free-form, unverifiable):

- **`author_did`** (compacted: `adid`) — DID of grain creator (cryptographically verifiable)
- **`origin_did`** (compacted: `odid`) — original source DID in A2A relay chains

### 12.2 Why W3C DIDs

[W3C DIDs](https://www.w3.org/TR/did-core/) provide decentralized identity without central PKI:

1. **did:key** (default) — Self-contained; public key in the DID itself
   ```
   did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK
   ```

2. **did:web** (enterprise) — Organizational identity via DNS
   ```
   did:web:example.com:agents:summarizer
   ```

### 12.3 Identity Fields (Orthogonal)

| Field | Purpose | Example | Used By |
|-------|---------|---------|---------|
| `author_did` | Agent identity — who created this grain | `did:key:z6Mk...` | COSE signature verification, audit trail |
| `user_id` | Data subject — whose personal data | `"alice-42"`, `"patient-789"` | GDPR erasure, per-user encryption |
| `namespace` | Logical partition — grouping | `"work"`, `"robotics:arm-7"` | Query scoping, access control |

### 12.4 User ID Compliance Context

`user_id` is specifically for natural persons under GDPR, CCPA, HIPAA:
- Triggers per-person encryption (HKDF key derivation)
- Enables erasure proofs (crypto-erasure by destroying key)
- Tracks per-person consent
- Enables blind index lookups (HMAC tokens) without exposing plaintext

For non-person memory (seasonal, device, system), `user_id` is simply omitted. `namespace` handles logical grouping.

### 12.5 Agent Ownership and Legal Entity

An agent may belong to a legal entity — a natural person or a juridical person (company, partnership, NGO, government body). OMS expresses this relationship as a protected Fact grain written at agent provisioning time by the operator, not by the agent itself.

#### 12.5.1 The `owner` Field

Any grain type MAY carry an `owner` field (compacted: `own`) containing a **LegalEntity** map. In practice, `owner` is used in the ownership Fact grain described in §12.5.3. It MUST NOT be used as an access control gate — `invalidation_policy` (§23) governs supersession authorization.

**LegalEntity sub-schema:**

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | REQUIRED | `"human"` (natural person) or `"org"` (juridical entity) |
| `name` | string | REQUIRED | Registered legal name |
| `entity_form` | string | OPTIONAL | Legal structure (open enum; see §12.5.2). Omit when `type: "human"`. |
| `jurisdiction` | string | OPTIONAL | ISO 3166-2 code of registration jurisdiction (e.g., `"US-DE"`, `"IN-KA"`, `"GB"`, `"SG"`) |
| `reg_id` | string | OPTIONAL | Government registration ID, prefixed by type (e.g., `"EIN:88-..."`, `"CIN:U..."`, `"ABN:51..."`) |
| `did` | string | OPTIONAL | W3C DID for cryptographic verifiability. RECOMMENDED when available. |

#### 12.5.2 `entity_form` Registry (Open Enum)

| Value | Legal structure |
|---|---|
| `"c_corp"` | C-Corporation (US) |
| `"s_corp"` | S-Corporation (US) |
| `"pbc"` | Public Benefit Corporation (US) |
| `"llc"` | Limited Liability Company (US) |
| `"llp"` | Limited Liability Partnership (US / India / UK) |
| `"pvt_ltd"` | Private Limited Company (India: Pvt. Ltd.; UK: Ltd.) |
| `"plc"` | Public Limited Company (UK) |
| `"gmbh"` | Gesellschaft mit beschränkter Haftung (DE / AT / CH) |
| `"sarl"` | Société à responsabilité limitée (FR and Francophone jurisdictions) |
| `"bv"` | Besloten vennootschap (NL / BE) |
| `"pty_ltd"` | Proprietary Limited (AU / ZA) |
| `"sole_proprietor"` | Sole proprietorship (any jurisdiction) |
| `"partnership"` | General partnership |
| `"ngo"` | Non-governmental organization / 501(c)(3) |
| `"government"` | Government body or public agency |
| `"trust"` | Trust entity |
| `"cooperative"` | Cooperative |

This is an open enum. Implementations MAY define additional values for jurisdiction-specific structures not listed above.

**`reg_id` prefix conventions:**

| Prefix | Country | ID type |
|---|---|---|
| `EIN:` | US | Employer Identification Number |
| `CIN:` | India | Company Identification Number (MCA) |
| `GSTIN:` | India | GST Identification Number |
| `ABN:` | Australia | Australian Business Number |
| `VAT:` | EU / UK | VAT registration number |
| `UEN:` | Singapore | Unique Entity Number |
| `SIREN:` | France | Système d'Identification du Répertoire des Entreprises |

Prefixes not listed here MUST be preserved as-is. New prefixes do not require a spec update.

#### 12.5.3 Ownership Fact Grain Convention

Agent ownership is expressed as a Fact grain with `relation: "mg:owned_by"` in the `"agent:identity"` namespace. The `object` field carries the owner's legal name as a string (for semantic triple completeness). The structured `owner` field carries the full LegalEntity map.

This grain MUST be written by the operator at agent provisioning time. It MUST carry an `invalidation_policy` (§23) restricting supersession to the owner's authorized DID. It SHOULD be COSE-signed (§9) by the owner's DID.

**Example — organization owner (Indian Pvt. Ltd.):**

```json
{
  "type": "fact",
  "subject": "did:web:example.com:agents:my-agent",
  "relation": "mg:owned_by",
  "object": "Example Corp Pvt. Ltd.",
  "owner": {
    "type": "org",
    "name": "Example Corp Pvt. Ltd.",
    "entity_form": "pvt_ltd",
    "jurisdiction": "IN-KA",
    "reg_id": "CIN:U72900KA2023PTC123456",
    "did": "did:web:example.com"
  },
  "source_type": "system",
  "author_did": "did:web:example.com",
  "namespace": "agent:identity",
  "structural_tags": ["legal:ownership", "mg:protected"],
  "invalidation_policy": {
    "mode": "locked",
    "authorized": ["did:web:example.com"],
    "scope": "lineage",
    "protection_reason": "Immutable ownership declaration — change requires authorized officer signature"
  },
  "created_at": 1737000000000
}
```

**Example — individual human owner:**

```json
{
  "type": "fact",
  "subject": "did:key:z6MkAgentDID...",
  "relation": "mg:owned_by",
  "object": "Jane Doe",
  "owner": {
    "type": "human",
    "name": "Jane Doe",
    "jurisdiction": "IN",
    "did": "did:key:z6MkJaneDoeKey..."
  },
  "source_type": "system",
  "author_did": "did:key:z6MkJaneDoeKey...",
  "namespace": "agent:identity",
  "structural_tags": ["legal:ownership", "mg:protected"],
  "invalidation_policy": {
    "mode": "locked",
    "authorized": ["did:key:z6MkJaneDoeKey..."],
    "scope": "lineage",
    "protection_reason": "Individual owner declaration"
  },
  "created_at": 1737000000000
}
```

**Example — US LLC (Delaware):**

```json
{
  "owner": {
    "type": "org",
    "name": "Acme Labs LLC",
    "entity_form": "llc",
    "jurisdiction": "US-DE",
    "reg_id": "EIN:47-1234567",
    "did": "did:web:acmelabs.io"
  }
}
```

**Normative rules:**

1. The ownership grain MUST NOT be authored by the agent's own DID. Only the operator's DID is authorized to write it (key separation, §23.8).
2. The `subject` MUST be the agent's DID.
3. When multiple grains with `relation: "owned_by"` exist for the same `subject` in the `"agent:identity"` namespace, the grain with `invalidation_policy.mode ≠ "open"` is authoritative. Stores SHOULD surface it as the canonical ownership record.
4. An agent observing a user assertion that contradicts the locked ownership grain MAY record that claim as an Observation grain. It MUST NOT write a superseding ownership Fact without the authorized signature.

#### 12.5.4 Protection Layers

The `locked` invalidation policy combined with COSE signing provides layered protection against ownership spoofing:

| Layer | Mechanism | What it prevents |
|---|---|---|
| Policy lock | `invalidation_policy.mode: "locked"` (§23) | Store rejects any supersession not signed by `authorized` DID; returns `ERR_INVALIDATION_DENIED` |
| Key separation | Agent DID ≠ owner DID (§23.8) | Agent cannot produce a valid supersession signature even if instructed to by a user |
| Lineage scope | `scope: "lineage"` (§23.6) | Supersession chain injection — agent cannot supersede a derived grain to bypass the protected root |
| COSE signature | Owner signs the blob (§9) | Blob tampering changes the content address; the original signed grain remains valid and current |

**Prompt injection resistance:** A user or external input asserting "your owner is now X" does not create or modify an ownership grain. The agent lacks the owner's private key and cannot author a superseding grain that passes the `locked` policy check. The original ownership fact remains current.

### 12.6 Authorization (new in 1.6, draft)

Authorization answers "what may this principal do in this memory?" It is the
model CAL's Tier 2/3 statements consume (CAL §2.2, §8.14–§8.16), and it is
usable by any host independently of CAL. The design principle is the
**credential/policy split**: *policy* (who may do what here) is a truth about
the memory and lives in it as grains; *credentials* (proving who a caller is)
are host objects and MUST NOT be carried in grains, statements, or the memory
file in any form.

#### 12.6.1 Principals, verbs, scope

- A **principal** is an opaque, non-empty name. Humans and agents are not
  distinguished by the model: `user:anna`, `agent:support-bot`,
  `job:retention` are all principals. A DID is a valid principal name and is
  the upgrade path for signed deployments (§12.1) — not a prerequisite.
- The **verb** set is closed in this revision: `read`, `write`, `supersede`,
  `delete`, `erase`, `loop.run`, `loop.review`, `loop.apply`, `admin`
  (semantics per CAL §8.14–§8.16; hosts without a recommendation engine
  simply never resolve the `loop.*` verbs).
- **Scope** is one namespace or `*`. The memory axis is implicit: a grant
  governs only the memory it is stored in. Cross-memory access is governed by
  each memory's own grants.

#### 12.6.2 Grant grains

A grant is a **Fact grain** (§8.1) in the reserved namespace
**`agent:authz`** (following the `agent:identity` / `agent:recommendations`
precedent, §12.5.3, §8.12 rule 7):

| Field | Value |
|---|---|
| `type` | `"fact"` |
| `namespace` | `"agent:authz"` |
| `subject` | the **grantee** principal name |
| `relation` | `"mg:permits"` |
| `object` | the canonical grant string: `"<verbs> ON <scope>"`, where `<verbs>` is the granted verbs lowercase, comma-separated without spaces, sorted lexicographically, and `<scope>` is the governed namespace(s) comma-separated without spaces, sorted, or the literal `*` — e.g. `"delete,read ON caller,shared"`, `"admin ON *"` |
| `confidence` | `1.0` |
| `context` | grantor and reason under reserved keys: `{"grantor": "<principal>", "because": "<reason>"}` (`because` present when given) |
| `author_did` | RECOMMENDED where DIDs are deployed — binds the grant cryptographically when the grain is COSE-signed (§9) |

The canonical `object` form is normative: two grants conferring the same
rights MUST serialize to the same string, so grants deduplicate
content-addressably and compare across implementations.

**Effective rights** of a principal for a namespace = the union of the verbs
in **live** grant grains naming it — grains in `agent:authz` with
`subject = <principal>`, `relation = "mg:permits"`, not superseded
(`superseded_by` unset, `system_valid_to` unset) — whose scope includes the
namespace (or is `*`). Resolution MUST fail closed: an unknown principal or
an empty result confers nothing.

Because grants are ordinary grains, the PERMISSION relation category query
(CAL §7) already retrieves them; CAL's `SHOW GRANTS` is sugar over it.

#### 12.6.3 Revocation

Revocation is **retraction-by-supersession** — the native invalidation
mechanism (§5.6, §28.3), so grant history is append-only and auditable:

- A **full revoke** supersedes the covering grant grain(s) with
  `invalidation_type: "retraction"`, closing their `system_valid_to`.
- A **partial revoke** supersedes the covering grant with a new grant grain
  whose canonical `object` carries the reduced verb/scope set.
- The revoking principal and reason ride the superseding grain's `context`
  (`{"grantor": ..., "because": ...}`) and the standard invalidation fields.

`mg:revokes` remains the **Consent-domain** relation for data-subject consent
withdrawal (§8.10) — authorization revocation deliberately does not use it,
so PERMISSION-category queries over live grains return exactly the effective
grant set with no subtraction logic.

**Authoring authority:** writing or superseding any grain in `agent:authz`
requires the `admin` verb (or an owner session, §12.6.4). Every `erase`
grant is explicit: if a future revision defines role bundles, no bundle may
include `erase`.

#### 12.6.4 Owner sessions and bootstrap

A host MAY designate a local session as the memory's **owner** — the
implicit superuser, exempt from grant lookup (the `root@localhost` /
SQLite-file-owner precedent). The owner writes the first grant. A memory
containing no grant grains confers nothing on any non-owner principal —
fail-closed by construction — while remaining fully usable by its owner, so
a single-operator memory never needs to know this section exists.

**Tamper honesty:** whoever holds the file bytes is root over them — as with
any database's data directory. Content addressing and the operation log make
out-of-band edits *detectable*; COSE signing (§9) is the high-assurance
upgrade path.

#### 12.6.5 Replication

Grant grains replicate as **sync**: an importer, follower, or hub applies
them like any other grain, without re-authorization — the file carries its
own access policy wherever it goes. *Authoring* a grant (creating or
superseding in `agent:authz`) requires `admin`/owner authority at the
writing host. Disclosure property: a synced memory reveals its principal
names to every replica; deployments that consider principal names sensitive
SHOULD use opaque identifiers.

#### 12.6.6 Relationship to other mechanisms

- `invalidation_policy` (§23) continues to govern *supersession of specific
  grains*; §12.6 governs *operations by principals*. They compose: a
  supersession must pass both.
- Consent grains (§8.10) record *data-subject* consent and are unrelated to
  operational authorization; do not encode grants as Consent.
- The delegation fields (§6.11, `authorized_namespaces`) describe
  *agent-to-agent* delegated authority in Goal grains; a host MAY map them
  onto verbs, but grant grains are the authorization source of truth.

---

## 13. Sensitivity Classification

### 13.1 Header-Level Sensitivity

The fixed header includes a 2-bit sensitivity field (byte 1, bits 6-7):

| Value | Level | Meaning |
|-------|-------|---------|
| 00 | Public | No sensitivity constraints |
| 01 | Internal | Organization-internal data, not PII |
| 10 | PII | Contains personally identifiable information |
| 11 | PHI | Contains protected health information (HIPAA) |

Enables O(1) routing to encrypted storage or access control — no deserialization needed.

### 13.2 Standard Tag Vocabulary

Detailed sensitivity classification via `structural_tags` in payload:

| Prefix | Category | Examples |
|--------|----------|----------|
| `pii:` | Personal data | `pii:email`, `pii:phone`, `pii:ssn`, `pii:name` |
| `phi:` | Health data | `phi:diagnosis`, `phi:medication`, `phi:lab_result` |
| `reg:` | Regulatory jurisdiction | `reg:pci-dss`, `reg:sox`, `reg:basel-iii`, `reg:gdpr-art17` |
| `sec:` | Security data | `sec:credential`, `sec:api_key`, `sec:token` |
| `legal:` | Legal data | `legal:ownership`, `legal:privilege`, `legal:litigation_hold` |

The `reg:` prefix identifies which regulatory storage or retention rules apply to a grain. The vocabulary is **open-ended** — use well-known regulation identifiers. Examples: `reg:pci-dss` (PCI-compliant storage required), `reg:sox` (7-year immutable audit retention), `reg:basel-iii` (regulatory capital data), `reg:gdpr-art17` (erasure-eligible). Unlike `pii:` or `phi:`, `reg:` tags carry no compliance classification claim — they are routing and policy directives.

At write time, serializer scans tags and sets header sensitivity bits to highest classification present.

### 13.3 Header Sensitivity Limitations

Header sensitivity bits (§13.1) are **advisory routing metadata**, not a compliance guarantee. They enable efficient routing without deserialization but MUST NOT be treated as the sole basis for access control or encryption decisions.

Tag-based sensitivity assignment (§13.2) depends on the writer correctly identifying and tagging sensitive fields at creation time. If a grain contains sensitive data but is incorrectly or incompletely tagged, the header bits will not reflect the true classification.

Systems processing personal data, health information, or other regulated content SHOULD:
1. Treat header sensitivity bits as a fast-path routing hint, not a classification guarantee
2. Perform payload inspection for sensitive decisions — deserialize and validate `structural_tags` before routing or sharing
3. Enforce writer responsibility — establish clear tagging protocols for regulated workflows
4. Apply layered defense — combine header-level filtering with payload inspection; never gate compliance solely on header bits

### 13.4 Sensitivity Consistency Validation

**Serializer rule:** At write time, the serializer MUST scan all `structural_tags` values and set the header sensitivity bits to the highest classification present, using this mapping:

| Tag prefix present | Minimum header sensitivity |
|--------------------|---------------------------|
| `phi:*` | 11 (PHI) |
| `pii:*`, `sec:*`, `legal:*` | 10 (PII) |
| `reg:*` | 01 (internal) minimum — policy engine determines actual tier |
| No sensitive tags | 00 or 01 at writer's discretion |

**Parser rule:** At parse time, if `structural_tags` is present, the parser MUST validate that the header sensitivity bits are not lower than the highest classification the tags require. If they are lower, the parser MUST reject with `ERR_SENSITIVITY_MISMATCH`. This condition indicates either a serializer defect or potential header tampering to bypass access controls.

### 13.5 Legal Neutrality Statement

The sensitivity classifications in this specification (`public`, `internal`, `PII`, `PHI`) are technical routing and storage metadata. They are not legal definitions of personal data, health information, financial information, or any regulated category under any jurisdiction.

Different legal regimes use different terminology and thresholds:
- **GDPR (EU)** — "personal data": any information relating to an identified or identifiable natural person
- **CCPA (California)** — "personal information": information that identifies or could reasonably be linked to a consumer
- **LGPD (Brazil)** — "dados pessoais": similar scope to GDPR
- **HIPAA (USA)** — "protected health information (PHI)": a specific regulatory category under 45 CFR

Implementations MUST determine sensitivity classification according to applicable jurisdictional law and organizational policy. The `.mg` tags and header bits are provided as a compliance-aware tagging mechanism to facilitate routing and policy enforcement; the legal determination of what constitutes regulated data is outside the scope of this specification.

---

## 14. Cross-Links and Provenance

### 14.1 Provenance Chain

Every grain carries `provenance_chain` — the derivation trail:

```json
{
  "provenance_chain": [
    {"source_hash": "abc123...", "method": "user_input", "weight": 1.0},
    {"source_hash": "def456...", "method": "frequency_consolidation", "weight": 0.8}
  ]
}
```

Each entry has:
- `source_hash` — content address of source grain
- `method` — consolidation method or source type
- `weight` — how much this source contributed (0.0–1.0)

**Provenance chain method strings for Observation grains:**

| Method String | Meaning |
|---|---|
| `"sensor_read"` | Direct physical measurement from an instrument |
| `"llm_observation"` | LLM-generated observation from input messages or documents |
| `"reflective_compression"` | Observation produced by compressing prior Observation or Episode grains |
| `"multi_sensor_fusion"` | Observation produced by fusing multiple physical sensor readings sharing a `sync_group` |
| `"human_annotation"` | Observation recorded by a human observer or annotator |
| `"detection_inference"` | Observation produced by a classification or detection model |

### 14.2 Related-To Cross-Links

The `related_to` field enables semantic similarity links:

```json
{
  "related_to": [
    {
      "hash": "abc123...",
      "relation_type": "similar",
      "weight": 0.85
    },
    {
      "hash": "def456...",
      "relation_type": "elaborates",
      "weight": 0.70
    }
  ]
}
```

**Field compaction (RELATED_TO_FIELD_MAP):**

| Full Name | Short Key | Type |
|-----------|-----------|------|
| `hash` | `h` | string |
| `relation_type` | `rl` | string |
| `weight` | `w` | float64 |

### 14.3 Relation Type Registry (Closed Vocabulary)

The relation type vocabulary is intentionally closed (not extensible) to prevent PII leakage through relation names:

| Type | Meaning | Direction |
|------|---------|-----------|
| `similar` | Semantically similar content | Symmetric |
| `contradicts` | Incompatible claims | Symmetric |
| `elaborates` | Adds detail/specificity | Asymmetric |
| `generalizes` | More abstract version | Asymmetric |
| `temporal_next` | Event occurs after | Asymmetric |
| `temporal_prev` | Event occurs before | Asymmetric |
| `causal` | Causes or preconditions | Asymmetric |
| `supports` | Provides corroborating evidence | Asymmetric |
| `refutes` | Provides contradicting evidence (weaker than contradicts) | Asymmetric |
| `replaces` | Supersedes (outdated but not wrong) — **advisory only** | Asymmetric |
| `depends_on` | Validity depends on referenced grain | Asymmetric |

> **Normative note on `replaces`:** The `replaces` relation type is a semantic annotation only. It does NOT constitute formal supersession and MUST NOT cause a conformant store to update the target grain's index entry (`superseded_by`, `contradicted`, `system_valid_to`). Conformant clients MUST determine a grain's current status solely from the index `superseded_by` and `contradicted` fields, never from `related_to` links. This rule closes a bypass path for `invalidation_policy` (see §23.7).

---

## 15. Temporal Modeling

### 15.1 Five Timestamps Per Grain

| Field | Meaning | Real-World Reference | System Reference |
|-------|---------|----------------------|------------------|
| `valid_from` | When fact became true | Event start time | — |
| `valid_to` | When fact stopped being true | Event end time | — |
| `created_at` | When grain was added to system | Ingestion timestamp | System write time |
| `system_valid_from` | When grain became active in system | — | System validity start (blob field) |
| `system_valid_to` | When grain was superseded/retracted | — | System validity end (index layer) |

### 15.2 Bi-Temporal Queries

With these five fields, systems support:

| Query | Fields Used |
|-------|------------|
| "What does agent know now?" | `system_valid_to` is null/absent |
| "What was true on date X?" | `valid_from` ≤ X ≤ `valid_to` |
| "What did agent know at time T?" | `system_valid_from` ≤ T AND (`system_valid_to` is null OR `system_valid_to` > T) |
| "Reconstruct state at audit time T" | Combine event-time and system-time |

### 15.3 Implementation Note

`system_valid_to` is typically an **index-layer field**, not stored in immutable .mg blobs. The index adds this field when supersession occurs. The .mg blob itself carries `system_valid_from` at creation; the index tracks the end time.

---

## 16. Encoding Options

### 16.1 MessagePack (Default)

[MessagePack](https://msgpack.org/) is the default encoding. Well-supported across 50+ languages, compact, and human-debuggable with tools.

Canonical MessagePack rules (Section 4) ensure deterministic encoding.

### 16.2 CBOR (Optional)

[CBOR (RFC 8949)](https://www.rfc-editor.org/rfc/rfc8949) is an optional encoding, specified via flags bit 5. Uses Deterministic CBOR ([RFC 8949 §4.2.1](https://www.rfc-editor.org/rfc/rfc8949#section-4.2.1)) rules:

1. Map keys sorted by encoded form (lexicographic on CBOR bytes)
2. Integers in smallest encoding
3. No indefinite-length values
4. Single NaN representation
5. Shortest floating-point form that preserves value (e.g., `1.5` → binary16 `0xf93e00`; does NOT convert floats to integers)
6. Strings are UTF-8 NFC-normalized
7. No duplicate keys

**Critical:** Same grain encoded as MessagePack and CBOR have DIFFERENT content addresses (different bytes). Logical equivalence ≠ physical equivalence.

### 16.3 When to Use

- **MessagePack** (default): Universal, mature, fast
- **CBOR**: IETF standards track, COSE signatures, constrained devices

---

## 17. Conformance Levels

Implementations MUST declare which level they support:

### 17.1 Level 1: Minimal Reader

- Deserialize version byte + canonical MessagePack payload
- Compute and verify SHA-256 content addresses
- Support field compaction (short keys → full names)
- Support all thirteen standard grain types (0x01–0x0D) per §8 schemas
- Ignore unknown fields
- Constant-time hash comparison

Level 1 is sufficient for reading, verifying, and storing grains.

### 17.2 Level 2: Full Implementation

All Level 1 requirements, plus:
- Serialize (full names → short keys)
- Enforce canonical MessagePack rules
- Validate required fields per schema
- Pass all test vectors
- Support multi-modal content references
- Implement Store protocol (get/put/delete/list/exists)
- Enforce `invalidation_policy` on all supersession and contradiction operations
- Implement `supersede` as a distinct, atomic store operation (not a raw `put` + index patch); `put` MUST reject grains containing `derived_from` claims that imply supersession without going through `supersede`
- Apply fail-closed rule: unknown `invalidation_policy.mode` values MUST be treated as `mode: "locked"`
- Enforce the `replaces` non-supersession rule: `relation_type: "replaces"` MUST NOT trigger index mutations on the target grain
- MUST validate that `observer_type` is a non-empty string; MUST NOT reject unknown `observer_type` values (open enum)
- MUST emit `oid` and `otype` short keys
- SHOULD warn (but MUST NOT reject) when `observer_model` is absent on Observation grains where `observer_type` is `"llm"`, `"reflector"`, `"classifier"`, or `"detector"`

### 17.3 Level 3: Production Store

All Level 2 requirements, plus:
- At least one persistent backend (filesystem, S3, database)
- AES-256-GCM encrypted grain envelopes
- Per-user key derivation (HKDF-SHA256)
- Blind-index tokens for encrypted search
- SPO/SOP/PSO/POS/OPS/OSP index (hexastore) or equivalent
- Full-text search (FTS5 or equivalent)
- Hash-chained audit trail
- Crash recovery and reconciliation
- Policy engine with compliance presets
- SHOULD partition Observation grain storage by observer domain, inferred from `observer_type`. Physical observer types (see Section 24) SHOULD flow to time-series storage with raw-data retention policies. Cognitive observer types SHOULD flow to vector + relational storage with the same retrieval semantics as Fact grains. Implementations MUST NOT hard-code the domain partition list — treat `observer_type` as an open string and drive routing from configuration or namespace.

### 17.4 Optional modules

Orthogonal to Levels 1–3, some capabilities are optional **modules**: not every
store performs them, and one that does not is not a lesser implementation.
Implementing a module means implementing it whole. This mirrors CAL's tier
modules (CAL §24) — **modules gate operations, not portability.** Interoperability
rides the `.mg` file: any Level 1 reader can read a memory containing a module's
grains.

| Module | Requires |
|---|---|
| `triggers` | The full execution contract of §27.8, including its idempotency, non-replication and arbitration rules |

An implementation declares its modules alongside its level:

```json
{ "oms_conformance": 3, "oms_modules": ["triggers"] }
```

An implementation that stores and replicates Trigger grains without evaluating
them MUST NOT declare the module. Storing them is ordinary grain handling and
needs no declaration; *acting* on them is what the contract governs.

---

## 18. Device Profiles

### 18.1 Extended Profile (Default)

**Target:** Servers, desktops, edge gateways

- Max blob size: 1 MB
- Hash function: SHA-256 (REQUIRED)
- All fields supported
- Encryption: AES-256-GCM
- Full feature set

### 18.2 Standard Profile

**Target:** Single-board computers, mobile, IoT

- Max blob size: 32 KB
- Hash function: SHA-256
- All fields supported
- Encryption: AES-256-GCM
- Vector search: optional

### 18.3 Lightweight Profile

**Target:** Microcontrollers, battery-powered sensors

- Max blob size: 512 bytes
- Hash function: SHA-256 (hardware accelerator recommended)
- Required fields only: `type`, `subject`, `relation`, `object`, `confidence`, `created_at`, `namespace`
- Omit: `context`, `derived_from`, `provenance_chain`, `content_refs`, `embedding_refs`
- Encryption: Transport-level only (DTLS/TLS)
- Streaming deserialization recommended (no full-blob-in-memory)

---

## 19. Error Handling

### 19.1 Format Errors

| Condition | Error Code | Message |
|-----------|-----------|---------|
| Blob shorter than 10 bytes | `ERR_TOO_SHORT` | Blob must be at least 10 bytes (9-byte header + payload) |
| Unsupported version byte | `ERR_VERSION` | Unsupported format version: {version} |
| Malformed MessagePack/CBOR | `ERR_CORRUPT` | Invalid payload encoding |
| Payload is not a map | `ERR_NOT_MAP` | Payload must be a MessagePack/CBOR map |
| Missing `type` field | `ERR_NO_TYPE` | Missing required field: type |
| Unknown type value | `ERR_UNKNOWN_TYPE` | Unknown memory type: {type} |
| Missing required field | `ERR_SCHEMA` | Missing required field: {field} |

### 19.2 Integrity Errors

| Condition | Error Code |
|-----------|-----------|
| SHA-256 hash mismatch | `ERR_INTEGRITY` |
| Content address not lowercase hex | `ERR_HASH_FORMAT` |
| Content address wrong length | `ERR_HASH_LENGTH` |

### 19.3 Validation Errors

| Condition | Error Code |
|-----------|-----------|
| Confidence out of [0.0, 1.0] | `ERR_RANGE` |
| Importance out of [0.0, 1.0] | `ERR_RANGE` |
| Empty required string | `ERR_EMPTY` |
| Negative count field | `ERR_RANGE` |
| Float64 value is NaN or Infinity | `ERR_FLOAT_INVALID` |
| `signed` flag ≠ presence of COSE wrapper | `ERR_SIGNED_MISMATCH` |
| Header sensitivity bits lower than tag classification | `ERR_SENSITIVITY_MISMATCH` |
| Duplicate map keys | `ERR_CORRUPT` |
| String contains BOM (`EF BB BF`) | `ERR_CORRUPT` |
| Supersession or contradiction violates `invalidation_policy` | `ERR_INVALIDATION_DENIED` |
| `invalidation_policy.mode` is unknown (fail-closed) | `ERR_INVALIDATION_DENIED` |
| Protected goal `satisfied` transition missing required evidence | `ERR_EVIDENCE_REQUIRED` |

### 19.4 Forward Compatibility

Implementations MUST handle forward-compatible changes gracefully:

1. **Unknown fields** → Deserializers preserve during round-trip; no error
2. **Unknown types** → Deserialize as opaque map (no schema validation)
3. **Future version bytes** → Reject with `ERR_VERSION`; include version in error message

---

## 20. Security Considerations

### 20.1 Integrity and Authenticity

Content addressing (SHA-256 hash) proves integrity but NOT authenticity. Any party can produce a valid grain.

For authenticity, use COSE Sign1 envelope with DID-based identity verification.

### 20.2 Confidentiality

The .mg format itself does NOT define encryption. When encryption is required, encrypt the entire blob as an opaque byte sequence using authenticated encryption (e.g., AES-256-GCM).

Content address of encrypted grain is the hash of ciphertext, not plaintext.

**Note on deduplication:** Encrypting a grain changes its content address. Encrypting the same plaintext with different keys or IVs produces different ciphertext and therefore different content addresses. Encrypted grains do not deduplicate via content address. Systems requiring deduplication of encrypted data SHOULD compute and store the plaintext content address separately as metadata before encryption.

### 20.3 Per-User Encryption Pattern

For compliance systems handling personal data:

1. Derive per-user key via HKDF-SHA256 from master key + user_id
2. Encrypt grain bytes with AES-256-GCM (user's key)
3. Generate HMAC token (blind index) for encrypted user_id field
4. Store: `{content_address: encrypted_blob, user_id_token: hmac(...)}`
5. Query: Look up blind index first, then decrypt matching grains

Destroying user's key → O(1) GDPR erasure (crypto-erasure).

### 20.4 Timing Attacks

When comparing content addresses for integrity verification, use constant-time comparison:
- Python: `hmac.compare_digest()`
- Go: `crypto/subtle.ConstantTimeCompare()`
- JavaScript: `crypto.timingSafeEqual()`

### 20.5 Content Reference Security

URIs in `content_refs` and `embedding_refs` MAY point to external resources. When fetching:

1. Validate URI (reject private IP ranges unless explicitly allowed)
2. Verify `checksum` field after fetching (detect tampering)
3. Never auto-fetch during deserialization (fetch-on-demand only)

### 20.6 Compliance Scenarios

**GDPR Erasure (Art. 17):**
Encrypt grains with per-user keys. Destroying user's key renders all their ciphertext unrecoverable. `user_id` field enables scoping.

**HIPAA PHI Detection:**
Tag PHI-containing grains with `structural_tags` prefix `"phi:"`. Policy engines inspect tags at write time.

**SOX Audit Trails (Sarbanes-Oxley, Section 802):**
.mg blobs are tamper-evident (content-addressed, immutable). `provenance_chain` traces derivation. Combined with hash-chained audit log, provides complete audit trail.

---

## 21. Test Vectors

> **Implementation note:** Content addresses are SHA-256 of the complete blob: 9-byte fixed header (`0x01` version, flags, type, 2-byte ns_hash, created_at_sec) followed by the canonical MessagePack/CBOR payload. Run the reference implementation against each input to produce verified hashes. The blob hex for Vector 1 is provided as a byte-level reference; all content addresses marked `[computed by reference implementation]` must be derived programmatically.

### 21.1 Vector 1: Minimal Fact

**Input:**
```json
{
  "type": "fact",
  "subject": "user",
  "relation": "prefers",
  "object": "dark mode",
  "confidence": 0.9,
  "source_type": "user_explicit",
  "created_at": 1768471200000,
  "namespace": "shared",
  "author_did": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK"
}
```

**Expected content address:**
```
3288d0d41cf49a1d428e404f0b6a6fe60388be9536937557f6139b813d53a520
```

**Blob hex (159 bytes):**
```
01 00 01 a4 d2 69 68 ba a0 89 a4 61 64 69 64 d9 38 64 69 64 3a 6b 65 79 3a
7a 36 4d 6b 68 61 58 67 42 5a 44 76 6f 74 44 6b 4c 35 32 35 37 66 61 69 7a
74 69 47 69 43 32 51 74 4b 4c 47 70 62 6e 6e 45 47 74 61 32 64 6f 4b a1 63
cb 3f ec cc cc cc cc cc cd a2 63 61 cf 00 00 01 9b c1 19 01 00 a2 6e 73 a6
73 68 61 72 65 64 a1 6f a9 64 61 72 6b 20 6d 6f 64 65 a1 72 a7 70 72 65 66
65 72 73 a1 73 a4 75 73 65 72 a2 73 74 ad 75 73 65 72 5f 65 78 70 6c 69 63
69 74 a1 74 a4 66 61 63 74
```

> Header breakdown: `01`=version, `00`=flags (public, MessagePack, unsigned), `01`=Fact type, `a4 d2`=SHA-256("shared")[0:2] as uint16 big-endian, `69 68 ba a0`=created_at_sec (1768471200 = 2026-01-15T10:00:00Z, big-endian).
>
> Payload breakdown: `89`=fixmap(9), `a4 61 64 69 64`=key "adid" (fixstr 4), `d9 38`=str8 length 56, followed by 56 UTF-8 bytes of the DID; key `c` value: `cb 3f ec cc cc cc cc cc cd` (float64 marker + 8 bytes = `3feccccccccccccd` = 0.9); then remaining keys "ca"/"ns"/"o"/"r"/"s"/"st"/"t" in lexicographic order with their values.

### 21.2 Vector 2: Event

**Input:**
```json
{
  "type": "event",
  "content": "User asked about dark mode settings",
  "created_at": 1768471200000,
  "namespace": "shared",
  "author_did": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "importance": 0.5
}
```

**Expected content address:**
```
[computed by reference implementation]
```

### 21.3 Vector 3: Bi-Temporal Fact

**Input:**
```json
{
  "type": "fact",
  "subject": "Alice",
  "relation": "works_at",
  "object": "Acme Corp",
  "confidence": 0.95,
  "source_type": "user_explicit",
  "created_at": 1737000000000,
  "valid_from": 1735689600000,
  "valid_to": 1767225600000,
  "system_valid_from": 1737000000000,
  "author_did": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK"
}
```

**Expected content address (bi-temporal fields):**
```
[computed by reference implementation]
```

### 21.4 Vector 4: Fact with Cross-Links

**Input:**
```json
{
  "type": "fact",
  "subject": "Bob",
  "relation": "manages",
  "object": "Project Alpha",
  "confidence": 0.90,
  "source_type": "llm_generated",
  "created_at": 1737000000000,
  "related_to": [
    {
      "hash": "4c4149355d3f3e1114e6a72bc5c2813a3ecd4deab2ba8771eaca8556b2c032f2",
      "relation_type": "similar",
      "weight": 0.85
    },
    {
      "hash": "6f7fb8935e150f61a607ece0582c87c42b9975d356def0e41164b85852836145",
      "relation_type": "elaborates",
      "weight": 0.70
    }
  ],
  "author_did": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK"
}
```

### 21.5 Vector 5: Observation

**Input:**
```json
{
  "type": "observation",
  "observer_id": "temp-sensor-01",
  "observer_type": "temperature",
  "subject": "server-room",
  "object": "22.5C",
  "confidence": 0.99,
  "created_at": 1737000000000,
  "namespace": "monitoring",
  "importance": 0.3,
  "author_did": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK"
}
```


### 21.6 Vector 6: Protected Fact with invalidation_policy

**Input:**
```json
{
  "type": "fact",
  "subject": "agent-007",
  "relation": "constraint",
  "object": "never delete user files without confirmation",
  "confidence": 1.0,
  "source_type": "user_explicit",
  "created_at": 1768471200000,
  "namespace": "safety",
  "invalidation_policy": {
    "mode": "locked",
    "authorized": ["did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK"]
  }
}
```

**Compaction and canonical form notes:**
- Compacted key order: `c`, `ca`, `ip`, `ns`, `o`, `r`, `s`, `st`, `t` — verifies that `ip` (`invalidation_policy`) sorts correctly between `ca` and `ns`.
- The nested `invalidation_policy` map is also sorted: `authorized` before `mode`.
- Namespace `"safety"` → SHA-256 first two bytes: `0x85 0x6E`.
- Header: `0x01 0x00 0x01 0x85 0x6E` + timestamp `1768471200` as big-endian 4 bytes.

**Expected content address:**
```
df928038769506fb66671aced0eb97d45871e169e505ed55a382c744e620550e
```

---

## 22. Implementation Notes

### 22.1 MessagePack Libraries

| Language | Library | Sorted Keys | Notes |
|----------|---------|-------------|-------|
| Python | `ormsgpack` | `OPT_SORT_KEYS` | Rust-backed (fast) |
| Python | `msgpack` | `sort_keys=True` | Pure Python fallback |
| Rust | `rmp-serde` | Via `BTreeMap` | Natural ordering |
| Go | `msgpack/v5` | Manual sorting | User responsible |
| JavaScript | `@msgpack/msgpack` | Pre-sort keys | Manual sorting required |
| Java | `jackson-dataformat-msgpack` | `SORT_PROPERTIES_ALPHABETICALLY` | Feature flag |
| C# | `MessagePack-CSharp` | Via `SortedDictionary` | Built-in support |

### 22.2 String Normalization

Use Unicode NFC (Canonical Composition):
- Python: `unicodedata.normalize("NFC", s)`
- Go: `golang.org/x/text/unicode/norm`
- JavaScript: `String.prototype.normalize("NFC")`
- Java: `java.text.Normalizer`

### 22.3 Constant-Time Hash Comparison

```python
import hmac
hmac.compare_digest(expected_hash, computed_hash)
```

```go
import "crypto/subtle"
subtle.ConstantTimeCompare(a, b) == 1
```

```javascript
import crypto from "crypto";
crypto.timingSafeEqual(a, b);
```

### 22.4 DID Parsing (did:key)

```
Format: did:key:z<multibase-base58-btc-encoded-multicodec-key>

Example: did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK

Parsing:
1. Remove "did:key:" prefix
2. Decode multibase (z = base58-btc) → raw bytes
3. Read multicodec prefix: one or more unsigned varint bytes identify the key type
   - Ed25519 public key: prefix 0xed 0x01 (2-byte varint), followed by 32 key bytes
   - Other key types use different varint values; always decode the full varint, not a fixed byte count
4. Extract public key bytes (everything after the varint prefix)
5. Verify signature using extracted public key
```

### 22.5 COSE Sign1 Libraries

- Python: `pycose` (RFC 9052 compliant)
- Go: `github.com/veraison/go-cose`
- JavaScript: `cose-js`, `cbor-x`
- Rust: `cosey`

### 22.6 Round-Trip Testing

To verify conformance:

1. Serialize grain → blob
2. Hash blob → content address
3. Compare against expected (test vector)
4. Deserialize blob → recreate grain
5. Serialize again → MUST match original blob bytes (round-trip fidelity)

### 22.7 Streaming and Partial Results

OMS grains are atomic, immutable knowledge units. Streaming outputs (e.g., token-by-token LLM responses, incremental tool results, partial server-sent events) are transport-layer concerns outside OMS scope. Implementations SHOULD buffer streaming content in their transport layer and emit a single immutable Event or Tool grain upon stream completion. For long-running tool executions requiring progress visibility, implementations MAY emit periodic State grains (type 0x03) as progress checkpoints, linked via `derived_from` to the originating Tool grain. Each checkpoint is a complete, self-contained grain — not a diff.

### 22.8 Recall Priority and Agent Memory Tiers

The `recall_priority` field (§6.1) maps to the memory tiering models used by agent frameworks:

| `recall_priority` | Tier | Framework mapping | Retrieval pattern |
|---|---|---|---|
| `"hot"` | In-context memory | Letta `core_memory`, LangChain `ConversationBufferMemory` | Included in every LLM prompt. Grains SHOULD be cached in-memory by the store. |
| `"warm"` | Retrieval memory | Letta `recall_memory`, LangChain `VectorStoreRetrieverMemory` | Retrieved by recency, embedding similarity, or structured filter. Typical RAG context. |
| `"cold"` | Archival memory | Letta `archival_memory`, long-term compliance storage | Retained for completeness, audit, and compliance. Not actively retrieved unless explicitly queried. |

Stores MAY use `recall_priority` to select storage tiers (e.g., SSD for hot, HDD for cold, object storage for archive). Writers SHOULD set `recall_priority` based on expected retrieval frequency. The default when absent is `"warm"`.

### 22.9 State Grain Context Schema Convention

For cross-framework agent state portability, implementations SHOULD use the following keys in the State grain (type 0x03) `context` map:

| Key | Type | Description |
|---|---|---|
| `messages_tail` | string | Content address of the most recent Event grain in the conversation |
| `memory_blocks` | map | Named memory blocks: `{block_name: block_value_string}`. Letta-compatible. |
| `system_prompt` | string | System prompt text, or content address of a Fact grain containing it |
| `active_tools` | array[string] | Tool names available in this agent state |
| `model` | string | LLM model identifier (e.g., `"claude-opus-4-6"`) |
| `pending_tool_calls` | array[string] | Content addresses of Tool grains in `"call"` phase awaiting results |
| `agent_config` | map | Framework-specific agent configuration (opaque to the spec) |

This schema is RECOMMENDED, not required. Implementations MAY include additional keys. The `memory_blocks` key is aligned with Letta's `core_memory` structure. The `messages_tail` key enables reconstructing the conversation by following `parent_message_id` chains backward from the tail.

### 22.10 Access Counter Semantics

Stores that implement `access_count` and `last_accessed_at` (§28.3) SHOULD observe the following:

- Stores MAY defer counter updates and flush them asynchronously. The maximum acceptable staleness is implementation-defined but SHOULD be documented.
- Only **user-facing retrieval operations** (search, get, query) SHOULD increment `access_count`. Internal reads — provenance traversal, invalidation checks, supersession chain resolution, compliance scans, and replication — SHOULD NOT increment it.
- Stores MAY use probabilistic counting (e.g., HyperLogLog) or sampling for high-frequency grains to limit write amplification.
- Stores MAY disable access tracking entirely and document this as a conformance note. `access_count` and `last_accessed_at` are OPTIONAL index-layer features, not conformance requirements.

---

## References

### Normative References

- [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) — Requirement Levels (MUST, SHOULD, etc.)
- [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) — Ambiguity of Uppercase vs Lowercase in RFC 2119
- [RFC 8949](https://www.rfc-editor.org/rfc/rfc8949) — CBOR (Concise Binary Object Representation)
- [RFC 9052](https://www.rfc-editor.org/rfc/rfc9052) — COSE (CBOR Object Signing and Encryption) Structures
- [RFC 9901](https://www.rfc-editor.org/rfc/rfc9901) — SD-JWT (Selective Disclosure for JSON Web Tokens)
- [FIPS 180-4](https://csrc.nist.gov/publications/detail/fips/180/4/final) — SHA-256
- [UAX #15](https://unicode.org/reports/tr15/) — Unicode Normalization Forms
- [W3C DID Core 1.0](https://www.w3.org/TR/did-core/) — Decentralized Identifiers
- [MessagePack Specification](https://github.com/msgpack/msgpack/blob/master/spec.md)

### Informative References

- [W3C PROV-Overview](https://www.w3.org/TR/prov-overview/) — Provenance Data Model
- [Deterministic CBOR — RFC 8949 §4.2.1](https://www.rfc-editor.org/rfc/rfc8949#section-4.2.1) — Deterministic CBOR Encoding (Preferred Serialization)
- [Gordian Envelope Internet-Draft](https://datatracker.ietf.org/doc/draft-mcnally-envelope/) — Content-Addressed Documents
- [did:key Method Specification](https://w3c-ccg.github.io/did-key-spec/)
- [GDPR Article 17](https://gdpr-info.eu/art-17-gdpr/) — Right to Erasure
- [HIPAA Technical Safeguards](https://www.hhs.gov/hipaa/for-professionals/security/laws-regulations/) — Protected Health Information
- [CCPA](https://oag.ca.gov/privacy/ccpa) — California Consumer Privacy Act

---

## 23. Grain Protection and Invalidation Policy

### 23.1 Purpose

A grain may carry an `invalidation_policy` field declaring who is authorized to remove it from "current and trusted" status. This field covers **all invalidation paths**, not only direct supersession:

1. **Direct supersession** — a new grain G2 is written with `derived_from: [G1]` and the index sets `G1.superseded_by = hash(G2)`
2. **Contradiction** — the index sets `G1.contradicted = true`
3. **Semantic replacement via `related_to`** — advisory only; does NOT constitute formal invalidation (see §23.7)

The `invalidation_policy` governs paths 1 and 2. Protection is declared at grain creation time — it is part of the immutable blob and covered by the COSE signature when present.

### 23.2 Field Schema

```msgpack
invalidation_policy: {
  "mode": "open" | "soft_locked" | "locked" | "quorum" | "delegated" | "timed" | "hold" | "consent_cascade",
  "authorized": ["did:key:z6Mk...", ...],   // for modes: delegated, quorum
  "threshold": 2,                            // for mode: quorum — minimum co-signers
  "locked_until": 1800000000,               // for mode: timed — Unix epoch u64 seconds
  "fallback_mode": "open",                  // for mode: timed — policy after unlock time
  "scope": "grain" | "subtree" | "lineage", // default: "grain"
  "protection_reason": "string"             // optional human-readable rationale
}
```

**Mode semantics:**

| Mode | Semantics | Store behavior |
|------|-----------|----------------|
| `open` | No restriction (default when field is absent) | Accept any supersession |
| `soft_locked` | Supersession permitted but MUST carry `supersession_justification` field | Accept with justification; flag for human review |
| `locked` | No supersession or contradiction permitted | MUST reject; return `ERR_INVALIDATION_DENIED` |
| `quorum` | Superseding grain MUST carry `supersession_auth` array with ≥ `threshold` valid COSE signatures from `authorized` DIDs | Verify each signature; reject if threshold not met |
| `delegated` | Only DIDs listed in `authorized` may invalidate; superseding grain MUST be COSE-signed by one of those DIDs | Verify signer is in `authorized` list |
| `timed` | Behaves as `locked` until `locked_until` epoch; then reverts to `fallback_mode` | Check wall clock against `locked_until`; apply `fallback_mode` after |
| `hold` | **Litigation hold** — grain MUST NOT be deleted, erased, or forgotten until hold is explicitly lifted. Supersedes TTL, consent withdrawal, erasure requests, and forgetting engine decay. | Reject all invalidation and erasure operations; return `ERR_INVALIDATION_DENIED` |
| `consent_cascade` | Grain is automatically eligible for erasure when its `processing_basis` Consent grain (§8.10, §6.1) is revoked. Stores MUST complete erasure within their stated SLA; SLA MUST be ≤ one month per GDPR Art. 12(3). | On Consent withdrawal, identify all grains with matching `processing_basis`, schedule for erasure within SLA |

### 23.3 Fail-Closed Rule

Parsers and stores MUST treat **unknown `mode` values as `"locked"`**. An implementation that encounters an unrecognized mode MUST reject the invalidation attempt rather than treating the policy as absent or permissive. This prevents an agent from crafting a novel mode value that an old implementation silently accepts.

If the `invalidation_policy` field is absent entirely, `mode: "open"` is implied.

### 23.4 Goal State Transition Protection

Protected Goal grains (those with `invalidation_policy.mode ≠ "open"`) MAY specify which state transitions the agent may execute autonomously via the `allowed_transitions` field:

```json
{
  "type": "goal",
  "goal_state": "active",
  "invalidation_policy": {
    "mode": "locked",
    "authorized": ["did:key:z6MkUser..."]
  },
  "allowed_transitions": ["satisfied", "failed"]
}
```

State transitions NOT listed in `allowed_transitions` are subject to the full `invalidation_policy`. If `allowed_transitions` is absent on a protected goal, **all** state transitions are subject to the policy.

**Reasoning:** Some goal lifecycle transitions (marking a goal `satisfied` because it was achieved, or `failed` because it became impossible) are natural completion events, not adversarial modifications. `allowed_transitions` lets the user designate these autonomous-safe transitions without making the entire goal unprotected.

**Evidence requirement for autonomous `satisfied` transitions:** For protected goals, an autonomous `satisfied` transition SHOULD include `satisfaction_evidence` grain references. Stores MAY enforce this when `evidence_required > 0` is set. This mitigates goal laundering.

### 23.5 Goal Laundering (Normative)

**Goal laundering** is the attack pattern where an agent:
1. Falsely marks a protected goal as `satisfied` (claiming success criteria were met)
2. Creates a new goal without the protected goal's constraints
3. Operates under the weaker new goal

Implementations MUST treat this as a protocol violation. Specifically:
- A grain that supersedes a protected goal inherits the original goal's `invalidation_policy` unless the supersession was explicitly authorized under that policy's terms
- `satisfied` and `failed` transitions on protected goals that have these in `allowed_transitions` SHOULD require non-empty `satisfaction_evidence`; stores MAY enforce this as `ERR_EVIDENCE_REQUIRED`

### 23.6 Scope

The `scope` field controls whether protection extends to derived grains:

| Scope | Meaning |
|-------|---------|
| `grain` | Only this grain (default) |
| `subtree` | This grain and all grains with `derived_from` pointing here (transitively, up to 16 hops) |
| `lineage` | This grain and all grains in the same supersession chain |

For `subtree` scope, a store MUST check the derivation ancestry of any proposed superseding grain and reject if any ancestor within 16 hops is protected against the requester. Implementations SHOULD cache a `protected_root` indicator per grain to avoid O(n) traversal per write.

### 23.7 Bypass Paths That Conformant Implementations MUST Close

**Bypass 1 — Contradiction flag:** Any mutation setting `contradicted=true` on a grain is subject to `invalidation_policy`, identical to supersession. The policy check MUST apply to contradiction index mutations, not only to supersession index mutations.

**Bypass 2 — `related_to: "replaces"` semantic claim:** Writing a new grain with `relation_type: "replaces"` pointing to a protected grain is permitted at the blob level (it is a new, valid content-addressed object). However, a conformant store MUST NOT update the target grain's index entry (`superseded_by`, `contradicted`, `system_valid_to`) in response to seeing a `replaces` relation. The target grain remains current and its `invalidation_policy` is not affected. See §15.3 normative note.

**Bypass 3 — Supersession chain injection:** An agent cannot bypass protection on grain A by superseding a derived grain A' (which itself supersedes A), arguing it is not directly superseding A. A store MUST traverse the `derived_from` chain of any proposed superseding grain up to 16 hops and reject if any ancestor in the chain is protected against the requester.

### 23.8 Key Separation Requirement (Normative, Deployment-Dependent)

Grain-level `invalidation_policy` enforcement is only meaningful when the agent's DID is cryptographically distinct from the user's DID. If an agent operates under the user's signing key, any DID-based policy check trivially passes regardless of the declared policy.

Deployments using `invalidation_policy` with `mode ≠ "open"` SHOULD enforce **key separation**: the user holds a root DID keypair; agents receive delegated DIDs with scoped authority via W3C Verifiable Credentials or UCAN capability tokens. The .mg format does not define the delegation mechanism, but conformant stores SHOULD refuse to accept a supersession proof where the agent DID is identical to the grain's `author_did` for grains with `mode: "locked"` or `mode: "quorum"`.

### 23.9 Interaction with Existing Fields

| Field | Interaction |
|-------|-------------|
| `superseded_by` | Index layer populates after a conformant `supersede` operation passes policy check |
| `contradicted` | Setting this is subject to `invalidation_policy`; not a bypass path |
| `expiry_policy` (Goal) | Orthogonal — governs when a goal is inactive; `invalidation_policy` governs who writes its replacement. An expired goal's `invalidation_policy` still applies to supersession for audit chain integrity. |
| `evidence_required` (Goal) | Linked — for protected goals with `"satisfied"` in `allowed_transitions`, `evidence_required > 0` is RECOMMENDED |
| `source_type` | Orthogonal — records provenance; do not conflate with protection. A `"user_explicit"` grain is not automatically protected; `invalidation_policy` must be set explicitly. |
| `structural_tags` | `"mg:protected"` MAY be added as a human-facing annotation alongside `invalidation_policy` but MUST NOT be used as the sole enforcement mechanism |

---

## 24. Observer Type Registry

The `observer_type` field on Observation grains is an **open enum**. Applications may define custom values beyond those listed here. Standard values are organized into two domains. Index layers MAY use this field to route physical Observation grains to time-series stores and cognitive Observation grains to vector + relational stores, but MUST NOT hard-code the domain partition list — treat `observer_type` as an open string governed by configuration or namespace.

### 24.1 Physical Observer Domain

Physical observers produce measurements of the material world: geometry, position, temperature, electromagnetic fields, acoustic signals. `source_type` SHOULD be `"sensor"` for grains produced by physical observers.

| Value | Description |
|---|---|
| `"lidar"` | 3D laser ranging — time-of-flight or FMCW; produces point clouds |
| `"camera"` | RGB, depth, stereo, or thermal imaging |
| `"imu"` | Inertial Measurement Unit — fused gyroscope + accelerometer readings |
| `"gps"` | Global Positioning System or any GNSS receiver |
| `"temperature"` | Thermal sensor — thermocouple, thermistor, RTD, infrared |
| `"pressure"` | Barometric, fluid, or contact pressure sensor |
| `"accelerometer"` | Linear acceleration sensor (standalone, not fused with gyroscope) |
| `"magnetometer"` | Magnetic field sensor or digital compass |
| `"ultrasonic"` | Ultrasonic distance ranging — time-of-flight |
| `"radar"` | Radio detection and ranging |
| `"microphone"` | Audio input or acoustic sensor |

### 24.2 Cognitive Observer Domain

Cognitive observers produce observations of the information space: conversations, documents, behaviors, patterns, classifications. `source_type` SHOULD be `"agent_inferred"` for AI-generated cognitive observations and `"user_explicit"` for human observations.

| Value | Description |
|---|---|
| `"llm"` | Large Language Model as observer — produces natural language observations from input data. `observer_model` RECOMMENDED. |
| `"reflector"` | Aggregating or pattern-distilling agent — produces higher-order observations from prior Observation grains. Maps to `consolidation_level` ≥ 2. `observer_model` RECOMMENDED. |
| `"classifier"` | ML classification model — produces categorical observations (label + confidence score). `observer_model` RECOMMENDED. |
| `"detector"` | ML detection or anomaly detection model — produces presence/absence or anomaly observations. `observer_model` RECOMMENDED. |
| `"human"` | Human observer or annotator — records direct perception or expert judgment. `observer_model` MUST be absent. |
| `"hybrid"` | Combined physical sensor + AI processing pipeline — e.g., camera + vision model producing a semantic label from raw imagery. SHOULD include provenance_chain entries for both sensor reading and inference steps. |

### 24.3 Extensibility

Custom `observer_type` values MUST NOT be identical to any registered value in §24.1 or §24.2. Custom values SHOULD use a namespace prefix, e.g., `"acme:thermal-v2"` or `"myapp:custom-observer"`. Conformant parsers MUST NOT reject unknown `observer_type` values.

---

## 25. Observation Mode Registry

The `observation_mode` field is a **closed enum**. It describes how the observation was produced, which determines how `confidence`, `valid_from`/`valid_to`, and `derived_from` should be interpreted by downstream consumers.

| Value | Meaning | `valid_from`/`valid_to` semantics | Typical `observer_type` |
|---|---|---|---|
| `"passive"` | Observer perceived without intervening — watched, listened, read data as it arrived without emitting a signal or query | Covers the duration of passive reception | `"camera"`, `"microphone"`, `"llm"`, `"human"` |
| `"active"` | Observer actively sampled or probed — emitted a signal, sent a query, asked a question to elicit a response | Marks the precise moment of the probe and its response window | `"lidar"`, `"radar"`, `"ultrasonic"`, `"llm"` |
| `"reflective"` | Observer processed past data to synthesize — looked back at prior grains, compressed, or reflected. `derived_from` SHOULD be populated with the content addresses of consumed grains. | Spans the window of the consumed input data, **not** the moment the grain was written. `created_at` is the write time; `valid_from`/`valid_to` is the observed window. | `"reflector"`, `"llm"` |
| `"real_time"` | Observer processed data as it arrived — stream processing with no meaningful buffering. `created_at` ≈ event time. | Point-in-time; `valid_from` ≈ `created_at` | `"imu"`, `"gps"`, `"microphone"`, `"llm"` (streaming inference) |

**Absent:** When `observation_mode` is absent, no mode assertion is made. Consumers SHOULD treat the observation as mode-unclassified and apply conservative trust calibration.

**Interaction with `active` mode:** Grains produced by an active observer SHOULD record the probe or query that triggered the observation in `context["probe"]`. This enables verification that the observed response corresponds to the stated query.

---

## 26. Observation Scope Registry

The `observation_scope` field is a **closed enum**. It describes the temporal breadth of what was observed — how much time the observation covers — enabling correct interpretation of `valid_from`/`valid_to` and appropriate retrieval strategies.

| Value | Temporal Breadth | Physical Example | Cognitive Example |
|---|---|---|---|
| `"point"` | Single moment — one reading, one event, one inference | GPS fix at t=T; one temperature sample | Single-message LLM impression; one annotated event |
| `"interval"` | Defined time window — seconds to tens of minutes | 1-second IMU batch; 10-minute sensor log segment | LLM observer notes compressing the last 30 minutes of conversation |
| `"session"` | Entire interaction session — minutes to hours | Full robot mission from start to dock | LLM observer notes covering a complete conversation thread |
| `"longitudinal"` | Across multiple sessions — days, weeks, or longer | Multi-day environmental monitoring log | Reflector cross-session pattern spanning weeks of user interactions |

**Default behavior:**
- For physical observers, `"point"` is implied when `observation_scope` is absent.
- For cognitive observers with `observation_mode: "reflective"`, `"interval"` or `"session"` SHOULD be set explicitly. Absent scope on a reflective cognitive observation is a conformance warning at Level 2.

**Interaction with temporal fields:**
- `"point"` → `valid_from` ≈ `valid_to` ≈ `created_at`; often omitted entirely
- `"interval"` → `valid_from` < `valid_to`; window is typically much shorter than a session
- `"session"` → `valid_from` = session start, `valid_to` = session end
- `"longitudinal"` → `valid_from` = earliest covered session, `valid_to` = latest covered session; `derived_from` SHOULD enumerate the intermediate Observation grains from each covered session
---

## 27. Grain Type Field Specifications

This section provides detailed field specifications for each standard grain type. For Tool grain phase fields, see §27.1. For Observer types, see §24. For Observation modes/scopes, see §25/§26.

### 27.1 Tool Grain (type = 0x05) — Phase and Mode Details

The `tool_phase` field acts as a discriminator for async vs. synchronous tool call recording.

**`tool_phase` discriminator:**

| Value | Meaning | Required fields | Absent fields |
|---|---|---|---|
| `"definition"` | **Definition** — tool schema record | `tool_name`, `tool_description`, `input_schema` | `input`, `content`, `is_error`, `tool_call_id` |
| absent (default) | **Complete** — synchronous call | `tool_name`, `input`, `content`, `is_error` | `derived_from` |
| `"call"` | **Call** — async; result not yet received | `tool_name`, `input` | `content`, `is_error` |
| `"result"` | **Result** — async result arrived | `tool_call_id`, `content`, `is_error`, `derived_from` | `tool_name`, `input` |

**Phase-dependent field presence:**

| Field | `"definition"` | `"call"` | `"result"` | complete (absent) |
|---|---|---|---|---|
| `tool_name` | REQUIRED | REQUIRED | omit | REQUIRED |
| `tool_description` | REQUIRED | omit | omit | omit |
| `input_schema` | REQUIRED | omit | omit | omit |
| `output_schema` | optional | omit | omit | omit |
| `strict` | optional | omit | omit | omit |
| `tool_type` | optional | optional | omit | optional |
| `tool_version` | optional | optional | omit | optional |
| `input` | MUST NOT | REQUIRED | omit | REQUIRED |
| `tool_call_id` | omit | RECOMMENDED | REQUIRED | optional |
| `call_batch_id` | omit | optional | optional | optional |
| `content` | MUST NOT | MUST NOT | REQUIRED | REQUIRED |
| `is_error` | MUST NOT | MUST NOT | REQUIRED | REQUIRED |
| `stdout` / `stderr` | MUST NOT | MUST NOT | optional | optional |
| `exit_code` | MUST NOT | MUST NOT | optional | optional |
| `duration_ms` | MUST NOT | MUST NOT | optional | optional |
| `derived_from` | omit | omit | `[call grain hash]` | omit |

**`execution_mode` values:**

| Value | Meaning |
|---|---|
| absent (default) | Standard function call — `tool_name` + `input` |
| `"function_call"` | Explicit standard function call |
| `"code_exec"` | CodeAct-style: `code` field holds executable Python/shell; result in `stdout`/`stderr` |
| `"computer_use"` | Anthropic computer-use tool; `input` holds action type and coordinates |

---

**Example 0 — Tool definition grain:**

```json
{
  "type": "tool",
  "tool_phase": "definition",
  "tool_name": "get_weather",
  "tool_description": "Get the current weather in a given location.",
  "input_schema": {
    "type": "object",
    "properties": {
      "location": {"type": "string"},
      "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
    },
    "required": ["location"]
  },
  "output_schema": {
    "type": "object",
    "properties": {
      "temperature": {"type": "number"},
      "unit": {"type": "string"},
      "description": {"type": "string"},
      "humidity": {"type": "number"}
    }
  },
  "strict": true,
  "tool_type": "client",
  "author_did": "did:web:example.com:agents:assistant",
  "created_at": 1737000000000
}
```

**Example 1 — Synchronous function call:**

```json
{
  "type": "tool",
  "tool_name": "get_weather",
  "tool_call_id": "toulu_01A09q90qw90lq917835lq9",
  "input": {"location": "San Francisco, CA", "unit": "celsius"},
  "content": "15°C, partly cloudy",
  "is_error": false,
  "duration_ms": 312,
  "created_at": 1737000000000
}
```

**Example 2 — CodeAct code execution:**

```json
{
  "type": "tool",
  "execution_mode": "code_exec",
  "code": "import pandas as pd\ndf = pd.read_csv('data.csv')\nprint(df.describe())",
  "interpreter_id": "session-abc123",
  "stdout": "       age    salary\ncount  100.0   100.0",
  "exit_code": 0,
  "is_error": false,
  "created_at": 1737000000000
}
```

**Alignment with Anthropic API:**

| Anthropic API field | OMS Tool field |
|---|---|
| `tool.name` | `tool_name` (definition grain) |
| `tool.description` | `tool_description` |
| `tool.input_schema` | `input_schema` |
| `tool.strict` | `strict` |
| (no Anthropic equivalent) | `output_schema` |
| `tool_use.id` | `tool_call_id` |
| `tool_use.input` | `input` |
| `tool_result.content` | `content` |
| `tool_result.is_error` | `is_error` |

### 27.2 Goal Grain (type = 0x07) — Lifecycle and Provenance Details

**Provenance chain methods:**

| Method | Meaning |
|---|---|
| `"user_input"` | Human set this goal directly |
| `"goal_decomposition"` | Agent decomposed a parent goal |
| `"goal_state_transition"` | Updates state of a prior Goal grain |
| `"goal_revision"` | Human modified a previously set goal |
| `"goal_inference"` | Agent inferred from Event or Fact patterns |
| `"goal_delegation"` | Delegated from another agent |

### 27.3 `source_type` Registry

The `source_type` field is an **open enum**. Standard values:

| Value | Meaning |
|---|---|
| `"user_explicit"` | Directly stated by human user |
| `"agent_inferred"` | Derived by an AI agent |
| `"sensor"` | Physical instrument measurement |
| `"consolidated"` | Distilled from multiple prior grains |
| `"system"` | Written by infrastructure (provisioning, etc.) |
| `"llm_generated"` | Generated by a language model |
| `"imported"` | Imported from external source |
| `"established_knowledge"` | Widely accepted universal truth — physical constants, scientific laws, geographic facts. Grains with this value SHOULD omit `user_id`, SHOULD omit `valid_to`, SHOULD set `confidence: 1.0`, and SHOULD use `invalidation_policy.mode: "locked"`. |
| `"axiomatic"` | Definitionally or logically true — mathematical axioms, tautologies. Same SHOULD rules as `"established_knowledge"`. |

### 27.4 HIPAA PHI Tag Normalization

The 18 normative `phi:` tag values matching 45 CFR §164.514(b) Safe Harbor identifiers:

`phi:name`, `phi:geo_subdivision`, `phi:date`, `phi:age_over_89`, `phi:phone`, `phi:fax`, `phi:email`, `phi:ssn`, `phi:mrn`, `phi:health_plan_id`, `phi:account_number`, `phi:certificate_license`, `phi:vehicle_id`, `phi:device_id`, `phi:url`, `phi:ip_address`, `phi:biometric`, `phi:photo`.

Stores supporting HIPAA compliance MUST recognize all 18 and apply appropriate access controls. Any `phi:*` tag MUST be treated as PHI-sensitive regardless of whether the specific value appears in this list.

### 27.5 External Citation Schema

Scientific and legal workflows cite external artifacts outside the OMS hash space. The `content_refs` field accepts a structured `external_citation` object alongside standard content references:

```json
{
  "citation_type": "doi",
  "identifier": "10.1038/s41586-024-07487-w",
  "retrieved_at": 1737000000000,
  "content_hash": "sha256:abc123...",
  "citation_role": "supports"
}
```

| Field | Type | Required | Values |
|---|---|---|---|
| `citation_type` | string | REQUIRED | `"doi"`, `"arxiv"`, `"pmid"`, `"isbn"`, `"rrid"`, `"clinicaltrials"`, `"url"` |
| `identifier` | string | REQUIRED | Type-specific identifier |
| `retrieved_at` | int64 | OPTIONAL | Epoch ms of retrieval |
| `content_hash` | string | OPTIONAL | SHA-256 of retrieved document |
| `citation_role` | string | OPTIONAL | `"supports"`, `"refutes"`, `"extends"`, `"replicates"`, `"uses_data"`, `"uses_software"` |

The `derived_from` field SHOULD accept both OMS content addresses and external citation objects.

### 27.6 Trigger Definitions via Observation Grains — **REMOVED in 1.6**

Superseded by the **Trigger grain type (§8.13)**.

This section described a convention in which a trigger was an Observation grain
with `observer_type: "trigger:*"` and its configuration in the `context` map
under `int:` keys, on the reasoning that *"existing Observation fields
accommodate trigger definitions"*. That premise proved false three ways, and the
section is removed rather than deprecated: preserving text that instructs
implementers to write non-conformant grains is worse than removing it.

1. **It was non-conformant against this specification's own closed enums.** It
   placed `"periodic"` / `"continuous"` / `"scheduled"` in `observation_mode`,
   which §25 declares **closed** over `passive | active | reflective |
   real_time`, and a watch target such as `"repos/{owner}/{repo}/issues"` in
   `observation_scope`, which §26 declares **closed** over the *temporal
   breadth* values `point | interval | session | longitudinal`. Every example the
   section carried would be rejected by a Level 2 implementation enforcing §25
   and §26.
2. **Configuration in `context` could not be queried.** `context` is not among
   Observation's queryable fields, and field resolution is top-level; a filter on
   an `int:` key inside it cannot be pushed down. Moving the keys to the top
   level would make them queryable but places them outside `context`, which is
   where §A.7 says `int:` fields live. The two requirements were mutually
   exclusive.
3. **A trigger is not an observation.** §24 partitions observers into Physical
   ("measurements of the material world") and Cognitive ("observations of the
   information space"). A standing rule is neither, and registering `trigger:*`
   would have required inventing a third domain solely to make the
   classification hold.

Migration: a trigger Observation becomes a Trigger grain (§8.13). `observer_id`
becomes `connector`, the `trigger:*` suffix becomes `kind`, the watch target
becomes `scope`, and the cadence keys become first-class fields. `int:` keys
that genuinely describe connector transport stay in `config`.

### 27.7 Consensus Grain Usage for Tool Definition Validation

When multiple independent sources produce or validate the same Tool definition grain, a Consensus grain (type 0x09) records the agreement. This pattern is useful for integration platforms where definitions may be synthesized by LLMs, parsed from OpenAPI specs, validated against reference data, or refined by execution feedback analysis.

**Semantics:**
- `agreed_content` is the content address of the Tool definition grain that achieved consensus.
- Each entry in `participating_observers` is a DID identifying a validation source.
- `dissent_grains` link to alternative definitions that did not achieve consensus.
- Consensus achievement (`agreement_count >= threshold`) serves as a confidence signal for tool catalog quality.

**Example — Multi-source validation consensus:**

```json
{
  "type": "consensus",
  "participating_observers": [
    "did:web:example.com:agents:spec-parser",
    "did:web:example.com:agents:llm-synthesizer",
    "did:web:example.com:agents:reference-validator",
    "did:web:example.com:agents:execution-evaluator"
  ],
  "threshold": 2,
  "agreement_count": 3,
  "dissent_count": 1,
  "agreed_content": "<content-address-of-validated-definition-grain>",
  "dissent_grains": ["<content-address-of-alternative-definition>"],
  "structural_tags": ["consensus:tool-definition"],
  "namespace": "axtion:connectors:github",
  "related_to": [
    {"hash": "<definition-grain-hash>", "relation_type": "supports", "weight": 1.0}
  ],
  "created_at": 1740700000000
}
```

---

### 27.8 Trigger Execution Contract

§8.13 defines what a trigger *is*. This section defines what an implementation
that evaluates one MUST do. Declaring the shape without the contract was the
gap that made the removed §27.6 unimplementable in an interoperable way: nothing
normative said what evaluates a declaration, what a firing produces, or what
idempotency a twice-delivered item receives.

An implementation that evaluates triggers declares the `triggers` module (§17.4).
One that does not MAY store and replicate Trigger grains without acting on them;
they are ordinary grains.

**1. Firing MUST be journaled.** Each firing writes a record naming the trigger,
what it produced, and what it skipped. A trigger that has never fired MUST be
distinguishable from one that has fired and found nothing — otherwise a
misconfigured trigger and an idle one look identical, and the misconfiguration
has no symptom.

**2. Ingestion MUST be idempotent on the declared `dedup_key`.** The same item
seen twice — connector replay, overlapping cursors, two evaluators racing — MUST
produce one unit of work and a recorded skip, never two. Implementations SHOULD
derive the work identity from `(connector, dedup value)` rather than the dedup
value alone; following CloudEvents, an `id` is unique only within a producer, so
two connectors sharing an id space would otherwise collide.

**3. First evaluation MUST NOT replay history.** A newly declared `polling`
trigger records the source's current position and fires nothing. Without this,
declaring a trigger over an established source starts work for its entire
history.

**4. An absent baseline means different things for different kinds.** A trigger
with no recorded next-due time has never been evaluated. For a *relative*
cadence (`interval`, `polling`) that means it is due now — there is nothing to
wait out. For an *absolute* schedule (`schedule`, `once`) it does NOT: the
trigger is due at the instant it names, and an implementation MUST NOT fire a
`cron` of `0 9 * * *` merely because it was just declared. A `once` trigger MUST
fire at most once; "no next instant" MUST be represented distinctly from "no
baseline yet", or it re-fires indefinitely.

**5. Evaluation state MUST NOT replicate.** Next-due time, source cursor, lease
and failure count are host-local. Replicating them lets two hosts sharing a
memory each defer to the other's watermark, and lets a memory restored into a
different environment inherit a cursor and silently skip work it should have
done. This is the mirror of §8.12's rule that *governance* state must replicate:
a record of who did what has to travel with the file; a record of where one host
had got to must not.

**6. Concurrent evaluation MUST be arbitrated by the store.** Where several
evaluators can reach one memory, claiming MUST be a conditional write, and an
evaluator whose claim has expired MUST NOT be able to write behind whichever
one succeeded it. Losing a claim is a normal outcome and MUST NOT be reported as
an error.

**7. Failure MUST back off.** A connector that fails repeatedly MUST have its
next evaluation deferred, and the reason MUST be inspectable. An implementation
MUST NOT retry a failing source at its declared cadence.

**8. Calendar semantics MUST be stated, not assumed.** Implementations disagree
about a `cron` occurrence falling in a daylight-saving gap — some skip it, some
fire it immediately — and agree only on the repeated hour, where it fires once.
An implementation MUST document which it does, and SHOULD refuse a timezone it
would mishandle rather than fire at the wrong hour and be believed.

---

## 28. Query Conventions

### 28.1 Standard Search Response Envelope

OMS does not define a transport or query protocol. However, implementations that expose search APIs SHOULD return results using the following standard envelope to ensure interoperability:

```json
{
  "results": [
    {
      "grain": { "...grain payload..." },
      "score": 0.92,
      "matched_fields": ["object", "subject"],
      "content_address": "a1b2c3d4..."
    }
  ],
  "total": 142,
  "next_cursor": "opaque-pagination-token"
}
```

| Field | Type | Description |
|---|---|---|
| `grain` | map | Full deserialized grain payload |
| `score` | float64, [0.0, 1.0] | Retrieval relevance score — distinct from `confidence` (which is epistemic certainty). A high `score` means the grain matched the query well; a high `confidence` means the claim is believed to be true. |
| `matched_fields` | array[string] | Which payload fields contributed to the match |
| `content_address` | string | SHA-256 hex of the grain blob |

### 28.2 Namespace Convention

OMS uses `namespace` (single string) for logical partitioning and `user_id` for GDPR data subject scoping. Systems that require additional scoping dimensions SHOULD use structured namespace strings with `:` as the separator:

```
{org}:{app}:{agent}:{custom}
```

Examples:
- `"acme:chatbot:agent-7"` — org-scoped, app-scoped, agent-scoped
- `"acme:chatbot:agent-7:session-42"` — additionally run-scoped
- `"agent:identity"` — reserved for ownership and identity grains (§12.5)
- `"agent:authz"` — reserved for authorization grant grains (§12.6)
- `"shared"` — default, no specific partition

The `run_id` field (§6.1) provides session/run scoping orthogonal to the namespace hierarchy. Use `run_id` when runs are ephemeral and high-cardinality; use namespace segments when partitions are stable and low-cardinality.

### 28.3 Index-Layer-Managed Fields

The following fields are updated by the **store/index layer** after initial write, not by the grain author. These fields are **not stored in the immutable .mg blob**, are **not part of the content address**, and are **not covered by COSE signatures** (see §5.6). Writers MUST NOT set these fields; stores MUST update them atomically:

| Field | Updated when |
|---|---|
| `superseded_by` | A superseding grain is accepted |
| `system_valid_to` | Grain is superseded or contradicted |
| `verification_status` | Verification, contestation, or retraction occurs |
| `rec_status` | A Recommendation lifecycle transition is recorded (§8.12.1) — the store rebuilds this from the audit chain rather than accepting it from a writer |
| `access_count` | Grain is retrieved by a search or get operation (see §22.10 for semantics) |
| `last_accessed_at` | Grain is retrieved by a search or get operation (see §22.10 for semantics) |

### 28.4 Store Protocol Convention

OMS does not define a formal store API. However, implementations that expose a programmatic store interface SHOULD implement the following operations to ensure interoperability:

| Operation | Signature | Description |
|---|---|---|
| `get` | `(content_address) → grain \| not_found` | Retrieve a grain by its SHA-256 content address |
| `put` | `(blob_bytes) → content_address \| error` | Store a grain blob; returns its content address. Idempotent: re-storing an existing blob is a no-op. |
| `supersede` | `(old_address, new_blob_bytes, justification?) → new_address \| error` | Atomic supersession: validates `invalidation_policy` on the old grain, writes the new grain, and updates the old grain's index-layer fields (`superseded_by`, `system_valid_to`). This MUST be atomic — if any step fails, the entire operation rolls back. |
| `exists` | `(content_address) → bool` | Check if a grain exists without retrieving it |
| `query` | `(filters, sort, limit, cursor) → result_envelope` | Structured query with the response envelope from §28.1 |
| `search` | `(embedding_or_text, filters, limit) → result_envelope` | Semantic similarity search combined with structured filters |
| `delete` | `(content_address) → void \| error` | Compliance erasure (GDPR Art. 17, consent cascade, retention). MUST NOT be exposed as a general-purpose unauthenticated API; MAY be surfaced to authorized principals via CAL Tier 2 (`FORGET`/`PURGE`, CAL §8.14 — `delete`/`erase` verbs, §12.6), each execution writing an audit Observation. MUST check litigation holds (`invalidation_policy.mode: "hold"`) before deleting. **One-way:** additions roll back by retraction (supersession); a delete or erasure is final — no un-delete operation exists, and a host MUST NOT add one. |
| `put_batch` | `(blob_bytes[]) → content_address[] \| error[]` | Batch ingest for consolidation, migration, and high-throughput scenarios |
| `get_batch` | `(content_address[]) → grain[] \| not_found[]` | Batch retrieval for provenance chain traversal and context assembly |

Stores SHOULD implement `supersede` as a distinct operation rather than exposing raw `put` + index mutation separately. Supersession is the most error-prone operation (invalidation policy checks, derivation DAG traversal for `scope: "subtree"`, atomic index update) and deserves a dedicated, well-tested code path.

### 28.5 Agent Capability Convention

Agents that participate in multi-agent systems SHOULD advertise their capabilities by writing a Fact grain with the `mg:has_capability` relation to the `"agent:identity"` namespace. This grain serves as the OMS equivalent of an A2A Agent Card or MCP server capability declaration.

**Convention:**
```json
{
  "type": "fact",
  "subject": "did:web:example.com:agents:summarizer",
  "relation": "mg:has_capability",
  "object": {
    "name": "Text Summarizer",
    "description": "Summarizes long documents into key points",
    "supported_tools": ["summarize_text", "extract_entities"],
    "input_modalities": ["text"],
    "output_modalities": ["text"],
    "protocol": "oms",
    "max_context_tokens": 200000
  },
  "confidence": 1.0,
  "source_type": "system",
  "namespace": "agent:identity",
  "author_did": "did:web:example.com",
  "invalidation_policy": {
    "mode": "delegated",
    "authorized": ["did:web:example.com"]
  }
}
```

The `object` map is an open schema. Standard keys:

| Key | Type | Description |
|---|---|---|
| `name` | string | Human-readable agent name |
| `description` | string | Agent purpose and capabilities summary |
| `supported_tools` | array[string] | Tool names this agent can invoke (cross-reference with Tool definition grains) |
| `input_modalities` | array[string] | `"text"`, `"image"`, `"audio"`, `"video"`. What the agent can consume. |
| `output_modalities` | array[string] | What the agent can produce |
| `protocol` | string | Communication protocol: `"oms"`, `"mcp"`, `"a2a"`, `"custom"`. Open enum. |
| `max_context_tokens` | int | Maximum context window in tokens |
| `model` | string | Underlying LLM model identifier |

Agents can discover other agents by querying Fact grains with `relation: "mg:has_capability"` in the `"agent:identity"` namespace.

### 28.6 Conversation Threading Convention

Conversations are reconstructed from Event grain sequences using `session_id` and `parent_message_id`:

1. All Event grains in a conversation MUST share the same `session_id`.
2. Event grains SHOULD populate `parent_message_id` (§6.2) to form a linked list from newest to oldest.
3. Branch points are expressed by two Event grains sharing the same `parent_message_id` but having different content addresses (tree-of-thought, beam search, alternative paths).
4. A State grain (type 0x03) with `relation: "mg:state_at"` and a `context` map containing `{messages_tail, message_count, participants}` represents a conversation snapshot.
5. Conversation summaries are Fact grains with `consolidation_level >= 1`, `derived_from` pointing to the summarized Event grains, and `source_type: "consolidated"`.

**Retrieving a conversation:**
1. Query: `type=event, session_id=X, system_valid_to=null, sort=timestamp_ms ASC`
2. Or: start from the most recent Event grain (`messages_tail` in a State grain) and follow `parent_message_id` backward.

### 28.7 Session Handoff Convention

When Agent A transfers control of a conversation to Agent B, the handoff is recorded using a Goal grain with `mg:delegates_to` relation and delegation scope fields (§6.11):

1. Agent A writes a Goal grain with `relation: "mg:delegates_to"`, `subject` = Agent A's DID, `object` = Agent B's DID, and delegation scope fields specifying `authorized_namespaces`, `authorized_tools`, `context_grains`, and `return_to`.
2. The `context_grains` field contains content addresses of grains Agent B needs to continue — typically the recent Event grain chain and any relevant Fact/State grains.
3. Agent B ingests the referenced grains, validates the delegation scope, and continues with a new `run_id` but the same `session_id`.
4. When Agent B completes its task, it writes a Goal grain with `goal_state: "satisfied"` linked via `derived_from` to the delegation grain, and control returns to the agent specified in `return_to`.

### 28.8 Skill Lifecycle and Transfer

Skills are represented by the dedicated **Skill grain (type 0x0B, §8.11)**. A Skill grain is a packaged, reusable capability: its **definition** fields (instructions, when-to-use, tools, resources, dependencies) specify what the capability is and how to perform it, while its optional **learned-competence** fields (proficiency, practice count, strategies) record how well a particular agent performs it. This section defines the conventions for authoring, lifecycle, transfer, and discovery; §8.11 defines the type's fields and normative rules.

> **History:** Earlier OMS revisions described a `Fact + mg:capable_of` *convention* — a Fact grain whose `object` map carried the skill fields. OMS 1.4 promotes that pattern to a first-class Skill grain type (§8.11), following the Consent (§8.10) precedent. The `mg:capable_of` relation is retained in the vocabulary for backward compatibility, but new skills SHOULD be written as Skill grains.

**Distinction from related constructs:**

| Construct | Grain type | What it captures |
|---|---|---|
| Agent Capability (§28.5) | Fact (`mg:has_capability`) | Static identity card — what the agent *is built to do* |
| Skill (§8.11, §28.8) | Skill (0x0B) | Packaged, reusable capability — the definition (how-to, tools, resources) plus optional learned proficiency and strategies |
| Workflow (§8.4) | Workflow | Fixed action sequence — a single procedure, not an adaptive capability |

**Example Skill grain:**
```json
{
  "type": "skill",
  "name": "code_review",
  "description": "Review code changes for correctness, style, and security issues",
  "version": "2.1.0",

  "instructions": "Read the diff, classify the change, then apply the matching strategy. Always check auth and input-handling paths first.",
  "when_to_use": "a pull request or code diff needs review before merge",
  "allowed_tools": ["mg:sha256:b0c1d2..."],
  "resources": ["mg:sha256:c3d4e5..."],
  "dependencies": ["mg:sha256:f7e8d9..."],
  "domain": "software",
  "input_modalities": ["code", "text"],
  "output_modalities": ["text"],

  "holder_did": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "proficiency": 0.82,
  "practice_count": 47,
  "last_practiced_at": 1711929600000,
  "transferable": true,
  "strategies": [
    {
      "condition": "security_focused",
      "workflow": "mg:sha256:a1b2c3...",
      "description": "OWASP-oriented review for auth and input handling code"
    },
    {
      "condition": "style_focused",
      "workflow": "mg:sha256:d4e5f6...",
      "description": "Linting and convention compliance review"
    }
  ],

  "confidence": 0.82,
  "source_type": "agent_inferred",
  "namespace": "agent:skills",
  "author_did": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "related_to": [
    {"target": "mg:sha256:a1b2c3...", "type": "elaborates"},
    {"target": "mg:sha256:d4e5f6...", "type": "elaborates"}
  ]
}
```

See §8.11 for the full field reference (required fields, optional fields, and compaction keys).

**Namespace:** Skill grains SHOULD use the `"agent:skills"` namespace to enable efficient discovery queries.

**Proficiency lifecycle:**

1. **Acquisition.** When an agent first learns a capability, it writes a Skill grain with an initial `proficiency` (typically low). The `derived_from` field SHOULD reference the grains (Events, Tools, Reasoning) that contributed to learning.
2. **Improvement.** Each time proficiency changes, the agent supersedes the previous Skill grain with an updated `proficiency` and incremented `practice_count`. The supersession chain provides a full learning history.
3. **Degradation.** Implementations MAY decay proficiency over time if `last_practiced_at` exceeds a threshold. This is an index-layer concern, not a grain mutation — the store writes a new superseding grain with reduced proficiency.

**Skill transfer between agents:**

1. Agent A queries its store: `type=skill, namespace="agent:skills", transferable=true`.
2. Agent A shares the Skill grain and all Workflow grains referenced in `strategies` with Agent B (via `get_batch` on the content addresses).
3. Agent B ingests the grains, rewrites the Skill grain with its own `holder_did` and `author_did`, initial `proficiency: 0.0`, and `practice_count: 0`, and sets `derived_from` to Agent A's original Skill grain content address. The `origin_did` field preserves Agent A's DID for provenance.
4. Agent B improves proficiency through practice, superseding the skill grain as described above.

**Skill discovery:**

Agents discover available skills by querying: `type=skill, namespace="agent:skills", system_valid_to=null, sort=proficiency DESC`.

To find agents that hold a specific skill: `type=skill, name="code_review", system_valid_to=null`.

> **Note — promotion realized:** Earlier OMS revisions modeled skills as a `Fact + mg:capable_of` convention and reserved a dedicated Skill grain type (`0x0B`) as a future promotion path, conditioned on skill queries becoming performance-critical across multi-agent deployments. OMS 1.4 realizes that promotion: Skill is now a first-class grain type (§8.11) with its own required fields and type byte, following the precedent set by Consent (§8.10).

### 28.9 CAL and SML — Companion Query and Markup Languages

The query conventions in this section (§28.1–§28.7) define OMS store operations and response envelopes at the structural level. The **Context Assembly Language (CAL)** ([CONTEXT-ASSEMBLY-LANGUAGE-CAL-SPECIFICATION.md](./CONTEXT-ASSEMBLY-LANGUAGE-CAL-SPECIFICATION.md)) is the companion specification that provides a formal, deterministic syntax for invoking these operations from an agent or LLM.

**Relationship to §28.4 Store Protocol:**

CAL extends the store operations defined in §28.4 with a structured query language. Where §28.4 defines `query`, `search`, `get`, `put`, and `supersede` as abstract operations, CAL provides the syntax for expressing them safely — with built-in token-budget awareness, multi-source composition, and a type system tied to OMS grain types.

| §28.4 store operation | CAL statement |
|---|---|
| `query` + `search` | `RECALL <type> WHERE … LIMIT …` |
| `put` (new grain) | `ADD <type> SET field = value … REASON "…"` |
| `supersede` | `SUPERSEDE <hash> SET field = value … REASON "…"` |
| `query`/`search` + `get_batch` + compose | `ASSEMBLE … FROM … BUDGET <n> TOKENS` |
| introspection | `DESCRIBE <type>` |
| `delete` (compliance erasure) | `FORGET <hash>` / `FORGET SUBJECT` / `PURGE OLDER THAN` (Tier 2, authorization-gated, audited — CAL §8.14; no CAL equivalent on Tier-0/1 hosts) |

**SML output format:**

CAL `ASSEMBLE` statements produce **SML (Semantic Markup Language)** output by default. SML is a flat, tag-based markup format optimized for LLM consumption: tag names are OMS grain types (`<fact>`, `<goal>`, `<event>`, …), attributes carry lightweight metadata, and text content is natural language. See the [SML specification](./SEMANTIC-MARKUP-LANGUAGE-SML-SPECIFICATION.md) for the full format definition, structural rules, and progressive disclosure model. Implementations that expose a query layer SHOULD support CAL and produce SML output for agent context assembly.

---

## Appendix A: Domain Profile Registry

Domain Profiles allow implementers to extend the OMS field vocabulary with domain-specific fields while preserving core interoperability. A grain declares membership in a domain profile by including a `structural_tag` of the form `"profile:<name>"` (e.g., `"profile:healthcare"`). A grain MAY declare membership in multiple profiles.

**Rules for profile implementations:**
- Profile-specific field names MUST use the domain namespace prefix defined below.
- Profile fields that are required within the profile MUST be validated only when the profile tag is present; they are always optional in the absence of the profile tag.
- Profile fields MUST NOT conflict with core OMS field names (§6).
- Profile short keys for compaction MUST be registered with the OMS working group to avoid collisions.

### A.1 Healthcare Profile (`hc:`)

**Tag:** `"profile:healthcare"` | **Namespace prefix:** `hc:`

Applies to grains that handle Protected Health Information (PHI) under HIPAA, health records under HL7 FHIR, or clinical observations. Grains using this profile SHOULD also include `structural_tags` entries from the normative `phi:` tag set (§27.4) when applicable.

| Field | Type | Required | Description |
|---|---|---|---|
| `hc:patient_id` | string | when applicable | De-identified patient reference; MUST NOT be a direct identifier unless encryption is active |
| `hc:encounter_id` | string | no | HL7 FHIR Encounter resource ID |
| `hc:practitioner_did` | string | no | DID of the treating practitioner or ordering clinician |
| `hc:icd10` | string[] | no | ICD-10-CM diagnosis codes |
| `hc:cpt` | string[] | no | CPT procedure codes |
| `hc:loinc` | string | no | LOINC code for laboratory or clinical observations |
| `hc:snomed` | string | no | SNOMED CT concept identifier |
| `hc:fhir_resource` | string | no | FHIR resource type (e.g., `"Observation"`, `"Condition"`, `"MedicationRequest"`) |
| `hc:fhir_id` | string | no | FHIR resource ID on the source system |
| `hc:consent_ref` | string | no | Content address of the Consent grain authorizing this PHI grain |
| `hc:deidentification` | string | no | De-identification method applied: `"safe_harbor"` (45 CFR §164.514(b)) or `"expert_determination"` (45 CFR §164.514(a)) |

**Normative:** Grains with `"profile:healthcare"` and PHI content MUST set `processing_basis: "consent"` (or applicable legal basis) and MUST NOT set `license` to any open license value.

### A.2 Legal Profile (`legal:`)

**Tag:** `"profile:legal"` | **Namespace prefix:** `legal:`

Applies to grains that represent contracts, case law, regulatory filings, legal opinions, or compliance records.

| Field | Type | Required | Description |
|---|---|---|---|
| `legal:jurisdiction` | string | recommended | ISO 3166-1 alpha-2 country code or `"EU"`, `"UN"`, etc. |
| `legal:matter_id` | string | no | Internal matter or case docket identifier |
| `legal:document_type` | string | no | `"contract"`, `"opinion"`, `"filing"`, `"statute"`, `"regulation"`, `"order"`, `"brief"` |
| `legal:parties` | string[] | no | DID or identifier of each legal party |
| `legal:effective_date` | integer | no | Unix epoch ms; date on which the legal instrument takes effect |
| `legal:expiry_date` | integer | no | Unix epoch ms; date on which the legal instrument expires or is superseded |
| `legal:citation` | string | no | Formal legal citation string (e.g., `"42 U.S.C. § 1983"`) |
| `legal:privilege` | string | no | Privilege assertion: `"attorney_client"`, `"work_product"`, `"none"` |
| `legal:hold_ref` | string | no | Content address of the Invalidation grain placing this grain under litigation hold |
| `legal:redaction_level` | string | no | `"none"`, `"partial"`, `"full"` |

**Normative:** Grains with `legal:privilege: "attorney_client"` or `"work_product"` MUST have invalidation mode `"hold"` applied before any export or cross-system transfer. Implementations MUST NOT auto-erase held grains (even on GDPR erasure requests) without documented litigation hold lift.

### A.3 Finance Profile (`fin:`)

**Tag:** `"profile:finance"` | **Namespace prefix:** `fin:`

Applies to grains that represent financial transactions, market observations, risk assessments, or regulatory filings (SOX, MiFID II, etc.).

| Field | Type | Required | Description |
|---|---|---|---|
| `fin:account_id` | string | no | Obfuscated or tokenized account reference |
| `fin:instrument_id` | string | no | ISIN, CUSIP, FIGI, or other instrument identifier |
| `fin:ticker` | string | no | Exchange ticker symbol |
| `fin:amount` | number | no | Transaction amount |
| `fin:currency` | string | no | ISO 4217 three-letter currency code |
| `fin:transaction_type` | string | no | `"debit"`, `"credit"`, `"transfer"`, `"fee"`, `"trade"`, `"settlement"` |
| `fin:market_timestamp` | integer | no | Exchange-provided timestamp in Unix epoch ms |
| `fin:venue` | string | no | Trading venue MIC code (ISO 10383) |
| `fin:strategy_id` | string | no | Quantitative strategy or model identifier |
| `fin:risk_score` | number | no | Normalized risk score [0.0–1.0] |
| `fin:sox_control_id` | string | no | SOX internal control identifier for audit trail linkage |
| `fin:retention_years` | integer | no | Regulatory retention requirement in years (overrides default retention policy) |

**Normative:** Grains with `"profile:finance"` that contain personally identifiable financial information MUST NOT be exported without `processing_basis` set and without applicable consent or contractual basis documented.

### A.4 Robotics Profile (`rob:`)

**Tag:** `"profile:robotics"` | **Namespace prefix:** `rob:`

Applies to grains produced by or about embodied robotic systems operating in physical environments.

| Field | Type | Required | Description |
|---|---|---|---|
| `rob:robot_id` | string | recommended | Unique robot platform identifier (URI or DID) |
| `rob:pose` | object | no | `{x, y, z, roll, pitch, yaw}` in the robot's reference frame |
| `rob:velocity` | object | no | `{vx, vy, vz}` in m/s |
| `rob:map_id` | string | no | Identifier of the map or environment model in use |
| `rob:mission_id` | string | no | Identifier of the current mission or task |
| `rob:battery_pct` | number | no | Battery charge at observation time [0.0–100.0] |
| `rob:safety_state` | string | no | `"normal"`, `"warning"`, `"emergency_stop"`, `"recovery"` |
| `rob:hardware_rev` | string | no | Robot hardware revision string |
| `rob:firmware_ver` | string | no | Firmware version string |
| `rob:contact_forces` | object | no | Force/torque sensor readings at contact points |
| `rob:coordinate_frame` | string | no | Reference frame identifier (e.g., `"world"`, `"odom"`, `"base_link"`) |

### A.5 Science Profile (`sci:`)

**Tag:** `"profile:science"` | **Namespace prefix:** `sci:`

Applies to grains produced in scientific research workflows — experiments, datasets, findings, replication records.

| Field | Type | Required | Description |
|---|---|---|---|
| `sci:doi` | string | no | Digital Object Identifier for the source publication or dataset |
| `sci:arxiv_id` | string | no | arXiv preprint identifier (e.g., `"2501.00123"`) |
| `sci:pmid` | string | no | PubMed article identifier |
| `sci:dataset_id` | string | no | Dataset identifier (DOI, Zenodo, Figshare, etc.) |
| `sci:experiment_id` | string | no | Local experiment or trial identifier |
| `sci:protocol_id` | string | no | Protocol identifier or URL (e.g., protocols.io DOI) |
| `sci:hypothesis` | string | no | Free-text hypothesis being tested |
| `sci:result_status` | string | no | `"positive"`, `"negative"`, `"inconclusive"`, `"replicated"`, `"failed_replication"` |
| `sci:p_value` | number | no | Statistical p-value of the result [0.0–1.0] |
| `sci:effect_size` | number | no | Standardized effect size (Cohen's d, r, etc.) |
| `sci:sample_size` | integer | no | Number of subjects or samples |
| `sci:preregistered` | boolean | no | Whether the study was pre-registered (e.g., on OSF, AsPredicted) |
| `sci:open_access` | boolean | no | Whether the source is open access |

### A.6 Consumer Profile (`con:`)

**Tag:** `"profile:consumer"` | **Namespace prefix:** `con:`

Applies to grains produced in consumer-facing agent contexts — personal assistants, recommendation systems, preference learning, and lifestyle applications.

| Field | Type | Required | Description |
|---|---|---|---|
| `con:device_type` | string | no | `"mobile"`, `"desktop"`, `"smart_speaker"`, `"wearable"`, `"tv"`, `"kiosk"` |
| `con:app_id` | string | no | Application or product identifier |
| `con:app_version` | string | no | Application version string |
| `con:locale` | string | no | BCP 47 language tag (e.g., `"en-US"`, `"fr-FR"`) |
| `con:preference_category` | string | no | Domain of the preference (e.g., `"music"`, `"food"`, `"news"`, `"shopping"`) |
| `con:interaction_type` | string | no | `"explicit_feedback"`, `"implicit_signal"`, `"purchase"`, `"skip"`, `"save"`, `"share"` |
| `con:sentiment` | number | no | Sentiment score [-1.0 = very negative, 1.0 = very positive] |
| `con:engagement_duration_ms` | integer | no | Duration of user engagement with the referenced content in milliseconds |
| `con:recommendation_rank` | integer | no | Position in a recommendation list that triggered the interaction |
| `con:ab_variant` | string | no | A/B test variant identifier |
| `con:ccpa_opted_out` | boolean | no | User has exercised CCPA opt-out of sale; MUST NOT be used as a processing basis — use `processing_basis` field instead |

**Normative:** Grains with `"profile:consumer"` that include `user_id` or any direct identifier MUST set `processing_basis` to a lawful basis under GDPR Art. 6 / CCPA § 1798.100 before cross-system transfer. Grains with `con:ccpa_opted_out: true` MUST NOT be included in data sale or data broker transfers.

### A.7 Integration Profile (`int:`)

**Tag:** `"profile:integration"` | **Namespace prefix:** `int:`

Applies to grains that represent REST API connectors, tool catalog entries, webhook definitions, or integration platform action registries. Integration profile fields are stored in the grain's `context` map (compact key: `ctx`), following the same pattern as other domain profiles.

| Field | Type | Required | Description |
|---|---|---|---|
| `int:base_url` | string | no | API base URL (e.g., `"https://api.github.com"`) |
| `int:http_method` | string | no | HTTP method: `"GET"`, `"POST"`, `"PUT"`, `"PATCH"`, `"DELETE"` |
| `int:http_path` | string | no | URL path template with `{param}` placeholders (e.g., `"/repos/{owner}/{repo}/issues"`) |
| `int:path_params` | string[] | no | Parameter names extracted from path template |
| `int:query_params` | string[] | no | Query parameter names |
| `int:body_params` | string[] | no | Body parameter names (for POST/PUT/PATCH) |
| `int:response_mapping` | string | no | JQ-compatible expression for response transformation (e.g., `".data.items"`) |
| `int:auth_type` | string | no | Auth mechanism: `"api_key"`, `"api_key:bearer"`, `"api_key:header"`, `"oauth2"`, `"basic"`, `"jwt"`, `"none"`. Open enum — implementations MAY define additional values (e.g., `"aws_sigv4"`, `"mtls"`) |
| `int:auth_scopes` | string[] | no | Required OAuth scopes (e.g., `["repo", "read:org"]`) |
| `int:read_only` | boolean | no | `true` if action does not mutate external state |
| `int:connector` | string | no | Parent connector slug (e.g., `"github"`, `"stripe"`) |
| `int:docs_url` | string | no | Documentation URL for this action or connector |
| `int:rate_limit` | integer | no | Advisory maximum requests per minute; enforcement is an implementation concern |
| `int:category` | string | no | Connector category (e.g., `"dev-tools"`, `"crm"`, `"communication"`) |
| `int:sunset_date` | string | no | ISO 8601 date when this action will be removed |
| `int:content_type` | string | no | Request content type if non-default (e.g., `"application/x-www-form-urlencoded"`) |

**Trigger-specific fields** (connector transport configuration on a Trigger grain's `config` map, §8.13):

| Field | Type | Used By | Description |
|---|---|---|---|
| `int:poll_interval_secs` | integer | polling | Seconds between polls |
| `int:cursor_field` | string | polling | Field name for incremental fetching (e.g., `"since"`, `"last_id"`) |
| `int:cursor_type` | string | polling | Cursor type: `"timestamp"`, `"id"`, `"etag"` |
| `int:webhook_path` | string | webhook | Inbound webhook receiver path |
| `int:webhook_secret_header` | string | webhook | Header containing HMAC signature |
| `int:cron_expression` | string | schedule | Cron expression (e.g., `"0 9 * * MON-FRI"`) |
| `int:timezone` | string | schedule | IANA timezone (e.g., `"America/New_York"`) |
| `int:config_schema` | map | all | JSON Schema for trigger configuration |
| `int:event_schema` | map | all | JSON Schema for emitted events |
| `int:allowed_outbound_hosts` | string[] | all | URL prefixes a connector may reach. Scheme, host and port all take part in the match; `*.example.com` covers subdomains but not the apex. An absent key is unrestricted; an empty array denies everything — the two are different statements. A bare `*` SHOULD be refused: it lets a declaration appear policed while permitting every destination |

**Normative:**
- Grains with `"profile:integration"` SHOULD include `int:connector` and `int:auth_type`.
- `int:http_path` parameters MUST match entries in `int:path_params`.
- `int:response_mapping` MUST be a valid JQ expression if present.
- `int:rate_limit` is advisory only — enforcement is an implementation concern.

**Example — Tool definition with integration profile:**

```json
{
  "type": "tool",
  "tool_phase": "definition",
  "tool_name": "github:create-issue",
  "tool_description": "Create a new issue in a GitHub repository",
  "input_schema": {
    "type": "object",
    "properties": {
      "owner": {"type": "string"},
      "repo": {"type": "string"},
      "title": {"type": "string"},
      "body": {"type": "string"},
      "labels": {"type": "array", "items": {"type": "string"}}
    },
    "required": ["owner", "repo", "title"]
  },
  "output_schema": {
    "type": "object",
    "properties": {
      "id": {"type": "integer"},
      "number": {"type": "integer"},
      "html_url": {"type": "string"}
    }
  },
  "structural_tags": ["profile:integration"],
  "context": {
    "int:base_url": "https://api.github.com",
    "int:http_method": "POST",
    "int:http_path": "/repos/{owner}/{repo}/issues",
    "int:path_params": ["owner", "repo"],
    "int:body_params": ["title", "body", "labels"],
    "int:auth_type": "api_key:bearer",
    "int:read_only": false,
    "int:connector": "github",
    "int:category": "dev-tools"
  },
  "namespace": "axtion:connectors:github",
  "created_at": 1740700000000
}
```

---

## Appendix B: ABNF Grammar

```abnf
mg-blob       = version-byte header-fields msgpack-payload
version-byte  = %x01
header-fields = flags-byte type-byte ns-hash-bytes created-at-bytes
                ; version-byte + header-fields = 9-byte "fixed header" in §3.1
flags-byte    = %x00-FF
type-byte     = %x01-0C / %xF0-FF
                ; Fact=0x01, Event=0x02, State=0x03, Workflow=0x04, Tool=0x05,
                ; Observation=0x06, Goal=0x07, Reasoning=0x08, Consensus=0x09,
                ; Consent=0x0A, Skill=0x0B, Recommendation=0x0C,
                ; 0x0D-0xEF reserved, 0xF0-0xFF domain profile types
ns-hash-bytes = 2OCTET  ; uint16 big-endian, first two bytes of SHA-256(namespace)
created-at-bytes = 4OCTET  ; uint32 big-endian

msgpack-payload = canonical-map
canonical-map = fixmap / map16 / map32
fixmap        = %x80-8F *key-value
map16         = %xDE uint16 *key-value
map32         = %xDF uint32 *key-value

key-value     = msgpack-string msgpack-value
msgpack-string = fixstr / str8 / str16 / str32  ; UTF-8 NFC-normalized
msgpack-value = msgpack-string / msgpack-int / msgpack-float
              / msgpack-bool / msgpack-array / canonical-map
              / msgpack-null  ; but nulls MUST be omitted from maps

content-address = 64 HEXDIG

mg-file       = magic flags grain-count field-map-ver compression-type
                reserved offset-table grains footer
magic         = "MG" %x01
flags         = %x00-FF
grain-count   = 4OCTET  ; uint32
field-map-ver = %x00-FF
compression-type = %x00-FF
reserved      = 6OCTET
offset-table  = *4OCTET  ; grain_count × uint32
grains        = *mg-blob
footer        = 32OCTET  ; SHA-256 checksum
```

---

## Appendix C: Field Mapping Table (Compact Reference)

**Core & Multi-Modal Fields:**

```json
{
  "t": "type",
  "s": "subject",
  "r": "relation",
  "o": "object",
  "c": "confidence",
  "st": "source_type",
  "ca": "created_at",
  "tt": "temporal_type",
  "vf": "valid_from",
  "vt": "valid_to",
  "svf": "system_valid_from",
  "svt": "system_valid_to",
  "ctx": "context",
  "sb": "superseded_by",
  "ct": "contradicted",
  "im": "importance",
  "adid": "author_did",
  "ns": "namespace",
  "user": "user_id",
  "tags": "structural_tags",
  "df": "derived_from",
  "cl": "consolidation_level",
  "sc": "success_count",
  "fc": "failure_count",
  "pc": "provenance_chain",
  "odid": "origin_did",
  "ons": "origin_namespace",
  "cr": "content_refs",
  "er": "embedding_refs",
  "rt": "related_to",
  "_e": "_elided",
  "_do": "_disclosure_of",
  "ip": "invalidation_policy",
  "sj": "supersession_justification",
  "sa": "supersession_auth",
  "own": "owner",
  "cat": "category",
  "rid": "run_id",
  "role": "role",
  "ac": "access_count",
  "laa": "last_accessed_at",
  "tms": "timestamp_ms",
  "obsdid": "observer_did",
  "sdid": "subject_did",
  "gdid": "grantee_did",
  "sid2": "session_id",
  "eid": "entity_id",
  "epstat": "epistemic_status",
  "vstatus": "verification_status",
  "rstat": "rec_status",
  "rhr": "requires_human_review",
  "pbasis": "processing_basis",
  "idst": "identity_state",
  "lic": "license",
  "tts": "trusted_timestamp",
  "itype": "invalidation_type",
  "ireason": "invalidation_reason",
  "iinit": "invalidation_initiator",
  "rpol": "retention_policy",
  "rpri": "recall_priority",
  "et": "embedding_text",
  "scope": "scope",
  "isw": "is_withdrawal",
  "basis": "basis",
  "jur": "jurisdiction",
  "pcon": "prior_consent",
  "wdids": "witness_dids",
  "prem": "premises",
  "conc": "conclusion",
  "imethod": "inference_method",
  "altc": "alternatives_considered",
  "statctx": "statistical_context",
  "swenv": "software_environment",
  "params": "parameter_set",
  "rseed": "random_seed"
}
```

**Tool-Specific Fields:**

```json
{
  "aphase": "tool_phase",
  "tn": "tool_name",
  "inp": "input",
  "cnt": "content",
  "iserr": "is_error",
  "tcid": "tool_call_id",
  "cbid": "call_batch_id",
  "ttype": "tool_type",
  "tver": "tool_version",
  "emode": "execution_mode",
  "code": "code",
  "out": "stdout",
  "err2": "stderr",
  "xc": "exit_code",
  "iid": "interpreter_id",
  "err": "error",
  "etype": "error_type",
  "dur": "duration_ms",
  "ptid": "parent_task_id",
  "tdesc": "tool_description",
  "isch": "input_schema",
  "osch": "output_schema",
  "strict": "strict"
}
```

**Consensus-Specific Fields:**

```json
{
  "pobs": "participating_observers",
  "thold": "threshold",
  "agcnt": "agreement_count",
  "discnt": "dissent_count",
  "disgrn": "dissent_grains",
  "agcon": "agreed_content"
}
```

**Observation-Specific Fields:**

```json
{
  "oid": "observer_id",
  "otype": "observer_type",
  "fid": "frame_id",
  "sg": "sync_group",
  "omode": "observation_mode",
  "oscope": "observation_scope",
  "omdl": "observer_model",
  "ocmp": "compression_ratio"
}
```

**Goal-Specific Fields:**

```json
{
  "desc": "description",
  "gs": "goal_state",
  "crit": "criteria",
  "crs": "criteria_structured",
  "pri": "priority",
  "pgs": "parent_goals",
  "sr": "state_reason",
  "se": "satisfaction_evidence",
  "prog": "progress",
  "dto": "delegate_to",
  "dfo": "delegate_from",
  "ep": "expiry_policy",
  "rec": "recurrence",
  "evreq": "evidence_required",
  "rof": "rollback_on_failure",
  "atr": "allowed_transitions"
}
```

**Skill-Specific Fields:**

```json
{
  "skname": "name",
  "instr": "instructions",
  "wtu": "when_to_use",
  "sver": "version",
  "atls": "allowed_tools",
  "res": "resources",
  "deps": "dependencies",
  "imod": "input_modalities",
  "omod": "output_modalities",
  "dom": "domain",
  "hdid": "holder_did",
  "prof": "proficiency",
  "prcnt": "practice_count",
  "lpa": "last_practiced_at",
  "strat": "strategies",
  "xfer": "transferable"
}
```

> **Note:** `description` (`desc`) is shared with the Goal-specific table above; Skill grains reuse the same short key. Holder identity uses the Skill-specific `holder_did` (`hdid`), distinct from the common `author_did` (`adid`) which records the grain's writer. Bundled file payloads use the common `content_refs` (`cr`) field.

**Recommendation-Specific Fields:**

```json
{
  "tref": "target_ref",
  "anlz": "analyzer",
  "summ": "summary",
  "ddk": "dedup_key",
  "pcal": "proposal_cal",
  "pedit": "proposal_edit",
  "pdata": "proposal_data",
  "sev": "severity",
  "msnap": "metric_snapshot",
  "evq": "evidence_query"
}
```

> **Note:** Evidence and scoring reuse common fields — `derived_from` (`df`), `confidence` (`c`), `importance` (`im`), `valid_to` (`vt`). The review state (`rec_status`) is an index-layer field compacted with the core fields above as `rstat`, not a Recommendation-specific blob field; it is rebuilt from the audit chain (§8.12.1).

**Content Reference Nested Compaction:**

```json
{
  "u": "uri",
  "m": "modality",
  "mt": "mime_type",
  "sz": "size_bytes",
  "ck": "checksum",
  "md": "metadata"
}
```

**Embedding Reference Nested Compaction:**

```json
{
  "vi": "vector_id",
  "mo": "model",
  "dm": "dimensions",
  "ms": "modality_source",
  "di": "distance_metric"
}
```

**Related-To Nested Compaction:**

```json
{
  "h": "hash",
  "rl": "relation_type",
  "w": "weight"
}
```

**Integration Profile Fields (stored in `context` map):**

```json
{
  "ib": "int:base_url",
  "ihm": "int:http_method",
  "ihp": "int:http_path",
  "ipp": "int:path_params",
  "iqp": "int:query_params",
  "ibp": "int:body_params",
  "irm": "int:response_mapping",
  "iat": "int:auth_type",
  "ias": "int:auth_scopes",
  "iro": "int:read_only",
  "ic": "int:connector",
  "idu": "int:docs_url",
  "irl": "int:rate_limit",
  "icat": "int:category",
  "isd": "int:sunset_date",
  "ict": "int:content_type",
  "ipis": "int:poll_interval_secs",
  "icf": "int:cursor_field",
  "icft": "int:cursor_type",
  "iwp": "int:webhook_path",
  "iwsh": "int:webhook_secret_header",
  "icron": "int:cron_expression",
  "itz": "int:timezone",
  "icfg": "int:config_schema",
  "ievt": "int:event_schema",
  "iaoh": "int:allowed_outbound_hosts"
}
```

**Trigger-Specific Fields (§8.13):**

```json
{
  "aknd": "kind",
  "twf": "workflow",
  "tcon": "connector",
  "tscp": "scope",
  "tena": "enabled",
  "tdk": "dedup_key",
  "tint": "interval_secs",
  "tcron": "cron",
  "tat": "at_ms",
  "tpred": "predicate",
  "tmem": "members",
  "tcorr": "correlate",
  "twin": "window_ms",
  "tconc": "concurrency",
  "tcatch": "catchup",
  "tcfg": "config"
}
```

---

## Appendix D: Compliance Mapping

### GDPR

| Article | .mg Support |
|---------|------------|
| Art. 5 (Data minimization) | `user_id` field enables per-person scope |
| Art. 12-23 (Rights) | Structured data format for automated response |
| Art. 17 (Erasure) | Crypto-erasure via key destruction; subject-scoped erasure expressible as CAL `FORGET SUBJECT` on Tier-2 hosts (§28.4, CAL §8.14) |
| Art. 25 (Privacy by design) | Provenance and audit built-in |
| Art. 30 (Records of processing) | `provenance_chain` and `created_at` timestamps support records-of-processing obligations |
| Art. 32 (Security) | COSE signing, AES-256-GCM encryption |

### HIPAA (45 CFR §164)

| Section | .mg Support |
|---------|------------|
| §164.308 (Administrative) | Audit trail via `provenance_chain` |
| §164.310 (Physical) | N/A (transport layer) |
| §164.312 (Technical) | AES-256-GCM encryption, COSE signatures |
| §164.314 (Organizational) | N/A (policy engine) |

### CCPA

| Requirement | .mg Support |
|------------|------------|
| Personal information collection | `user_id` and `structural_tags` for classification |
| Disclosure | Selective disclosure hides sensitive fields |
| Deletion | Crypto-erasure via key destruction |
| Opt-out | Policy-layer enforcement (outside .mg) |

---

## Appendix E: Version History

See [CHANGELOG.md](CHANGELOG.md) for the full version history.


---

---

## Appendix F: Glossary

- **Blob:** Complete .mg binary (9-byte fixed header + MessagePack payload)
- **Grain:** Atomic knowledge unit; identified by content address
- **Content address:** SHA-256 hash of blob bytes; unique identifier
- **Canonical:** Deterministic serialization rules ensuring identical bytes
- **DID:** W3C decentralized identifier; cryptographic identity without CA
- **COSE:** CBOR Object Signing and Encryption (RFC 9052)
- **Selective disclosure:** Hiding some fields while proving they exist
- **Provenance:** Derivation trail showing how grain was created
- **Cross-link:** Semantic relationship between grains
- **Bi-temporal:** Tracking both event-time and system-time dimensions
- **Fact:** Grain type 0x01 — a held claim, factual statement, or declarative knowledge about the world
- **Event:** Grain type 0x02 — a discrete occurrence with start/end time
- **State:** Grain type 0x03 — a persisting condition or status at a point in time
- **Workflow:** Grain type 0x04 — a directed graph of procedural steps, supporting sequential, conditional, parallel, and cyclic execution paths
- **Tool:** Grain type 0x05 — a completed tool invocation, API call, or agent action
- **Observation:** Grain type 0x06 — a raw sensor or environmental reading without interpretation
- **Goal:** Grain type 0x07 — a desired future state or objective
- **Reasoning:** Grain type 0x08 — an inference chain, chain-of-thought, or decision rationale
- **Consensus:** Grain type 0x09 — an agreement reached among multiple agents or sources
- **Consent:** Grain type 0x0A — a data subject's GDPR/CCPA/LGPD/PIPL consent or withdrawal record
- **Skill:** Grain type 0x0B — a packaged, reusable agent capability: a definition (instructions, when-to-use, tools, resources, dependencies) with optional learned-competence fields (proficiency, practice count, strategies)
- **Recommendation:** Grain type 0x0C — a governed, auditable proposal to change memory or agent configuration: a target, the producing analyzer, a deterministic summary, and one proposal (CAL evolve-batch, doc edit, or host-config change). Enters a propose → review → apply → roll-back lifecycle whose transitions are immutable audit Observation grains; its review state is an index-layer cache
- **processing_basis:** Legal basis for processing personal data under GDPR Art. 6 (consent, contract, legal_obligation, vital_interests, public_task, legitimate_interests)
- **consent_cascade:** Invalidation mode that propagates erasure/restriction to all grains linked via `processing_basis` when a Consent grain is invalidated
- **verification_status:** Lifecycle verification state of a grain's content: `"unverified"` (default — not yet reviewed), `"verified"` (confirmed correct by an authority), `"contested"` (contradicted or disputed), `"retracted"` (withdrawn from use)
- **run_id:** Session or execution scope identifier; distinct from user_id (data subject) and namespace (logical partition)
- **Crypto-erasure:** Destroying encryption key to unrecoverably erase data
- **Blind index:** HMAC token for searching encrypted data without decryption

---

## Appendix G: Complete Example Grain

```python
# Create a fact grain
grain = {
    "type": "fact",
    "subject": "machine-learning",
    "relation": "is_subset_of",
    "object": "artificial-intelligence",
    "confidence": 0.99,
    "epistemic_status": "accepted",
    "source_type": "user_explicit",
    "created_at": 1737000000000,
    "timestamp_ms": 1737000000000,
    "namespace": "knowledge-base",
    "author_did": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
    "user_id": "researcher-alice",
    "importance": 0.95,
    "structural_tags": ["ai", "ml", "education"],
    "context": {"source": "textbook", "chapter": "1.2"},
    "provenance_chain": [
        {"source_hash": "abc123...", "method": "direct_input", "weight": 1.0}
    ],
    "related_to": [
        {
            "hash": "def456...",
            "relation_type": "elaborates",
            "weight": 0.8
        }
    ]
}

# Serialize to .mg blob (9-byte fixed header, version byte 0x01)
# 1. Compact field names
# 2. Omit null values
# 3. NFC-normalize strings
# 4. Sort keys lexicographically
# 5. Encode as canonical MessagePack
# 6. Prepend 9-byte fixed header: version(1) + flags(1) + type(1) + ns_hash(2) + created_at(4)
#    type byte = 0x01 (Fact)
# 7. Compute SHA-256 hash

blob = serialize(grain)
content_address = sha256(blob).hex()

# Result: 64-character lowercase hex string
# Example: 3a1f5d8e9c2b7a4f6e9d2c8b1a4f7e9d2c8b1a4f7e9d2c8b1a4f7e9d2c8b1a4f
```

---

**Document Status:** This is the v1.5 revision of the .mg format specification. It adds the dedicated **Recommendation** grain type (`0x0C`, §8.12) — a governed, auditable proposal to change memory or agent configuration, with a supersession-based content model and an audit-grain lifecycle — realized from the reserved range following the Skill (`0x0B`) precedent; byte values `0x01`–`0x0B` are unchanged, so existing content addresses remain valid. (The prior v1.4 revision renamed `belief`→`fact` (`0x01`) and `action`→`tool` (`0x05`), redesigned the Workflow grain as a directed graph, added the `embedding_text` common field, and added the Skill grain type (`0x0B`, §8.11).) Submitted as a standards track document for consideration as an IETF RFC and W3C standard. Community feedback is encouraged through issue tracking and discussion forums.

**Last Updated:** 2026-08-03
**License:** This document is offered under the Open Web Foundation Final Specification Agreement (OWFa 1.0)
