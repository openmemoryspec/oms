# Open Memory Specification (OMS)
## Memory Grain (.mg) Container Definition

**Version:** 1.1
**Status:** Standards Track
**Category:** Data Formats
**Date:** February 2026
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
8. [Memory Types](#memory-types)
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
- Index layer queries and optimization
- Policy engines and compliance rule evaluation
- Transport protocols (HTTP, MQTT, Kafka)
- Encryption at rest (applications of per-grain encryption are external to this spec)
- Agent-to-agent communication protocol (which uses .mg format)

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

**Byte 2 — Type (memory type enum):**

| Value | Type |
|-------|------|
| 0x01 | Fact |
| 0x02 | Episode |
| 0x03 | Checkpoint |
| 0x04 | Workflow |
| 0x05 | ToolCall |
| 0x06 | Observation |
| 0x07 | Goal |
| 0x08-0xEF | Reserved for future standard types |
| 0xF0-0xFF | Application-defined types |

**Bytes 3-4 — Namespace Hash:** First two bytes of SHA-256(namespace), encoded as `uint16` big-endian. Provides 65,536 routing buckets without deserialization. Full namespace string remains authoritative in payload. This field is a routing hint only and MUST NOT be used for security decisions (see §13.3, §20).

**Bytes 5-8 — Created-at:** `uint32` epoch seconds (1970-01-01 onwards). Range: 1970 to 2106. Full millisecond precision in payload field.

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

> **Note — `source_type` for Observation grains:** Use `"sensor"` when `observer_type` is a physical instrument; `"agent_inferred"` when `observer_type` is a cognitive AI observer (`"llm"`, `"reflector"`, `"classifier"`, `"detector"`); `"user_explicit"` for human observers.

### 6.2 Episode-Specific Fields

| Full Name | Short Key | Type |
|-----------|-----------|------|
| `content` | `content` | string |
| `consolidated` | `consolidated` | bool |

### 6.3 Checkpoint-Specific Fields

| Full Name | Short Key | Type |
|-----------|-----------|------|
| `plan` | `plan` | array[string] |
| `history` | `history` | array[map] |

### 6.4 Workflow-Specific Fields

| Full Name | Short Key | Type |
|-----------|-----------|------|
| `steps` | `steps` | array[string] |
| `trigger` | `trigger` | string |

### 6.5 ToolCall-Specific Fields

| Full Name | Short Key | Type |
|-----------|-----------|------|
| `tool_name` | `tn` | string |
| `arguments` | `args` | map |
| `result` | `res` | any |
| `success` | `ok` | bool |
| `error` | `err` | string |
| `duration_ms` | `dur` | int |
| `parent_task_id` | `ptid` | string |

### 6.6 Observation-Specific Fields

| Full Name | Short Key | Type | Deprecated Full Name (v1.0) | Deprecated Short Key (v1.0) |
|-----------|-----------|------|-----------------------------|-----------------------------|
| `observer_id` | `oid` | string | `sensor_id` | `sid` |
| `observer_type` | `otype` | string | `sensor_type` | `stype` |
| `frame_id` | `fid` | string | — | — |
| `sync_group` | `sg` | string | — | — |
| `observation_mode` | `omode` | string | — (new in v1.1) | — |
| `observation_scope` | `oscope` | string | — (new in v1.1) | — |
| `observer_model` | `omdl` | string | — (new in v1.1) | — |
| `compression_ratio` | `ocmp` | float64 | — (new in v1.1) | — |

**Backward compatibility (v1.1):** Readers MUST accept the deprecated short keys `sid` and `stype` and map them to `observer_id` and `observer_type` respectively. Writers MUST emit `oid` and `otype` and MUST NOT emit `sid` or `stype`. The deprecated aliases will be removed in v2.0. All existing v1.0 Observation grains remain valid under v1.1 readers without any migration of stored data.

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

### 6.8 Compaction Rules

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

## 8. Memory Types

Each grain MUST contain a `type` field indicating its memory type. Seven standard types are defined:

### 8.1 Fact

Structured knowledge claim with confidence and temporal validity. Modeled as semantic triple (subject-relation-object).

**Required fields:**
- `type` = "fact"
- `subject` (non-empty string)
- `relation` (non-empty string)
- `object` (non-empty string)
- `confidence` (float64, [0.0, 1.0])
- `source_type` (non-empty string)
- `created_at` (int64, epoch ms)

**Optional fields:**
- `temporal_type` ("state" | "observation", default "observation")
- `valid_from`, `valid_to` (int64, epoch ms) — when fact was true in real world
- `system_valid_from`, `system_valid_to` (int64) — when fact was active in system
- `context` (map string→string) — contextual metadata
- `superseded_by` (string, content address of superseding grain)
- `contradicted` (bool, default false)
- `importance` (float64, [0.0, 1.0], default 0.7)
- `author_did` (string) — DID of creating agent
- `namespace` (string, default "shared")
- `user_id` (string) — data subject identifier
- `structural_tags` (array[string]) — classification tags
- `derived_from` (array[string]) — parent content addresses
- `consolidation_level` (int, [0, 3])
- `success_count`, `failure_count` (int, non-negative)
- `origin_did` (string) — original source agent DID
- `origin_namespace` (string) — original source namespace
- `provenance_chain` (array[map]) — derivation trail
- `content_refs` (array[map]) — references to external content
- `embedding_refs` (array[map]) — references to embeddings
- `related_to` (array[map]) — cross-links to related grains

**Provenance chain entry:**
```json
{
  "source_hash": "abc123...",
  "method": "frequency_consolidation",
  "weight": 0.8
}
```

**RDF mapping:** `<grain:subject> <grain:relation> "grain:object" .`

**Design note:** The Fact type intentionally serves as a broad knowledge representation primitive, accommodating optional fields for temporal validity, bi-temporal system tracking, consolidation history, confidence, and cross-links. This allows simple grains (subject, relation, object, confidence) to coexist with richly annotated ones in the same store without separate type hierarchies. Implementations SHOULD populate only the fields appropriate to their use case and treat unused optional fields as absent. Future versions may introduce specialized types for semantic patterns (e.g., goals, learned models) that require distinct lifecycle semantics not expressible as Fact optional fields.

### 8.2 Episode

Raw, unstructured interaction record. Episodes are input to consolidation, which extracts structured Facts.

**Required fields:**
- `type` = "episode"
- `content` (non-empty string) — raw text of episode
- `created_at` (int64, epoch ms)

**Optional fields:**
- `user_id` (string)
- `author_did` (string)
- `namespace` (string, default "shared")
- `importance` (float64, [0.0, 1.0], default 0.5)
- `consolidated` (bool, default false)
- `structural_tags` (array[string])
- `content_refs` (array[map])

### 8.3 Checkpoint

Agent state snapshot for save/restore.

**Required fields:**
- `type` = "checkpoint"
- `context` (map) — agent state snapshot
- `created_at` (int64, epoch ms)

**Optional fields:**
- `plan` (array[string]) — planned actions
- `history` (array[map]) — action history
- `user_id` (string)
- `structural_tags` (array[string])

### 8.4 Workflow

Procedural memory — learned sequence of actions.

**Required fields:**
- `type` = "workflow"
- `steps` (non-empty array[string])
- `trigger` (non-empty string)
- `created_at` (int64, epoch ms)

**Optional fields:**
- `importance` (float64, [0.0, 1.0], default 0.7)
- `namespace` (string, default "shared")

### 8.5 ToolCall

Record of tool/function invocation and result.

**Required fields:**
- `type` = "tool_call"
- `tool_name` (non-empty string)
- `arguments` (map)
- `result` (any MessagePack-serializable value)
- `success` (bool)
- `created_at` (int64, epoch ms)

**Optional fields:**
- `error` (string) — error message if success=false
- `duration_ms` (int) — execution time
- `author_did` (string)
- `user_id` (string)
- `namespace` (string, default "shared")
- `parent_task_id` (string) — parent task content address
- `structural_tags` (array[string])
- `content_refs` (array[map])

### 8.6 Observation

Sensor reading, environmental measurement, or cognitive observation. Captures what an observer — a physical instrument, an AI agent, or a human — perceived at a moment in time, and with what certainty. Designed for high-volume, time-critical data with optional spatial or contextual framing.

**Epistemological note:** An Observation is `"I (observer) perceived X"` — anchored to a specific perceiver, moment, and method. This is distinct from a Fact, which is `"X is true"` — a knowledge claim derived from one or more observations. An LLM that hallucinates can produce a technically valid Observation grain (the observer genuinely produced that output) while the derived Fact is false. Keeping these as separate grain types preserves the full audit chain: `raw input → Observation → Fact → Goal`. See Section 24 for observer type taxonomy.

**Required fields:**
- `type` = "observation"
- `observer_id` (non-empty string) — unique identifier of the observing entity. For physical sensors: device serial number or path (e.g., `"robot-arm-7/lidar-front"`). For AI agents: the agent's DID (e.g., `"did:key:z6Mk..."`). For humans: a pseudonymous or DID-based identifier.
- `observer_type` (non-empty string) — category of observer. See Observer Type Registry (Section 24). Open enum; applications may define custom values.
- `created_at` (int64, epoch ms)

**Optional fields:**
- `subject` (string) — the entity being observed
- `object` (string) — the observed value, measurement, or summary
- `confidence` (float64, [0.0, 1.0])
- `frame_id` (string) — the reference frame for interpreting this observation. For physical observers: spatial coordinate frame (e.g., `"map"`, `"base_link"`, `"odom"`). For cognitive observers: contextual frame such as a thread ID (`"thread:conv-abc123"`), task label, or session identifier. In both cases this field answers: "relative to what context should this observation be interpreted?"
- `sync_group` (string) — temporal alignment group. All observations sharing a `sync_group` value SHOULD be interpreted together at the same logical moment. For physical: multi-sensor fusion (camera + lidar + IMU at same timestamp). For cognitive: multi-agent concurrent observations of the same event.
- `namespace` (string, default "shared")
- `author_did` (string)
- `importance` (float64, [0.0, 1.0], default 0.3)
- `structural_tags` (array[string])
- `content_refs` (array[map]) — references to raw data: point clouds, images, audio recordings, conversation transcripts
- `context` (map) — observer-specific metadata (sensor calibration parameters, model prompt configuration, annotation guidelines, etc.)
- `derived_from` (array[string]) — content addresses of grains consumed to produce this observation; REQUIRED for `observation_mode: "reflective"` to enable full compression provenance tracing
- `consolidation_level` (int, [0, 3]) — maps onto observer hierarchy: 0=raw single reading, 1=frequency-aggregated, 2=pattern-distilled, 3=longitudinal sequence
- `valid_from`, `valid_to` (int64, epoch ms) — for `observation_mode: "reflective"`, these describe the real-world window that was observed, not the moment the grain was written
- `provenance_chain` (array[map]) — derivation trail; use method strings from Section 14.1
- `observation_mode` (string enum) — how the observation was made. See Observation Mode Registry (Section 25). Valid values: `"passive"`, `"active"`, `"reflective"`, `"real_time"`. Default: absent.
- `observation_scope` (string enum) — temporal breadth of what was observed. See Observation Scope Registry (Section 26). Valid values: `"point"`, `"interval"`, `"session"`, `"longitudinal"`. Default: absent (`"point"` implied for physical observers).
- `observer_model` (string) — for AI observers: the model or software version identifier. RECOMMENDED when `observer_type` is `"llm"`, `"reflector"`, `"classifier"`, or `"detector"`. Example: `"claude-sonnet-4-6"`, `"gpt-4o"`, `"yolov8n"`. Absent for physical observer types.
- `compression_ratio` (float64) — for aggregating observers with `observation_mode: "reflective"`: the ratio of input content volume to output content volume. Computed as `input_tokens / output_tokens` or `input_bytes / output_bytes`. A value of `12.4` means the observer compressed 12.4×. Provides trust calibration signal: higher compression implies more aggressive summarization and potentially lower fidelity. Absent for point observers.

**Example — physical sensor:**
```json
{
  "type": "observation",
  "observer_id": "robot-arm-7/lidar-front",
  "observer_type": "lidar",
  "subject": "obstacle",
  "object": "2.3m at bearing 045°",
  "confidence": 0.97,
  "frame_id": "base_link",
  "sync_group": "scan_000412",
  "observation_mode": "active",
  "observation_scope": "point",
  "created_at": 1737000000000,
  "namespace": "robotics:arm-7",
  "source_type": "sensor"
}
```

**Example — cognitive observer (LLM, reflective, interval-scoped):**
```json
{
  "type": "observation",
  "observer_id": "did:key:z6MkLLMObserverAgent...",
  "observer_type": "llm",
  "observer_model": "claude-sonnet-4-6",
  "subject": "user:alice",
  "object": "Building a Next.js app with Supabase auth, deadline in 1 week. Prefers TypeScript. Currently blocked on OAuth callback configuration.",
  "confidence": 0.85,
  "observation_mode": "reflective",
  "observation_scope": "interval",
  "compression_ratio": 12.4,
  "frame_id": "thread:conv-abc123",
  "valid_from": 1736993000000,
  "valid_to": 1737000000000,
  "created_at": 1737000001000,
  "derived_from": ["a1b2c3d4...", "e5f6a7b8...", "c9d0e1f2..."],
  "consolidation_level": 1,
  "source_type": "agent_inferred",
  "namespace": "app:alice",
  "user_id": "alice",
  "provenance_chain": [
    {"source_hash": "a1b2c3d4...", "method": "llm_observation", "weight": 0.4},
    {"source_hash": "e5f6a7b8...", "method": "llm_observation", "weight": 0.35},
    {"source_hash": "c9d0e1f2...", "method": "llm_observation", "weight": 0.25}
  ]
}
```

**Example — human observer:**
```json
{
  "type": "observation",
  "observer_id": "did:key:z6MkDrJones...",
  "observer_type": "human",
  "subject": "patient:p-987",
  "object": "Patient appears anxious, reports intermittent chest pain rated 7/10. Diaphoretic. Reports onset 3 hours ago.",
  "confidence": 0.91,
  "observation_mode": "passive",
  "observation_scope": "point",
  "created_at": 1737000000000,
  "namespace": "clinical:ward-3",
  "user_id": "patient:p-987",
  "structural_tags": ["phi:clinical_observation", "phi:vital_signs"],
  "source_type": "user_explicit",
  "context": {"encounter_id": "enc-55123", "observation_setting": "emergency_department"}
}
```

**Design rationale — why Observation is not collapsed into Fact:**
The Observation type must remain distinct from Fact for four reasons:
1. **Observer accountability:** `observer_id` + `observer_type` + `observer_model` enables per-observer reliability scoring at query time without payload deserialization. Miscalibrated sensors or high-hallucination AI observers can be batch-penalized or re-evaluated.
2. **Multi-observer consensus:** `sync_group` enables fusing N concurrent observations into a consensus Fact. This pattern has no clean analog in the Fact triple structure.
3. **Raw-vs-derived separation:** Observations are epistemically raw; Facts are derived claims. Keeping them separate enables re-deriving Facts from raw Observations after updating a model, and rolling back Facts when supporting Observations are discredited.
4. **O(1) header routing:** The `0x06` type byte enables routing physical-sensor Observations to time-series stores and cognitive Observations to vector+relational stores without deserialization.

### 8.7 Goal

Explicit objective set by a human or inferred by an agent. Goals have lifecycle semantics (active → satisfied | failed | suspended) that are distinct from Fact confidence revisions. Each state transition creates a new immutable grain; the supersession chain carries the transition history.

**Why not Fact with `relation="has_goal"`:** At scale, querying active goals via Fact requires full payload deserialization to inspect the `relation` field. A dedicated type byte enables O(1) header-level filtering before any MessagePack decode. Goal state (`goal_state`) is a first-class indexable field, not buried in a `context` map.

**Required fields:**
- `type` = "goal"
- `subject` (non-empty string) — agent or entity this goal belongs to
- `description` (non-empty string) — natural language statement of the objective
- `goal_state` (string, enum) — one of: `"active"`, `"satisfied"`, `"failed"`, `"suspended"`
- `source_type` (non-empty string) — `"user_explicit"`, `"agent_inferred"`, `"system"`, `"consolidated"` (typical values for Goal grains; see §6.1 for full open enum). Required (not optional) because human-vs-agent origin is a first-class routing concern.
- `created_at` (int64, epoch ms)

**Optional fields (Goal-specific):**
- `criteria` (array[string]) — semi-structured success conditions, human-readable and LLM-evaluable: `["p99_latency_ms < 100", "error_rate < 0.001"]`
- `criteria_structured` (array[map]) — machine-evaluable criteria (see schema below)
- `priority` (int, [1, 5]) — 1=critical, 5=low. Integer not float for discrete scheduling.
- `parent_goals` (array[string]) — content addresses of parent Goal grains; supports DAG hierarchy (multiple parents)
- `state_reason` (string) — audit rationale for the current state transition
- `satisfaction_evidence` (array[string]) — content addresses of ToolCall, Fact, or Observation grains that substantiate a `satisfied` transition
- `progress` (float64, [0.0, 1.0]) — agent-assessed progress; subjective estimate, not mechanically derived from criteria
- `delegate_to` (string) — DID of agent to whom this goal is delegated
- `delegate_from` (string) — content address of the grain that originated the delegation
- `expiry_policy` (string, enum) — `"hard"` (fail if `valid_to` exceeded), `"soft"` (suspend), `"extend"` (agent may extend `valid_to`)
- `recurrence` (string) — cron-like recurrence expression for recurring goals
- `evidence_required` (int) — minimum number of grains in `satisfaction_evidence` required before `satisfied` transition is valid
- `rollback_on_failure` (array[string]) — content addresses of Workflow or ToolCall grains to execute on `failed` transition
- `allowed_transitions` (array[string]) — goal state transitions the agent may execute autonomously without satisfying `invalidation_policy`; valid values: `"satisfied"`, `"failed"`, `"suspended"`, `"active"`. Absent on a protected goal means all transitions require policy authorization.
- `invalidation_policy` (map) — protection policy restricting who may supersede this goal or set `contradicted=true`; see §23. **Note on goal laundering:** An agent MUST NOT use `satisfied` transitions to escape a protected goal's constraints. Protected goals with `"satisfied"` in `allowed_transitions` SHOULD include `satisfaction_evidence` references; stores MAY enforce `evidence_required > 0` for such goals.

**Optional fields (inherited from core):**
- `confidence` (float64, [0.0, 1.0]) — certainty the goal is correctly understood
- `importance` (float64, [0.0, 1.0])
- `author_did` (string)
- `user_id` (string) — data subject (GDPR scope for human-set goals)
- `namespace` (string, default "shared")
- `valid_from` (int64, epoch ms) — when the goal activates
- `valid_to` (int64, epoch ms) — deadline
- `superseded_by` (string) — content address of next state grain in the transition chain
- `derived_from` (array[string]) — parent content addresses
- `provenance_chain` (array[map]) — who set this goal and by what method
- `related_to` (array[map]) — cross-links to relevant Facts, Episodes, or other Goals
- `structural_tags` (array[string])
- `embedding_refs` (array[map]) — for semantic similarity search across goals
- `context` (map string→string)

**`criteria_structured` entry schema:**
```json
{
  "metric": "p99_latency_ms",
  "operator": "lt",
  "threshold": 100,
  "window_ms": 300000,
  "measurement_ns": "monitoring:metrics"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `metric` | string | REQUIRED | Metric identifier to evaluate |
| `operator` | string | REQUIRED | Comparison: `"lt"`, `"gt"`, `"lte"`, `"gte"`, `"eq"`, `"neq"` |
| `threshold` | number | REQUIRED | Comparison value |
| `window_ms` | int | OPTIONAL | Measurement evaluation window in milliseconds |
| `measurement_ns` | string | OPTIONAL | Namespace or system to query for the metric |

**State transition via supersession chain:**

```
G1 (active):  goal_state="active",  derived_from=[]
G2 (suspended): goal_state="suspended", derived_from=[<hash-G1>],
                provenance_chain=[{source_hash:<hash-G1>, method:"goal_state_transition", weight:1.0}],
                state_reason="Higher priority goal preempted"
G3 (satisfied): goal_state="satisfied", derived_from=[<hash-G2>],
                satisfaction_evidence=[<toolcall-hash>, <observation-hash>],
                state_reason="All criteria verified"
```

The index layer fills in `superseded_by` on prior grains after new state grains are written, following the same pattern as `system_valid_to` in §15.3.

**Provenance chain methods for Goal grains:**

| Method string | Meaning |
|--------------|---------|
| `"user_input"` | Human set this goal directly |
| `"goal_decomposition"` | Agent broke a parent goal into sub-goals |
| `"goal_state_transition"` | This grain updates the state of a prior Goal grain |
| `"goal_revision"` | Human modified a previously set goal |
| `"goal_inference"` | Agent inferred a goal from Episode or Fact patterns |
| `"goal_delegation"` | Goal was delegated from another agent |

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
| Footer   | SHA-256 checksum |  32 bytes (over header + index + grains)
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
| 4-7 | Reserved |

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

SHA-256 over: `header (16 bytes) || index (grain_count*4 bytes) || grains (variable)`

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
| `legal:` | Legal data | `legal:privilege`, `legal:litigation_hold` |

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

**Provenance chain method strings for Observation grains (v1.1):**

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
- Support all 7 memory types
- Ignore unknown fields
- Constant-time hash comparison
- **v1.1:** MUST accept deprecated short keys `sid` and `stype` in Observation grains and map them to `observer_id` and `observer_type` respectively

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
- **v1.1:** MUST validate that `observer_type` is a non-empty string; MUST NOT reject unknown `observer_type` values (open enum)
- **v1.1:** MUST emit `oid` and `otype` short keys; MUST NOT emit deprecated `sid` or `stype`
- **v1.1:** SHOULD warn (but MUST NOT reject) when `observer_model` is absent on Observation grains where `observer_type` is `"llm"`, `"reflector"`, `"classifier"`, or `"detector"`

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
- **v1.1:** SHOULD partition Observation grain storage by observer domain, inferred from `observer_type`. Physical observer types (see Section 24) SHOULD flow to time-series storage with raw-data retention policies. Cognitive observer types SHOULD flow to vector + relational storage with the same retrieval semantics as Fact grains. Implementations MUST NOT hard-code the domain partition list — treat `observer_type` as an open string and drive routing from configuration or namespace.

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

### 21.2 Vector 2: Episode

**Input:**
```json
{
  "type": "episode",
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

**Note:** v1.1 serializers emit short keys `oid` and `otype` for this grain. v1.0 legacy grains using `sid`/`stype` short keys are accepted by v1.1 readers under the backward-compatibility aliasing rule (§6.6).

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
  "mode": "open" | "soft_locked" | "locked" | "quorum" | "delegated" | "timed",
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

## Appendix A: ABNF Grammar

```abnf
mg-blob       = version-byte header-fields msgpack-payload
version-byte  = %x01
header-fields = flags-byte type-byte ns-hash-bytes created-at-bytes
                ; version-byte + header-fields = 9-byte "fixed header" in §3.1
flags-byte    = %x00-FF
type-byte     = %x01-FF
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

## Appendix B: Field Mapping Table (Compact Reference)

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
  "sa": "supersession_auth"
}
```

**Observation-Specific Fields (v1.1):**

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

> **Deprecated short keys (v1.0, accepted by readers until v2.0):** `"sid"` → `observer_id`, `"stype"` → `observer_type`

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

---

## Appendix C: Compliance Mapping

### GDPR

| Article | .mg Support |
|---------|------------|
| Art. 5 (Data minimization) | `user_id` field enables per-person scope |
| Art. 12-23 (Rights) | Structured data format for automated response |
| Art. 17 (Erasure) | Crypto-erasure via key destruction |
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

## Appendix D: Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-02-17 | Initial publication — 9-byte fixed header (version 0x01, 2-byte NS hash), canonical serialization, content addressing, seven memory types (incl. Goal 0x07), COSE signing, selective disclosure, .mg container files, W3C DIDs, sensitivity bits, reg: tag vocabulary, provenance, temporal modeling, conformance levels, device profiles |
| 1.1 | 2026-02-20 | Generalized Observer: renamed `sensor_id`→`observer_id` (short key `sid`→`oid`) and `sensor_type`→`observer_type` (short key `stype`→`otype`); added four new Observation-specific fields (`observation_mode`, `observation_scope`, `observer_model`, `compression_ratio`); generalized `frame_id` and `sync_group` semantics to cover both physical-coordinate and cognitive-context framing; added Observer Type Registry (Section 24), Observation Mode Registry (Section 25), Observation Scope Registry (Section 26); extended elidability table, provenance method strings, and conformance level requirements; deprecated v1.0 short keys `sid`/`stype` with read-only compatibility until v2.0 |

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

## Appendix E: Glossary

- **Blob:** Complete .mg binary (version byte + optional header + payload)
- **Grain:** Atomic knowledge unit; identified by content address
- **Content address:** SHA-256 hash of blob bytes; unique identifier
- **Canonical:** Deterministic serialization rules ensuring identical bytes
- **DID:** W3C decentralized identifier; cryptographic identity without CA
- **COSE:** CBOR Object Signing and Encryption (RFC 9052)
- **Selective disclosure:** Hiding some fields while proving they exist
- **Provenance:** Derivation trail showing how grain was created
- **Cross-link:** Semantic relationship between grains
- **Bi-temporal:** Tracking both event-time and system-time dimensions
- **Crypto-erasure:** Destroying encryption key to unrecoverably erase data
- **Blind index:** HMAC token for searching encrypted data without decryption

---

## Appendix F: Complete Example Grain

```python
# Create a fact grain
grain = {
    "type": "fact",
    "subject": "machine-learning",
    "relation": "is_subset_of",
    "object": "artificial-intelligence",
    "confidence": 0.99,
    "source_type": "user_explicit",
    "created_at": 1737000000000,
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
# 6. Prepend 9-byte fixed header (1-byte version + 8-byte header: flags, type, ns_hash[2], created_at[4])
# 7. Compute SHA-256 hash

blob = serialize(grain)
content_address = sha256(blob).hex()

# Result: 64-character lowercase hex string
# Example: 3a1f5d8e9c2b7a4f6e9d2c8b1a4f7e9d2c8b1a4f7e9d2c8b1a4f7e9d2c8b1a4f
```

---

**Document Status:** This is the initial publication of the .mg format specification, submitted as a standards track document for consideration as an IETF RFC and W3C standard. Community feedback is encouraged through issue tracking and discussion forums.

**Last Updated:** 2026-02-17
**License:** This document is offered under the Open Web Foundation Final Specification Agreement (OWFa 1.0)
