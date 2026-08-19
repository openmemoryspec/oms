# Changelog

All notable changes to the Open Memory Specification are documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Version numbers follow the specification's own versioning scheme: `MAJOR.MINOR` where:

- **MAJOR** — Backwards-incompatible changes to the binary format or semantics
- **MINOR** — Backwards-compatible additions or clarifications

---

## [1.6-draft] — 2026-08-10 (RFC, open for comment; anonymization batch added 2026-08-16, trigger batch 2026-08-19, saved-query, template-identity, blob-lifecycle, subject-reachability, mail-profile and housekeeping batches 2026-08-19)

### Added in the saved-query batch (2026-08-19)

- **CAL §8.18 Saved queries** — `DEFINE QUERY "name"($params) [DESCRIPTION …]
  AS { body }`, `RUN "name"($p = v)`, `DROP QUERY`, `DESCRIBE QUERIES`. Answers
  `openmemoryspec/oms#12 §1`: 1.0–1.2 restricted saved-query bodies in four
  places (§8.14.3, §8.16.1, §8.17, `CAL-E125`) and made `query:<ns>/<name>` a
  Recommendation target while defining no statement that creates one. Bodies are
  **Tier 0 only**, non-recursive, and **parsed at definition time with the
  declared parameters standing in** — a stored query that cannot run is refused
  where it is written, not left as a landmine for an unattended first caller.
  Defining is Tier 3 (`admin`); **running is a Tier 0 read**, because the body
  is structurally read-only and re-verified at execution.
- **OMS §28.4.1 Host metadata table** — `meta_get`/`meta_put`/`meta_delete`/
  `meta_scan`, and the reserved prefix registry (`qry:`, `tpl:`, `retention:`,
  `anon:`, `vault:`, `trg:`). §10.5.1, §10.5.2 and §27.8 already referred to
  "reserved store-metadata prefixes"; this section is the definition they
  referenced. Metadata rows are **not grains** — not content-addressed, not
  returned by a read, no supersession — and replication is per-prefix with the
  same split the trigger and governance sections draw: *what a definition is*
  replicates, *where one host got to* does not.
- **CAL error codes `CAL-E130`–`CAL-E137`** (saved query), and §24's
  **Definitions module**.
- **CAL §13 / OMS §28.9** gain the definition-surface and blob rows.

### Changed in the saved-query batch (2026-08-19)

- **`DROP` leaves CAL's §2.4 grammar-exclusion list**, admitted by a production
  with a closed two-target set (`DROP TEMPLATE`, `DROP QUERY`) that reaches no
  grain. §2.4 records why the permanence claim survives: what was permanent is
  the exclusion of unbounded destruction *of memory*, and `DROP` was blocked as
  the nearest available proxy for that property — this revision replaces the
  proxy with the property. `UNDEFINE` stays reserved and unspecified rather than
  becoming a second spelling; implementations MUST NOT accept it as an alias.

### Fixed in the housekeeping batch (2026-08-19)

Answers `openmemoryspec/oms#12 §7`.

- **`mg:step_action:<node_id>` joins the relation vocabularies** (OMS §8, CAL
  §7.1). It was used normatively in §8.4 while appearing in no vocabulary list,
  so a conformant validator warned `CAL-W001` — "unknown `mg:` relation" — on
  this specification's own relation. It is the first **parameterized** standard
  relation (the node id is part of the value), so CAL §7.3 now states the
  matching rule: parameterized rows match by prefix, everything else by exact
  value.
- **CAL §14.3 had two disclosure models and admitted one.** The grammar has read
  `disclosure_level = "summary" | "headlines" | "full"` since 1.0, against a
  §14.3 table naming `summary | standard | full`. They were never the same
  control: one is **metadata density** (how many attributes an element carries),
  the other is **body extent** (how much free text survives). §14.3 now
  documents both axes and how they compose, including that `full` is *additive*
  — it carries a Skill's `instructions`/`when_to_use`, which no other level
  does at any budget — and that omitting the option is **not** a level but a
  request to leave each format's established output alone.
- **Version markers reconciled.** `SPECIFICATION.md`'s footer said "the v1.5
  revision" under a 1.6-draft header; it now describes the 1.6 draft, and the
  byte-range sentence reads `0x01`–`0x0C` unchanged (Trigger takes `0x0D`).
  Footer dates across all three documents moved to 2026-08-19.
- **Duplicate section number `§6.14`.** The trigger batch added "§6.14
  Trigger-Specific Fields" without renumbering the "§6.14 Compaction Rules" that
  followed it — the same bump the Recommendation type did on its way in.
  Compaction Rules is now §6.15; nothing referenced it by number.
- **`CONTRIBUTING.md` pointed at `oms-specification.md`**, a file that has never
  existed in this repository. The checklist now names the three documents that
  do, and asks for the changelog entry.

### Added in the mail-profile batch (2026-08-19)

- **OMS Appendix A.8 Mail Profile (`mail:`)** — `mail:message_id`,
  `mail:in_reply_to`, `mail:references`, `mail:from`/`to`/`cc`/`bcc`,
  `mail:subject`, `mail:date`, `mail:folder`, `mail:direction`,
  `mail:transport`, plus a normative RFC 5322 → OMS field mapping. Event grains
  model LLM turns — `role` is a chat enum — and nothing in the specification
  named a sender, recipient, subject line or `Message-ID`, so every mail-shaped
  agent invented its own `context` keys and lost portability on exactly the
  grains most worth porting. Answers `openmemoryspec/oms#12 §3` by its option
  (a). Option (b), a first-class `thread_id` common field, was **rejected**: it
  adds a compact key to a normative frozen field map for a concept `session_id`
  already carries. The thread *is* the session (§28.6); `In-Reply-To` maps to
  both `parent_message_id` (the parent grain's content address) and
  `mail:in_reply_to` (the transport id — the only form available when the
  parent was never ingested).
- Two prohibitions carry privacy weight: `mail:message_id` MUST NOT be used as
  or derived into a content address, and `mail:bcc` MUST NOT be populated from a
  *received* message. A correspondent an agent asserts facts about SHOULD also
  appear in `subject`, because §28.4.4 reaches structured references — a person
  reachable only through a `context` key is not reachable by erasure or DSAR.
- **CAL `domain_prefix` admits `mail`** — the production is closed by design (an
  open prefix would turn every misspelled field into a valid domain field
  instead of `CAL-E004`), so a new OMS profile needs this CAL change to be
  queryable.
- **Status:** this is the one part of the 1.6 draft written *ahead of* a
  reference implementation rather than from working code, and it says so in the
  spec. The field set is a starting point for comment.

### Added in the subject-reachability batch (2026-08-19)

- **OMS §28.4.4 Subject reachability** — prior revisions defined the erasure
  selector without ever saying what it must **reach**, and a conforming
  implementation was found to under-erase silently because of it: a grain
  carrying a `subject` but no relation or object reached no structural index, so
  the identity's own transcript survived an erasure **that reported success**
  and went undisclosed in a DSAR **that reported completeness**. Now normative:
  reachability is a property of the identity, not of grain shape; reach extends
  to object position, session/run keys and partition-key derivatives; **erasure
  and disclosure MUST share one selector**; a known-incomplete scope MUST NOT
  report success; a reachability fix MUST heal existing memories by index
  rebuild rather than only indexing correctly going forward; residual limits
  MUST be documented. Carries a **required conformance case** — store a grain
  with only a subject, assert it is reported and then erased. Answers
  `openmemoryspec/oms#12 §6`.
- **CAL §8.19 `REPORT SUBJECT`** — the read-only mirror of `FORGET SUBJECT`,
  and the statement §28.4.4 rule 3 is about. Tier 0, `read` verb, same selector
  and same `WITH text_mentions` precondition as the erasure. It MUST NOT sit
  behind the cap that disables destructive operations — a deployment that turned
  destruction off still owes data subjects access — and it writes **no** audit
  grain: access destroys nothing, and an audit grain naming the subject of a
  DSAR would itself be personal data about them in an immutable replicating
  store. GDPR Art. 15/20 in §19.2 now point at it instead of at a `RECALL`
  approximation.

### Added in the blob-lifecycle batch (2026-08-19)

- **OMS §7.1.1 the `cas:` URI scheme** — `cas://sha256:<64-hex>`, used once as
  an example URI since 1.0 and never defined. The address is the SHA-256 of the
  **plaintext** bytes (so content addresses identically in an encrypted memory
  and a plaintext one), a reader MUST verify it, resolution is memory-local and
  never a network fetch, a reference is **not** a promise of presence, and a
  malformed address fails closed before any lookup. Answers
  `openmemoryspec/oms#12 §5`.
- **OMS §28.4 `put_blob` / `get_blob`** — the operation table was grains-only.
  `put_blob` is idempotent by construction (the address *is* the content);
  `get_blob` verifies the digest before returning bytes.
- **OMS §28.4.2 Blob concurrency** — a blob read needs no lock, because an
  immutable object whose address is its checksum poses no consistency question.
  Stated normatively because the alternative starves real callers: where a
  memory lock is exclusive, a run holding the memory would otherwise block the
  subprocess it spawned to process an attachment, for a lock protecting nothing.
- **OMS §28.4.3 Blob lifecycle under erasure** — the GDPR-grade half.
  Sole-referenced attachments MUST be reclaimed by subject erasure; reclamation
  MUST be **targeted, never a store-wide sweep** (a census races an upload that
  has happened while its referencing grain has not, and collecting it destroys
  an unrelated caller's data); the surviving-reference check MUST run under the
  erasure's own serialization; the report MUST count reclaimed blobs separately;
  and crypto-erasure MUST reach blobs, since encrypting the database while
  leaving attachments in the clear beside it encrypts the index and publishes
  the documents. The byte-identical-concurrent-upload window is stated rather
  than implied.
- **CAL §8.14.1** carries the same obligation onto `FORGET SUBJECT`.

### Added in the template-identity batch (2026-08-19)

- **OMS §28.4.1 definition identity** — every stored definition (`qry:`, `tpl:`)
  carries **two** stamps that an implementation MUST NOT collapse: `updated_at`
  (the latest-wins replication tiebreaker and the "when did this change") and
  `definition_hash` (SHA-256 over the NFC-normalized body — the stable identity
  of *this* revision). Neither substitutes for the other: a timestamp cannot
  identify a revision (two hosts editing to the same body get different stamps
  for identical content), and a hash cannot order revisions. Answers
  `openmemoryspec/oms#12 §2`.
- **CAL §10.6.3 Template identity, revision, and `DROP TEMPLATE`** — templates
  are stored definitions under `tpl:<name>` on §8.18.1's terms; both stamps are
  reported by `DESCRIBE TEMPLATES`; built-ins are code, not metadata, and are
  neither droppable nor persistable; dropping a template another template
  `EXTENDS` is refused rather than left dangling.
- **`template:<ns>/<name>@<definition_hash>`** — Recommendation targets may be
  revision-qualified, and a Recommendation applied to a definition MUST record
  the **replaced body** on its applied audit grain, so its inverse reinstates
  the revision it displaced rather than "whatever was there before". §8.12
  rule 4 additionally asks `summary.template_id` to carry its revision: a bare
  name renders a summary whose text can silently change after a reviewer
  approved it.

### Added in the trigger batch (2026-08-19)

- **OMS §8.13 Trigger (type `0x0D`)**: a standing rule that starts a Workflow —
  interval, calendar schedule, one-shot, polling an external source, a condition
  over this memory, a host-delivered webhook, a manual dispatch, or a boolean
  gate over other triggers. Typed, queryable fields; connector transport
  configuration stays in `config` under the §A.7 `int:` keys. Binding is
  normatively **trigger → workflow**: a Workflow is content-addressed, so a plan
  accumulating trigger references would change address whenever one was added.
- **OMS §27.8 Trigger Execution Contract**: what an implementation that
  *evaluates* triggers must do — journal every firing, deduplicate on the
  declared key, seed rather than replay history on first evaluation, keep
  evaluation state host-local, arbitrate concurrent evaluation through the
  store, back off on failure, and state its DST semantics rather than assume
  them. Answers `openmemoryspec/oms#12 §4`.
- **OMS §17.4 Optional modules** and the `triggers` module declaration,
  mirroring CAL's tier modules: modules gate operations, not portability.
- **§A.7 `int:allowed_outbound_hosts`**: the destinations a connector may reach.
- **CAL: `RECALL triggers`** — the closed grain-type set grows to 13, with a
  queryable field set (`kind`, `workflow`, `connector`, `scope`, `enabled`,
  `cron`).

### Removed in the trigger batch (2026-08-19)

- **OMS §27.6 "Trigger Definitions via Observation Grains"** — removed, not
  deprecated. It was non-conformant against this specification's own closed
  enums (§25 `observation_mode`, §26 `observation_scope`), its `context`-carried
  configuration could not be queried, and §24's observer domains have no bucket a
  standing rule belongs in. Keeping text that instructs implementers to write
  invalid grains is worse than removing it. §8.13 replaces it, and §27.6 carries
  a migration note.
- **`Workflow.trigger` (§8.4)** — **breaking.** A free-text "activation
  condition" that no implementation evaluated, so it described an activation
  that could not activate anything. Old blobs are unaffected: an unknown field
  is preserved and ignored (§19.4). CAL's `ON "…"` clause on `ADD`/`SUPERSEDE
  workflow` goes with it; `ON` remains a keyword, used by `GRANT`/`REVOKE`.

### Added in the anonymization batch (2026-08-16)

- **OMS §10.5 Pseudonymized Egress**: the placeholder disclosure model for
  external models; the reserved `anon:<namespace>` policy prefix
  (write-if-absent replication, fail-closed unreadable rows,
  minimum-reader-version stamping, keyed value-derived pseudonyms only on
  encrypted files); the reserved never-replicating `vault:` prefix (sealed
  under a file-key-derived subkey, erased with the subject, revealed only
  under grant + fingerprint-only audit); the `anonymized` response report
  (mapping ids only).
- **CAL §8.1.2 / §8.2.1 `WITH anonymize("<level>")`**: strengthen-only, on
  RECALL and per-ASSEMBLE-source — the file-declared policy remains the
  gate; query text can only add severity.
- **CAL §8.17 `REHYDRATE`** (Tier 3): the round trip's return leg;
  classification stated (re-identification is never a plain read), gated,
  audited by fingerprint; `CAL-E127 MappingUnknown`.

> This entry describes a **draft**. The `release/oms-v1.6` branch is the
> comment window; nothing below is normative until release.

### CAL 1.3 — the pillar changes, on purpose and in the open

CAL 1.0–1.2 promised "non-destructive by grammar; no unsafe mode." The
promise was true and had a hidden cost: real deployments still needed
erasure — GDPR, retention — so destruction happened anyway, outside the
language, ungoverned by any spec. Exiling destruction never prevented it; it
only prevented *specifying* it. 1.3 brings it inside, where grammar bounds
its shape (whole grains by hash, identity, or age — never a predicate, never
a key, no history rewrite, no `UNFORGET`), grants gate it per principal, and
the audit trail sees it. Every 1.0–1.2 document remains valid Tier-0/1 CAL;
for any session without grants, CAL is still exactly the language those
releases promised — now the shared floor, not a ceiling that pushed
dangerous operations off the books.

#### CAL 1.3 — Added

- **Four-tier capability model** (§2.2): Tier 0 Core, Tier 1 Evolve, Tier 2
  Destroy, Tier 3 Control. **Tiers gate operations, not portability** —
  interop rides the `.mg` file and Tier-0 reads; a read-only implementation
  remains fully conformant forever. New `cal_tiers` conformance declaration;
  `DESCRIBE capabilities` reports the host's tiers.
- **Tier 2 — Destroy** (§8.14): `FORGET <hash>` (reason optional but
  recorded), `FORGET SUBJECT <id> [WITH text_mentions] BECAUSE "…"`, and
  `PURGE OLDER THAN <duration> [TYPE <t>] BECAUSE "…"`. Requires the
  `delete`/`erase` verbs; fail-closed; one-way; every execution writes an
  in-memory audit Observation (§8.14.4) following the §8.12.1 audit pattern.
- **Tier 3 — Control** (§8.15): `GRANT`/`REVOKE` verbs on a namespace for a
  principal, `SHOW GRANTS`, `DESCRIBE PRINCIPAL` — over grant grains stored
  in the memory itself (OMS §12.6). `REVOKE` is retraction-by-supersession;
  grant history is append-only. `DEFINE ROLE` reserved, deferred.
- **Tier 3 — Governance** (§8.16): `APPROVE`/`REJECT`/`APPLY`/`ROLLBACK`
  with `BECAUSE` mandatory as *syntax*, `RUN LOOP [FULL]`, and the
  `DESCRIBE loop|analyzers|outcomes|policy` reads. Self-approval refused
  against the creating actor and the triggering principal; observer type
  derives from the principal's host record, never statement text; refused
  inside `BATCH`, saved queries, and `proposal_cal`. Loop *policy* writes
  stay permanently outside CAL (self-licensing).
- **Principal-bound sessions and the credential/policy split** (§18.4):
  hosts authenticate and resolve principals; grants resolve from the
  memory; credentials never enter grains or statements; owner sessions
  preserve the single-operator experience.
- **Authorization error codes** CAL-E121–E126, with CAL-E121 messages
  naming the missing verb and the `GRANT` that would confer it.

#### CAL 1.3 — Changed

- §2 Safety Model restated (see narrative above); `FORGET`/`PURGE` and
  `GRANT`/`REVOKE` moved from the blocked-token list to tier-gated
  keywords. `DELETE`, `ERASE`, `DESTROY`, `TRUNCATE` and all key/credential
  vocabulary remain permanently excluded.

### OMS 1.6 — Added

- **§12.6 Authorization**: principals (humans and agents uniformly, DIDs as
  the signed upgrade path), the closed verb set, namespace-scoped grants;
  **grant grains** — Facts in the new reserved `agent:authz` namespace
  (subject = grantee, `mg:permits`, canonical object string, grantor/reason
  in `context`); effective rights = live heads, fail-closed; revocation by
  retraction-by-supersession (`mg:revokes` stays consent-domain); owner
  bootstrap; grants replicate as sync while authoring requires
  `admin`/owner; the credential/policy split.

### OMS 1.6 — Changed

- §28.4 `delete`: may surface to authorized principals via CAL Tier 2; gains
  the one-way rule (additions roll back by retraction; erasure is final).
- §28.9: the "no CAL equivalent — structurally excluded" row now maps to
  Tier-2 `FORGET`/`PURGE` on hosts that implement them.
- §8.12 `proposal_cal`: MAY carry Tier-2 statements where the host
  authorizes destructive proposals (apply recorded not rollbackable);
  governance statements refused unconditionally.

---

## [1.5] — 2026-08-03

### Added

- **Recommendation grain type (`0x0C`, §8.12)** — new dedicated cognitive grain type for a governed, auditable **proposal to change memory or agent configuration**. Realized from the `0x0C–0xEF` reserved range following the Skill (`0x0B`) precedent. A Recommendation names a `target_ref` (`grain:` / `entity:` / `query:` / `template:` / `doc:` / `host:` scheme), the producing `analyzer` (`{id, params}`), a deterministic `summary` (`{template_id, args}` — never analyzer free prose), a computed `dedup_key`, and exactly one proposal (`proposal_cal` / `proposal_edit` / `proposal_data`). Optional: `severity`, `metric_snapshot`, `evidence_query`; evidence and scoring reuse the common `derived_from` (evidence hashes), `confidence`, `importance`, and `valid_to` (expiry). It never mutates memory by itself — it enters a propose → review → apply → roll-back lifecycle (§8.12.1) whose transitions are immutable audit Observation grains, and its review state (`rec_status`) is a rebuildable index-layer cache, so the content address is stable across lifecycle transitions while a change in *content* is a supersession sharing one `dedup_key` — `dedup_key`, not the content address, is what stays constant across a supersession chain. A supersession resets `rec_status` to `pending`, so an approval never carries forward to content no reviewer has seen. Byte values `0x01`–`0x0B` are unchanged; existing content addresses remain valid.
- **§6.13 Recommendation-Specific Fields** — compaction table for the type's fields (`tref`, `anlz`, `summ`, `ddk`, `pcal`, `pedit`, `pdata`, `sev`, `msnap`, `evq`). (Former §6.13 Compaction Rules renumbered to §6.14.)
- **Recommendation compaction (Appendix C)** — added a Recommendation-Specific Fields compaction block.
- **Grain-type table (§3.1), glossary, ABNF `type-byte` (Appendix B), the §8 type count, and conformance Level 1** updated to include Recommendation (`0x0C`); reserved range narrowed to `0x0D–0xEF`.
- **`rec_status` as an index-layer field** — enumerated in §5.6, §6.1 (short key `rstat`), §11.7 and §28.3 alongside `superseded_by` and `verification_status`, so the general prohibition on writers setting index-layer fields binds it. §11.7 classifies it **Rebuildable**: an importer MUST reconstruct it from the audit chain and MUST NOT trust an imported value.
- **Normative `dedup_key` construction (§8.12 rule 5)** — a concrete recipe (lowercase hex SHA-256 over NUL-separated analyzer-family, `target_ref`, action-kind, with defined normalization), so independent implementations derive identical keys and dedup survives import, federation and forking.
- **Applier obligations (§8.12 rules 8–9)** — a stored `proposal_cal` executes under the **approving** principal's capability (never the analyzer's), MUST NOT write outside the recommendation's own namespace, and is bounded by CAL's `MAX_QUERY_LENGTH` and Tier-1 write quotas. `proposal_edit` requires a `doc:` allowlist check and a `base_digest` match; `proposal_data` requires host-schema validation.

### CAL 1.2 — Added

- **Recommendation grain support** — `recommendation`/`recommendations` added to the closed grain-type set (§5.1); `RECALL recommendations` with a type-specific field set (`target_ref`, `analyzer`, `severity`, `dedup_key`, `rec_status`); `<recommendation>` content-projection rule; TOON column set; field-count row; grammar productions `grain_type_plural`, `grain_type_singular`, `grain_field_name`, and new `recommendation_field`; the `DESCRIBE` type listing and the JSON `valid_values` enum. Recommendation is **query-only** — engine-emitted and lifecycle-gated — so it is deliberately absent from the CAL-addable set (no `ADD recommendation`; lifecycle transitions never occur via `ADD`/`SUPERSEDE SET`). Backward-compatible.
- **`ELEMENT` shorthand for custom templates (§10.6.1)** — a template whose only section is `ELEMENT` may now be written as a single string: `DEFINE TEMPLATE oneliner AS "{{grain.subject}}: {{grain.content}}"` and, inline, `FORMAT TEMPLATE "<text>"`. Both are defined **by equivalence** to `ELEMENT { <text> }`, so the shorthand is per-element rather than whole-result and introduces no second rendering model, no new variables, and no new scope; `EXTENDS` (§10.7), the §10.8 limits, the §10.2 Mustache subset and the §10.5 variable set all apply unchanged. Named and inline forms are disambiguated by token class — `TEMPLATE <identifier>` is a reference, `TEMPLATE <string>` is a body, `TEMPLATE { ... }` is a section list — which is why `template_name` remains `identifier`. New grammar production `template_shorthand`; `define_template_stmt` takes `( template_body | template_shorthand )`. A definition may not combine the two forms, and a template needing `HEADER`, `FOOTER`, an explicit `ELEMENT_SUMMARY` or `SOURCE_BREAK` still uses the section list. Backward-compatible.

### CAL 1.2 — Fixed

- **`format_spec` did not admit the string-bodied inline template it documents** — the §10.1.1 alias examples (`FORMAT [json AS structured, TEMPLATE "{{subject}}: {{object}}" AS oneliner]`), the §14.2.1 multi-format response example, and the §27 quick reference have used `TEMPLATE "<text>"` since CAL 1.1, but the §4 grammar offered only `"TEMPLATE" , template_name` (an identifier) and `"TEMPLATE" , "{" , template_body , "}"` (a section list) — a quoted string matched neither, making four of the specification's own examples ungrammatical. `format_spec` now includes `"TEMPLATE" , string_literal`, and §10.6.1 gives it semantics. Errata for CAL 1.1; no conforming query changes meaning.
- **Those same examples used variables outside the closed set** — they interpolated bare `{{subject}}`/`{{object}}`, which predate the §10.3–10.5 Content Projection Model. §10.5 defines only the `grain.` namespace and §10.8 fixes the variable set as **Closed** with undefined variables rendering as the empty string, so the §14.2.1 example claimed an output (`"alice: dark mode"`) that a conforming engine would have rendered as `": "`. Updated to `{{grain.subject}}`/`{{grain.object}}` in §10.1.1, §14.2.1 and §27. Errata for CAL 1.1.

### SML 1.1 — Added

- **`<recommendation>` element** — SML tag for the Recommendation grain, rendering the proposal summary with `target`/`severity` attributes; added to the flat tag set and the all-types rendering example.
- **SML bumped 1.0 → 1.1** with a version-history appendix and document-status footer. The element set changed, so the version had to move: two documents both labelled "SML 1.0" would otherwise differ in element set with nothing to distinguish them. Additive — every SML 1.0 document remains a valid SML 1.1 document.

---

## [1.4] — 2026-06-13

### Added

- **Skill grain type (`0x0B`, §8.11)** — new dedicated cognitive grain type for packaged, reusable agent capabilities. Promotes the former §28.8 Skill Convention (a `Fact + mg:capable_of` pattern) to a first-class type, following the precedent set by Consent (§8.10). A Skill grain is **hybrid**: durable *definition* fields specify what the capability is and how to perform it, while optional *learned-competence* fields record a given agent's mastery. Required fields: `name`, `description`, `created_at`. Optional definition fields: `instructions`, `when_to_use`, `version`, `allowed_tools`, `resources`, `dependencies`, `input_modalities`, `output_modalities`, `domain`. Optional learned-competence fields: `holder_did`, `proficiency`, `practice_count`, `last_practiced_at`, `strategies` (each referencing a Workflow grain), `transferable`. A grain may be a pure definition (no `holder_did`/`proficiency`) or a held instance. Byte values `0x01`–`0x0A` are unchanged; existing content addresses remain valid.
- **§6.12 Skill-Specific Fields** — compaction table for the Skill grain's type-specific fields, grouped definition/learned (`skname`, `instr`, `wtu`, `sver`, `atls`, `res`, `deps`, `imod`, `omod`, `dom`, `hdid`, `prof`, `prcnt`, `lpa`, `strat`, `xfer`). (Former §6.12 Compaction Rules renumbered to §6.13.)
- **Skill compaction (Appendix C)** — added a Skill-Specific Fields compaction block.
- **`embedding_text` common field (§6.1)** — optional `string` (compact key: `et`) providing source text for vector embedding and full-text indexing. When present, implementations SHOULD use this value instead of the grain's default per-type text representation. Enables document-derived grains to preserve source paragraph context for retrieval while maintaining structured subject/relation/object triples. Benchmarked at +29.4pp Recall@10 improvement on a 28-grain refund policy dataset. See `proposals/embedding-text-field.md`.
- **Appendix C** — added `"et": "embedding_text"` to core field compaction table.
- **Workflow grain type redesigned as directed graph (§8.4)** — Workflow grains now model directed graphs instead of flat step lists. New fields: `nodes` (array[string]), `edges` (array[map] with `src`, `dst`, `cond`, `max_cycles`), `bindings` (map[string→string] mapping node IDs to Tool definition grain hashes), `retries` (map[string→int]). Supports sequential, parallel fork/join, conditional branching, and bounded cycles. Three-tier node-to-Tool resolution (bound → named → abstract). Replaces `steps` array.
- **Workflow structural semantics (§8.4)** — node types inferred from graph topology: fork (multiple outgoing edges), AND-join (multiple incoming edges), decision point (conditional edges), terminal (no outgoing edges). Entry point is first element of `nodes`.
- **`mg:has_graph` relation** — replaces `mg:requires_steps` for Workflow grains. Reflects the directed graph model.

### Changed

- **Type renames (breaking):** `"belief"` → `"fact"` (0x01), `"action"` → `"tool"` (0x05). Byte values unchanged; existing content addresses remain valid. The `Fact` grain is the (subject, relation, object) semantic triple with confidence; the `Tool` grain records tool/action invocations and executions. Updates the grain-type table (§5), the type-specific field headings, and the relation vocabulary. Legacy `"belief"`/`"action"` type strings are not accepted.
- **§28.8 retitled** "Skill Convention" → "Skill Lifecycle and Transfer" — now defines lifecycle, transfer, and discovery over the dedicated Skill grain (§8.11) rather than the legacy `Fact + mg:capable_of` object-map pattern; discovery/transfer queries use `type=skill`. `mg:capable_of` now maps to the Skill grain in the §8 relation vocabulary (retained for backward compatibility).
- **Grain-type table (§3.1), glossary, ABNF/CDDL `type-byte`, and conformance Level 1** updated to include Skill (`0x0B`); reserved range narrowed to `0x0C–0xEF`.
- **Workflow grain description** — "Learned action sequence — procedural memory" → "Directed graph of procedural steps — plans, pipelines, processes".
- **Workflow fields** — `steps` (array[string]) and `trigger` (string) replaced by `nodes`, `edges`, `bindings`, `retries`, and optional `trigger`. Edge schema uses nested `src`/`dst`/`cond`/`max_cycles` fields with compaction keys.
- **`mg:requires_steps`** renamed to **`mg:has_graph`** in the `mg:` standard relation vocabulary.

### CAL 1.1 — Added

- **Skill grain support** — `skill`/`skills` added to the closed grain-type set (§5.1); `RECALL skills` with type-specific fields (`name`, `version`, `domain`, `holder_did`, `proficiency`, `transferable`, `practice_count`, `last_practiced_at`); `ADD skill` (required SET fields `name`, `description`); `<skill>` content-projection rule; TOON column set; field-count row. Grammar productions `grain_type_plural`, `grain_type_singular`, `grain_field_name`, and new `skill_field` updated. `mg:capable_of` added to the `AGENCY` relation-category shortcut.
- **Multi-format output (§10.1.1)** — `FORMAT` and `AS` clauses now accept bracketed format lists (`FORMAT [markdown, json]`) with optional aliases (`FORMAT [json AS structured, markdown AS report]`). Single query execution produces multiple renderings. New multi-format response shape (`"formats"` object, §14.2.1). Maximum 5 formats per list. New error codes: `CAL-E110` (too many formats), `CAL-E113` (duplicate format key).
- **Workflow graph syntax for ADD/SUPERSEDE (§8.8.1)** — dedicated graph syntax for workflow grains using `->` (sequential edges), `()` (parallel fork/join), `WHEN` (conditional edges), `* N` (retry bounds), `BIND` (node-to-Action mapping), and `ON` (trigger clause). Full graph replacement on SUPERSEDE.
- **`BECAUSE` alias** — accepted as synonym for `REASON` in ADD, SUPERSEDE, and workflow statements.

### CAL 1.1 — Changed

- **Pipeline operators removed** — bare pipe syntax (`| ORDER BY`, `| LIMIT`, `| COUNT`, etc.) replaced by direct clause syntax (`ORDER BY`, `LIMIT`, `COUNT`). All examples and grammar productions updated. Backward-compatible at the semantic level.
- **Workflow query fields** — `steps` field replaced by `node` and `binding` for grain-type-specific querying.
- **Workflow content projection** — `nodes` joined with `->` arrow syntax replaces numbered `steps` list in SML/template output.
- **`GrainTypeNotAddable` (CAL-E051)** — addable grain-type set is now Fact, Observation, Goal, Workflow, Skill; the error message was updated to match.

### CAL 1.1 — Fixed

- **Stale `belief`/`action` literals** — corrected leftovers the rename missed: `grain_type_plural` (`beliefs`/`actions` → `facts`/`tools`), `RECALL MY beliefs` → `RECALL MY facts`, the TOON column table and examples, and a `"grain_type": "beliefs"` JSON example.

### SML — Added

- **`<skill>` element** — new tag mapping to the Skill grain; added to the all-grain-types comprehensive example. Default content is the skill `description`; default attributes `name`, `proficiency`?, `domain`?.

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
