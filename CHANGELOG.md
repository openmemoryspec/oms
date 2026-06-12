# Changelog

All notable changes to the Open Memory Specification are documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Version numbers follow the specification's own versioning scheme: `MAJOR.MINOR` where:

- **MAJOR** — Backwards-incompatible changes to the binary format or semantics
- **MINOR** — Backwards-compatible additions or clarifications

---

## [1.4] — 2026-06-12

### Added

- **`embedding_text` common field (§6.1)** — optional `string` (compact key: `et`) providing source text for vector embedding and full-text indexing. When present, implementations SHOULD use this value instead of the grain's default per-type text representation. Enables document-derived grains to preserve source paragraph context for retrieval while maintaining structured subject/relation/object triples. Benchmarked at +29.4pp Recall@10 improvement on a 28-grain refund policy dataset. See `proposals/embedding-text-field.md`.
- **Appendix C** — added `"et": "embedding_text"` to core field compaction table.
- **Workflow grain type redesigned as directed graph (§8.4)** — Workflow grains now model directed graphs instead of flat step lists. New fields: `nodes` (array[string]), `edges` (array[map] with `src`, `dst`, `cond`, `max_cycles`), `bindings` (map[string→string] mapping node IDs to Tool definition grain hashes), `retries` (map[string→int]). Supports sequential, parallel fork/join, conditional branching, and bounded cycles. Three-tier node-to-Tool resolution (bound → named → abstract). Replaces `steps` array.
- **Workflow structural semantics (§8.4)** — node types inferred from graph topology: fork (multiple outgoing edges), AND-join (multiple incoming edges), decision point (conditional edges), terminal (no outgoing edges). Entry point is first element of `nodes`.
- **`mg:has_graph` relation** — replaces `mg:requires_steps` for Workflow grains. Reflects the directed graph model.

### Changed

- **Type renames (breaking):** `"belief"` → `"fact"` (0x01), `"action"` → `"tool"` (0x05). Byte values unchanged; existing content addresses remain valid. The `Fact` grain is the (subject, relation, object) semantic triple with confidence; the `Tool` grain records tool/action invocations and executions. Updates the grain-type table (§5), the type-specific field headings, the relation vocabulary, and the §28.8 Skill Convention `mg:capable_of` mapping (now a `Fact` grain). Legacy `"belief"`/`"action"` type strings are not accepted.
- **Workflow grain description** — "Learned action sequence — procedural memory" → "Directed graph of procedural steps — plans, pipelines, processes".
- **Workflow fields** — `steps` (array[string]) and `trigger` (string) replaced by `nodes`, `edges`, `bindings`, `retries`, and optional `trigger`. Edge schema uses nested `src`/`dst`/`cond`/`max_cycles` fields with compaction keys.
- **`mg:requires_steps`** renamed to **`mg:has_graph`** in the `mg:` standard relation vocabulary.

### CAL 1.1 — Added

- **Multi-format output (§10.1.1)** — `FORMAT` and `AS` clauses now accept bracketed format lists (`FORMAT [markdown, json]`) with optional aliases (`FORMAT [json AS structured, markdown AS report]`). Single query execution produces multiple renderings. New multi-format response shape (`"formats"` object, §14.2.1). Maximum 5 formats per list. New error codes: `CAL-E110` (too many formats), `CAL-E113` (duplicate format key).
- **Workflow graph syntax for ADD/SUPERSEDE (§8.8.1)** — dedicated graph syntax for workflow grains using `->` (sequential edges), `()` (parallel fork/join), `WHEN` (conditional edges), `* N` (retry bounds), `BIND` (node-to-Action mapping), and `ON` (trigger clause). Full graph replacement on SUPERSEDE.
- **`BECAUSE` alias** — accepted as synonym for `REASON` in ADD, SUPERSEDE, and workflow statements.

### CAL 1.1 — Changed

- **Pipeline operators removed** — bare pipe syntax (`| ORDER BY`, `| LIMIT`, `| COUNT`, etc.) replaced by direct clause syntax (`ORDER BY`, `LIMIT`, `COUNT`). All examples and grammar productions updated. Backward-compatible at the semantic level.
- **Workflow query fields** — `steps` field replaced by `node` and `binding` for grain-type-specific querying.
- **Workflow content projection** — `nodes` joined with `->` arrow syntax replaces numbered `steps` list in SML/template output.
- **`GrainTypeNotAddable` (CAL-E051)** — Workflow added to the set of addable grain types (Fact, Observation, Goal, Workflow).

---

## [1.3] — 2026-03-03

### Added

- **Action grain `output_schema` field** — JSON Schema (draft-07 compatible) describing the action's return value; optional in definition phase, omitted in all other phases. Compact key: `osch`. Enables LLMs to plan multi-step workflows without trial executions by knowing what each tool produces.
- **Integration Domain Profile (`profile:integration`)** — new Appendix A.7 entry with `int:` namespace prefix for REST API connectors, tool catalogs, and action registries. 16 action fields (`int:base_url`, `int:http_method`, `int:http_path`, `int:auth_type`, etc.) and 9 trigger-specific fields (`int:poll_interval_secs`, `int:webhook_path`, `int:cron_expression`, etc.). All fields stored in `context` map.
- **Integration Profile compact keys** — 25 new `int:*` short key mappings registered in Appendix C to avoid collisions.
- **§27.6 Trigger Definitions via Observation Grains** — documented convention for mapping trigger definitions (polling, webhook, schedule, listener) to Observation grains using `observer_type` values prefixed with `"trigger:"`. `"trigger:listener"` uses `observation_mode: "continuous"`, same as webhook.
- **§27.7 Consensus Grain Usage for Action Definition Validation** — documented usage pattern for recording multi-source agreement on Action definition grains via the Consensus grain type.
- **Updated §27.1 tool definition example** to include `output_schema` field.
- **Updated Anthropic API alignment table** in §27.1 to note `output_schema` has no Anthropic equivalent.
- **`CONTEXT-ASSEMBLY-LANGUAGE-CAL-SPECIFICATION.md` — Context Assembly Language v1.0** bundled into this repository as part of OMS v1.3. CAL is a non-destructive, deterministic, LLM-native language for assembling agent context from OMS stores. **SML (Semantic Markup Language)** is now its own standalone specification at `SEMANTIC-MARKUP-LANGUAGE-SML-SPECIFICATION.md`. OMS spec gains §1.5 (Companion Specifications) and §28.9 (CAL/SML store operation mapping).

### Changed

- **Integration Profile compact key renames (Appendix C)** — fixed collisions and improved consistency:
  - `"im"` → `"ihm"` (`int:http_method`) — `"im"` collided with `importance`
  - `"ip"` → `"ihp"` (`int:http_path`) — `"ip"` collided with `invalidation_policy`
  - `"ict2"` → `"icft"` (`int:cursor_type`) — aligned with field name
  - `"icfgs"` → `"icfg"` (`int:config_schema`) — shortened for consistency
  - `"ievs"` → `"ievt"` (`int:event_schema`) — aligned with field name
- **§27.6 polling trigger example** — added missing `int:path_params` to align with A.7 normative rule requiring path template parameters to match `int:path_params` entries.
- **§27.6 `observation_mode` mapping** — clarified that `"trigger:listener"` uses `"continuous"` (same as webhook).
- **§6.1 `category` field** — fixed stale cross-reference from "§27 Category Registry" to "§27 Grain Type Field Specifications" (renamed in v1.2).

---

## [1.2] — 2026-02-23

### Breaking changes

- **Type renames are not backwards-compatible:** `"fact"` → `"belief"`, `"episode"` → `"event"`, `"checkpoint"` → `"state"`, `"tool_call"` → `"action"`. Legacy type strings are not accepted; no prior implementation exists.
- **Removed legacy Action short keys** `arguments`/`args`, `result`/`res`, `success`/`ok` — use `input`/`inp`, `content`/`cnt`, `is_error`/`iserr` exclusively.
- **Removed legacy Observation short keys** `sid` (`sensor_id`) and `stype` (`sensor_type`) — use `oid` and `otype` exclusively.
- **Removed `contradicted` field** — use `verification_status` exclusively.
- Binary wire format is otherwise unchanged from v1.1. The fixed header returns to 9 bytes (the Category byte introduced in v1.2-draft is removed).

### Added

- **New type 0x08: Reasoning** — inference chain and thought audit trail; fields: `premises`, `conclusion`, `inference_method`, `alternatives_considered`, `requires_human_review`, `thinking_content`, `thinking_redacted`, `statistical_context`, `software_environment`, `parameter_set`, `random_seed`
- **New type 0x09: Consensus** — multi-agent agreement record; fields: `participating_observers`, `threshold`, `agreement_count`, `dissent_count`, `dissent_grains`, `agreed_content`
- **New type 0x0A: Consent** — DID-scoped permission grant or withdrawal; fields: `subject_did`, `grantee_did`, `scope`, `is_withdrawal`, `basis`, `jurisdiction`, `prior_consent`, `witness_dids`
- **Action grain additions:** `action_phase` (`definition`/`call`/`result`), `tool_call_id`, `call_batch_id`, `tool_type`, `tool_version`, `execution_mode` (`function_call`/`code_exec`/`computer_use`), `code`, `stdout`, `stderr`, `exit_code`, `interpreter_id`, `tool_description`, `input_schema`, `strict`
- **New core common fields:** `timestamp_ms` (authoritative payload timestamp, epoch ms), `observer_did`, `subject_did`, `session_id`, `entity_id`, `epistemic_status`, `verification_status`, `requires_human_review`, `processing_basis`, `identity_state`, `license`, `trusted_timestamp` (RFC 3161)
- **New common fields from §6.1:** `run_id`, `role`, `access_count`, `last_accessed_at`, `owner`, `category`
- **New invalidation fields:** `invalidation_type` (retraction/erratum/corrigendum/retraction_with_replacement/expression_of_concern), `invalidation_reason`, `invalidation_initiator`
- **New `retention_policy` field** — minimum retention requirements distinct from `invalidation_policy`; fields: `minimum_retention_years`, `regulation`, `deletion_requires`
- **New `recall_priority` field** — storage tier hint: `"hot"`, `"warm"`, `"cold"`
- **New `invalidation_policy` modes:** `"hold"` (litigation hold — blocks all erasure until explicitly lifted) and `"consent_cascade"` (auto-erasure within ≤ 30 days when linked Consent grain is revoked)
- **`source_type` new values:** `"established_knowledge"` (physical constants, scientific laws) and `"axiomatic"` (mathematical axioms, tautologies)
- **`mg:` standard relation vocabulary** — 21 standard relations: `mg:perceives`, `mg:knows`, `mg:said`, `mg:did`, `mg:infers`, `mg:agrees_with`, `mg:state_at`, `mg:has_graph`, `mg:intends`, `mg:permits`, `mg:revokes`, `mg:prohibits`, `mg:requires`, `mg:prefers`, `mg:avoids`, `mg:delegates_to`, `mg:owned_by`, `mg:has_capability`, `mg:handed_off_to`, `mg:depends_on`, `mg:assigned_to`
- **HIPAA PHI tag normalization** — 18 normative `phi:` tag values per 45 CFR §164.514(b) Safe Harbor
- **External citation schema** in `content_refs` — structured citations for doi/arxiv/pmid/isbn/rrid/clinicaltrials/url with `citation_role`
- **§12.5 Agent Ownership and Legal Entity** — Belief grain pattern with `mg:owned_by` relation
- **§28 Query Conventions** — standard search response envelope, namespace convention, index-layer-managed fields, store protocol convention, agent capability convention, conversation threading convention, session handoff convention
- **Event grain additions for LLM conversation turns:** `content_blocks` (structured multi-block content matching Anthropic/OpenAI message format), `model_id`, `stop_reason`, `token_usage`, `parent_message_id` (conversation threading)
- **Embedding chunk metadata:** `chunk_index`, `chunk_text`, `chunk_strategy`, `chunk_overlap` — enables RAG round-trip from vector search hit back to source grain
- **Goal grain task dependency fields:** `depends_on` (DAG-structured task ordering for plan-and-execute agents), `assigned_agent`, `expected_output`, `output_grain`, `deadline`
- **Delegation scope fields (§6.11):** `authorized_namespaces`, `authorized_types`, `authorized_tools`, `delegation_depth`, `delegation_expiry`, `context_grains`, `return_to` — scoped authority grants for multi-agent orchestration and session handoff
- **Action grain `error_type`** — structured error classification (`"timeout"`, `"rate_limit"`, `"auth_failure"`, etc.) for retry policy decisions
- **§22.7 Streaming non-pattern note** — grains are atomic; streaming is transport-layer
- **§22.8 Recall priority framework mapping** — maps `hot`/`warm`/`cold` to Letta and LangChain memory tiers
- **§22.9 State grain context schema convention** — recommended keys for cross-framework agent state portability
- **§22.10 Access counter semantics** — async flush, no internal reads counted, probabilistic counting guidance
- **§28.4 Store Protocol Convention** — get/put/supersede/query/search/delete/put_batch/get_batch operations
- **§28.5 Agent Capability Convention** — Belief grain pattern with `mg:has_capability` for agent discovery (Agent Card equivalent)
- **§28.6 Conversation Threading Convention** — session_id + parent_message_id linked-list pattern
- **§28.7 Session Handoff Convention** — delegation + context_grains pattern for inter-agent control transfer
- **Domain Profile Extension Model** — profile declaration via `structural_tags`; namespace prefixes `hc:`, `legal:`, `fin:`, `rob:`, `sci:`, `con:`
- **Appendix A: Domain Profile Registry** — Healthcare (0xF0–0xF3), Legal (0xF4–0xF6), Finance (0xF7–0xFA), Robotics (0xFB–0xFF), Science (field extensions), Consumer (field extensions)

### Changed

- **Header reverts to 9 bytes** — Category byte (Byte 3) removed from fixed header; `category` remains as optional payload field
- **`created_at` in header is now normatively a routing hint only** — MUST NOT be used as authoritative event timestamp; use `timestamp_ms` payload field instead
- **Type renames:** Fact → Belief (0x01), Episode → Event (0x02), Checkpoint → State (0x03), ToolCall → Action (0x05). Workflow, Observation, Goal unchanged.
- **Ownership grain** uses `mg:owned_by` relation on Belief type instead of the old `"fact"` type with `"owned_by"` relation
- **Application-defined type range** (0xF0–0xFF) documented as domain profile conventions in Appendix A
- **Minimum blob size** is 10 bytes (9-byte header + 1-byte empty MessagePack map)
- **§27 Category Registry** replaced by §27 Grain Type Field Specifications
- **Belief `source_type` is no longer required** — moved to optional common fields; existing grains without `source_type` are valid
- **Belief `object` now accepts map in addition to string** — enables structured object values for complex relation triples
- **Goal `subject` and `source_type` are no longer required** — moved to optional fields; `description`, `goal_state`, and `created_at` are the only required fields

### Removed

- **Action fields `arguments`/`args`, `result`/`res`, `success`/`ok`** — removed immediately; no prior implementation exists. Use `input`/`inp`, `content`/`cnt`, `is_error`/`iserr`.
- **`contradicted` (bool, short key `ct`)** — removed immediately; no prior implementation exists. Use `verification_status`.

---

## [1.1] — 2026-02-20

### Added

- Generalized Observer model — renamed `sensor_id` → `observer_id` (short key `sid` → `oid`) and `sensor_type` → `observer_type` (short key `stype` → `otype`)
- Four new Observation-specific fields: `observation_mode`, `observation_scope`, `observer_model`, `compression_ratio`
- Generalized `frame_id` and `sync_group` semantics to cover both physical-coordinate and cognitive-context framing
- Observer Type Registry (§24) — Physical domain: lidar, camera, imu, gps, temperature, pressure, accelerometer, magnetometer, ultrasonic, radar, microphone; Cognitive domain: llm, reflector, classifier, detector, human, hybrid
- Observation Mode Registry (§25) — passive, active, reflective, real_time
- Observation Scope Registry (§26) — point, interval, session, longitudinal
- Extended elidability table with Observation-specific fields
- Provenance chain method strings for Observation grains
- Conformance level requirements for v1.1 Observation field handling

### Changed (backwards-compatible)

- `observer_model` RECOMMENDED (but not required) when `observer_type` is `"llm"`, `"reflector"`, `"classifier"`, or `"detector"`
- Level 2 conformance: SHOULD warn when `observer_model` is absent for cognitive observer types
- Level 3 conformance: SHOULD partition Observation grain storage by observer domain

### Removed in v1.2

- Short keys `sid` (`sensor_id`) and `stype` (`sensor_type`) from v1.0 — removed in v1.2; no prior implementation exists. Use `oid` and `otype`.

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
