# Changelog

All notable changes to the Open Memory Specification are documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Version numbers follow the specification's own versioning scheme: `MAJOR.MINOR` where:

- **MAJOR** — Backwards-incompatible changes to the binary format or semantics
- **MINOR** — Backwards-compatible additions or clarifications

---

## [1.0] — 2026-02-19

### Added

- Initial release of the Open Memory Specification
- Memory Grain (`.mg`) binary container format
- 9-byte fixed header: version, flags, type, namespace hash, timestamp
- MessagePack canonical serialization with deterministic key ordering
- CBOR alternative encoding (`cbor_encoding` flag)
- SHA-256 content addressing
- Field compaction (short key mappings for storage efficiency)
- Seven standard memory types: Fact, Episode, Checkpoint, Workflow, ToolCall, Observation, Goal
- Application-defined types (`0xF0–0xFF`)
- COSE Sign1 cryptographic signing envelope
- AES-256-GCM payload encryption
- zstd payload compression
- Selective disclosure via hash commitments
- Multi-modal content references (image, audio, video, point cloud, 3D mesh, embedding, binary)
- W3C DID-based identity and authorization model
- Sensitivity classification: public, internal, PII, PHI
- Cross-link and provenance tracking
- Temporal modeling (created_at, valid_from, valid_to, superseded_by)
- `.mg` file container format with grain index and checksum
- Grain protection and invalidation policy
- Compliance metadata (GDPR, CCPA, HIPAA primitives)
- Three conformance levels: Level 1 (Minimal Reader), Level 2 (Full Implementation), Level 3 (Production Store)
- Device profiles for constrained environments
- Error codes and error handling specification
- Security considerations section
- Test vectors for all standard memory types
- Namespace routing hint (NS hash) — 65,536 routing buckets
