# CAL (Context Assembly Language) Specification v1.3

**Status:** Draft for Comment (CAL 1.3 RFC) | **Date:** 2026-08-10 | **Version:** 1.3-draft | **Classification:** Experimental
**Part of:** [Open Memory Specification (OMS) v1.6 draft](./SPECIFICATION.md)

> **Draft status.** Version 1.3 restates the safety pillar — from "non-destructive
> by grammar" to **append-only by construction for evolution, authorization-gated
> for destruction** — and adds the Tier 2 (Destroy) and Tier 3 (Control) statement
> families. Every CAL 1.0–1.2 document remains valid, and a session without grants
> is exactly the language 1.0–1.2 specified. Sections changed by this draft are
> normative only upon release.
---

## Table of Contents

- [Abstract](#abstract)
1. [Introduction](#1-introduction)
2. [The Safety Model](#2-the-safety-model)
3. [Lexical Structure](#3-lexical-structure)
4. [Grammar (EBNF)](#4-grammar-ebnf)
5. [Type System](#5-type-system)
6. [OMS Grain Type Integration](#6-oms-grain-type-integration)
7. [mg: Relation Vocabulary](#7-mg-relation-vocabulary)
8. [Statement Semantics](#8-statement-semantics)
9. [Semantic Shortcuts](#9-semantic-shortcuts)
10. [FORMAT System](#10-format-system)
11. [Streaming Protocol](#11-streaming-protocol)
12. [Domain Profile Querying](#12-domain-profile-querying)
13. [Store Protocol Mapping](#13-store-protocol-mapping)
14. [Response Model](#14-response-model)
15. [Dual Wire Format](#15-dual-wire-format)
16. [Internationalization](#16-internationalization)
17. [Execution Model](#17-execution-model)
18. [Capability Token Model](#18-capability-token-model)
19. [Policy Integration](#19-policy-integration)
20. [Threat Model](#20-threat-model)
21. [Audit Trail](#21-audit-trail)
22. [Error Model](#22-error-model)
23. [Compliance Checks](#23-compliance-checks)
24. [Conformance Levels](#24-conformance-levels)
25. [Versioning and Evolution](#25-versioning-and-evolution)
26. [Interface Integration](#26-interface-integration)
27. [LLM System Prompt Template](#27-llm-system-prompt-template)
- [Appendix A: Complete EBNF Grammar](#appendix-a-complete-ebnf-grammar)
- [Appendix B: JSON Schema References](#appendix-b-json-schema-references)
- [Appendix C: Error Code Registry](#appendix-c-error-code-registry)
- [Appendix D: Reserved Words](#appendix-d-reserved-words)
- [Appendix E: Queryable Fields Reference](#appendix-e-queryable-fields-reference)
- [Appendix F: Version History](#appendix-f-version-history)

---

## Abstract

The **Context Assembly Language (CAL)** is a companion specification to the Open Memory Specification (OMS). It defines a non-destructive, deterministic, LLM-native language for **assembling agent context from persistent memory**.

CAL allows AI agents to **recall** memory, **assemble** context windows from multiple memory sources with budget constraints, **evolve** memory append-only, and -- for principals explicitly granted the capability -- **destroy** whole grains (tombstone or bulk erasure) and **control** who may do which of those. The core safety guarantee is two-part: *history cannot be rewritten* (structural -- no statement mutates a stored blob, no destruction takes a predicate), and *destruction and control execute only for an authorized principal* (Tier 2/3, fail-closed). A session without grants is exactly the non-destructive CAL that versions 1.0--1.2 specified -- that language is the floor of every conforming implementation, not its ceiling.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

---

## 1. Introduction

### 1.1 What is CAL?

CAL is a **non-destructive, deterministic, LLM-native context assembly and evolution language** for memory databases that implement the Open Memory Specification.

CAL is a **non-destructive, deterministic, LLM-native context assembly and evolution language** -- answering "what should be in the agent's context window right now?"

Key capabilities:

| Dimension | CAL (Context Assembly Language) |
|-----------|---------------------------------|
| **Primary question** | "What should be in the context window?" |
| **Core operation** | ASSEMBLE (compose context from multiple sources) |
| **Output model** | Flat, semantic, LLM-native context (content projection from OMS grains) |
| **Token awareness** | Native (BUDGET clause, progressive disclosure) |
| **Multi-source** | First-class (FROM clause with priority) |
| **Format control** | Built-in (FORMAT clause, AS clause, custom templates) |
| **Progressive disclosure** | Native (WITH progressive_disclosure) |
| **Batching** | BATCH statement for multiple queries |
| **Schema discovery** | DESCRIBE statement for introspection |
| **Streaming** | Native (STREAM clause on ASSEMBLE) |

### 1.2 Design Goals

1. **Non-destructive by grammar, not by convention.** The parser rejects destructive tokens. There is no "unsafe mode."
2. **Append-only evolution.** Writes create new grains -- they never modify or delete existing ones. Every change is traceable and revertible.
3. **Context-window-aware.** CAL understands that its output will be consumed by an LLM with finite context. Budget allocation, progressive disclosure, content projection, and format control are first-class concerns. Output is shaped for LLM comprehension, not storage fidelity.
4. **Multi-source composition.** ASSEMBLE makes composing context from multiple memory sources a single, declarative operation.
5. **Bounded execution.** Every query has a compile-time-determinable upper bound on work.
6. **Policy-transparent.** CAL queries execute within the active policy (GDPR, HIPAA, etc.). CAL cannot override policy.
7. **LLM-ergonomic.** Keywords read like English. Common patterns have shortcuts (ABOUT, RECENT, SINCE, MY). Errors include suggestions. The grammar fits in approximately 1200 tokens.
8. **Deterministic.** Same query + same state = same results + same order. No randomness.
9. **Composable.** Queries nest, pipe, combine with set operations, and compose into ASSEMBLE blocks.
10. **Internationally aware.** CAL handles multilingual content natively -- Unicode normalization, cross-lingual search, bidi text, and locale-aware sorting are specified behaviors.
11. **Dual-format.** Every CAL statement has a bijective mapping between human-readable text (`text/cal`) and machine-readable JSON (`application/json+cal`). Neither is canonical -- they are equivalent.
12. **Versionable.** `CAL/1 RECALL ...` explicitly targets a spec version.

### 1.3 What CAL Is NOT

- **Not SQL.** No tables, no joins, no transactions. (Since 1.3, CAL does have a DCL -- `GRANT`/`REVOKE` -- the way SQL has since SQL-89.)
- **Not Turing-complete.** No loops, no recursion, no persistent variables.
- **Not an unguarded destructive language.** Destruction exists only as Tier 2: a tombstone of one grain by content address, or bulk erasure by identity or age -- never by predicate, and only for a principal granted `delete`/`erase`. CAL can never rewrite history: no statement mutates a stored blob. CAL cannot touch encryption keys or credentials -- ever, at any tier.
- **Not an authentication or key-management system.** The host proves who a principal is, custodies keys, and transports files; CAL governs what an authenticated principal may do inside a memory.
- **Not a transport protocol.** CAL defines a language. Transport (HTTP, gRPC, MCP, etc.) is implementation-specific.
- **Not a rendering engine.** CAL's FORMAT clause specifies *semantic structure*, not pixel-level presentation. The agent or UI decides how to render.
- **Not a storage mirror.** CAL output is a *projection* optimized for LLM consumption, not a serialization of the underlying OMS grain structure. Hashes, namespaces, and internal metadata stay in the machine envelope.

### 1.4 The Git Analogy

CAL's safety model maps directly to git:

| Git Operation | CAL Equivalent | Destructive? | In CAL? |
|---------------|---------------|-------------|---------|
| `git log` | `RECALL` | No | Yes (Tier 0) |
| `git show` | `RECALL WHERE hash = ...` | No | Yes (Tier 0) |
| `git add` + `git commit` (new file) | `ADD` | No (append-only) | Yes (Tier 1) |
| `git commit` (amend existing) | `SUPERSEDE` | No (append-only) | Yes (Tier 1) |
| `git revert` | `REVERT` | No (creates new commit) | Yes (Tier 1) |
| `git reset --hard` | -- (rewrites history) | **Yes** | **No -- structurally impossible** |
| Repo deletion under admin rights | `FORGET` / `PURGE` | **Yes** (whole grains, one-way) | Tier 2 (authorization-gated) |
| `git push --force` | Crypto-erasure | **Yes** (destroys keys) | **No** |

The line between the append-only tiers (0/1) and Tier 2 is: **can the operation be undone by another append-only operation?** If yes, it needs no special authority. If no -- a tombstone or an erasure -- it executes only for a principal explicitly granted it, it writes an audit Observation, and it is final: destruction takes a hash, an identity, or an age, never a predicate, and there is no un-forget. Key destruction stays out of the language entirely.

### 1.5 Relationship to OMS

CAL operates on the 13 grain types defined by OMS v1.6: Fact, Event, State, Workflow, Tool, Observation, Goal, Reasoning, Consensus, Consent, Skill, Recommendation, Trigger. CAL treats this as a **closed set** -- custom types are not queryable via CAL.

CAL extends the Store Protocol Convention defined in OMS §28.4 ([SPECIFICATION.md](./SPECIFICATION.md)) with a formal query language. Where OMS defines the `query`, `search`, and `supersede` store operations, CAL provides a structured, deterministic syntax for invoking them safely.

---

## 2. The Safety Model

### 2.1 The Core Guarantee

> **CAL cannot rewrite history. This is not a policy check. It is a structural impossibility.**
>
> CAL *can* evolve data -- by creating new grains that supersede old ones; the old grains survive, and every evolution is traceable and revertible. Since 1.3, CAL *can also* destroy whole grains -- a tombstone by content address, or bulk erasure by identity or age -- but **only** for a principal explicitly granted that verb, only with an audit Observation written, and only in a shape the grammar bounds: **a hash, an identity, or an age -- never a predicate, and never a key.** For any session without such grants, CAL is exactly the non-destructive language of versions 1.0--1.2.

The guarantee is enforced at three reinforcing levels:

| Level | Mechanism | What It Prevents |
|-------|-----------|-----------------|
| **Grammar** | No production mutates a stored blob; no destruction production accepts a predicate (`FORGET WHERE` does not parse); no key/credential vocabulary exists | Parser cannot produce a history-rewriting or predicate-destruction AST node |
| **Type System** | `CalStatement` is a closed, per-tier statement set (§8). Destruction variants exist only in the Tier 2 family; adding any other destructive variant requires modifying this specification | No code path from a read/evolve AST to any destructive method |
| **Authorization + API Surface** | The executor receives a constrained facade; destructive facade methods execute only after the session principal's grants (OMS §12.6) allow the statement's verb; key management is never on the facade | Destruction without an authorized principal is refused fail-closed; key destruction is structurally inaccessible |

### 2.2 Four-Tier Capability Model

| Tier | Name | What It Can Do | Gate |
|------|------|---------------|------|
| **0** | Core (default) | Query, count, explain, assemble, describe, batch. Cannot modify anything. | None. Default for all CAL sessions. |
| **1** | Evolve (opt-in) | Add new grains, supersede, revert, view history. Append-only; never deletes. | Server opt-in; `write`/`supersede` verbs where Tier 3 is implemented, capability token with write quotas otherwise. |
| **2** | Destroy | `FORGET <hash>`, `FORGET SUBJECT`, `PURGE OLDER THAN`. Whole-grain tombstone and bulk erasure -- one-way, audited. | `delete`/`erase` verbs. Requires *an* authorization model: host-defined authorization suffices (Tier 2 without Tier 3 is legal). A host with no authorization model MUST refuse Tier 2. |
| **3** | Control | `GRANT`/`REVOKE` (DCL), the governance lifecycle (`APPROVE`/`REJECT`/`APPLY`/`ROLLBACK`/`RUN LOOP`), the definition surface (`DEFINE QUERY`/`DROP QUERY`, `DEFINE TEMPLATE`/`DROP TEMPLATE`), principal-bound sessions. | `admin` and `loop.*` verbs; implies the grant model (OMS §12.6) that Tier 2 consumes. |

**Tiers gate operations, not portability.** Interoperability rides the `.mg` file and Tier-0 reads: any Core implementation can read any memory. A read-only implementation declares Tier 0 and remains fully conformant forever -- nothing in Tier 2/3 is required of it. Hosts declare their tiers, and `DESCRIBE capabilities` MUST report them, so an agent (or a test suite) learns what it may attempt before attempting it.

### 2.3 Formal Safety Claims

**"CAL cannot rewrite history because..."** No statement mutates a stored blob. `SUPERSEDE` and `REVERT` create new grains and update index-layer fields only; `FORGET` tombstones a whole grain (the store's compliance `delete`, OMS §28.4) and is **one-way** -- there is no `UNFORGET`, and a host MUST NOT provide an un-tombstone mechanism. Additions roll back by *retraction* (a supersession with `invalidation_type: "retraction"`); tombstones and erasure are final.

**"CAL destruction cannot happen without an authorized principal because..."** Every Tier-2 statement executes only after the session principal's grants allow its verb (`delete` for `FORGET <hash>`, `erase` for `FORGET SUBJECT`/`PURGE`). Sessions without grants are Tier 0/1. Enforcement is fail-closed: an unknown principal, an absent grant, or a host without an authorization model all refuse. Every Tier-2 execution writes an audit Observation grain (§8.14.4).

**"CAL cannot destroy by predicate because..."** The destruction grammar accepts exactly three target shapes: a content address, a subject identity, or an age window. There is no production for `FORGET WHERE` or any filter-driven deletion, so a single statement's blast radius is always explicit in its text.

**"CAL cannot trigger key destruction because..."** Crypto-erasure requires key management, and no key vocabulary exists in the grammar (§2.4) and no key method exists on the executor facade. Keys and credentials are host objects at every tier.

**"CAL ADD cannot destroy data because..."** ADD creates a new grain via the OMS store `put` operation (OMS §28.4). It does not modify or reference any existing grain. The grain count increases by one.

**"CAL SUPERSEDE cannot destroy the original grain because..."** The OMS store protocol defines `supersede` as: write the new grain, then update the old grain's index-layer fields (`superseded_by`, `system_valid_to`). The old grain's blob is never touched. It remains readable.

**"CAL REVERT cannot destroy data because..."** REVERT creates a *new* grain (copying content from a previous version) and then supersedes the current head. Three grains exist afterward: original, supersession, and revert. Nothing is deleted.

**"CAL cannot cross namespace boundaries because..."** Every CAL query carries a `CapabilityToken` cryptographically bound to a namespace. The executor overwrites any namespace in the parsed query with the token's namespace.

**"CAL cannot exhaust resources because..."** Hard limits are specified: `MAX_LIMIT=1000`, `MAX_QUERY_LENGTH=8192 bytes`, `QUERY_TIMEOUT=5000ms`. Tier 1 operations have additional write quotas: `MAX_ADD_PER_MINUTE=20`, `MAX_SUPERSEDE_PER_MINUTE=10`, `MAX_REVERT_PER_MINUTE=5`. These cannot be overridden by query syntax.

### 2.4 Grammar-Level Exclusions

The following tokens do **not exist** in CAL's lexer or grammar, at any tier:

```
DELETE, ERASE, DESTROY, TRUNCATE,                           -- Predicate/unbounded destruction
INSERT, CREATE, WRITE, STORE,                               -- Unconstrained creation
KEY, ENCRYPT, DECRYPT, ROTATE, MASTER, DEK, SECRET, TOKEN,  -- Key/credential management
POLICY, SEAL, UNSEAL, CONSENT, RESTRICT,                    -- Sealed policy / consent records
SCHEMA, PARTITION, INDEX, MIGRATION                         -- Schema
```

If these appear in a query, they are parse errors, not recognized keywords. This list is permanent: no future tier unblocks it. (`POLICY` here means the sealed policy and loop-policy *writes*; the read-only `DESCRIBE policy` target is an identifier, not this keyword.)

**Changed in 1.3:** `FORGET` and `PURGE` moved from this list to the Tier 2 keyword set, and `GRANT`/`REVOKE` to the Tier 3 keyword set (§3.2). The block on them existed to force a specification-level decision before any destructive or control surface entered the language -- this revision is that decision, and the statements it admits are bounded by §2.1--§2.3.

**Also changed in 1.3 -- `DROP`, and why the permanence claim survives it.**
`DROP` leaves this list, admitted by a production with a **closed two-target
set**: `DROP TEMPLATE` (§10.6.3) and `DROP QUERY` (§8.18.4). Neither target is
memory. A template and a saved query are *host metadata* -- definitions the
memory carries so they travel with the file (OMS §28.4.1), never grains, never
content-addressed, never returned by a read. There is no production by which
`DROP` reaches a grain, so the claim this list encodes -- that no CAL statement
destroys memory outside the audited, authorization-gated Tier-2 shapes -- is
unchanged. What was permanent was the exclusion of unbounded destruction *of
memory*; the token was blocked as the nearest available proxy for that, and
this revision replaces the proxy with the property itself. Removing a
definition is idempotent and lossless against memory: every grain the query
ever recalled is still there, and re-running the `DEFINE` restores it exactly.

`UNDEFINE` remains reserved (Appendix D) and unspecified. It was held for these
semantics; `DROP TEMPLATE`/`DROP QUERY` are the spelling that shipped, so
`UNDEFINE` stays reserved rather than becoming a second one -- an
implementation MUST NOT accept it as an alias.

Note: Tier 1/2/3 keywords are always **parsed** (so `EXPLAIN FORGET ...` works as a dry-run even where execution is unavailable). The **executor** rejects non-EXPLAIN statements above the session's effective tier: `CAL-E044: Tier1NotEnabled` for Tier 1, `CAL-E121`/`CAL-E122` for Tier 2/3 (§22). This two-layer approach ensures `EXPLAIN` can always preview an operation without risk.

---

## 3. Lexical Structure

### 3.1 Character Set

CAL queries are UTF-8 encoded. Keywords are case-insensitive (`RECALL` = `recall` = `Recall`). Implementations MUST reject queries containing invalid UTF-8 sequences (error `CAL-E070: InvalidUTF8`).

### 3.2 Keywords

All keywords are listed exhaustively.

**Tier 0 (Read) keywords:**
```
RECALL, ASSEMBLE, WHERE, AND, OR, NOT, IN, BETWEEN, LIMIT, OFFSET,
ORDER, BY, ASC, DESC, WITH, EXPLAIN, SCOPE,
UNION, INTERSECT, EXCEPT,
SELECT, COUNT, FIRST, GROUP, SUBJECTS, OBJECTS, HASHES, PROJECT,
INCLUDE, EXCLUDE, IS, NULL, TRUE, FALSE,
EXISTS, HISTORY, DESCRIBE, BATCH, COALESCE, REPORT,
ABOUT, RECENT, SINCE, LIKE, MY, CONTRADICTIONS, AS,
FOR, FROM, BUDGET, PRIORITY, FORMAT,
LET, THREAD,
STREAM, TEMPLATE, DEFINE, EXTENDS,
HEADER, ELEMENT, ELEMENT_SUMMARY, ELEMENT_OMIT, SOURCE_BREAK, FOOTER,
DIFF, PROJECT,
CAL                                                       -- version prefix
```

**Tier 1 (Evolve) keywords (always parsed; execution requires Tier 1 enabled -- see section 2.4):**
```
ADD, SUPERSEDE, REVERT, SET, REASON
```

**Tier 2 (Destroy) keywords (new in 1.3; always parsed; execution requires the `delete`/`erase` verb -- see sections 2.2 and 8.13):**
```
FORGET, PURGE, SUBJECT, OLDER, THAN, TYPE, BECAUSE
```

**Tier 3 (Control) keywords (new in 1.3; always parsed; execution requires the `admin`/`loop.*` verbs -- see sections 8.14 and 8.15):**
```
GRANT, REVOKE, ON, TO, SHOW, GRANTS, PRINCIPAL,
APPROVE, REJECT, APPLY, ROLLBACK, RUN, LOOP, FULL,
QUERY, QUERIES, DESCRIPTION, DROP                         -- definition surface (§8.18, §10.6.3)
```

`RUN` serves two statements distinguished by their next token: `RUN LOOP`
(§8.16) and `RUN "<name>"` (§8.18.3). A saved query name is a **string
literal**, so the two never collide -- `LOOP` is a keyword and can never be a
quoted name.

`SUBJECT` is shared: `FORGET SUBJECT` is Tier 2, and `REPORT SUBJECT` (§8.19) is the Tier-0 read over the same selector. `REPORT` itself is a Tier-0 keyword -- disclosure is a read at every tier, and a host that has disabled destruction still answers access requests.

`BECAUSE` was already an alias of `REASON` in `reason_clause`; Tier 2/3 statements conventionally write `BECAUSE`, and this specification uses that spelling for them throughout. `DEFINE ROLE` is **reserved but not specified** in 1.3 -- role bundles are deferred to a future revision; the 1.3 grant surface is verbs-only.

**Relation category keywords:**
```
PREFERENCE, KNOWLEDGE, PERMISSION, INTERACTION, AGENCY, LIFECYCLE, OBSERVATION
```

### 3.3 Identifiers

Field names are a closed set (not user-definable).

**Common fields (available on all grain types):**
```
query, subject, relation, object, user_id, namespace,
confidence, importance, tags, score, type, time, hash,
verification_status, source_type, contradicted,
recall_priority, epistemic_status
```

**Grain-type-specific fields (see section 6 for which types unlock which fields):**
```
role, session_id, parent_message_id, model_id, content,
context, plan,
nodes, edges, bindings, retries,
tool_name, tool_phase, is_error, tool_call_id,
observer_id, observer_type,
goal_state, assigned_agent, deadline, depends_on,
reasoning_type, premises, conclusion,
threshold, agreement_count, participating_observers,
consent_action, purpose, grantor_did, grantee_did, scope, expires_at,
name, version, domain, holder_did, proficiency, transferable,
practice_count, last_practiced_at,
target_ref, analyzer, severity, dedup_key, rec_status
```

### 3.4 Literals

| Type | Syntax | Example |
|------|--------|---------|
| String | Double-quoted, `\"` escape | `"alice"`, `"last 7 days"` |
| Number | Optional sign, digits, optional decimal | `0.8`, `-1`, `42` |
| Boolean | `true` / `false` | `true` |
| Array | Square brackets, comma-separated | `["tag1", "tag2"]` |
| Hash | `sha256:` + 8-64 hex chars | `sha256:a1b2c3d4...` |
| Parameter | `$` + identifier | `$user_id`, `$limit` |

### 3.5 Comments

Line comments only: `-- comment text`

### 3.6 Reserved Words (Future-Proofing)

See [Appendix D](#appendix-d-reserved-words) for the complete list. Reserved words cannot be used as unquoted identifiers even if not yet functional.

---

## 4. Grammar (EBNF)

This section provides the unified CAL/1 grammar. See [Appendix A](#appendix-a-complete-ebnf-grammar) for the complete, unabridged grammar.

```ebnf
(* CAL/1 Grammar -- Tier 0 + Tier 1 + All Extensions *)

query           = [ version_prefix ] , [ let_block ] , statement ;
version_prefix  = "CAL" , "/" , major_version ;
major_version   = digit+ ;

let_block       = { let_binding } ;
let_binding     = "LET" , "$" , identifier , "=" , recall_stmt , [ "|" , extractor ] , ";" ;
extractor       = "SUBJECTS" | "OBJECTS" | "HASHES" ;

statement       = explain_stmt | recall_stmt | assemble_stmt | set_stmt
                | exists_stmt | history_stmt | describe_stmt | batch_stmt
                | coalesce_stmt | run_query_stmt | report_subject_stmt
                | add_stmt | supersede_stmt | revert_stmt
                | forget_stmt | purge_stmt                        (* Tier 2 *)
                | grant_stmt | revoke_stmt | show_grants_stmt     (* Tier 3 *)
                | approve_stmt | reject_stmt | apply_stmt
                | rollback_stmt | run_loop_stmt
                | define_template_stmt | drop_template_stmt       (* Tier 3 *)
                | define_query_stmt | drop_query_stmt             (* Tier 3 *)
                | rehydrate_stmt ;                                (* Tier 3 *)

(* --- Tier 0: Read --- *)

explain_stmt    = "EXPLAIN" , ( recall_stmt | assemble_stmt | set_stmt
                | add_stmt | supersede_stmt | revert_stmt | batch_stmt
                | coalesce_stmt
                | forget_stmt | purge_stmt | grant_stmt | revoke_stmt ) ;

set_stmt        = "(" , query , ")" , set_op , "(" , query , ")" ;
set_op          = "UNION" | "INTERSECT" | "EXCEPT" ;

recall_stmt     = "RECALL" , [ "MY" ] , [ grain_type_plural ] , [ in_clause ] ,
                  [ about_clause ] , [ like_clause ] , [ since_clause ] ,
                  [ between_clause ] , [ thread_clause ] ,
                  [ where_clause ] , [ with_clause ] , [ pipeline ] ,
                  [ recent_clause ] , [ contradictions_clause ] , [ as_clause ] ;

assemble_stmt   = "ASSEMBLE" , [ context_name ] ,
                  [ for_clause ] ,
                  from_clause ,
                  [ budget_clause ] ,
                  [ priority_clause ] ,
                  [ format_clause ] ,
                  [ stream_clause ] ,
                  [ with_clause ] ;

exists_stmt     = "EXISTS" , ( hash_literal | parameter ) ;

history_stmt    = "HISTORY" , ( hash_literal | parameter ) , [ diff_clause ]
                | "HISTORY" , [ in_clause ] , "WHERE" , subject_clause , "AND" , relation_clause ,
                  [ as_of_clause ] ;

describe_stmt   = "DESCRIBE" , describe_target ;
describe_target = "grain_types" | "fields" , [ grain_type_singular ]
                | "capabilities" | "server" | "templates" | "grammar"
                | "PRINCIPAL" , principal                       (* Tier 3 read *)
                | "loop" | "analyzers" | "outcomes" | "policy" ; (* Tier 3 reads *)

batch_stmt      = "BATCH" , "{" , batch_entry , { "," , batch_entry } , "}" ;
batch_entry     = label , ":" , ( recall_stmt | exists_stmt | history_stmt
                | describe_stmt | coalesce_stmt ) ;
                (* Tier 2/3 statements are NOT batch entries -- see §8.14.3 *)

coalesce_stmt   = "COALESCE" , "(" , recall_stmt , "," , recall_stmt ,
                  { "," , recall_stmt } , ")" ;

define_template_stmt = "DEFINE" , "TEMPLATE" , template_name ,
                       [ extends_clause ] ,
                       ( template_body | template_shorthand ) ;

(* --- Clauses --- *)

context_name    = identifier ;
label           = identifier ;

for_clause      = "FOR" , string_literal ;
from_clause     = "FROM" , source , { "," , source } ;
source          = [ label , ":" ] , "(" , recall_stmt , ")"
                | [ label , ":" ] , let_ref ;
let_ref         = "$" , identifier ;

budget_clause   = "BUDGET" , positive_integer , ( "tokens" | "grains" ) ;
priority_clause = "PRIORITY" , label , { ">" , label } ;
format_clause   = "FORMAT" , format_value ;
format_value    = format_spec | "[" , aliased_format , { "," , aliased_format } , "]" ;
aliased_format  = format_spec , [ "AS" , identifier ] ;
format_spec     = format_type
                | preset_name
                | "TEMPLATE" , template_name
                | "TEMPLATE" , string_literal
                | "TEMPLATE" , "{" , template_body , "}" ;
format_type     = "markdown" | "json" | "yaml" | "text" | "sml" | "triples" | "toon" ;
preset_name     = "structured" | "readable" | "compact" | "data" ;

stream_clause   = "STREAM" , [ "{" , stream_option , { "," , stream_option } , "}" ] ;
stream_option   = "progress" | "budget" | "chunks" | "all"
                | "chunk_size" , "=" , positive_integer ;

about_clause    = "ABOUT" , string_literal ;
recent_clause   = "RECENT" , positive_integer ;
since_clause    = "SINCE" , string_literal ;
like_clause     = "LIKE" , string_literal ;
between_clause  = "BETWEEN" , value , "AND" , value ;
contradictions_clause = "CONTRADICTIONS" ;
as_clause       = "AS" , ( format_spec | "[" , aliased_format , { "," , aliased_format } , "]" ) ;

thread_clause   = "THREAD" , thread_target ;
thread_target   = string_literal | "FROM" , hash_literal ;

diff_clause     = "DIFF" , ( hash_literal | parameter ) ;
as_of_clause    = "AS" , "OF" , string_literal ;

in_clause       = "IN" , ( string_literal | "SCOPE" , string_literal ) ;

where_clause    = "WHERE" , condition , { "AND" , condition } ;

condition       = field_condition | grain_field_condition | query_condition
                | time_condition | type_condition | tag_condition
                | in_condition | hash_condition | meta_condition
                | relation_shortcut | domain_field_condition ;

field_condition   = field_name , comparator , value ;
grain_field_condition = grain_field_name , comparator , value ;
meta_condition    = meta_field_name , comparator , value ;
query_condition   = "query" , "=" , string_literal ;
time_condition    = "time" , "=" , string_literal
                  | "time" , "BETWEEN" , value , "AND" , value ;
type_condition    = "type" , "=" , string_literal ;
tag_condition     = "tags" , ( "INCLUDE" | "EXCLUDE" ) , array_literal ;
in_condition      = field_name , "IN" , "(" , ( value_list | subquery_extract ) , ")" ;
hash_condition    = "hash" , "=" , ( hash_literal | parameter ) ;
relation_shortcut = "relation" , "IS" , relation_category ;
domain_field_condition = domain_field , comparator , value ;

domain_field    = domain_prefix , ":" , identifier ;
domain_prefix   = "hc" | "legal" | "fin" | "rob" | "sci" | "con" | "int"
                | "mail" ;   (* new in 1.3, OMS Appendix A.8 *)

relation_category = "PREFERENCE" | "KNOWLEDGE" | "PERMISSION" | "INTERACTION"
                  | "AGENCY" | "LIFECYCLE" | "OBSERVATION" ;

subquery_extract = recall_stmt , "|" , extractor ;

meta_field_name = "recall_priority" | "epistemic_status" | "verification_status"
                | "source_type" ;

grain_field_name = event_field | state_field | workflow_field
                 | action_field | observation_field | goal_field
                 | reasoning_field | consensus_field | consent_field
                 | skill_field | recommendation_field ;

event_field     = "role" | "session_id" | "parent_message_id" | "model_id" | "content" ;
state_field     = "context" | "plan" ;
workflow_field  = "node" | "binding" ;
action_field    = "tool_name" | "tool_phase" | "is_error" | "tool_call_id" ;
observation_field = "observer_id" | "observer_type" ;
goal_field      = "goal_state" | "assigned_agent" | "deadline" | "depends_on" ;
reasoning_field = "reasoning_type" | "premises" | "conclusion" ;
consensus_field = "threshold" | "agreement_count" | "participating_observers" ;
consent_field   = "consent_action" | "purpose" | "grantor_did" | "grantee_did"
                | "scope" | "expires_at" ;
skill_field     = "name" | "version" | "domain" | "holder_did" | "proficiency"
                | "transferable" | "practice_count" | "last_practiced_at" ;
recommendation_field = "target_ref" | "analyzer" | "severity" | "dedup_key"
                | "rec_status" ;

comparator      = "=" | "!=" | ">=" | "<=" | ">" | "<" ;

field_name      = "subject" | "relation" | "object" | "user_id" | "namespace"
                | "confidence" | "importance" | "score"
                | "verification_status" | "source_type" | "contradicted" ;

subject_clause  = "subject" , "=" , value ;
relation_clause = "relation" , "=" , value ;

with_clause     = "WITH" , with_option , { "," , with_option } ;
with_option     = "superseded" | "score_breakdown" | "explanation" | "provenance"
                | "contradiction_detection" | "progressive_disclosure"
                | "summarize"
                | "diversity" , "(" , diversity_spec , ")"
                | "consistency" , "(" , consistency_level , ")"
                | "progressive_disclosure" , "(" , disclosure_level , ")"
                | "dedup" , "(" , field_name , ")"
                | "locale" , "(" , string_literal , ")"
                | "cache" , "(" , "ttl" , "=" , positive_integer , ")"
                | "anonymize" , "(" , anonymize_level , ")"
                | extension_option ;
diversity_spec  = "mmr" , [ "," , "lambda" , "=" , number ]
                | "threshold" , "," , number ;
consistency_level = "eventual" | "bounded" , "(" , number , ")" | "linearizable" ;
disclosure_level = "summary" | "headlines" | "full" ;
anonymize_level = "standard" | "strict" ;
extension_option = "x_" , identifier , [ "(" , value_list , ")" ] ;

pipeline        = { pipe_stage } ;
pipe_stage      = select_stage | order_stage | limit_stage | offset_stage
                | count_stage | first_stage | subjects_stage | objects_stage
                | hashes_stage | group_stage | project_stage ;
select_stage    = "SELECT" , field_name , { "," , field_name } ;
order_stage     = "ORDER" , "BY" , field_name , [ "ASC" | "DESC" ] ;
limit_stage     = "LIMIT" , positive_integer ;
offset_stage    = "OFFSET" , ( positive_integer | parameter ) ;
count_stage     = "COUNT" ;
first_stage     = "FIRST" ;
subjects_stage  = "SUBJECTS" ;
objects_stage   = "OBJECTS" ;
hashes_stage    = "HASHES" ;
group_stage     = "GROUP" , "BY" , field_name ;
project_stage   = "PROJECT" , project_spec , { "," , project_spec } ;
project_spec    = "content" , "(" , project_field , { "," , project_field } , ")"
                | "attr" , "(" , project_field , { "," , project_field } , ")" ;
project_field   = field_name | grain_field_name | domain_field ;

(* --- Tier 1: Evolve --- *)

add_stmt        = "ADD" , grain_type_singular , add_clause , { add_clause } , reason_clause
                | workflow_add_stmt ;
supersede_stmt  = "SUPERSEDE" , ( hash_literal | parameter ) , set_clause , { set_clause } , reason_clause
                | workflow_supersede_stmt ;
revert_stmt     = "REVERT" , ( hash_literal | parameter ) , reason_clause ;

add_clause      = "SET" , ( add_field | grain_add_field ) , "=" , value ;
add_field       = "subject" | "relation" | "object"
                | "confidence" | "importance" | "tags" ;
grain_add_field = goal_add_field | observation_add_field ;
goal_add_field  = "goal_state" | "assigned_agent" | "deadline" | "depends_on" ;
observation_add_field = "observer_id" | "observer_type" ;

set_clause      = "SET" , evolve_field , "=" , value ;
evolve_field    = "object" | "confidence" | "importance" | "tags" ;
reason_clause   = ( "REASON" | "BECAUSE" ) , string_literal ;

(* --- Tier 2: Destroy (new in 1.3) --- *)

(* Tier 0: the read-only mirror of FORGET SUBJECT -- §8.19 *)
report_subject_stmt = "REPORT" , "SUBJECT" , string_literal , [ with_clause ] ;

forget_stmt     = "FORGET" , ( hash_literal | parameter ) , [ reason_clause ]
                | "FORGET" , "SUBJECT" , string_literal ,
                  [ with_clause ] , reason_clause ;
purge_stmt      = "PURGE" , "OLDER" , "THAN" , duration_literal ,
                  [ "TYPE" , grain_type_singular ] , reason_clause ;
duration_literal = digit+ , ( "d" | "h" | "m" ) ;

(* --- Tier 3: Control (new in 1.3) --- *)

grant_stmt      = "GRANT" , verb_list , "ON" , ns_scope , "TO" , principal ,
                  [ with_clause ] ;
revoke_stmt     = "REVOKE" , verb_list , "ON" , ns_scope , "FROM" , principal ,
                  [ with_clause ] ;
show_grants_stmt = "SHOW" , "GRANTS" , [ "FOR" , principal ] ;
rehydrate_stmt  = "REHYDRATE" , string_literal ,
                  "WITH" , "mapping" , "(" , string_literal , ")" ;
verb_list       = verb , { "," , verb } ;
verb            = "read" | "write" | "supersede" | "delete" | "erase"
                | "loop.run" | "loop.review" | "loop.apply" | "admin" ;
ns_scope        = identifier | string_literal | "*" ;
principal       = string_literal ;   (* e.g. "agent:support-bot" — quoted:
                                        principal names carry ":" and "-",
                                        which identifiers do not *)

approve_stmt    = "APPROVE"  , ( hash_literal | parameter ) , reason_clause ;
reject_stmt     = "REJECT"   , ( hash_literal | parameter ) , reason_clause ;
apply_stmt      = "APPLY"    , ( hash_literal | parameter ) , reason_clause ;
rollback_stmt   = "ROLLBACK" , ( hash_literal | parameter ) , reason_clause ;
run_loop_stmt   = "RUN" , "LOOP" , [ "FULL" ] , [ with_clause ] ;

(* --- Tier 3: the definition surface (new in 1.3) --- *)

define_query_stmt = "DEFINE" , "QUERY" , query_name ,
                    [ "(" , param_decl , { "," , param_decl } , ")" ] ,
                    [ "DESCRIPTION" , string_literal ] ,
                    "AS" , "{" , query_body , "}" ;
param_decl        = parameter , [ "=" , value ] ;
query_body        = statement ;   (* Tier 0 only -- §8.18.2 *)
query_name        = string_literal ;

drop_query_stmt    = "DROP" , "QUERY" , query_name ;
drop_template_stmt = "DROP" , "TEMPLATE" , ( template_name | string_literal ) ;

(* RUN of a saved query is Tier 0: it classifies as its body does (§8.18.3). *)
run_query_stmt    = "RUN" , query_name ,
                    [ "(" , param_bind , { "," , param_bind } , ")" ] ;
param_bind        = parameter , "=" , value ;

(* --- Workflow graph syntax --- *)

workflow_add_stmt       = "ADD" , "workflow" , string_literal ,
                          graph_line , { graph_line } ,
                          { bind_clause } , reason_clause ;
workflow_supersede_stmt = "SUPERSEDE" , hash_literal ,
                          graph_line , { graph_line } ,
                          { bind_clause } , reason_clause ;
(* `on_clause` removed in 1.3: it set OMS's `Workflow.trigger`, which 1.6
   removes. A trigger is a Trigger grain that names its workflow (OMS §8.13).
   "ON" remains a keyword, used by GRANT and REVOKE. *)
bind_clause             = "BIND" , node_name , "=" , hash_literal ;

graph_line              = node_or_group , { "->" , node_or_group , [ when_mod ] , [ repeat_mod ] } ;
node_or_group           = node_name | "(" , node_name , { "," , node_name } , ")" ;
node_name               = identifier | string_literal ;
when_mod                = "WHEN" , string_literal ;
repeat_mod              = "*" , positive_integer ;

(* --- Template definitions --- *)

template_name   = identifier ;
extends_clause  = "EXTENDS" , ( preset_name | template_name ) ;
template_body   = section+ ;
template_shorthand = "AS" , string_literal ;
section         = header_section | element_section | element_summary_section
                | element_omit_section | source_break_section | footer_section ;
header_section           = "HEADER" , "{" , template_text , "}" ;
element_section          = "ELEMENT" , "{" , template_text , "}" ;
element_summary_section  = "ELEMENT_SUMMARY" , "{" , template_text , "}" ;
element_omit_section     = "ELEMENT_OMIT" , "{" , template_text , "}" ;
source_break_section     = "SOURCE_BREAK" , "{" , template_text , "}" ;
footer_section           = "FOOTER" , "{" , template_text , "}" ;

(* --- Shared terminals --- *)

value           = string_literal | number | boolean | parameter
                | array_literal | hash_literal ;
value_list      = value , { "," , value } ;
string_literal  = '"' , { any_char - '"' | '\\"' } , '"' ;
number          = [ "-" ] , digit+ , [ "." , digit+ ] ;
boolean         = "true" | "false" ;
parameter       = "$" , identifier ;
array_literal   = "[" , [ value_list ] , "]" ;
hash_literal    = "sha256:" , hex_char{8,64} ;
identifier      = letter , { letter | digit | "_" } ;
positive_integer = digit+ ;

grain_type_plural   = "facts" | "events" | "states" | "workflows" | "tools"
                    | "observations" | "goals" | "reasonings" | "consensuses"
                    | "consents" | "skills" | "recommendations"
                    | "triggers" ;

grain_type_singular = "fact" | "event" | "state" | "workflow" | "tool"
                    | "observation" | "goal" | "reasoning" | "consensus"
                    | "consent" | "skill" | "recommendation"
                    | "trigger" ;
```

---

## 5. Type System

### 5.1 Grain Types (Closed Set)

| Type | Plural (after RECALL) | Singular (in ADD/WHERE) | OMS Type Code |
|------|----------------------|------------------------|---------------|
| Fact | `facts` | `fact` | 0x01 |
| Event | `events` | `event` | 0x02 |
| State | `states` | `state` | 0x03 |
| Workflow | `workflows` | `workflow` | 0x04 |
| Tool | `tools` | `tool` | 0x05 |
| Observation | `observations` | `observation` | 0x06 |
| Goal | `goals` | `goal` | 0x07 |
| Reasoning | `reasonings` | `reasoning` | 0x08 |
| Consensus | `consensuses` | `consensus` | 0x09 |
| Consent | `consents` | `consent` | 0x0A |
| Skill | `skills` | `skill` | 0x0B |
| Recommendation | `recommendations` | `recommendation` | 0x0C |
| Trigger | `triggers` | `trigger` | 0x0D |

### 5.2 Common Field Types

| Field | Type | Operators | Notes |
|-------|------|-----------|-------|
| `query` | String | `=` only | Triggers semantic (BM25/vector) search |
| `subject` | String | `=`, `!=`, `IN` | Triple subject lookup |
| `relation` | String | `=`, `!=`, `IN`, `IS` | Triple relation lookup. `IS` used with relation category shortcuts. |
| `object` | String | `=`, `!=`, `IN` | Triple object lookup |
| `user_id` | String | `=`, `!=` | User isolation |
| `namespace` | String | `=` | Namespace isolation (overwritten by token) |
| `confidence` | Number | `=`, `!=`, `>=`, `<=`, `>`, `<` | Range [0.0, 1.0] |
| `importance` | Number | `=`, `!=`, `>=`, `<=`, `>`, `<` | Range [0.0, 1.0] |
| `score` | Number | `>=`, `>` | Post-retrieval filter |
| `tags` | Array | `INCLUDE`, `EXCLUDE` | Tag set operations |
| `type` | GrainType | `=` | One of 10 types |
| `time` | Temporal | `=`, `BETWEEN` | Natural language or epoch |
| `hash` | Hash | `=` | Content-address lookup |
| `contradicted` | Boolean | `=` | `true` or `false` |
| `verification_status` | String | `=` | `"unverified"`, `"verified"`, `"contested"`, `"retracted"` |
| `source_type` | String | `=` | Source type |
| `recall_priority` | String | `=` | `"hot"`, `"warm"`, `"cold"` |
| `epistemic_status` | String | `=` | `"certain"`, `"probable"`, `"uncertain"`, `"estimated"`, `"derived"` |

### 5.3 Evolve Fields (Tier 1, Closed Set)

**ADD fields** -- these can appear in an `ADD` statement's `SET` clauses. The first three are **required**:

| Field | Type | Required? | Constraint |
|-------|------|-----------|-----------|
| `subject` | String | **Yes** | The entity |
| `relation` | String | **Yes** | The predicate |
| `object` | String | **Yes** | The value |
| `confidence` | Number | No | Range [0.0, 1.0]. Default: implementation-defined |
| `importance` | Number | No | Range [0.0, 1.0]. Default: implementation-defined |
| `tags` | Array | No | Tag set. Default: empty |

Namespace and user_id are taken from the capability token -- they cannot appear in SET clauses.

**SUPERSEDE fields** -- only these fields can appear in a `SUPERSEDE` statement's `SET` clauses:

| Field | Type | Constraint |
|-------|------|-----------|
| `object` | String | The new value of the fact |
| `confidence` | Number | Range [0.0, 1.0] |
| `importance` | Number | Range [0.0, 1.0] |
| `tags` | Array | Replaces the tag set |

### 5.4 NULL Semantics

Missing field = no match (never errors). `WHERE confidence >= 0.8` on a grain without `confidence` returns no match, not an error.

---

## 6. OMS Grain Type Integration

### 6.1 Design Principle

When a `RECALL` statement specifies a grain type (e.g., `RECALL tools`), the parser unlocks a **type-specific field set** for use in `WHERE` clauses. This enables precise querying of OMS-native fields that only exist on specific grain types, without polluting the global field namespace.

The type-specific field set is a **compile-time guarantee**: the parser MUST reject field references that do not belong to the specified grain type. When no grain type is specified (`RECALL WHERE ...`), only the common field set is available.

### 6.2 Field Resolution Rules

1. **Phase 1 -- Common fields.** The common field set (section 5.2) is always available.
2. **Phase 2 -- Type-specific fields.** When the statement specifies a grain type plural, the grain-type-specific field set is additionally available.

**Validation rule:** If a `grain_field_condition` references a field not in the declared grain type's field set, the parser MUST return error `CAL-E060: FieldNotOnGrainType` with a suggestion listing valid fields for that type.

### 6.3 Grain-Type-Specific Queryable Fields

#### Fact (0x01) -- `RECALL facts`

All Fact fields are in the common set (`subject`, `relation`, `object`, `confidence`). No additional type-specific fields.

#### Event (0x02) -- `RECALL events`

| Field | Type | Operators | Notes |
|-------|------|-----------|-------|
| `role` | String | `=`, `!=` | `"user"`, `"assistant"`, `"system"`, `"tool"` |
| `session_id` | String | `=` | Conversation session identifier |
| `parent_message_id` | String | `=` | Threading: parent message reference |
| `model_id` | String | `=`, `!=` | LLM model identifier |
| `content` | String | `=` | Semantic search on event content |

#### State (0x03) -- `RECALL states`

| Field | Type | Operators | Notes |
|-------|------|-----------|-------|
| `context` | String | `=`, `!=` | State context identifier |
| `plan` | String | `=` | Semantic search on plan content |

#### Workflow (0x04) -- `RECALL workflows`

| Field | Type | Operators | Notes |
|-------|------|-----------|-------|
| `node` | String | `=` | Match workflows containing a specific node |
| `binding` | String | `=` | Match workflows binding to a specific Tool grain hash |

#### Tool (0x05) -- `RECALL tools`

| Field | Type | Operators | Notes |
|-------|------|-----------|-------|
| `tool_name` | String | `=`, `!=`, `IN` | Tool identifier |
| `tool_phase` | String | `=` | `"definition"`, `"call"`, `"result"`, `"complete"` |
| `is_error` | Boolean | `=` | Whether the tool resulted in error |
| `tool_call_id` | String | `=` | Correlation ID across tool phases |

#### Observation (0x06) -- `RECALL observations`

| Field | Type | Operators | Notes |
|-------|------|-----------|-------|
| `observer_id` | String | `=`, `!=` | Identifier of the observing entity |
| `observer_type` | String | `=`, `!=` | Type classifier (e.g., `"agent:monitor"`) |

#### Goal (0x07) -- `RECALL goals`

| Field | Type | Operators | Notes |
|-------|------|-----------|-------|
| `goal_state` | String | `=`, `!=` | `"active"`, `"completed"`, `"abandoned"`, `"blocked"` |
| `assigned_agent` | String | `=`, `!=` | DID of responsible agent |
| `deadline` | Temporal | `=`, `BETWEEN` | ISO 8601 or epoch |
| `depends_on` | String | `=`, `IN` | Content address(es) of prerequisite goals |

#### Reasoning (0x08) -- `RECALL reasonings`

| Field | Type | Operators | Notes |
|-------|------|-----------|-------|
| `reasoning_type` | String | `=` | `"deductive"`, `"inductive"`, `"abductive"`, `"analogical"` |
| `premises` | String | `=` | Semantic search on premises |
| `conclusion` | String | `=`, `!=` | Semantic or exact match on conclusion |

#### Consensus (0x09) -- `RECALL consensuses`

| Field | Type | Operators | Notes |
|-------|------|-----------|-------|
| `threshold` | Number | `=`, `>=`, `<=`, `>`, `<` | Agreement threshold [0.0, 1.0] |
| `agreement_count` | Number | `=`, `>=`, `<=`, `>`, `<` | Number of agreeing observers |
| `participating_observers` | Array | `INCLUDE` | Filter by participating observer IDs |

#### Consent (0x0A) -- `RECALL consents`

| Field | Type | Operators | Notes |
|-------|------|-----------|-------|
| `consent_action` | String | `=` | `"grant"` or `"withdraw"` |
| `purpose` | String | `=`, `!=` | Purpose-binding (e.g., `"personalization"`) |
| `grantor_did` | String | `=` | DID of consent grantor |
| `grantee_did` | String | `=` | DID of consent recipient |
| `scope` | String | `=` | Consent scope identifier |
| `expires_at` | Temporal | `=`, `BETWEEN` | Consent expiration |

#### Skill (0x0B) -- `RECALL skills`

| Field | Type | Operators | Notes |
|-------|------|-----------|-------|
| `name` | String | `=`, `!=`, `IN` | Machine-readable skill identifier (e.g., `"code_review"`) |
| `version` | String | `=`, `IN` | Skill-defined version string (e.g., `"2.1.0"`) |
| `domain` | String | `=`, `IN` | Domain context (e.g., `"software"`) |
| `holder_did` | String | `=` | DID of the agent that holds the skill (learned instances) |
| `proficiency` | Number | `=`, `>=`, `<=`, `>`, `<` | Mastery level [0.0, 1.0] (learned instances) |
| `transferable` | Boolean | `=` | Whether the skill can be transferred to another agent |
| `practice_count` | Number | `=`, `>=`, `<=`, `>`, `<` | Successful applications |
| `last_practiced_at` | Temporal | `=`, `BETWEEN` | Most recent successful application |

#### Recommendation (0x0C) -- `RECALL recommendations`

| Field | Type | Operators | Notes |
|-------|------|-----------|-------|
| `target_ref` | String | `=`, `!=`, `IN` | The change target (`entity:…`, `grain:sha256:…`, `doc:…`, etc.) |
| `analyzer` | String | `=`, `IN` | The producing analyzer's id (e.g., `"waiser.duplicate_sweep/1"`). **Projection:** OMS stores `analyzer` as a map (`{id, params?}`, §8.12 of the OMS spec); CAL matches against its `id` member only. `params` is not queryable. |
| `severity` | String | `=`, `IN` | `"info"`, `"low"`, `"medium"`, `"high"` |
| `dedup_key` | String | `=` | Stable proposal identity (a supersession chain shares one) |
| `rec_status` | String | `=`, `IN` | Review state (index-layer): `"pending"`, `"approved"`, `"rejected"`, `"applied"`, `"rolled_back"`, `"expired"` |

Recommendation grains are engine-emitted proposals and are **not CAL-addable** — like Events, Tools, and States, they are absent from the addable whitelist enforced by `CAL-E051` (Appendix C). `RECALL recommendations` reads the proposal queue, and lifecycle transitions occur only through the host's review/apply path (§8.12.1 of the OMS spec), never via `ADD`/`SUPERSEDE SET`.

> **Note — why `rec_status` is type-scoped:** `verification_status` is a `meta_field_name` (§4), queryable on every grain type. `rec_status` is instead a `recommendation_field`, because it is meaningful only for this one type — a Recommendation's review state has no analogue on a Fact or an Event. Both are index-layer fields in OMS (§6.1 of the OMS spec); the asymmetry in CAL is deliberate scoping, not an oversight.

#### Trigger (0x0D) -- `RECALL triggers`

| Field | Type | Operators | Notes |
|-------|------|-----------|-------|
| `kind` | String | `=`, `!=`, `IN` | `"interval"`, `"schedule"`, `"once"`, `"polling"`, `"memory"`, `"webhook"`, `"manual"`, `"composite"` |
| `workflow` | String | `=`, `IN` | Content address of the Workflow this trigger starts |
| `connector` | String | `=`, `!=`, `IN` | External system name, e.g. `"gmail"` |
| `scope` | String | `=`, `CONTAINS`, `STARTS WITH` | What is watched |
| `enabled` | Boolean | `=`, `!=` | Whether the trigger is live |
| `cron` | String | `=`, `CONTAINS` | Calendar expression |

These are **first-class fields, not `context` keys**, which is the point: a
trigger's cadence and target must be filterable. `RECALL triggers WHERE enabled
= true AND connector = "gmail"` is the query this type exists to make possible,
and it is why §8.13 of the OMS spec declares them rather than carrying them in a
`context`-shaped map. Connector transport configuration remains in `config` and
is not queryable.

Operational state — when a trigger last fired, where its cursor is, whether it
is currently claimed — is deliberately **not** in the grain and therefore not
queryable through CAL. It is host-local (OMS §27.8) and does not travel with the
memory.

### 6.4 Type-Specific ADD Extensions

When adding Goal, Observation, or Skill grains, type-specific fields are available in SET clauses:

```sql
ADD goal
  SET subject = "alice"
  SET relation = "mg:intends"
  SET object = "complete quarterly review"
  SET goal_state = "active"
  SET assigned_agent = "did:web:assistant.example.com"
  SET deadline = "2026-03-15T00:00:00Z"
  SET importance = 0.9
  REASON "user created objective during planning session"

ADD observation
  SET subject = "system"
  SET relation = "mg:perceives"
  SET object = "alice works late on Fridays"
  SET observer_id = "obs-activity-monitor"
  SET observer_type = "agent:activity-tracker"
  SET confidence = 0.7
  REASON "observed pattern across last 4 weeks"

ADD skill
  SET name = "code_review"
  SET description = "Review code changes for correctness, style, and security issues"
  SET instructions = "Read the diff, classify the change, then apply the matching strategy; check auth and input-handling paths first."
  SET when_to_use = "a pull request or code diff needs review before merge"
  SET version = "2.1.0"
  SET domain = "software"
  REASON "authoring reusable code-review skill definition"
```

### 6.5 Field Count Summary

| Grain Type | Common Fields | Type-Specific Fields | Total Queryable |
|-----------|--------------|---------------------|----------------|
| Fact | 18 | 0 | 18 |
| Event | 18 | 5 | 23 |
| State | 18 | 2 | 20 |
| Workflow | 18 | 2 | 20 |
| Tool | 18 | 4 | 22 |
| Observation | 18 | 2 | 20 |
| Goal | 18 | 4 | 22 |
| Reasoning | 18 | 3 | 21 |
| Consensus | 18 | 3 | 21 |
| Consent | 18 | 6 | 24 |
| Skill | 18 | 8 | 26 |
| Recommendation | 18 | 5 | 23 |
| _(no type)_ | 18 | 0 | 18 |

---

## 7. mg: Relation Vocabulary

### 7.1 Standard mg: Relations

OMS defines a standard `mg:` relation vocabulary. CAL provides first-class support for `mg:` relations: they are valid string literals, the parser recognizes them for validation, and common patterns have semantic shortcuts.

| Relation | Category | Typical Subject | Typical Object | Description |
|----------|----------|----------------|----------------|-------------|
| `mg:perceives` | Observation | Agent/Observer | Phenomenon | Sensory/cognitive input |
| `mg:knows` | Knowledge | Entity | Fact | Knowledge assertion |
| `mg:said` | Interaction | Entity | Statement | Recorded utterance |
| `mg:did` | Interaction | Entity | Action description | Recorded action |
| `mg:infers` | Knowledge | Agent | Conclusion | Inference result |
| `mg:agrees_with` | Consensus | Agent | Proposition | Agreement record |
| `mg:state_at` | Observation | Agent | State snapshot | Point-in-time state |
| `mg:has_graph` | Workflow | Process | Graph of steps | Workflow definition |
| `mg:intends` | Lifecycle | Entity | Objective | Goal declaration |
| `mg:permits` | Permission | Grantor DID | Action/scope | Permission grant |
| `mg:revokes` | Permission | Grantor DID | Action/scope | Permission withdrawal |
| `mg:prohibits` | Permission | Authority | Action/scope | Prohibition |
| `mg:requires` | Preference | Entity | Requirement | Requirement assertion |
| `mg:prefers` | Preference | Entity | Preference | Preference assertion |
| `mg:avoids` | Preference | Entity | Aversion | Avoidance assertion |
| `mg:delegates_to` | Agency | Entity | Agent DID | Delegation |
| `mg:owned_by` | Knowledge | Resource | Entity | Ownership |
| `mg:has_capability` | Agency | Agent DID | Capability | Agent capability |
| `mg:capable_of` | Agency | Agent DID | Skill | Learned skill (Skill grain, 0x0B) |
| `mg:handed_off_to` | Interaction | Agent DID | Agent DID | Agent handoff |
| `mg:depends_on` | Lifecycle | Goal | Goal | Goal dependency |
| `mg:assigned_to` | Agency | Task | Agent DID | Task assignment |

### 7.2 Relation Category Shortcuts

CAL defines **relation category shortcuts** as syntactic sugar for common multi-relation queries. These expand to `IN` conditions at parse time:

| Shortcut | Expands To |
|----------|-----------|
| `relation IS PREFERENCE` | `relation IN ("mg:prefers", "mg:avoids", "mg:requires")` |
| `relation IS KNOWLEDGE` | `relation IN ("mg:knows", "mg:infers")` |
| `relation IS PERMISSION` | `relation IN ("mg:permits", "mg:revokes", "mg:prohibits")` |
| `relation IS INTERACTION` | `relation IN ("mg:said", "mg:did", "mg:handed_off_to")` |
| `relation IS AGENCY` | `relation IN ("mg:delegates_to", "mg:has_capability", "mg:capable_of", "mg:assigned_to")` |
| `relation IS LIFECYCLE` | `relation IN ("mg:intends", "mg:depends_on")` |
| `relation IS OBSERVATION` | `relation IN ("mg:perceives", "mg:state_at")` |

**Examples:**

```sql
-- All preference-related facts about alice
RECALL facts WHERE subject = "alice" AND relation IS PREFERENCE
  ORDER BY confidence DESC

-- All permission records for a DID
RECALL WHERE subject = "did:key:z6Mk..." AND relation IS PERMISSION
```

### 7.3 mg: Relation Validation

The parser SHOULD validate `mg:` prefixed relation values against the known vocabulary. Unknown `mg:` relations produce warning `CAL-W001` (not an error).

---

## 8. Statement Semantics

CAL/1 statements are organized into four tiers (§2.2):

| Statement | Tier | Description |
|-----------|------|-------------|
| **RECALL** | 0 | Retrieve grains matching filters |
| **ASSEMBLE** | 0 | Compose context from multiple sources with budget |
| **EXISTS** | 0 | Check grain existence by content address |
| **HISTORY** | 0 | Version history with AS OF and DIFF |
| **EXPLAIN** | 0 | Execution plan preview (any tier's statement, as dry-run) |
| **DESCRIBE** | 0 | Schema introspection (`PRINCIPAL`/`loop`-family targets read Tier-3 state) |
| **BATCH** | 0 | Multiple independent queries in one request |
| **COALESCE** | 0 | Fallback chain of RECALL queries |
| *Set operations* | 0 | `UNION`, `INTERSECT`, `EXCEPT` |
| **ADD** | 1 | Create a new grain (append-only) |
| **SUPERSEDE** | 1 | Create a new version of an existing grain |
| **REVERT** | 1 | Restore a previous version |
| **FORGET** | 2 | Tombstone one grain by hash, or erase a subject's grains (§8.14) |
| **PURGE** | 2 | Age-scoped retention erasure (§8.14) |
| **GRANT / REVOKE** | 3 | Confer or withdraw verbs on a namespace for a principal (§8.15) |
| **SHOW GRANTS** | 3 (read) | List live grants (§8.15) |
| **APPROVE / REJECT / APPLY / ROLLBACK** | 3 | Recommendation lifecycle transitions (§8.16) |
| **RUN LOOP** | 3 | Trigger an analysis pass of the host's improvement engine (§8.16) |

### 8.1 RECALL (Tier 0)

Retrieves grains matching the given filters. Returns results using the OMS Standard Search Response Envelope.

```sql
RECALL facts WHERE subject = "alice" AND relation = "prefers"
  WITH contradiction_detection
  ORDER BY confidence DESC
  LIMIT 10
```

RECALL supports semantic shortcuts (ABOUT, RECENT, SINCE, LIKE, MY, CONTRADICTIONS -- see section 9), grain-type-specific fields (section 6), thread shorthand (section 8.1.1), and per-query format control via AS.

```sql
-- With grain-type-specific fields
RECALL tools WHERE tool_name = "get_weather" AND is_error = false
  ORDER BY time DESC LIMIT 20

-- With domain profile fields
RECALL facts WHERE tags INCLUDE ["profile:healthcare"]
  AND hc:patient_id = "P-12345" AND relation = "mg:knows"
```

#### 8.1.1 THREAD Shorthand

The `THREAD` keyword provides concise syntax for conversation retrieval:

```sql
-- Full conversation in a session
RECALL events THREAD "sess-123"
-- Expands to: RECALL events WHERE session_id = "sess-123" ORDER BY time ASC

-- Full thread containing a specific message
RECALL events THREAD FROM sha256:a1b2c3d4...
```

#### 8.1.2 WITH anonymize (new in 1.3)

```sql
RECALL facts WHERE subject = "caller:john" WITH anonymize("strict")
```

Requests pseudonymized results for this one query. **Strengthen-only**: the
option MAY raise the treatment an active egress policy (OMS Section 10.5)
would apply -- `"standard"` asserts the declared policy (a no-op where one
is active), `"strict"` treats every category at redaction severity -- and
MUST NOT weaken or disable a file-declared policy. Two reasons the
weakening form does not exist, both load-bearing:

1. Unknown WITH options warn-and-skip (Section 25 forward compatibility),
   so on an older implementation this option silently does nothing. That is
   survivable only because it was never the gate -- the file-declared
   policy is.
2. A weakening spelling would make every query author a policy author. The
   file's declaration MUST outrank query text, or the policy is advisory.

Implementations without anonymization support treat the option as unknown
(warn-and-skip); consumers MUST NOT read the absence of the response's
`anonymized` report as "values are safe".

### 8.2 ASSEMBLE (Tier 0)

The flagship new statement. Composes a context block from multiple RECALL sources with token budgets, priority ordering, format control, and progressive disclosure.

```sql
CAL/1 ASSEMBLE user_context
  FOR "conversation about alice's preferences and goals"
  FROM
    facts:    (RECALL facts ABOUT "alice" WHERE relation = "prefers" LIMIT 20),
    goals:    (RECALL goals ABOUT "alice" RECENT 10),
    events:   (RECALL events WHERE user_id = "alice" RECENT 5),
    history:  (RECALL facts ABOUT "alice"
                WHERE relation = "prefers" WITH superseded
                ORDER BY time DESC LIMIT 3)
  BUDGET 2000 tokens
  PRIORITY facts > goals > events > history
  FORMAT markdown
  WITH progressive_disclosure, dedup(subject)
```

**Execution Phases:**

1. **Source Resolution.** Each source in the FROM clause is an independent RECALL. They execute in parallel. Results MUST be deterministic.
2. **Deduplication.** If `WITH dedup(field)` is specified, grains in multiple sources are deduplicated. The copy from the highest-priority source is kept.
3. **Budget Allocation.** The budget allocator distributes tokens according to the PRIORITY clause. Default weights: 2 sources [0.65, 0.35]; 3 sources [0.50, 0.30, 0.20]; 4 sources [0.40, 0.28, 0.20, 0.12]; 5+ sources: exponential decay. Surplus from under-utilizing sources redistributes to remaining sources.
4. **Progressive Disclosure.** When enabled, the response includes three tiers: Summary, Headlines, Full.
5. **Formatting.** The FORMAT clause determines output structure.

**Budget Units:**

| Unit | Meaning | Default | Max |
|------|---------|---------|-----|
| `tokens` | Approximate token count | 4000 | 16000 |
| `grains` | Maximum total grain count | 50 | 200 |

Token estimation is approximate by design. The response MUST report actual tokens used.

**ASSEMBLE Constraints:**

| Constraint | Limit |
|-----------|-------|
| Max sources in FROM | 8 |
| Max LET bindings per ASSEMBLE | 5 |
| Max total BUDGET (tokens) | 16,000 |
| Max total BUDGET (grains) | 200 |
| Max context_name length | 64 characters |
| Max FOR string length | 256 characters |
| ASSEMBLE timeout | 10,000ms |

#### 8.2.1 Per-source anonymize override (new in 1.3)

A named source MAY carry its own `WITH anonymize(...)`, same strengthen-only
rule as Section 8.1.2: a mounted source's own file policy applies first, and
the override can only add severity on top. This is the multi-file case where
per-source treatment genuinely differs -- a mounted org replica may warrant
stricter handling than the session's own memory.

```sql
ASSEMBLE briefing FOR "customer call prep"
  FROM
    work:       (RECALL facts ABOUT "caller:john" LIMIT 20) WITH anonymize("strict"),
    background: (RECALL org.facts ABOUT "acme" LIMIT 10)
  BUDGET 1500 tokens
```

### 8.3 EXISTS (Tier 0)

Checks if a specific grain exists by content address. Returns boolean. O(1) via hash lookup.

```sql
EXISTS sha256:a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2
```

### 8.4 HISTORY (Tier 0)

Retrieves the version history for a grain or a (subject, relation) triple. Returns versions in reverse chronological order. Capped at 100 versions.

```sql
-- By hash: show version chain
HISTORY sha256:a1b2c3d4...

-- By triple: show all versions
HISTORY WHERE subject = "alice" AND relation = "prefers"

-- AS OF: temporal snapshot
HISTORY WHERE subject = "alice" AND relation = "prefers" AS OF "2025-06-15"

-- DIFF: show changes between two versions
HISTORY sha256:aaa... DIFF sha256:bbb...
```

### 8.5 EXPLAIN (Tier 0)

Returns the execution plan without running the query. Works with all statement types including ASSEMBLE.

```sql
EXPLAIN RECALL facts WHERE query = "alice preferences" LIMIT 10
EXPLAIN ASSEMBLE user_context
  FOR "conversation about alice"
  FROM facts: (RECALL facts ABOUT "alice"),
       goals: (RECALL goals ABOUT "alice" RECENT 5)
  BUDGET 2000 tokens
```

### 8.6 DESCRIBE (Tier 0)

Schema introspection for grain types, fields, capabilities, server metadata, templates, and grammar.

```sql
CAL/1 DESCRIBE grain_types        -- list available grain types
CAL/1 DESCRIBE fields             -- list all queryable fields
CAL/1 DESCRIBE fields fact      -- list fields for a specific grain type
CAL/1 DESCRIBE capabilities       -- server capabilities and conformance
CAL/1 DESCRIBE server             -- server metadata
CAL/1 DESCRIBE templates          -- list registered templates
CAL/1 DESCRIBE grammar            -- return EBNF (optional, Extended conformance)
```

### 8.7 BATCH (Tier 0)

Multiple independent queries in a single request. Each sub-query gets its own result slot. Only Tier 0 (read) statements are allowed in BATCH.

```sql
CAL/1 BATCH {
  preferences: RECALL facts ABOUT "alice" WHERE relation = "prefers",
  recent:      RECALL events ABOUT "alice" RECENT 5,
  team:        RECALL facts WHERE relation = "member_of" AND object = "team-alpha"
}
```

**Constraints:** Max 10 queries per BATCH. LET bindings within a BATCH are scoped to that BATCH block.

### 8.8 ADD (Tier 1)

Creates a **new** grain. Pure append-only -- does not modify or reference any existing grain.

```sql
ADD fact
  SET subject = "alice"
  SET relation = "prefers"
  SET object = "dark mode"
  SET confidence = 0.9
  SET tags = ["preference", "ui"]
  REASON "user stated preference during onboarding conversation"
```

**Addable grain types:** Fact, Observation, Goal, Workflow, Skill. Events, Tools, States, and other types represent system-generated records and are not user-creatable.

**Required SET fields (Fact):** `subject`, `relation`, `object`. `REASON` is mandatory.

**Required SET fields (Skill):** `name`, `description`. `REASON` is mandatory.

#### 8.8.1 ADD Workflow (Graph Syntax)

Workflows use a dedicated graph syntax instead of SET clauses. The graph is expressed with arrows (`->`), parallel groups (`()`), conditions (`WHEN`), and repeat bounds (`* N`).

```sql
-- Simple linear workflow
ADD workflow "nightly backup"
  snapshot -> compress -> upload
  REASON "automate database backups"

-- Parallel fork/join
ADD workflow "code review"
  lint -> (security_review, compliance_review) -> evaluate
  REASON "parallel review gates"

-- Conditional branching
ADD workflow "release gate"
  evaluate -> implement WHEN "approved"
  evaluate -> reject WHEN "rejected"
  REASON "approval-based routing"

-- Retry on failure
ADD workflow "resilient deploy"
  build -> deploy * 3 -> notify
  REASON "retry deploy up to 3 times"

-- Full pipeline with bindings
ADD workflow "release pipeline"
  build -> (unit_test, lint) -> integration_test
  integration_test -> stage_deploy * 3
  stage_deploy -> approval
  approval -> prod_deploy WHEN "approved"
  approval -> rollback WHEN "rejected"
  prod_deploy -> notify
  rollback -> notify
  BIND build = sha256:def111...
  BIND stage_deploy = sha256:def333...
  BIND prod_deploy = sha256:def444...
  REASON "standard release process"
```

**Clause order (fixed):** name → graph lines → BIND → REASON

**Graph operators:**

| Operator | Meaning | Example |
|----------|---------|---------|
| `->` | Sequential edge | `build -> test` |
| `(a, b)` | Parallel fork/join | `(lint, test) -> merge` |
| `WHEN "cond"` | Conditional edge | `-> deploy WHEN "approved"` |
| `* N` | Repeat up to N times on failure | `deploy * 3` |

**Operator precedence:** `* N` (highest) > `WHEN` > `->` (lowest).

**Structural inference:** Node types are inferred from graph topology — a node with multiple unconditional outgoing edges is a fork, a node receiving multiple edges is an AND-join, a node with `WHEN` edges is a decision point.

**BIND clause:** Maps a node name to a Tool definition grain hash (`tool_phase: "definition"`). The executor fetches the definition to discover `tool_name`, `input_schema`, etc. Unbound nodes are resolved by name convention or treated as abstract steps.

**Node names:** Bare identifiers (`build`, `unit_test`) or quoted strings (`"send welcome email"`). Reserved words (`ADD`, `workflow`, `ON`, `WHEN`, `BIND`, `REASON`, `BECAUSE`) must be quoted.

### 8.9 SUPERSEDE (Tier 1)

Creates a **new** grain that supersedes an existing one. The old grain is preserved and remains queryable via `WITH superseded`.

```sql
SUPERSEDE sha256:target_hash
  SET object = "light mode"
  SET confidence = 0.95
  REASON "user explicitly changed preference"
```

**Fact supersession:** At least one `SET` clause and `REASON` are required.

**Workflow supersession:** Uses graph syntax (full graph replacement):

```sql
SUPERSEDE sha256:abc123...
  build -> (unit_test, lint, security_scan) -> integration_test
  integration_test -> canary_deploy * 3
  canary_deploy -> approval
  approval -> prod_deploy WHEN "approved"
  approval -> rollback WHEN "rejected"
  BIND security_scan = sha256:def888...
  REASON "added security scan, replaced stage with canary"
```

### 8.10 REVERT (Tier 1)

Creates a **new** grain that restores content from the version before the target. Like `git revert`, this does not undo history -- it creates a new version.

```sql
REVERT sha256:target_hash
  REASON "supersession was based on misunderstood context"
```

### 8.11 Set Operations

```sql
(RECALL WHERE user_id = "alice" AND query = "project status")
EXCEPT
(RECALL WHERE user_id = "bob" AND query = "project status")
```

Each operand executes independently. Set operations are applied post-retrieval:
- **UNION:** Deduplicate by content_address, merge scores (max)
- **INTERSECT:** Keep only grains present in both, merge scores (min)
- **EXCEPT:** Keep left grains absent from right

### 8.12 LET Bindings

LET bindings name intermediate RECALL results that can be referenced by `$name` in subsequent FROM clauses or WHERE IN sub-expressions.

```sql
CAL/1
LET $team_members = RECALL facts
  WHERE relation = "member_of" AND object = "team-alpha" SUBJECTS;

LET $team_prefs = RECALL facts
  WHERE subject IN ($team_members) AND relation = "prefers";

ASSEMBLE team_context
  FOR "team alpha's collective preferences"
  FROM prefs: ($team_prefs),
       goals: (RECALL goals WHERE subject IN ($team_members) RECENT 10)
  BUDGET 3000 tokens
  PRIORITY prefs > goals
  FORMAT markdown
```

**LET constraints:**
- Max 5 LET bindings per request
- Evaluated once, in declaration order
- Within a standalone query: cannot reference other LET bindings
- Within ASSEMBLE: LET bindings can reference prior bindings (linear chaining only, max depth 3)
- Scoped to the enclosing BATCH or single-statement context

### 8.13 COALESCE

Evaluates argument queries left-to-right. Returns the result of the **first query that returns at least one grain**. Remaining queries are not executed (short-circuit evaluation).

```sql
COALESCE(
  RECALL facts WHERE subject = "alice" AND relation = "favorite_color",
  RECALL facts WHERE subject = "alice" AND relation = "prefers"
    AND tags INCLUDE ["color"],
  RECALL facts ABOUT "alice" LIKE "color preference"
)
```

**Constraints:** Max 5 branches. All branches MUST be RECALL statements.

### 8.14 FORGET and PURGE (Tier 2 -- new in 1.3)

Destruction of whole grains, in exactly three shapes. Maps to the OMS store's
compliance `delete` operation (OMS §28.4) and host bulk-erasure operations.

```sql
FORGET sha256:a1b2c3d4...                                   -- one grain, by address
FORGET sha256:a1b2c3d4... BECAUSE "user retracted consent"  -- reason optional here, recorded when given
FORGET SUBJECT "patient-789" BECAUSE "GDPR Art. 17 request" -- every grain referencing an identity
FORGET SUBJECT "patient-789" WITH text_mentions BECAUSE "…" -- + grains whose indexed text mentions it
PURGE OLDER THAN 90d BECAUSE "retention policy"             -- age-scoped sweep
PURGE OLDER THAN 30d TYPE event BECAUSE "transcript TTL"    -- age + one grain type
```

#### 8.14.1 Semantics

- `FORGET <hash>` tombstones one grain: it leaves the default recall path
  permanently. Requires the `delete` verb. `BECAUSE` is OPTIONAL on this form
  (it predates 1.3 in deployed dialects and MUST remain valid without a
  reason) and MUST be recorded in the audit Observation when given.
- `FORGET SUBJECT <id>` erases every grain referencing the identity --
  history included -- per the host's erasure procedure; `WITH text_mentions`
  extends the scope to grains whose indexed text mentions the identity.
  Requires the `erase` verb. `BECAUSE` is REQUIRED (parse error without).
- `PURGE OLDER THAN <duration> [TYPE <t>]` erases grains older than the
  window, optionally limited to one grain type. Requires the `erase` verb.
  `BECAUSE` is REQUIRED.
- All three are **one-way** (§2.3): there is no `UNFORGET`, and a host MUST
  NOT offer an un-tombstone or un-erase mechanism through any interface.
- An erasure execution SHOULD produce the host's erasure report; the audit
  Observation (§8.14.4) references it. Where the erased grains referenced
  content-addressed attachments, erasure MUST also reclaim the ones no
  surviving grain references, and the report MUST carry that count separately
  from the grain count (OMS §28.4.3). A statement that erased a subject's
  grains and left the subject's documents on disk has not erased the subject.

#### 8.14.2 Authorization

Tier-2 statements execute only when the session principal (§18.4) holds the
required verb for the target namespace, resolved from the memory's own grant
grains (OMS §12.6) or, on a Tier-2-without-Tier-3 host, from the host's
authorization model. Enforcement is fail-closed: no principal, no grants, or
no authorization model at all → `CAL-E121`/`CAL-E122`, nothing executes.
A host session designated *owner* (OMS §12.6.4) is exempt from grant lookup.

#### 8.14.3 Context restrictions

Tier-2 and Tier-3 statements MUST be top-level, single statements. They are
refused inside `BATCH`, inside saved-query bodies, and inside a
Recommendation's `proposal_cal` **except** that a `proposal_cal` MAY carry
Tier-2 statements where the host authorizes destructive proposals (OMS
§8.12); governance statements (§8.16) are refused inside `proposal_cal`
unconditionally. `EXPLAIN` of any Tier-2/3 statement is always available as
a dry-run and executes nothing.

#### 8.14.4 The Tier-2 audit Observation

Every Tier-2 execution MUST write one audit Observation grain in the memory
it acted on, following the audit pattern of OMS §8.12.1: `observer_id` = the
session principal, `observer_type` from the principal's host record (§18.4),
the statement's verb and target (hash, subject identity, or age window) and
the `BECAUSE` reason (where given) carried in the Observation's `object`,
and `derived_from` referencing the erasure report where one exists. The
audit grain is ordinary memory: it replicates with the file and is
`RECALL`-able. A destruction that cannot write its audit Observation MUST
fail before destroying anything.

### 8.15 GRANT, REVOKE, SHOW GRANTS (Tier 3 -- new in 1.3)

The control plane: confer or withdraw verbs on a namespace for a principal.
Grants are stored **in the memory itself** as grant grains (OMS §12.6) --
they travel, replicate, and are queryable like any other memory.

```sql
GRANT read, write ON caller TO "agent:support-bot"
GRANT delete ON * TO "user:anna" WITH because("support rotation")
REVOKE write ON caller FROM "agent:support-bot" WITH because("offboarded")
SHOW GRANTS                          -- every live grant
SHOW GRANTS FOR agent:support-bot    -- one principal's live grants
DESCRIBE PRINCIPAL agent:support-bot -- effective verbs per namespace
```

#### 8.15.1 Semantics

- **Verbs:** `read`, `write`, `supersede`, `delete`, `erase`, `loop.run`,
  `loop.review`, `loop.apply`, `admin`. The verb set is closed in 1.3.
- **Scope:** one namespace or `*`. The memory axis is implicit -- a grant
  governs the memory it is stored in, never another memory.
- **Principals** are opaque names (`user:anna`, `agent:support-bot`).
  Identity *binding* -- proving a caller is that principal -- is the host's
  job (§18.4); CAL never sees or carries credentials.
- `GRANT` writes a grant grain (OMS §12.6.2). `REVOKE` retracts the covering
  grant grain(s) by supersession -- a full revoke retracts, a partial revoke
  supersedes with the reduced grant -- so grant history is append-only and
  nothing is deleted. GRANT/REVOKE are therefore **Tier-1-shaped writes
  gated by Tier-3 authority**, not destructive operations.
- **Who may GRANT:** a principal holding `admin` (on the target namespace),
  or the owner session. Every `erase` grant is explicit -- if a future
  revision defines role bundles, no bundle may include `erase`.
  There is no `WITH GRANT OPTION`.
- `SHOW GRANTS` is a read (sugar over `RECALL` of the PERMISSION relation
  category); it requires only `read` on the authz namespace.

#### 8.15.2 Fail-closed resolution

A session principal's effective rights are the union of live (unsuperseded)
grant grains naming it, intersected with the statement's target namespace.
Unknown principal or empty result → Tier 0/1 only. A memory containing no
grant grains confers nothing on anyone except the owner session.

### 8.16 Governance statements (Tier 3 -- new in 1.3)

The Recommendation lifecycle (OMS §8.12), expressible in the language whose
data it governs.

```sql
RUN LOOP                                      -- incremental analysis pass
RUN LOOP FULL WITH min_new(3), if_stale("6h") -- full sweep, gated
APPROVE sha256:9c41... BECAUSE "matches our deploy policy"
REJECT  sha256:9c41... BECAUSE "false positive - seasonal"
APPLY   sha256:9c41... BECAUSE "approved in standup"
ROLLBACK sha256:9c41... BECAUSE "regressed at the 30-day checkpoint"
DESCRIBE loop        -- engine health   (read)
DESCRIBE analyzers   -- registered analyzers (read)
DESCRIBE outcomes    -- the Verify gate's measurements (read)
DESCRIBE policy      -- the effective host policy (read)
```

#### 8.16.1 Semantics and gates

- `BECAUSE` is REQUIRED on every lifecycle transition -- a missing reason is
  a **parse error**, and hosts MUST additionally validate it non-empty at
  execution. The reason lands in the transition's audit Observation
  (OMS §8.12.1).
- Verb mapping: `RUN LOOP` requires `loop.run`; `APPROVE`/`REJECT` require
  `loop.review`; `APPLY`/`ROLLBACK` require `loop.apply`; a destructive
  apply additionally requires the `delete`/`erase` verb its proposal needs
  (two-key property).
- **Self-approval MUST be refused** against the recommendation's creating
  actor and, for a recommendation of LLM or external-command origin, against
  the principal that triggered the run that authored it.
- **Observer type MUST derive from the session principal's host record**
  (§18.4) -- never from statement text. A statement cannot claim to be
  human.
- `RUN LOOP` carries no credentials and no model configuration -- analysis
  backends are host-configured. `WITH` options are limited to gating knobs
  (e.g. `min_new(n)`, `if_stale("6h")`).
- Loop *policy* writes (auto-apply grants, analyzer deny-lists) are
  permanently outside CAL: the policy is what gates these statements, and a
  language able to edit its own gate would be self-licensing.
- Governance statements are refused inside `BATCH`, saved-query bodies, and
  `proposal_cal` (§8.14.3) -- no one-round-trip approve-and-apply macro, and
  no recommendation whose payload approves other recommendations.

### 8.17 REHYDRATE (Tier 3 -- new in 1.3)

```sql
REHYDRATE "Hi [PERSON_1], I've verified pin [PIN_1]." WITH mapping("a1b2c3d4e5f60718")
```

The return leg of the pseudonymization round trip (OMS Section 10.5):
replaces exact placeholder tokens in the text with their originals from the
mapping the id names. Unmatched tokens are left intact and reported, never
guessed.

**Classification: Control -- stated rather than hidden, because rehydration
is re-identification.** The statement MUST NOT classify as a plain read: an
implementation with authorization MUST gate it on the `admin` verb (or an
equivalent dedicated grant) and MUST write a Tier-2 audit Observation per
execution carrying the revealed values' *fingerprints*, never the
identities -- the same non-re-identifying audit rule bulk erasure follows
(Section 21). An implementation whose statement classification is
exhaustive-without-wildcard is forced to decide this at build time, which
is the point.

**Resolution.** The mapping id resolves against the session's live mappings
first, then the file's sealed vault (OMS Section 10.5.2). An id whose
mapping no longer exists is an error (CAL-E127), not an empty substitution
-- silently returning the placeholders would be indistinguishable from a
model echoing them.

Refused inside `BATCH`, saved-query bodies, and `proposal_cal`, like every
Tier 2/3 statement (Section 8.14.3).

### 8.18 Saved queries (Tier 3 to define, Tier 0 to run -- new in 1.3)

A **saved query** is a named, parameterized CAL body stored with the memory and
executed by name. It exists so that prompt-assembly logic -- which `ASSEMBLE`
runs on every turn, and which is edited far more often than the agent around it
-- can live as a definition the memory carries rather than as a string literal
compiled into a host.

Versions 1.0--1.2 restricted saved-query bodies in four places (§8.14.3,
§8.16.1, §8.17, `CAL-E125`) and made `query:<namespace>/<name>` a valid
Recommendation `target_ref` (OMS §8.12) while defining no statement that
creates one. This section is the missing definition; the restrictions that
referenced it are unchanged and now have a referent.

#### 8.18.1 Definitions are host metadata, not memory

A saved query is **not a grain.** It is not content-addressed, it is not
returned by `RECALL`, it does not participate in supersession, and defining one
writes no grain. It is host metadata carried in the memory's own metadata table
(OMS §28.4.1) under the reserved key prefix `qry:<name>`, one row per
definition.

That placement is the substantive design decision, and it is deliberate on both
sides:

- **Carried by the memory, not the client.** A saved query travels with the
  `.mg` file. Every surface opening that memory -- CLI, MCP server, console,
  language bindings -- sees the same set, so "which queries exist" is a
  property of the memory rather than of whichever process happens to be asking.
- **Not memory.** A definition is configuration. Making it a grain would put
  editable operational config into an immutable, content-addressed,
  replicating, erasure-relevant structure, and would put a query body into
  every `RECALL` that did not ask for one.

**Replication (normative).** Definitions MUST replicate with the memory (bundle
and stream export/import), converging **latest-wins** on the definition's
update time. Usage state MUST NOT replicate: a `last_run_at`-style
last-execution stamp is a record of what one host did, not of what the
definition *is*, and an incoming update MUST leave the local stamp intact. This
is the same split §27.8 draws for triggers and §8.12 draws for governance
state: **what a definition is replicates; where one host got to does not.**

**Identity.** A saved query carries the two stamps OMS §28.4.1 requires of any
stored definition: `updated_at` (the replication tiebreaker) and
`definition_hash` (the SHA-256 identity of this revision's body). Both are
reported by `DESCRIBE QUERIES` (§8.18.4). This is what makes
`query:<namespace>/<name>` a pinnable Recommendation target (OMS §8.12) — a
governed change to a saved query can name the revision it replaced and store an
inverse that reinstates exactly that one.

A point-in-time restore reconstructs grain history and MUST skip the
definition registry -- a restore to last Tuesday is a statement about memory,
not an instruction to revert an operator's query edits -- and MUST report that
it did.

An entry the running implementation cannot load (written by a newer revision,
or against a limit since tightened) MUST be skipped with a warning rather than
failing the open, and MUST be left in the file so another implementation can
still read it.

#### 8.18.2 DEFINE QUERY

```sql
DEFINE QUERY "session_prompt"($user, $session, $n = 10)
  DESCRIPTION "standard session bootstrap"
AS {
  ASSEMBLE "session" FROM
    profile: (RECALL facts  WHERE subject = $user),
    recent:  (RECALL events WHERE session_id = $session RECENT $n)
  BUDGET 1200 FORMAT sml
}
```

Normative rules:

1. **Bodies are Tier 0 only.** A body MUST contain exactly one statement, and
   that statement MUST classify as Tier 0 (read). A Tier 1/2/3 statement in a
   body is refused at definition time (`CAL-E136`). This is what makes `RUN`
   itself a read (§8.18.3), and it is why the Tier-2/3 context restrictions
   (§8.14.3) can name saved-query bodies without further qualification.
2. **No recursion.** A body MUST NOT contain `RUN` (`CAL-E135`). Bodies compose
   through subqueries and `ASSEMBLE` sources, not through each other; this
   makes execution cost analyzable from the body's own text.
3. **Define-time validation MUST parse the body as it would run.** An
   implementation MUST parse the body at definition time, with the declared
   parameters standing in at their call-site positions, and refuse a body that
   fails (`CAL-E137`). A body whose parameter sits where the grammar demands a
   literal (`RECENT $n`) is valid: the check substitutes before parsing, so
   only a body that is malformed *however* it is bound is refused.

   This rule is normative because of who pays for its absence. A stored query
   that cannot parse is a latent failure whose first caller is typically an
   unattended agent, long after its author has moved on. Validation at
   definition time turns that into an error for the person who can still fix
   it.
4. **Redefinition replaces.** `DEFINE QUERY` on an existing name replaces the
   definition and updates its revision stamp; it is not an error. There is no
   definition history -- a saved query is configuration, and its audit trail is
   the Recommendation lifecycle (OMS §8.12) where a governed change produced
   it.
5. **Authorization.** `DEFINE QUERY` and `DROP QUERY` are Tier 3 and require
   the `admin` verb for the target namespace. A definition is executed by
   principals who may not be permitted to write it; that gap is the point.
6. **Verification is not one-shot.** An implementation MUST re-verify the
   read-only property when the body executes, not only when it is defined. A
   definition can arrive by replication from a peer running a different
   revision, so a definition-time check alone is a check the attacker chooses
   the timing of.

**Limits.** An implementation MUST enforce, and `DESCRIBE capabilities` SHOULD
report:

| Constraint | Limit |
|---|---|
| Max saved queries per namespace | 100 |
| Max query body size | 8192 bytes |
| Max declared parameters | 10 |
| Query name | string literal, 64 characters |

Exceeding them is `CAL-E131` / `CAL-E132` / `CAL-E133`.

A parameter MAY declare a default (`$n = 10`). A parameter without a default is
required at the call site.

#### 8.18.3 RUN

```sql
RUN "session_prompt"($user = "john", $session = "call-42")
```

`RUN "<name>"` binds arguments to the declared parameters and executes the
body. A parameter with neither an argument nor a default is `CAL-E134`; an
unknown name is `CAL-E130`.

**`RUN` classifies as its body does.** Because bodies are structurally Tier 0
(§8.18.2 rule 1, re-verified at execution by rule 6), running a saved query is
a **read**, requires only the `read` verb, and is available wherever `RECALL`
is. An implementation MUST NOT gate `RUN` behind the `admin` verb that
`DEFINE QUERY` requires: defining is control, running is reading, and
collapsing the two would make every consumer of a saved query an
administrator.

Substitution happens **before** parsing, so an implementation caching parsed
statements by text reuses a plan per distinct argument set, and a
zero-parameter saved query hits that cache on every call after the first.
Values bind as values, not as text splices: a bound argument can never
introduce a token, so no argument can change the body's classification.

#### 8.18.4 DROP QUERY and DESCRIBE QUERIES

```sql
DROP QUERY "session_prompt"
DESCRIBE QUERIES
```

`DROP QUERY` removes a definition. It is Tier 3 (`admin`), it touches no grain,
and it is lossless against memory (§2.4). Dropping an absent name is
`CAL-E130`.

`DESCRIBE QUERIES` is Tier 0 and lists the registered definitions -- name,
description, parameter count, body size, both revision stamps (`updated_at` and
`definition_hash`, §8.18.1), and the last-execution stamp where the host keeps
one. It is the discovery surface an agent uses to
find what it may `RUN`.

`DESCRIBE capabilities` SHOULD report saved-query support and the limits of
§8.18.2.


### 8.19 REPORT SUBJECT (Tier 0 -- new in 1.3)

The read-only mirror of `FORGET SUBJECT`: the same selection, disclosed instead
of destroyed. It is the in-language answer to GDPR Art. 15 (access) and Art. 20
(portability), and the safe rehearsal for an erasure.

```sql
REPORT SUBJECT "pat"
REPORT SUBJECT "pat" WITH text_mentions
```

**One selector, two verbs.** `REPORT SUBJECT <id>` MUST select exactly the set
`FORGET SUBJECT <id>` would erase, through the same code path and the same
indexes (OMS §28.4.4 rule 3) -- the identity, its partition-style derived keys,
grains carrying it in either triple position, its thread and run records, and
under `WITH text_mentions` its indexed text mentions. `WITH text_mentions`
carries the same hard precondition on both statements: where the text index is
absent or incomplete the statement MUST refuse rather than return a quietly
partial answer.

Sharing the selector is the point. It makes "show me everything you hold, then
delete it" two statements over one set, and it means a disclosure cannot
promise less than an erasure removes or more than it reaches.

**It is a read, and it is classified as one.** `REPORT SUBJECT` requires the
`read` verb, not `erase`. It MUST NOT sit behind whatever process-level cap
disables destructive operations: a deployment that has turned destruction off
still owes data subjects access, and coupling the two would make an
implementation choose between refusing erasure and refusing disclosure.

**It writes nothing.** No audit Observation, no grain of any kind. Tier 2 writes
an audit record because destruction is irreversible and its record must outlive
it (§8.14.4); an access request destroys nothing and needs no such record. A
host that logs access does so in host logs, not in memory -- and an audit grain
naming the subject of a DSAR would itself become personal data about that
subject, in an immutable replicating store.

**Result.** The response carries every matched identity string (so the requester
can see *which* keys matched, including derived ones) and the selected grains.
Hosts SHOULD also offer the result as a portable export, since Art. 20 asks for
a machine-readable form rather than a rendering.


---

## 9. Semantic Shortcuts

Shortcuts are **syntactic sugar** -- they desugar to standard WHERE/pipeline clauses. The desugared form is always valid and produces identical results.

### 9.1 ABOUT

```sql
RECALL facts ABOUT "alice"
-- Desugars to: RECALL facts WHERE subject = "alice"
-- Falls back to: RECALL facts WHERE query = "alice" (if no structural match)
```

### 9.2 RECENT

```sql
RECALL events ABOUT "alice" RECENT 5
-- Desugars to: RECALL events WHERE subject = "alice" ORDER BY time DESC LIMIT 5
```

### 9.3 SINCE

```sql
RECALL events SINCE "last week"
-- Desugars to: RECALL events WHERE time = "last week"
```

### 9.4 LIKE

```sql
RECALL LIKE "machine learning best practices"
-- Desugars to: RECALL WHERE query = "machine learning best practices"
```

### 9.5 MY

```sql
RECALL MY facts
-- Desugars to: RECALL facts WHERE user_id = $current_user_id
```

### 9.6 CONTRADICTIONS

```sql
RECALL facts ABOUT "alice" CONTRADICTIONS
-- Desugars to: RECALL facts WHERE subject = "alice" AND contradicted = true
--              WITH contradiction_detection
```

### 9.7 BETWEEN

```sql
RECALL events BETWEEN 1709251200 AND 1709337600
-- Desugars to: RECALL events WHERE time BETWEEN 1709251200 AND 1709337600
```

### 9.8 Shortcut Combination Rules

| Combination | Valid? | Notes |
|------------|--------|-------|
| ABOUT + WHERE | Yes | ABOUT becomes an additional AND condition |
| ABOUT + LIKE | No | Ambiguous -- error `CAL-E060` |
| RECENT + LIMIT | No | Ambiguous -- error `CAL-E060` |
| RECENT + ORDER BY | No | Ambiguous -- error `CAL-E060` |
| SINCE + WHERE time | No | Ambiguous -- error `CAL-E060` |
| SINCE + BETWEEN | No | Ambiguous -- error `CAL-E060` |
| MY + WHERE user_id | No | Ambiguous -- error `CAL-E060` |
| CONTRADICTIONS + WITH contradiction_detection | Yes | Redundant but not an error |
| AS + FORMAT (in ASSEMBLE) | Yes | AS controls per-source; FORMAT controls assembly |

---

## 10. FORMAT System

### 10.1 Semantic Presets

| Preset | Output Format | Description |
|--------|--------------|-------------|
| `structured` / `sml` | SML-based | Semantic tag structure optimised for LLM consumption (see [SML spec](./SEMANTIC-MARKUP-LANGUAGE-SML-SPECIFICATION.md)) |
| `readable` / `markdown` | Markdown | Human-readable, default for ASSEMBLE |
| `compact` / `text` | Plain text | Minimal, token-efficient |
| `data` / `json` | JSON | Machine-readable structured data |
| `yaml` | YAML | YAML structure |
| `triples` | Triples | Subject-relation-object triples |
| `toon` | TOON | Token-Oriented Object Notation — CSV-tabular for uniform grain arrays; ~40% fewer tokens vs JSON. Optimised for large RECALL result sets and budget-constrained ASSEMBLE. See Section 10.9. |

### 10.1.1 Multi-Format Output

The `FORMAT` and `AS` clauses accept either a single format name or a bracketed list of format names. When a list is provided, the implementation executes the query **once** and renders the result set into **each** requested format, returning all renderings in a single response.

**Single format** (existing behavior):

```
CAL/1 RECALL facts ABOUT "alice" FORMAT markdown
CAL/1 RECALL facts ABOUT "alice" AS json
```

**Multi-format:**

```
CAL/1 RECALL facts ABOUT "alice" FORMAT [markdown, json]
CAL/1 RECALL facts ABOUT "alice" AS [markdown, json]
```

**Multi-format with aliases:**

Each format in a bracketed list MAY include an `AS <identifier>` alias. When present, the alias becomes the key in the multi-format response object (Section 14.2.1) instead of the canonical format name. Aliases are particularly useful when the list contains multiple templates (which would otherwise all share the key `"template"`) or when the client wants semantically meaningful keys.

```
CAL/1 RECALL facts FORMAT [json AS customers, markdown AS report]
CAL/1 RECALL facts FORMAT [json AS structured, TEMPLATE "{{grain.subject}}: {{grain.object}}" AS oneliner]
CAL/1 RECALL facts FORMAT [TEMPLATE "{{grain.subject}}" AS names, TEMPLATE "{{grain.object}}" AS values]
```

**Rules:**

1. The query executes once. All format renderings share the same result set.
2. When a single format is specified, the response uses the `"format"` / `"content"` fields (Section 14.2). When a format list is specified, the response uses a `"formats"` object keyed by format name or alias (Section 14.2.1).
3. `FORMAT [json]` (single-element list) uses the multi-format response shape (`"formats"` object with one key), not the single-format shape.
4. `FORMAT []` (empty list) is a parse error.
5. Duplicate format names in the list (without aliases) are deduplicated silently; each format is rendered at most once.
6. Maximum 5 formats per list. Exceeding this limit produces a `CAL-E110` error.
7. When aliases are used, the effective key (alias or canonical name) MUST be unique across the list. Duplicate keys where at least one is an explicit alias produce a `CAL-E113` error.
8. Aliases follow identifier syntax (`[a-z][a-z0-9_]*`). Aliases are only supported in bracketed format lists, not in bare comma-separated lists.
9. A `TEMPLATE "<text>"` entry is an inline template in the `ELEMENT` shorthand form (Section 10.6.1) — the string renders one grain and the engine iterates. `TEMPLATE <identifier>` is a reference to a registered template; the two are distinguished by token class (quoted = body, bare = name).

### 10.2 Custom Templates (Mustache-subset)

CAL templates use a strict subset of Mustache:
- Variable interpolation: `{{variable}}`
- Sections (conditional blocks): `{{#section}}...{{/section}}`
- Inverted sections (if-not): `{{^section}}...{{/section}}`
- Comments: `{{! comment }}`
- Constrained iteration: `{{#each}}` block (capped at 200 iterations)

**Excluded:** Lambdas, partials, set delimiter, unescaped interpolation.

### 10.3 Content Projection Model

CAL output is consumed by LLMs, not database clients. The **Content Projection Model** defines how OMS grain fields map to LLM-friendly output -- what becomes readable text content, what becomes lightweight metadata attributes, and what stays in the machine envelope only.

#### 10.3.1 Design Principle

> **The output format reflects the consumer's mental model, not the storage system's data model.**
>
> OMS stores grains as structured triples with rich metadata (hashes, namespaces, short keys, provenance chains). LLMs think in natural language with lightweight structural hints. The projection model bridges these two worlds: it composes grain fields into readable text with just enough structure for the LLM to categorize and weight information.

#### 10.3.2 Per-Grain-Type Content Projection

Each grain type defines a **content rule** (what becomes the text content of the output element) and an **attribute set** (what becomes metadata on the element). All other fields remain in the machine envelope (Section 14.1) and never appear in formatted output.

| Grain Type | Text Content Rule | Default Attributes |
|-----------|------------------|-------------------|
| **Fact** | `humanize(relation) + " " + object` | `subject`, `confidence`? |
| **Event** | `content` | `role`, `time`? |
| **Goal** | `object` (the objective description) | `subject`, `state`?, `deadline`? |
| **Tool** | `object` (tool result summary) | `tool`, `phase`? |
| **Observation** | `object` (what was observed) | `observer`? |
| **Reasoning** | `conclusion` | `type`? |
| **State** | `plan` (summary) | `context`? |
| **Workflow** | `nodes` (joined as readable text) | `name`? |
| **Trigger** | what the rule watches | `kind`, `scope`? |
| **Consensus** | `object` (the agreed claim) | `threshold`?, `count`? |
| **Consent** | `purpose` | `action`, `grantor`, `grantee` |
| **Skill** | `description` (what the skill enables) | `name`, `proficiency`?, `domain`? |
| **Recommendation** | `summary` (the rendered proposal one-liner) | `target`, `severity`?, `analyzer`? |

Attributes marked with `?` are included at `standard` and `full` disclosure levels only, omitted at `summary` level.

#### 10.3.3 The `humanize()` Function

The `humanize()` function transforms OMS relation strings into human-readable text:

1. **Strip namespace prefix:** `"mg:prefers"` → `"prefers"`
2. **Replace underscores with spaces:** `"works_at"` → `"works at"`
3. **Preserve custom relations as-is after stripping:** `"acme:similar_to"` → `"similar to"`

Implementations MUST apply `humanize()` to relation strings in formatted output. The raw relation value remains available in the machine envelope.

#### 10.3.4 Time Humanization

Timestamps in formatted output SHOULD use relative human-readable form by default:

| Age | Formatted As |
|-----|-------------|
| < 1 hour | `"Nm ago"` (e.g., `"23m ago"`) |
| < 24 hours | `"Nh ago"` (e.g., `"3h ago"`) |
| < 7 days | `"yesterday"`, `"2d ago"`, etc. |
| < 30 days | `"2w ago"`, `"3w ago"` |
| < 1 year | `"Mar 1"`, `"Jan 15"` |
| >= 1 year | `"Mar 2025"`, `"2024"` |

Full ISO 8601 timestamps remain in the machine envelope. Implementations MAY provide a `WITH iso_timestamps` option to override humanization.

#### 10.3.5 The PROJECT Clause

The `PROJECT` clause overrides default content projection, allowing queries to surface custom or domain-specific fields:

```sql
CAL/1 RECALL facts ABOUT "alice"
  PROJECT content(relation, object), attr(confidence, x_department)
  LIMIT 10 AS sml
```

**Syntax:**
```
PROJECT content(field, ...), attr(field, ...)
```

- `content(...)` -- fields composed into text content via concatenation with space separator. Relation-type fields are passed through `humanize()`.
- `attr(...)` -- fields rendered as element attributes.
- Fields not listed in either `content()` or `attr()` are excluded from formatted output.

**Without PROJECT**, the per-grain-type defaults (Section 10.3.2) apply. This is the common case.

**With PROJECT**, the query author has explicit control:

```sql
-- Surface domain profile fields
RECALL observations WHERE tags INCLUDE ["profile:healthcare"]
  PROJECT content(object), attr(observer, hc:patient_id, hc:encounter_id)
  AS sml

-- Produces:
-- <observation observer="dr-smith" hc:patient_id="P-1234" hc:encounter_id="E-567">
--   elevated heart rate detected
-- </observation>
```

### 10.4 Semantic Markup Language (SML)

SML is now a standalone specification. See **[SEMANTIC-MARKUP-LANGUAGE-SML-SPECIFICATION.md](./SEMANTIC-MARKUP-LANGUAGE-SML-SPECIFICATION.md)** for the full SML definition, structural rules, comprehensive example, and progressive disclosure model.

SML is the default output format for the `structured` / `sml` preset. The Content Projection Model (Section 10.3) and template engine (Section 10.2) apply to SML output as defined in this CAL specification.

### 10.5 Template Variables

#### Assembly-Level Variables

| Variable | Type | Description |
|----------|------|-------------|
| `{{assembly.name}}` | string | Context name |
| `{{assembly.intent}}` | string | FOR clause text |
| `{{assembly.source_count}}` | integer | Number of sources |
| `{{assembly.grain_count}}` | integer | Total grains included |
| `{{budget.total}}` | integer | Total budget |
| `{{budget.used}}` | integer | Budget consumed |
| `{{budget.remaining}}` | integer | Remaining budget |
| `{{budget.unit}}` | string | `"tokens"` or `"grains"` |
| `{{budget.utilization}}` | number | 0.0-1.0 utilization ratio |
| `{{disclosure.level}}` | string | Disclosure level |
| `{{timestamp}}` | string | ISO 8601 assembly timestamp |

#### Source-Level Variables

| Variable | Type | Description |
|----------|------|-------------|
| `{{source.label}}` | string | Source label |
| `{{source.index}}` | integer | 0-based position |
| `{{source.priority}}` | integer | 1-based priority rank |
| `{{source.grain_count}}` | integer | Grains in this source |
| `{{source.tokens_used}}` | integer | Tokens consumed |
| `{{source.truncated}}` | boolean | Whether grains were cut for budget |

#### Grain-Level Variables

| Variable | Type | Description |
|----------|------|-------------|
| `{{grain.content}}` | string | **Projected text content** (per Section 10.3.2 content rules) |
| `{{grain.type}}` | string | Grain type (used as SML element name) |
| `{{grain.subject}}` | string | Triple subject |
| `{{grain.relation}}` | string | Raw triple relation (with namespace) |
| `{{grain.humanized_relation}}` | string | Humanized relation (namespace stripped, underscores replaced) |
| `{{grain.object}}` | string | Triple object |
| `{{grain.confidence}}` | number | Confidence [0.0, 1.0] |
| `{{grain.importance}}` | number | Importance [0.0, 1.0] |
| `{{grain.tags}}` | string | Comma-separated tags |
| `{{grain.created_at}}` | string | ISO 8601 timestamp |
| `{{grain.relative_time}}` | string | Humanized relative time (e.g., "2h ago") |
| `{{grain.score}}` | number | Relevance score |
| `{{grain.hash}}` | string | Content address (for machine envelope use, not LLM output) |
| `{{#grain.is_full}}` | section | True when disclosure = full |
| `{{#grain.is_summary}}` | section | True when disclosure = summary |

### 10.6 DEFINE TEMPLATE

Templates use the flat semantic model. The `ELEMENT` section defines how each grain renders, and elements are emitted directly without group wrappers:

```sql
CAL/1 DEFINE TEMPLATE semantic_sml
  EXTENDS structured
  HEADER {
<context intent="{{assembly.intent}}">
  }
  ELEMENT {
  <{{grain.type}} subject="{{grain.subject}}"{{#grain.confidence}} confidence="{{grain.confidence}}"{{/grain.confidence}}>{{grain.content}}</{{grain.type}}>
  }
  ELEMENT_SUMMARY {
  <{{grain.type}} subject="{{grain.subject}}">{{grain.content}}</{{grain.type}}>
  }
  SOURCE_BREAK {

  }
  FOOTER {
</context>
  }
```

**Usage:**
```sql
CAL/1 ASSEMBLE conversation_context
  FOR "helping alice with her project"
  FROM facts: (RECALL facts ABOUT "alice" LIMIT 20),
       goals: (RECALL goals ABOUT "alice" RECENT 5)
  BUDGET 3000 tokens
  FORMAT TEMPLATE semantic_sml
```

**Inline templates:**

An inline template supplies a body at the point of use instead of naming a
registered one. Two forms are available — a braced section list, and a bare
string:

```sql
FORMAT TEMPLATE {
  ELEMENT {
- [{{grain.type}}] {{grain.content}}{{#grain.confidence}} ({{grain.confidence}}){{/grain.confidence}}
  }
}

FORMAT TEMPLATE "- [{{grain.type}}] {{grain.content}}"
```

#### 10.6.1 The `ELEMENT` Shorthand

A template whose only section is `ELEMENT` — by far the most common case —
MAY be written as a single string in place of the section list. The string
form is defined by equivalence:

> `TEMPLATE "<text>"` is exactly equivalent to `TEMPLATE { ELEMENT { <text> } }`,
> and `DEFINE TEMPLATE <name> AS "<text>"` is exactly equivalent to
> `DEFINE TEMPLATE <name> ELEMENT { <text> }`.

The shorthand is therefore **per-element**, not whole-result: the string is the
rendering of *one* grain, the engine iterates, and elements are emitted exactly
as they are for an `ELEMENT` section. It introduces no second rendering model,
no additional variables, and no separate scope — `{{grain.*}}` is bound per
grain, as in any `ELEMENT` body.

```sql
-- These two definitions are indistinguishable to the engine.
CAL/1 DEFINE TEMPLATE oneliner AS "{{grain.subject}}: {{grain.content}}"

CAL/1 DEFINE TEMPLATE oneliner
  ELEMENT {
{{grain.subject}}: {{grain.content}}
  }
```

Because the shorthand *is* an `ELEMENT` section, everything defined for
sections applies to it unchanged: `EXTENDS` composes normally (Section 10.7 —
undefined sections, including `ELEMENT_SUMMARY` under budget pressure, come
from the parent preset), the Section 10.8 limits apply to the string as the
template body, and the Section 10.2 Mustache subset and Section 10.5 variable
set are the same.

The named and inline forms are disambiguated by token class, not by lookahead:
`TEMPLATE <identifier>` is a reference to a registered template,
`TEMPLATE <string>` is a shorthand body, `TEMPLATE { ... }` is a section list.
Template names are identifiers (`template_name = identifier`) precisely so that
a quoted argument is always a body and never a name.

A template that needs more than one section — a `HEADER`, a `FOOTER`, an
explicit `ELEMENT_SUMMARY`, a `SOURCE_BREAK` — MUST use the section list. A
definition MUST NOT combine the two forms.

**All 13 grain types rendered:**
```sml
<context intent="helping alice prepare her Q1 engineering review">

  <fact subject="alice" confidence="0.95">prefers dark mode in all tools</fact>
  <fact subject="alice" confidence="0.88">requires keyboard shortcuts for productivity</fact>
  <fact subject="alice" confidence="0.82">works best in deep-focus blocks of 90 minutes</fact>

  <goal subject="alice" state="active" deadline="2026-03-15">complete Q1 engineering review presentation</goal>
  <goal subject="alice" state="active">reduce P0 incident rate by 20% in Q2</goal>

  <event role="user" time="10m ago">Can you help me pull together the Q1 metrics?</event>
  <event role="assistant" time="10m ago">Sure — retrieving deployment counts, incident data, and velocity now.</event>
  <event role="user" time="8m ago">Focus on the reliability numbers first.</event>

  <tool tool="query_metrics" phase="completed">retrieved 47 deployments and 3 P0 incidents for Q1 2026</tool>
  <tool tool="search_docs" phase="completed">found Q1 review template in confluence/engineering/reviews</tool>

  <observation observer="system">alice opened incident-dashboard at 09:14 UTC</observation>
  <observation observer="system" source="calendar">Q1 review presentation scheduled for 2026-03-15 14:00 UTC</observation>

  <reasoning type="deductive">alice is prioritising reliability given 3 P0 incidents; lead with incident reduction narrative</reasoning>
  <reasoning type="abductive">low velocity in week 8 likely caused by the infra migration; flag as contextual outlier</reasoning>

  <state context="q1_review_prep">outlining slides: 1. headline metrics  2. incident retrospective  3. velocity trend  4. Q2 goals</state>

  <workflow name="review prep">retrieve_metrics -> identify_narrative -> draft_outline -> populate_data -> send_for_review</workflow>

  <consensus threshold="3" count="4">Q1 deployment frequency improved 18% over Q4 2025</consensus>

  <consent action="granted" grantor="alice" grantee="agent">access engineering metrics dashboards for review preparation</consent>

  <skill name="metrics_review" proficiency="0.82" domain="software">summarise quarterly engineering metrics and surface the reliability narrative</skill>

  <recommendation target="entity:alice/velocity" severity="low">consolidate 2 duplicate "week 8 velocity" observations into one</recommendation>

</context>
```

### 10.6.3 Template identity, revision, and DROP TEMPLATE (new in 1.3)

A registered template is a **stored definition** and lives in the memory's
metadata table under `tpl:<name>` (OMS §28.4.1), on exactly the terms §8.18.1
sets out for saved queries: carried by the memory rather than the client, not a
grain, replicating latest-wins, with usage stamps that do not replicate.

**Identity.** A template's identity is `<namespace>/<name>` — the form OMS
§8.12 already uses as a Recommendation `target_ref`. Versions 1.0–1.2 gave a
template a bare 64-character name and nothing else, which left that target
unpinnable: a Recommendation applied against `template:acct/invoice_row` could
not say which revision it changed, and so could not store an honest inverse.

Every stored template therefore carries the two stamps OMS §28.4.1 requires of
any definition — `updated_at` (the replication tiebreaker) and
`definition_hash` (the SHA-256 identity of this revision's body). A host MUST
report both from `DESCRIBE TEMPLATES`, so a reviewer can see which revision a
proposal targets and an auditor can tell whether the template that rendered a
`summary` is still the template that would render it now.

The revision covers the template's **own** body, not its parent's. A template
that `EXTENDS` a preset resolves against whatever that preset is at render
time; presets are implementation-shipped and versioned with the
implementation, so they are not stored definitions and have no
`definition_hash`.

**Built-ins are not stored definitions.** The presets and built-in templates an
implementation ships are code, not metadata. They MUST NOT be droppable, MUST
NOT be overwritten by `DEFINE TEMPLATE`, and MUST NOT be persisted into the
metadata table — a memory that carried a copy of a built-in would pin one
implementation's rendering of it forever.

**DROP TEMPLATE.**

```sql
DROP TEMPLATE "invoice_row"
DESCRIBE TEMPLATES
```

`DROP TEMPLATE` removes a stored template definition. It is Tier 3 (`admin`),
it touches no grain, and it is lossless against memory (§2.4): the grains the
template ever rendered are untouched, and re-running the `DEFINE` restores it
byte-for-byte. Dropping a name that does not exist, or a built-in, is an error;
dropping a template another template `EXTENDS` MUST be refused rather than
leaving a dangling parent.

Nothing here is retroactive. Output already rendered through a dropped or
replaced template is text that was already produced; a template is a rendering
instruction (§10.8), so changing one changes what renders *next*.

`UNDEFINE` remains reserved and unspecified (§2.4, Appendix D). It was the
placeholder for these semantics, `DROP TEMPLATE` is the spelling that shipped,
and an implementation MUST NOT accept `UNDEFINE` as an alias for it.


### 10.7 Template Inheritance

Templates inherit from presets via `EXTENDS`. Sections not defined in the template use the parent preset's definition. Inheritance depth is limited to 1 (template -> preset only). Default parent is `readable`.

The `data` preset cannot be extended (it outputs structural JSON, not template-driven text).

Inheritance is defined over sections, so it applies to the `ELEMENT` shorthand (Section 10.6.1) without special-casing: a shorthand template defines `ELEMENT` and inherits every other section — including the `ELEMENT_SUMMARY` the engine substitutes under budget pressure — from its parent.

### 10.8 Template Safety Model

Templates are rendering instructions, not programs:
- No file system access, no code execution, no network requests
- No access to environment variables or other namespaces
- Undefined variables render as empty string
- `{{#each}}` capped at 200 iterations
- Output bounded by budget * 2 characters
- Validated at definition time for syntax, known variables, section balance, and size

**Template Constraints:**

| Constraint | Limit |
|-----------|-------|
| Max template body size | 4096 bytes |
| Max templates per namespace | 50 |
| Max nesting depth | 5 levels |
| Max `{{#each}}` iterations | 200 |
| Template name length | 64 characters |
| Inheritance depth | 1 |
| Variable set | Closed |
| Revision stamps | `updated_at` + `definition_hash` (§10.6.3, OMS §28.4.1) |

### 10.9 TOON — Token-Oriented Object Notation

#### 10.9.1 What is TOON?

**TOON (Token-Oriented Object Notation)** is a compact, LLM-native encoding format defined by the [TOON specification](https://github.com/toon-format/spec) (v3.0). It combines:

- **YAML-like indentation** for nested or non-uniform objects
- **CSV-style tabular layout** for uniform arrays of objects

For CAL's primary output shape — arrays of grains of the same type — TOON's tabular mode achieves approximately **40% fewer tokens** compared to JSON while preserving full semantic fidelity. The same content projection rules from Section 10.3 apply: `humanize()`, time humanization, and per-grain-type content rules all carry through.

TOON is complementary to SML, not a replacement:

| Property | SML | TOON |
|----------|-----|------|
| Semantic tag names | Yes (`<fact>`, `<goal>`, …) | No — grain type in section header only |
| Token efficiency | Moderate | High (~40% fewer vs JSON) |
| Uniform arrays | One element per line | CSV table — optimal |
| Mixed grain types | Natural (each type has its own tag) | Grouped sections |
| LLM-native | Yes | Yes |
| Best for | Rich context with clear epistemic signals | Large result sets, tight budgets |

#### 10.9.2 When to Use TOON

Prefer `FORMAT toon` / `AS toon` when:

1. **Large RECALL result sets** — uniform grain arrays of 20+ grains where token savings matter.
2. **Tight ASSEMBLE budgets** — when the BUDGET clause is at or near the limit of available context.
3. **Homogeneous source queries** — ASSEMBLE sources that each contain a single grain type.

Prefer SML when:
- The LLM must make epistemic decisions per grain (trust calibration based on tag name).
- Mixed grain types appear within a single source without logical grouping.
- The downstream prompt system is tuned for `<tag>` signals.

#### 10.9.3 TOON Rendering Rules for RECALL Results

For a `RECALL` result returning N grains of a single type, the TOON output is a **root-level array document**. The first line is the TOON array header (detected as root array by §5 of the TOON spec); rows follow at depth 0 (no indentation):

```
type[N]{col1,col2,...}:
value1,value2,...
value1,value2,...
...
```

Where:
- `type` is the grain type (plural form, lowercase): `facts`, `events`, `goals`, etc.
- `[N]` is the count of rows.
- `{col1,col2,...}` are the projected field names.
- The trailing `:` on the header is **required** by the TOON grammar (`header = [key] bracket-seg [fields-seg] ":"`).
- Rows follow at depth 0 — no indentation — because the array is the root document.

**Column set for each grain type** (same content projection as Section 10.3.2):

| Grain Type | Columns (standard disclosure) |
|-----------|-------------------------------|
| `facts` | `subject`, `content`, `confidence` |
| `events` | `role`, `time`, `content` |
| `goals` | `subject`, `content`, `state` |
| `tools` | `tool`, `phase`, `content` |
| `observations` | `observer`, `content` |
| `reasonings` | `type`, `content` |
| `states` | `context`, `content` |
| `workflows` | `name`, `content` |
| `consensuses` | `threshold`, `count`, `content` |
| `consents` | `grantor`, `grantee`, `action`, `content` |
| `skills` | `name`, `content`, `proficiency` |
| `recommendations` | `target`, `content`, `severity` |
| `triggers` | `kind`, `content` |

At `summary` disclosure, `confidence`, `state`, `phase`, and `type` columns are omitted.
At `full` disclosure, additional columns `source` and `observed` are appended.

**Example — `RECALL facts ABOUT "alice" LIMIT 3 AS toon`:**
```
facts[3]{subject,content,confidence}:
alice,prefers dark mode,0.95
alice,prefers vim,0.9
alice,works best in deep-focus blocks of 90 minutes,0.82
```

**Example — `RECALL events WHERE user_id = "alice" RECENT 3 AS toon`:**
```
events[3]{role,time,content}:
user,10m ago,Can you help me pull together the Q1 metrics?
assistant,10m ago,Sure — retrieving deployment counts and incident data.
user,8m ago,Focus on the reliability numbers first.
```

**String quoting.** A value in a tabular row MUST be double-quoted if it: contains the active delimiter (comma by default), has leading or trailing whitespace, is empty, matches a reserved literal (`true`, `false`, `null`), matches a numeric pattern, contains a leading hyphen, or contains any of `:`, `"`, `\`, `[`, `]`, `{`, `}`, or control characters. Only five escape sequences are valid inside quoted strings: `\\`, `\"`, `\n`, `\r`, `\t`. No `\u` escapes — Unicode characters appear as literal UTF-8. Implementations MUST apply `humanize()` to relation fields and time humanization to timestamp fields, identical to SML.

**Number canonicalization.** Numeric values (confidence, importance, scores) MUST be emitted in canonical decimal form: no exponent notation, no leading zeros except `0` itself, no trailing fractional zeros (`0.90` → `0.9`, `1.5000` → `1.5`). `NaN` and `±Infinity` map to `null`.

#### 10.9.4 TOON Rendering Rules for ASSEMBLE Results

For an `ASSEMBLE` result, the TOON output is a **root-level object document**. The first line is a metadata key-value pair, which causes the TOON parser to detect root form as "object" (per §5 of the TOON spec). Named grain-type arrays are then properties of that object; their tabular rows are indented **2 spaces** (depth+1):

```
context: <context_name>
intent: <for_clause_text>
tokens: <used>/<total>
<grain_type>[N]{col1,col2,...}:
  row1_val1,row1_val2,...
  row2_val1,row2_val2,...
<grain_type>[N]{col1,col2,...}:
  row1_val1,row1_val2,...
  ...
```

Rules:
- The metadata header uses `key: value` format (colon-space separator).
- The trailing `:` on every array header is **required** by the TOON grammar.
- Tabular rows are indented **2 spaces** because they are named properties of the root object.
- No blank lines between tabular rows within a group; one blank line between groups.
- Source labels are omitted from the output (ASSEMBLE TOON is grain-group-centric). To expose source attribution, use `FORMAT sml`.
- Within a group, all grains MUST be of the same type. If a source returns mixed types, the executor MUST split them into separate same-type groups.
- Groups are ordered by priority (highest priority first, matching the `PRIORITY` clause).

**Example — ASSEMBLE FORMAT toon:**

```
context: agent_context
intent: helping alice prepare her Q1 engineering review
tokens: 1847/2000
facts[3]{subject,content,confidence}:
  alice,prefers dark mode,0.95
  alice,prefers vim,0.9
  alice,works best in deep-focus blocks of 90 minutes,0.82
goals[2]{subject,content,state,deadline}:
  alice,complete Q1 engineering review,active,2026-03-15
  alice,reduce P0 incident rate by 20% in Q2,active,-
events[3]{role,time,content}:
  user,10m ago,Can you help me pull together the Q1 metrics?
  assistant,10m ago,Sure — retrieving deployment counts and incident data.
  user,8m ago,Focus on the reliability numbers first.
```

#### 10.9.5 Auto-TOON (Budget Pressure Hint)

When no explicit `FORMAT` is specified and all of the following conditions hold, implementations MAY automatically select `toon` as the output format instead of the default `sml`:

1. A `BUDGET` clause is present.
2. Estimated token utilization (from the EXPLAIN plan) exceeds **85%** of the budget.
3. All ASSEMBLE sources return a single grain type each (enabling full tabular mode).

When auto-TOON activates, the response MUST include a warning:
```json
{ "code": "CAL-W005", "message": "FORMAT auto-selected as toon due to budget pressure (>85% utilization estimate). Specify FORMAT explicitly to suppress this warning." }
```

Auto-TOON is opt-in at the server level. Servers report whether it is active via `DESCRIBE capabilities` (`auto_toon_enabled`).

#### 10.9.6 TOON and PROJECT

The `PROJECT` clause (Section 10.3.5) works with TOON. The projected fields become the TOON column headers:

```sql
CAL/1 RECALL observations WHERE tags INCLUDE ["profile:healthcare"]
  PROJECT content(object), attr(hc:patient_id, hc:encounter_id)
  LIMIT 10 AS toon
```

Output (root array — RECALL result):
```
observations[2]{content,hc:patient_id,hc:encounter_id}:
elevated heart rate detected,P-1234,E-567
blood pressure within normal range,P-1235,E-568
```

#### 10.9.7 TOON and Streaming

TOON output is compatible with the STREAM protocol (Section 11). When streaming TOON, each `source_data` chunk carries one or more complete CSV rows — never partial rows. The metadata header is emitted in the first chunk.

#### 10.9.8 TOON Wire Format in application/json+cal

In `application/json+cal`, the formatted TOON output is a plain string in `formatted_context.text` (for ASSEMBLE) or `formatted` (for RECALL). The media type annotation uses `"format": "toon"`.

---

## 11. Streaming Protocol

### 11.1 Event Types

Streaming ASSEMBLE uses a typed event stream. Events are delivered in causal order.

| Event Type | Phase | Description |
|-----------|-------|-------------|
| `assembly_started` | Init | Stream opened, assembly ID assigned |
| `source_started` | Source Resolution | A RECALL query has begun |
| `source_completed` | Source Resolution | A RECALL query has finished |
| `dedup_completed` | Deduplication | Cross-source dedup finished |
| `budget_allocated` | Budget Allocation | Token budget distributed |
| `disclosure_decided` | Progressive Disclosure | Disclosure levels assigned |
| `chunk` | Formatting | A chunk of formatted output |
| `assembly_completed` | Done | All phases complete |
| `error` | Any | An error occurred |
| `cancelled` | Any | Stream cancelled |

**Ordering invariant:**
```
assembly_started
  -> source_started(s1) -> source_completed(s1)
  -> source_started(s2) -> source_completed(s2)
  -> ...                                          (sources may interleave)
  -> dedup_completed
  -> budget_allocated
  -> chunk(1) -> chunk(2) -> ... -> chunk(n)
  -> assembly_completed
```

### 11.2 STREAM Clause

```sql
ASSEMBLE user_context
  FROM facts: (RECALL facts ABOUT "alice")
  BUDGET 2000 tokens
  STREAM { all }                              -- all events
  -- or: STREAM { progress, chunks }          -- specific events
  -- or: STREAM { all, chunk_size = 200 }     -- custom chunk size
  -- or: STREAM                               -- bare = all events
```

| Option | Events Emitted |
|--------|---------------|
| `progress` | assembly_started, source_started, source_completed, assembly_completed |
| `budget` | dedup_completed, budget_allocated, disclosure_decided |
| `chunks` | chunk (formatted output) |
| `all` | All of the above |
| `chunk_size = N` | Target tokens per chunk (default 100, min 20, max 1000) |

`error` and `cancelled` events are ALWAYS emitted regardless of options.

### 11.3 Transport Bindings

#### SSE (Server-Sent Events) -- RECOMMENDED

```http
POST /memories/{id}/cal HTTP/1.1
Content-Type: application/json+cal
Accept: text/event-stream

event: assembly_started
data: {"type":"assembly_started","assembly_id":"asm_a1b2c3d4",...}

event: chunk
data: {"type":"chunk","chunk_index":0,"content":"## Context...","tokens":18,...}

event: assembly_completed
data: {"type":"assembly_completed","summary":{...}}
```

#### NDJSON (Fallback)

```http
POST /memories/{id}/cal HTTP/1.1
Accept: application/x-ndjson

{"type":"assembly_started","assembly_id":"asm_a1b2c3d4",...}
{"type":"chunk","chunk_index":0,...}
{"type":"assembly_completed",...}
```

#### WebSocket

Full-duplex with explicit pause/resume/cancel:

```json
{"action": "assemble", "request_id": "req_001", "payload": {...}}
{"action": "cancel", "assembly_id": "asm_a1b2c3d4"}
{"action": "pause", "assembly_id": "asm_a1b2c3d4"}
{"action": "resume", "assembly_id": "asm_a1b2c3d4"}
```

### 11.4 Progressive Budget Updates

Budget information is refined through the streaming phases: `assembly_started` (total known), `source_completed` (per-source estimates), `budget_allocated` (final allocation), `chunk` (running countdown via `budget_remaining`), `assembly_completed` (final utilization).

If a source fails, its allocated budget is redistributed and a revised `budget_allocated` event is emitted.

### 11.5 Cancellation

- **HTTP:** Client closes connection. Server also supports `DELETE /memories/{id}/cal/stream/{assembly_id}`.
- **WebSocket:** Client sends `{"action": "cancel", "assembly_id": "..."}`.
- Cancellation is best-effort. Partial results are valid and usable.
- Cancellation MUST be recorded in the audit trail.

### 11.6 Backpressure

- **SSE:** TCP-level flow control. Server buffers max 64KB unsent events. Stall timeout: 30 seconds.
- **WebSocket:** Explicit `pause`/`resume` actions. Chunk emission pauses; progress events continue.

**Streaming Constraints:**

| Constraint | Limit |
|-----------|-------|
| Max concurrent streams per client | 3 |
| Max event buffer per stream | 64 KB |
| Stream reconnection window | 10 s |
| Min chunk_size | 20 tokens |
| Max chunk_size | 1000 tokens |
| Default chunk_size | 100 tokens |
| Backpressure stall timeout | 30 s |

---

## 12. Domain Profile Querying

OMS defines domain profiles (healthcare, legal, finance, robotics, science, consumer, integration, mail). CAL provides structured access to domain-tagged grains.

`domain_prefix` is a **closed** production (§4), so a new OMS profile is not
usable from CAL until this specification admits its prefix. `mail:` is admitted
in 1.3 alongside OMS Appendix A.8; the closure is deliberate -- an open prefix
would make every misspelled field name a valid domain field rather than
`CAL-E004`.

### 12.1 Profile Querying via Tags

```sql
RECALL WHERE tags INCLUDE ["profile:healthcare"]
RECALL facts WHERE tags INCLUDE ["profile:healthcare"]
  AND subject = "patient:P-12345" AND relation IS PREFERENCE
```

### 12.2 Domain-Prefixed Fields

Domain-specific fields use OMS domain prefix convention:

| Domain | Prefix | Example Fields |
|--------|--------|---------------|
| Healthcare | `hc:` | `hc:patient_id`, `hc:encounter_id`, `hc:provider_id`, `hc:condition_code`, `hc:phi_category` |
| Legal | `legal:` | `legal:case_id`, `legal:jurisdiction`, `legal:privilege_status`, `legal:retention_category` |
| Finance | `fin:` | `fin:account_id`, `fin:transaction_id`, `fin:risk_category`, `fin:compliance_flag` |
| Robotics | `rob:` | `rob:device_id`, `rob:coordinate_frame`, `rob:safety_zone` |
| Science | `sci:` | `sci:experiment_id`, `sci:dataset_id`, `sci:methodology`, `sci:reproducibility_status` |
| Consumer | `con:` | `con:session_context`, `con:interaction_channel` |
| Integration | `int:` | `int:source_system`, `int:correlation_id`, `int:sync_status` |
| Mail | `mail:` | `mail:message_id`, `mail:from`, `mail:to`, `mail:subject`, `mail:folder` |

**Example:**
```sql
RECALL facts WHERE tags INCLUDE ["profile:healthcare"]
  AND hc:patient_id = "P-12345"
  AND hc:condition_code IN ("J06.9", "J20.9")
  AND relation = "mg:knows"
  ORDER BY time DESC LIMIT 20
```

The parser SHOULD emit warning `CAL-W002` if a domain field is used without the corresponding `profile:` tag.

---

## 13. Store Protocol Mapping

Every CAL statement maps to one or more OMS Store Protocol operations (OMS §28.4). This mapping is deterministic.

| CAL Statement | Min Store Ops | Max Store Ops | Operations |
|--------------|--------------|--------------|------------|
| RECALL | 1 | 1 | `query` or `search` |
| EXISTS | 1 | 1 | `exists` |
| HISTORY (hash) | 1 | 101 | `get` + chain walk |
| HISTORY (triple) | 1 | 1 | `query(include_superseded=true)` |
| EXPLAIN | 0 | 0 | Compile-time only |
| ADD | 1 | 1 | `put` |
| SUPERSEDE | 2 | 3 | `get` + `supersede` |
| REVERT | 3 | 4 | `get` + `get` + `supersede` |
| Set operation | 2 | 2 | One query per operand |
| ASSEMBLE | N | N | N x `query`/`search` (one per source) |
| DEFINE QUERY / DROP QUERY | 1 | 1 | `meta_put` / `meta_delete` (OMS §28.4.1) -- no grain operation |
| DEFINE TEMPLATE / DROP TEMPLATE | 1 | 1 | `meta_put` / `meta_delete` (OMS §28.4.1) -- no grain operation |
| RUN "<name>" | 1 | N | `meta_get` + the body's own operations |
| DESCRIBE QUERIES / TEMPLATES | 1 | 1 | `meta_scan` (OMS §28.4.1) |

---

## 14. Response Model

### 14.1 Machine Envelope

Every CAL response includes a `_cal` metadata block:

```json
{
  "_cal": {
    "version": "1.0",
    "statement_type": "recall",
    "tier": 0,
    "query_hash": "sha256:...",
    "duration_ms": 42,
    "budget": {
      "tokens_used": 1847,
      "grains_returned": 8,
      "grains_scanned": 156
    }
  },
  "results": [...],
  "total": 42,
  "next_cursor": "cursor:eyJ..."
}
```

### 14.2 LLM Content Layer

A formatted representation for direct insertion into LLM context windows. The content layer uses the **Content Projection Model** (Section 10.3) to transform grain fields into natural language with lightweight structural hints.

**SML format** (default for `structured` / `sml`):
```sml
<context intent="helping alice with project">

  <fact subject="alice" confidence="0.92">prefers dark mode</fact>
  <fact subject="alice" confidence="0.88">requires keyboard shortcuts</fact>

  <goal subject="alice" state="active">complete Q1 review</goal>

</context>
```

**Markdown format** (default for `readable` / `markdown`):
```markdown
## Context: helping alice with project

**Facts**
- alice prefers dark mode (confidence: 0.92)
- alice requires keyboard shortcuts (confidence: 0.88)

**Goals**
- alice: complete Q1 review (active)
```

**Compact format** (for `compact` / `text`):
```
[fact] alice prefers dark mode (0.92)
[fact] alice requires keyboard shortcuts (0.88)
[goal] alice: complete Q1 review (active)
```

The machine envelope (Section 14.1) carries hashes, namespaces, full timestamps, and other storage metadata. These MUST NOT appear in the LLM content layer.

#### 14.2.1 Multi-Format Response

When the query specifies a format list (`FORMAT [markdown, json]`), the response replaces the single `"format"` / `"content"` fields with a `"formats"` object keyed by format name (or alias, if provided). Each value is the rendered text for that format.

```json
{
  "_cal": {
    "version": "1.0",
    "statement_type": "recall",
    "query_hash": "sha256:...",
    "duration_ms": 38
  },
  "formats": {
    "markdown": "## Facts\n- alice prefers dark mode (confidence: 0.92)\n",
    "json": [
      {"subject": "alice", "relation": "prefers", "object": "dark mode", "confidence": 0.92}
    ]
  },
  "grain_count": 1,
  "total": 1
}
```

**With aliases** (`FORMAT [json AS structured, TEMPLATE "{{grain.subject}}: {{grain.object}}" AS oneliner]`):

```json
{
  "_cal": { "version": "1.0", "statement_type": "recall", "query_hash": "sha256:...", "duration_ms": 42 },
  "formats": {
    "structured": [{"subject": "alice", "relation": "prefers", "object": "dark mode"}],
    "oneliner": "alice: dark mode\n"
  },
  "grain_count": 1,
  "total": 1
}
```

**Discriminator:** Clients distinguish single-format from multi-format responses by checking for `"format"` (string) vs `"formats"` (object). Exactly one of the two keys is present when a FORMAT clause was specified. When aliases are used, the `"formats"` keys are the alias identifiers, not the canonical format names.

### 14.3 Progressive Disclosure

| Level | Metadata Density | When Used |
|-------|-----------------|-----------|
| `summary` | Tag name + subject + content only | Token budget tight (<1000 tokens) |
| `standard` | + confidence, role, state, time | Default |
| `full` | + source_type, importance, tags, verification_status | Token budget generous or LIMIT <= 5 |

Progressive disclosure controls **metadata density on a flat structure**, not nesting depth. The element shape stays the same across all levels -- only the number of attributes changes.

### 14.4 Per-Grain-Type Content Projection

Each grain type projects its fields into a **text content** string and **attribute set** using the rules defined in Section 10.3.2. The following table shows the projected output for each type:

| Grain Type | Projected Text Content | Example Output (sml) |
|-----------|----------------------|---------------------|
| Fact | `humanize(relation) + " " + object` | `<fact subject="alice" confidence="0.95">prefers dark mode in all tools</fact>` |
| Event | `content` | `<event role="user" time="10m ago">Can you help me pull together the Q1 metrics?</event>` |
| Goal | `object` | `<goal subject="alice" state="active" deadline="2026-03-15">complete Q1 engineering review presentation</goal>` |
| Tool | `object` (tool result) | `<tool tool="query_metrics" phase="completed">retrieved 47 deployments and 3 P0 incidents for Q1 2026</tool>` |
| Observation | `object` | `<observation observer="system">alice opened incident-dashboard at 09:14 UTC</observation>` |
| Reasoning | `conclusion` | `<reasoning type="deductive">alice is prioritising reliability given 3 P0 incidents; lead with incident reduction narrative</reasoning>` |
| State | `plan` summary | `<state context="q1_review_prep">outlining slides: 1. headline metrics  2. incident retrospective  3. velocity trend  4. Q2 goals</state>` |
| Workflow | `nodes` joined | `<workflow name="review prep">retrieve_metrics -> identify_narrative -> draft_outline -> populate_data -> send_for_review</workflow>` |
| Trigger | what it watches | `<trigger kind="polling" scope="mailbox:accounts@example.com">poll the accounting mailbox every 2 minutes for invoices</trigger>` |
| Consensus | `object` | `<consensus threshold="3" count="4">Q1 deployment frequency improved 18% over Q4 2025</consensus>` |
| Consent | `purpose` | `<consent action="granted" grantor="alice" grantee="agent">access engineering metrics dashboards for review preparation</consent>` |
| Skill | `description` | `<skill name="incident_retro" proficiency="0.82" domain="engineering">run a blameless incident retrospective and distil the recurring themes</skill>` |
| Recommendation | `summary` (rendered) | `<recommendation target="entity:eng/alice" severity="low" analyzer="waiser.duplicate_sweep/1">merge 3 near-duplicate facts about alice's dashboard preferences</recommendation>` |

The `PROJECT` clause (Section 10.3.5) overrides these defaults when custom or domain-specific fields must be surfaced.

> **Note:** Section 10.3.2 is the normative source for content rules and attribute sets; this table illustrates the resulting output. When a grain type is added, Section 10.3.2 and this table MUST be updated together.

---

## 15. Dual Wire Format

### 15.1 Media Types

| Format | Media Type | Use Case |
|--------|-----------|----------|
| Text | `text/cal` | LLM generation, human authoring, documentation |
| JSON | `application/json+cal` | Programmatic construction, structured output |

### 15.2 Bijective Mapping

Every valid CAL statement has exactly one representation in each format, and conversion between them is lossless.

**text/cal:**
```
CAL/1 RECALL facts ABOUT "alice" WHERE confidence >= 0.8 RECENT 5 AS markdown
```

**application/json+cal:**
```json
{
  "cal_version": 1,
  "statement": "recall",
  "grain_type": "facts",
  "about": "alice",
  "where": [{ "field": "confidence", "op": ">=", "value": 0.8 }],
  "recent": 5,
  "as": "markdown"
}
```

### 15.3 Round-Trip Guarantee

`parse(serialize(parse(text))) == parse(text)` and `serialize(parse(serialize(json))) == serialize(json)`. Whitespace may differ; semantic content is identical.

### 15.4 Content Negotiation

Standard HTTP content negotiation applies. The `Accept` header controls response format. If absent, response format matches request format.

### 15.5 JSON Schema

The JSON format has published JSON Schemas (draft 2020-12):
- Request: `https://cal-spec.org/schema/v1/cal-request.schema.json`
- Response: `https://cal-spec.org/schema/v1/cal-response.schema.json`

Implementations MUST validate incoming `application/json+cal` against the schema before execution.

---

## 16. Internationalization

### 16.1 Character Encoding

CAL queries and responses MUST be UTF-8 encoded. Invalid UTF-8 sequences produce error `CAL-E070: InvalidUTF8`.

### 16.2 Unicode Normalization

All string comparisons use **NFC** normalization. String literals are NFC-normalized at parse time. Stored grain content is NFC-normalized at write time. Implementations MUST normalize.

### 16.3 Bidirectional Text (Bidi)

Grain content is stored in logical order. CAL rejects string literals containing bidi override characters (U+202A-U+202E, U+2066-U+2069) to prevent bidi-based spoofing attacks (error `CAL-E071: BidiOverrideRejected`).

### 16.4 Cross-Lingual Search

When `query = "..."`, `LIKE "..."`, or `ABOUT "..."` triggers semantic search, the search SHOULD work across languages when multilingual embeddings are available. Cross-lingual search is REQUIRED at Extended conformance level.

Implementations MUST declare cross-lingual capability in `DESCRIBE capabilities`:
```json
{
  "cross_lingual_search": true,
  "embedding_model": "multilingual-e5-large",
  "supported_languages": ["en", "es", "fr", "de", "ja", "zh", "ar"]
}
```

### 16.5 Locale-Aware Sorting

Default: Unicode code point order (binary sort). Locale-aware sorting requested via `WITH locale("xx")`:

```sql
RECALL facts ABOUT "alice" ORDER BY object ASC WITH locale("de")
```

Locale-aware sorting is optional. Implementations that do not support it MUST ignore the `locale()` option with a warning.

### 16.6 Identifier Safety

Field names and keywords are ASCII-only, never subject to Unicode normalization. This prevents confusion attacks with visually similar Unicode characters.

---

## 17. Execution Model

### 17.1 Query Pipeline (Tier 0)

```
CAL String
    |
    v
+----------+    +---------+    +----------+    +-----------+    +----------+
|  LEXER   |--->| PARSER  |--->|VALIDATOR |--->| PLANNER   |--->| EXECUTOR |
| Tokens   |    |CalStmt  |    |Type chk  |    |Query plan |    | Results  |
+----------+    +---------+    +----------+    +-----------+    +----------+
                                    |               |                |
                                    |          +----v------+   +----v----+
                                    |          |POLICY GATE|   | AUDIT   |
                                    |          |check_read |   | TRAIL   |
                                    |          +-----------+   +---------+
                               +----v-----+
                               | FIREWALL |
                               |complexity|
                               |deny list |
                               +----------+
```

### 17.2 Evolve Pipeline (Tier 1)

```
CAL String (ADD, SUPERSEDE, or REVERT)
    |
    v
+----------+    +---------+    +----------+    +-----------+
|  LEXER   |--->| PARSER  |--->|VALIDATOR |--->| TIER CHECK|
| Tokens   |    |CalStmt  |    |Type chk  |    |Token ok?  |
+----------+    +---------+    +----------+    +-----+-----+
                                                      |
                   +----------------------------------+
                   |
             +-----v------+    +-----------+    +----------+
             |POLICY GATE |--->| EXECUTOR  |--->|  AUDIT   |
             |check_write |    | add() or  |    |  TRAIL   |
             +------------+    | supersede |    +----------+
                               +-----------+
```

### 17.3 Resource Limits

| Resource | Limit | Spec-mandated? |
|----------|-------|---------------|
| Max query string length | 8,192 bytes | Yes |
| Max LIMIT value | 1,000 | Implementation-configurable |
| Default LIMIT (if omitted) | 20 | Yes |
| Max subquery nesting | 3 levels | Yes |
| Max pipeline stages | 5 | Yes |
| Max IN literal set size | 100 | Implementation-configurable |
| Max set operands | 5 | Yes |
| Max parameters per query | 20 | Yes |
| Query timeout | 5,000ms | Implementation-configurable |
| ASSEMBLE timeout | 10,000ms | Implementation-configurable |
| Parse time budget (queries <=4KB) | <1ms | Yes |
| Max ADD per minute | 20 | Implementation-configurable |
| Max SUPERSEDE per minute | 10 | Implementation-configurable |
| Max REVERT per minute | 5 | Implementation-configurable |
| Max SET clauses per ADD | 6 | Yes (3 required + 3 optional base) |
| Max SET clauses per SUPERSEDE | 4 | Yes |
| Max REASON length | 500 chars | Yes |
| Max BATCH queries | 10 | Yes |
| Max COALESCE branches | 5 | Yes |
| Max LET bindings | 5 | Yes |
| Max ASSEMBLE sources | 8 | Yes |
| Max BUDGET tokens | 16,000 | Yes |
| Max BUDGET grains | 200 | Yes |

### 17.4 Determinism Guarantees

| Property | Guarantee |
|----------|-----------|
| Same query + same state = same results | Yes |
| Tiebreaking for equal scores | Lexicographic hash order (ascending) |
| Parser is stateless | Yes |
| Decidability | Every string terminates in bounded time |
| ADD is idempotent | No (unique content address per call) |
| SUPERSEDE is idempotent | No (returns SupersessionConflict) |

---

## 18. Capability Token Model

### 18.1 Token Structure

**Tier 0 (read-only) token:**
```json
{
  "token_id": "uuid-v4",
  "namespace": "authorized-namespace",
  "user_id": "on-whose-behalf",
  "tier": 0,
  "allowed_ops": ["Recall", "Assemble", "Count", "Exists", "Explain",
                   "History", "Describe", "Batch", "Coalesce"],
  "issued_at": 1709337600000,
  "expires_at": 1709337900000,
  "max_uses": 1,
  "allowed_grain_types": [],
  "write_quota_remaining": 0,
  "signature": "hmac-sha256-signature"
}
```

**Tier 1 (evolve) token:**
```json
{
  "token_id": "uuid-v4",
  "namespace": "authorized-namespace",
  "user_id": "on-whose-behalf",
  "tier": 1,
  "allowed_ops": ["Recall", "Assemble", "Count", "Exists", "Explain",
                   "History", "Describe", "Batch", "Coalesce",
                   "Add", "Supersede", "Revert"],
  "issued_at": 1709337600000,
  "expires_at": 1709337900000,
  "max_uses": 1,
  "allowed_grain_types": ["fact"],
  "write_quota_remaining": 10,
  "signature": "hmac-sha256-signature"
}
```

### 18.2 Two-Phase Execution

```
1. LLM generates CAL string
2. Agent harness -> prepare endpoint
   -> Server authenticates, parses, validates, creates token
   -> For Tier 1: shows what will be added/superseded/reverted
   -> Returns {token, plan, tier, side_effects}
3. Agent harness reviews plan (REQUIRED for Tier 1, RECOMMENDED for Tier 0)
   -> Execute endpoint
   -> Server verifies token, checks expiration/replay, executes, returns results
```

### 18.3 Namespace Enforcement

The namespace is ALWAYS taken from the token, never from the query. Implementations MUST overwrite any namespace specified in the CAL string with the token's namespace.

### 18.4 Principal-Bound Sessions (Tier 2/3 -- new in 1.3)

Tier-2 and Tier-3 execution requires a **principal-bound session**: the host
authenticates a caller, resolves it to a principal name, and every statement
in the session executes *as* that principal.

- **The credential/policy split.** The host's credential store maps secrets
  (tokens, passwords, OS identity) to principal *names*; the memory's grant
  grains (OMS §12.6) map principal names to *rights*. Credentials MUST NOT
  be carried in grains, in CAL statements, or in the memory file in any
  form. A CAL statement can name a principal (`GRANT ... TO "agent:x"`); it
  can never mint, present, or reveal a credential.
- **The boundary rule.** Anything that needs a filesystem path, a
  credential, or a process to exist is a host operation; everything that
  acts on grains in an open memory is CAL. Hosts MUST NOT extend CAL past
  this boundary.
- **Observer class.** Each principal's host record carries its observer
  class (`human` or an agent class per OMS §24); audit grains take
  `observer_type` from that record, never from statement text.
- **Owner sessions.** A host MAY designate a local session as *owner* (OMS
  §12.6.4) -- the implicit superuser of the memory it opens, exempt from
  grant lookup. This preserves the single-operator experience: a memory
  with no grant grains is fully usable by its owner and confers nothing on
  anyone else.
- **Fail-closed.** An unauthenticated or unknown caller where a credential
  map is configured resolves to an anonymous principal with at most `read`;
  where no authorization model exists at all, Tier 2/3 statements MUST be
  refused (`CAL-E122`).

---

## 19. Policy Integration

### 19.1 CAL Inherits Sealed Policy

CAL queries execute through the same read path as all other interfaces. No CAL syntax can weaken the active policy.

| Policy Constraint | CAL Behavior |
|---|---|
| `encryption_required` | Transparent -- CAL reads decrypted grains via normal path |
| `consent_level = Explicit` | Grains without consent silently excluded |
| `processing_restriction` | Restricted users' data invisible in results |
| `pii_detection` / `phi_detection` | PII/PHI-tagged fields subject to policy redaction |
| `audit_required` | Every CAL query produces audit entry |

### 19.2 GDPR Implications

- **Art. 15 (Right of Access):** `REPORT SUBJECT "<id>"` (Tier 0, `read` verb, §8.19) -- the erasure selector in disclose-mode, so the answer matches what an erasure would remove.
- **Art. 16 (Right to Rectification):** CAL SUPERSEDE enables correction of inaccurate personal data.
- **Art. 17 (Right to Erasure):** Expressible in-language since 1.3 -- `FORGET SUBJECT <id> BECAUSE "..."` (Tier 2, `erase` verb, audited, one-way; §8.14). Hosts without Tier 2 continue to serve erasure via implementation-specific APIs.
- **Art. 20 (Data Portability):** `REPORT SUBJECT` (§8.19), whose result hosts SHOULD offer as a portable export.
- **Art. 25 (By Design):** Grammar-level safety qualifies as "by design" protection.

### 19.3 HIPAA Implications

- **Minimum Necessary:** Under HIPAA policy, CAL SHOULD enforce stricter default LIMIT and require field projection for PHI-containing results.
- **Audit:** CAL query audit entries MUST use pseudonymized user IDs and query hashes.

### 19.4 EU AI Act Implications

- **Transparency:** Results include `provenance_id` linking to immutable provenance chain.
- **Explanations:** `WITH explanation` provides compliant explanations.
- **Tier 1 Traceability:** Every SUPERSEDE/REVERT includes a mandatory REASON.

---

## 20. Threat Model

### 20.1 Attack Vectors and Defenses

| Attack | Severity | Defense |
|--------|----------|---------|
| **Prompt injection** | CRITICAL | Grammar-level exclusion + capability token scoping + query firewall |
| **Query injection** | HIGH | Parameterized queries (`$param`) -- no string concat |
| **Memory spam via ADD** | HIGH | Write quota (20/min); single-use tokens; mandatory REASON |
| **Hallucinated ADD** | HIGH | Two-phase prepare/execute; mandatory REASON; REVERT enables correction |
| **Supersede injection** | HIGH | Two-phase prepare/execute; write quotas; REVERT recovery |
| **Supersede storm** | HIGH | Write quotas; per-token single-use; rate limiting |
| **Resource exhaustion** | HIGH | Hard compiled limits, timeout enforcement |
| **Cross-namespace disclosure** | CRITICAL | Token-bound namespace enforcement |
| **Timing side-channel** | MEDIUM | Response jitter; identical error responses |
| **Privilege escalation** | HIGH | Token tier checked before execution |
| **Template injection** | MEDIUM | Closed variable set; no code execution; validated at definition time |
| **Streaming resource exhaustion** | MEDIUM | Max concurrent streams (3); backpressure; stall timeout (30s) |

### 20.2 Query Firewall

Implementations SHOULD perform static analysis between parsing and execution:
- Maximum query complexity score
- Deny patterns
- Mandatory namespace filter
- Maximum Tier 1 operations per session

### 20.3 Kill Switch

Implementations MUST support disabling CAL at runtime:
- **Master switch:** Disables all CAL operations (503).
- **Tier 1 switch:** Disables only ADD/SUPERSEDE/REVERT (403 for evolve, reads continue).

---

## 21. Audit Trail

Every CAL execution MUST produce an audit entry.

**Tier 0 (Read):**

| Field | Type | Description |
|-------|------|-------------|
| `token_id` | string | Capability token correlation |
| `query_hash` | string | SHA-256 of normalized CAL string |
| `namespace` | string | Token's namespace |
| `actor_id` | string | Pseudonymized (HMAC-SHA256) |
| `agent_id` | string? | Which LLM generated this query |
| `result_count` | integer | Number of grains returned |
| `tier` | integer | 0 |
| `duration_ms` | integer | Execution time |

**Tier 1 (Evolve):**

| Field | Type | Description |
|-------|------|-------------|
| `token_id` | string | Capability token correlation |
| `query_hash` | string | SHA-256 of normalized CAL string |
| `namespace` | string | Token's namespace |
| `actor_id` | string | Pseudonymized |
| `agent_id` | string? | Which LLM generated this query |
| `operation` | string | `"add"`, `"supersede"`, or `"revert"` |
| `target_hash` | string? | Target grain's content address |
| `new_hash` | string | Newly created grain's content address |
| `reason` | string | Mandatory reason text |
| `tier` | integer | 1 |
| `duration_ms` | integer | Execution time |

**Tier 2 (Destroy) and Tier 3 (Control) -- new in 1.3:**

| Field | Type | Description |
|-------|------|-------------|
| `query_hash` | string | SHA-256 of normalized CAL string |
| `namespace` | string | Effective namespace |
| `principal` | string | The session principal (§18.4) -- not pseudonymized; accountability requires the real name |
| `operation` | string | `"forget"`, `"forget_subject"`, `"purge"`, `"grant"`, `"revoke"`, `"approve"`, `"reject"`, `"apply"`, `"rollback"`, `"run_loop"` |
| `target` | string | Hash, subject identity, age window, or `principal/verbs/ns` for DCL |
| `reason` | string? | The `BECAUSE` text (absent only on bare `FORGET <hash>`) |
| `audit_grain` | string | Content address of the in-memory audit Observation (§8.14.4, OMS §8.12.1) |
| `tier` | integer | 2 or 3 |
| `duration_ms` | integer | Execution time |

Unlike Tier 0/1 audit entries -- host-side logs -- Tier 2/3 executions
additionally write their audit **into the memory itself** as an Observation
grain (§8.14.4, §8.16.1), so accountability replicates with the file.

**Streaming audit fields (additional):**

| Field | Type | Description |
|-------|------|-------------|
| `stream_enabled` | boolean | Whether streaming was requested |
| `stream_options` | array | Active stream options |
| `events_emitted` | integer | Total events sent |
| `cancelled` | boolean | Whether assembly was cancelled |
| `cancel_reason` | string? | Cancellation reason |
| `sources_failed` | integer | Number of failed sources |

---

## 22. Error Model

### 22.1 Error Format

Errors are stable across spec versions. Every error MUST include: code, message, and suggestion. Errors SHOULD include: position, expected alternatives, and example correction.

```json
{
  "error": {
    "code": "CAL-E003",
    "message": "Unknown grain type \"belief\".",
    "position": {"start": 7, "end": 13, "line": 1, "col": 8},
    "suggestion": "Did you mean \"fact\"? (OMS renamed Belief -> Fact in v1.4)",
    "example": "RECALL facts WHERE subject = \"alice\"",
    "valid_values": ["fact","event","state","workflow","tool","observation","goal","reasoning","consensus","consent","skill","recommendation"]
  }
}
```

### 22.2 Error Code Summary

See [Appendix C](#appendix-c-error-code-registry) for the complete registry. Error codes are organized by category:

| Range | Category | Count |
|-------|----------|-------|
| CAL-E001 -- CAL-E019 | Parse | 19 |
| CAL-E020 -- CAL-E022 | Type | 3 |
| CAL-E030 -- CAL-E031 | Execution | 2 |
| CAL-E040 -- CAL-E052 | Evolve | 10 |
| CAL-E060 -- CAL-E066 | Grain Type | 7 |
| CAL-E070 -- CAL-E071 | i18n | 2 |
| CAL-E075 -- CAL-E082 | Streaming | 8 |
| CAL-E085 -- CAL-E096 | Template | 12 |
| CAL-E100 | Version | 1 |
| CAL-E110, CAL-E113 | Multi-Format | 2 |
| CAL-E121 -- CAL-E127 | Authorization (Tier 2/3, new in 1.3) | 7 |
| CAL-E130 -- CAL-E137 | Saved query (new in 1.3) | 8 |

### 22.3 Warning Codes

| Code | Category | Description |
|------|----------|-------------|
| CAL-W001 | Warning | Unknown `mg:` relation (not in standard vocabulary) |
| CAL-W002 | Warning | Domain field used without matching `profile:` tag |
| CAL-W003 | Warning | Unknown domain prefix |
| CAL-W004 | Warning | Unknown extension option (ignored) |
| CAL-W005 | Warning | FORMAT auto-selected as `toon` due to budget pressure (>85% utilization estimate). Specify FORMAT explicitly to suppress. |

---

## 23. Compliance Checks

CAL introduces compliance verification checks that implementations MUST validate:

| Check | Regulation | Severity |
|-------|-----------|----------|
| `cal_grammar_safety` | All | Critical |
| `cal_default_minimization` | GDPR Art.25, HIPAA | Critical |
| `cal_audit_logging` | All | Critical |
| `cal_authz_enforcement` | HIPAA, SOX | Critical |
| `cal_no_policy_override` | All | Critical |
| `cal_injection_prevention` | All | Critical |
| `cal_tier1_audit` | All | Critical |
| `cal_tier1_policy_gate` | All | Critical |
| `cal_provenance_tracking` | EU AI Act | High |
| `cal_ai_marking` | EU AI Act | High |
| `cal_dsar_completeness` | GDPR Art.15 | High |
| `cal_hipaa_minimum_necessary` | HIPAA | High |
| `cal_phi_in_queries` | HIPAA | High |
| `cal_rate_limiting` | All | High |
| `cal_portability_format` | GDPR Art.20 | Medium |
| `cal_consent_on_read` | GDPR Art.6, LGPD | Medium |

---

## 24. Conformance Levels

### Level 1: Core (MUST implement)

- `RECALL` with `WHERE`, `IN`, `LIMIT`, `ABOUT`, `RECENT`
- `EXISTS`
- Parameter binding (`$param`)
- Hash literals (`sha256:...`)
- Error codes CAL-E001 through CAL-E031
- All safety invariants (section 2)
- Policy enforcement, audit integration
- Determinism guarantees (section 17.4)
- `text/cal` wire format

### Level 2: Extended (SHOULD implement)

Everything in Core, plus:
- Pipeline clauses (`SELECT`, `ORDER BY`, `LIMIT`, `COUNT`, `FIRST`, `GROUP BY`)
- Set operators (`UNION`, `INTERSECT`, `EXCEPT`)
- Subqueries (`WHERE field IN (subquery | EXTRACTOR)`)
- `EXPLAIN` mode
- `HISTORY` statement (including AS OF and DIFF)
- `DESCRIBE` statement (grain_types, fields, capabilities, server)
- `BATCH` statement
- `COALESCE` statement
- `LET` bindings
- All semantic shortcuts (SINCE, LIKE, MY, CONTRADICTIONS, BETWEEN)
- `ASSEMBLE` statement with BUDGET, PRIORITY, FORMAT
- Advanced `WITH` options (diversity, score_breakdown, explanation, provenance, progressive_disclosure, anonymize)
- `AS` per-query format control
- `application/json+cal` wire format (dual wire format)
- Error suggestion system ("did you mean?")
- Cross-lingual search
- Grain-type-specific fields (section 6)
- `mg:` relation category shortcuts (section 7)
- Domain profile querying (section 12)
- `THREAD` shorthand

### Level 3: Evolve (MAY implement)

Everything in Extended, plus:
- `ADD` with grain type, `SET` clauses, and `REASON`
- `SUPERSEDE` with `SET` clauses and `REASON`
- `REVERT` with `REASON`
- Tier 1 capability tokens with write quotas
- Error codes CAL-E040 through CAL-E052
- Two-phase prepare/execute with side-effect preview

### Level 4: Full (MAY implement)

Everything in Evolve, plus:
- Streaming ASSEMBLE (`STREAM` clause, SSE transport, cancellation)
- Custom FORMAT templates (`DEFINE TEMPLATE`, inline templates, named references)
- The `ELEMENT` shorthand (`DEFINE TEMPLATE x AS "..."`, `FORMAT TEMPLATE "..."`)
- Template inheritance from presets
- Template validation (error codes CAL-E085 through CAL-E096)
- Content Projection Model and `PROJECT` clause
- `DESCRIBE grammar` (returns EBNF)
- `DESCRIBE templates`
- WebSocket transport for streaming

### Tier modules (MAY implement -- new in 1.3)

Orthogonal to Levels 1--4, the Tier 2/3 statement families are optional
**modules**. Implementing one means implementing it whole:

- **Destroy (Tier 2):** `FORGET <hash>`, `FORGET SUBJECT`, `PURGE OLDER
  THAN`; an authorization model (host-defined suffices); the audit
  Observation (§8.14.4); error codes CAL-E121--E126. Tier 2 without Tier 3
  is legal.
- **Control (Tier 3):** `GRANT`/`REVOKE`/`SHOW GRANTS`/`DESCRIBE PRINCIPAL`
  over in-memory grant grains (OMS §12.6); principal-bound sessions
  (§18.4); the governance statements (§8.16) where the host has a
  recommendation engine. Implies the grant model Tier 2 consumes.
- **Definitions (Tier 3 to write, Tier 0 to read):** `DEFINE QUERY`/`DROP
  QUERY`/`RUN`/`DESCRIBE QUERIES` (§8.18) and `DEFINE TEMPLATE`/`DROP
  TEMPLATE`/`DESCRIBE TEMPLATES` (§10.6) over the memory's metadata table
  (OMS §28.4.1), including the replication rules of §8.18.1, the
  define-time body validation of §8.18.2, and error codes CAL-E130--E137.
  An implementation MAY support templates without saved queries; both are
  read surfaces at Tier 0 once defined.

A Tier-0/1 implementation is not diminished by declining both modules --
**tiers gate operations, not portability** (§2.2), and conformance claims
are per-tier.

Implementations MUST declare conformance, including their tiers:
```json
{"cal_conformance": "extended", "cal_version": "1.3", "cal_tiers": [0, 1]}
```

`DESCRIBE capabilities` MUST report the same tier list.

---

## 25. Versioning and Evolution

### 25.1 Semver for Specs

- **Major (CAL/1 -> CAL/2):** Breaking changes to grammar or semantics. Extremely rare.
- **Minor (e.g. 1.0 -> 1.1):** Additive only — new keywords, operators, or WITH options.
- **Patch (e.g. 1.0.0 -> 1.0.1):** Clarifications only. No grammar changes.

### 25.2 Extension Mechanism

Implementation-specific hints via `WITH x_prefix_name(...)`:

```sql
RECALL WHERE query = "..." WITH x_hnsw_ef(200)
```

Rules:
1. Extensions MUST use `x_` prefix
2. Extensions MUST NOT change core query semantics
3. Unknown extensions produce warning (not error)
4. Extensions MUST NOT enable destructive operations

---

## 26. Interface Integration

CAL is transport-agnostic. Implementations MAY expose CAL through any combination of interfaces:

| Interface | Endpoint Pattern | Input | Output |
|-----------|-----------------|-------|--------|
| REST | `POST /memories/{id}/cal` | `{"query": "...", "params": {...}}` | Response Envelope |
| REST | `POST /memories/{id}/cal/prepare` | `{"query": "...", "params": {...}}` | `{token, plan, tier}` |
| REST | `POST /memories/{id}/cal/execute` | `{"token": "..."}` | Response |
| REST (stream) | `POST /memories/{id}/cal` | `Accept: text/event-stream` | SSE stream |
| gRPC | `CalQuery(CalRequest)` | query string + params | `CalResponse` |
| MCP | Tool: `cal` | `{"query": "...", "params": {...}}` | JSON |
| A2A | Skill: `memory_cal` | CAL in task input | Task artifact |
| CLI | `<impl> cal "..."` | String or file | JSON or table |
| WebSocket | `/memories/{id}/cal/ws` | JSON frames | JSON frames |

All interfaces SHOULD support both Tier 0 and Tier 1 (when enabled). The two-phase prepare/execute flow is REQUIRED for Tier 1 and RECOMMENDED for Tier 0.

---

## 27. LLM System Prompt Template

Implementations SHOULD provide this reference to LLMs generating CAL queries (~1200 tokens):

```
## CAL Quick Reference

CAL is a non-destructive context assembly language for OMS memory databases.
It can read, assemble, and evolve memories, but never delete them.

### Read operations:
  RECALL [MY] [type] [IN "ns"] [ABOUT "entity"] [WHERE conditions] [WITH options]
    [clauses] [RECENT n] [SINCE "time"] [AS format]
  ASSEMBLE name FOR "intent" FROM label:(RECALL ...), ...
    BUDGET n tokens PRIORITY l1 > l2 FORMAT markdown [STREAM]
  EXISTS sha256:hash
  HISTORY WHERE subject = "s" AND relation = "r" [AS OF "date"]
  HISTORY sha256:hash [DIFF sha256:other]
  EXPLAIN <any statement>
  DESCRIBE grain_types | fields | capabilities | server
  BATCH { label: RECALL ..., label: RECALL ... }
  COALESCE(RECALL ..., RECALL ...)

### Evolve operations (when enabled):
  ADD <type> SET subject = "s" SET relation = "r" SET object = "o" [SET ...] REASON "why"
  SUPERSEDE sha256:hash SET field = value [SET ...] REASON "why"
  REVERT sha256:hash REASON "why"

### Types (queryable via RECALL -- all 12):
  facts, events, states, workflows, tools, observations, goals,
  reasonings, consensuses, consents, skills, recommendations

### Types creatable via ADD (subset -- others rejected with CAL-E051):
  facts, observations, goals, workflows, skills

### WHERE conditions (combine with AND):
  query = "search text"           -- semantic search
  subject = "entity"              -- triple subject
  relation = "predicate"          -- triple relation
  relation IS PREFERENCE          -- category shortcut
  object = "value"                -- triple object
  user_id = "uid"                 -- user filter
  hash = sha256:abcd...           -- exact lookup
  time = "last 7 days"            -- natural language time
  time BETWEEN epoch1 AND epoch2  -- epoch range
  confidence >= 0.8               -- min confidence
  tags INCLUDE ["tag1"]           -- required tags
  type = "fact"                 -- grain type

### Shortcuts: ABOUT, RECENT n, SINCE, LIKE, MY, CONTRADICTIONS, BETWEEN

### Clauses: SELECT f1,f2 ORDER BY field [ASC|DESC] LIMIT n COUNT
             FIRST SUBJECTS OBJECTS HASHES GROUP BY field
             PROJECT content(f1,f2), attr(f3,f4)

### Parameters: Use $name for dynamic values

### Streaming:
  ASSEMBLE ... STREAM                          -- stream all events
  ASSEMBLE ... STREAM { progress, chunks }     -- specific events

### Custom templates:
  FORMAT TEMPLATE name                         -- use named template (bare = name)
  FORMAT TEMPLATE { ELEMENT { <{{grain.type}}>{{grain.content}}</{{grain.type}}> } }
  FORMAT TEMPLATE "{{grain.subject}}: {{grain.content}}"   -- ELEMENT shorthand (quoted = body)
  DEFINE TEMPLATE oneliner AS "{{grain.subject}}: {{grain.content}}"

### Format aliases (in bracketed lists):
  FORMAT [json AS customers, markdown AS report]
  FORMAT [TEMPLATE "{{grain.subject}}" AS names, TEMPLATE "{{grain.object}}" AS values]

### Output formats:
  sml (default structured): flat tag-based — <fact subject="alice" confidence="0.92">prefers dark mode</fact>
  toon: CSV-tabular, ~40% fewer tokens — facts[3]{subject,content,confidence}:\nalice,prefers dark mode,0.95
  markdown: human-readable prose
  json: machine-readable structured data
  text: minimal plain text
  triples: subject-relation-object triples
  -- Use AS toon on large RECALL sets; FORMAT toon on budget-constrained ASSEMBLE

### Rules:
- LIMIT is always enforced (default 20, max 1000)
- CAL cannot rewrite history. Assume you cannot delete, erase, forget, or
  destroy data: destruction and GRANT/APPROVE-class statements execute only
  for sessions explicitly granted them -- check DESCRIBE capabilities before
  attempting any, and expect CAL-E121/E122 refusals otherwise.
- REASON is mandatory for all evolve operations; BECAUSE is mandatory for
  governance and bulk-erasure statements.
- Use HISTORY to check current version before SUPERSEDE.
- WITH anonymize("standard"|"strict") can only STRENGTHEN a file-declared
  pseudonymization policy, never weaken one; REHYDRATE (reverse lookup)
  is grant-gated and audited like destruction.
```

---

## Appendix A: Complete EBNF Grammar

The complete EBNF grammar is provided in section 4. This appendix restates it as a single unbroken production set for implementer convenience. The grammar in section 4 is the normative reference.

Implementations seeking the EBNF as machine-readable text SHOULD support `DESCRIBE grammar` which returns the productions in EBNF notation.

---

## Appendix B: JSON Schema References

CAL defines two JSON Schemas for the dual wire format:

| Schema | URI | Purpose |
|--------|-----|---------|
| Request | `https://cal-spec.org/schema/v1/cal-request.schema.json` | Validates `application/json+cal` requests |
| Response | `https://cal-spec.org/schema/v1/cal-response.schema.json` | Validates CAL responses |

The schemas are published alongside this specification in `schemas/v1/`. Implementations MUST validate incoming `application/json+cal` against the request schema. The schemas use JSON Schema draft 2020-12.

A collection of 50 example request/response pairs is provided in `schemas/v1/cal-examples.json` as a conformance test suite.

---

## Appendix C: Error Code Registry

All error codes use the `CAL-E` prefix.

### Parse Errors (CAL-E001 -- CAL-E019)

| Code | Description |
|------|-------------|
| CAL-E001 | Query exceeds maximum length (8192 bytes) |
| CAL-E002 | Unexpected token (includes expected list) |
| CAL-E003 | Unknown grain type (includes "did you mean?") |
| CAL-E004 | Unknown field name (includes suggestions) |
| CAL-E005 | Unterminated string literal |
| CAL-E006 | Invalid number |
| CAL-E007 | Subquery nesting exceeds depth 3 |
| CAL-E008 | Unbound parameter |
| CAL-E009 | Duplicate parameter binding |
| CAL-E010 | LIMIT exceeds maximum |
| CAL-E011 | IN set too large |
| CAL-E012 | Too many pipeline stages |
| CAL-E013 | Too many set operands |
| CAL-E014 | Empty query |
| CAL-E015 | Invalid hash literal (must be sha256: + hex) |
| CAL-E016 | REASON text exceeds maximum length |
| CAL-E017 | Unknown evolve field in SET clause |
| CAL-E018 | Missing REASON clause |
| CAL-E019 | Missing SET clause (SUPERSEDE requires at least one) |

### Type Errors (CAL-E020 -- CAL-E022)

| Code | Description |
|------|-------------|
| CAL-E020 | Incompatible types in comparison |
| CAL-E021 | Pipeline stage type mismatch |
| CAL-E022 | SUBJECTS/OBJECTS requires fact-type input |

### Execution Errors (CAL-E030 -- CAL-E031)

| Code | Description |
|------|-------------|
| CAL-E030 | Grain budget exceeded |
| CAL-E031 | Query timeout |

### Evolve Errors (CAL-E040 -- CAL-E052)

| Code | Description |
|------|-------------|
| CAL-E040 | SupersessionConflict -- target grain already superseded |
| CAL-E041 | NoPreviousVersion -- REVERT target is the original grain |
| CAL-E042 | GrainTypeNotEvolvable -- only Fact grains can be superseded |
| CAL-E043 | WriteQuotaExceeded -- too many evolve operations |
| CAL-E044 | Tier1NotEnabled -- requires Tier 1 capability |
| CAL-E045 | NamespaceMismatch -- target grain in different namespace |
| CAL-E046 | TargetNotFound -- target hash does not exist |
| CAL-E050 | MissingRequiredField -- ADD requires subject, relation, object |
| CAL-E051 | GrainTypeNotAddable -- only Fact, Observation, Goal, Workflow, Skill can be created |
| CAL-E052 | AddQuotaExceeded -- too many ADD operations |

### Multi-Format Errors (CAL-E110, CAL-E113)

| Code | Description |
|------|-------------|
| CAL-E110 | Too many formats in multi-format list (maximum 5) |
| CAL-E113 | Duplicate format key in multi-format list (alias or canonical name collision) |

### Authorization Errors (CAL-E121 -- CAL-E126, new in 1.3)

> The block begins at E121, not E120: the deployed dialect's registry had
> already assigned CAL-E120 (JSON+CAL parse failure) before this draft, and
> codes are append-only in every registry. E121 matches its shipped
> `NotAuthorized` exactly.

| Code | Description |
|------|-------------|
| CAL-E121 | NotAuthorized — the session principal lacks the statement's verb on the target namespace. The message SHOULD name the missing verb, the resource, and the `GRANT` statement that would confer it |
| CAL-E122 | TierNotSupported — the host does not implement this statement's tier (or has no authorization model for Tier 2/3) |
| CAL-E123 | UnknownPrincipal — the named or session principal does not resolve |
| CAL-E124 | SelfApproval — the reviewing principal created, or triggered the run that authored, this recommendation |
| CAL-E125 | ContextRefused — Tier 2/3 statement inside `BATCH`, a saved-query body, or `proposal_cal` where forbidden (§8.14.3) |
| CAL-E126 | ReasonRequired — a required `BECAUSE` reason is missing or empty at execution |
| CAL-E127 | MappingUnknown — `REHYDRATE` named a mapping id that resolves to neither a live session mapping nor a vault entry |

### Saved Query Errors (CAL-E130 -- CAL-E137, new in 1.3)

> **Why this block is not CAL-E053--E059.** Those codes sit in the free gap
> between the Evolve block (ends E052) and the grain-type block (starts E060),
> and are the natural home for a new family. They are not used here because the
> reference implementation had already assigned CAL-E040--E059 to its template
> and saved-query errors before this section existed, overlapping the Evolve
> codes this registry published in 1.5 -- only CAL-E044 (`Tier1NotEnabled`)
> agrees between the two. Codes are append-only in every registry, so neither
> side can renumber, and putting the saved-query family at E053--E059 would
> place it directly against the half of that implementation's block that is
> *already* colliding. A fresh block collides with nothing, and the divergence
> below is a defect to reconcile in the implementation rather than a
> disagreement to encode in the registry.

| Code | Description |
|------|-------------|
| CAL-E130 | QueryNotFound — `RUN` or `DROP QUERY` named a definition that does not exist |
| CAL-E131 | TooManyQueries — the namespace already holds the maximum number of saved queries (§8.18.2) |
| CAL-E132 | QueryBodyTooLarge — body exceeds the maximum body size (§8.18.2) |
| CAL-E133 | TooManyQueryParams — more declared parameters than the maximum (§8.18.2) |
| CAL-E134 | MissingQueryParam — a parameter with no default received no argument at the `RUN` call site |
| CAL-E135 | RecursiveQuery — `RUN` appears inside a `DEFINE QUERY` body (§8.18.2 rule 2) |
| CAL-E136 | QueryBodyNotReadOnly — a body contains a statement above Tier 0 (§8.18.2 rule 1) |
| CAL-E137 | InvalidQueryBody — the body failed to parse at definition time with its parameters standing in (§8.18.2 rule 3) |

### Shortcut and Grain Type Errors (CAL-E060 -- CAL-E066)

| Code | Description |
|------|-------------|
| CAL-E060 | AmbiguousShortcut / FieldNotOnGrainType |
| CAL-E061 | Grain-type-specific field used without declaring grain type |
| CAL-E062 | Invalid `tool_phase` value |
| CAL-E063 | Invalid `goal_state` value |
| CAL-E064 | Invalid `consent_action` value |
| CAL-E065 | Invalid `recall_priority` value |
| CAL-E066 | Invalid `epistemic_status` value |

### Internationalization Errors (CAL-E070 -- CAL-E071)

| Code | Description |
|------|-------------|
| CAL-E070 | InvalidUTF8 -- query contains invalid UTF-8 sequences |
| CAL-E071 | BidiOverrideRejected -- bidi override characters not allowed |

### ASSEMBLE Errors (CAL-E075 -- CAL-E076)

| Code | Description |
|------|-------------|
| CAL-E075 | ASSEMBLE timeout exceeded (10s) |
| CAL-E076 | All ASSEMBLE sources failed |

### Streaming Errors (CAL-E077 -- CAL-E082)

| Code | Description |
|------|-------------|
| CAL-E077 | InvalidStreamOption -- unknown option in STREAM clause |
| CAL-E078 | ChunkSizeOutOfRange -- chunk_size must be 20-1000 |
| CAL-E079 | StreamNotSupported -- server does not support streaming |
| CAL-E080 | GrainFormatError -- individual grain could not be formatted |
| CAL-E081 | StreamReconnectExpired -- assembly completed before reconnect |
| CAL-E082 | AssemblyCancelled -- assembly was cancelled |

### Template Errors (CAL-E085 -- CAL-E096)

| Code | Description |
|------|-------------|
| CAL-E085 | TemplateNotFound -- named template does not exist |
| CAL-E086 | CannotExtendData -- templates cannot extend 'data' preset |
| CAL-E087 | CannotExtendCustom -- templates can only extend presets |
| CAL-E088 | DuplicateTemplateName -- template already exists |
| CAL-E089 | TooManyTemplates -- namespace at 50-template limit |
| CAL-E090 | UnknownTemplateVariable -- variable not in known set |
| CAL-E091 | UnbalancedTemplateSection -- opening tag without closing |
| CAL-E092 | TemplateTooLarge -- exceeds 4096 bytes |
| CAL-E093 | TemplateNestingTooDeep -- conditional nesting exceeds 5 |
| CAL-E094 | InvalidTemplateSyntax -- unrecognized Mustache syntax |
| CAL-E095 | DuplicateSection -- same section defined twice |
| CAL-E096 | ConflictingElementSections -- ELEMENT vs ELEMENT_SUMMARY conflict |

### Version Errors (CAL-E100)

| Code | Description |
|------|-------------|
| CAL-E100 | Unsupported CAL version |

### Warning Codes (CAL-W001 -- CAL-W005)

| Code | Description |
|------|-------------|
| CAL-W001 | Unknown `mg:` relation (not in standard vocabulary) |
| CAL-W002 | Domain field used without matching profile tag |
| CAL-W003 | Unknown domain prefix |
| CAL-W004 | Unknown extension option (ignored) |
| CAL-W005 | FORMAT auto-selected as `toon` due to budget pressure (>85% utilization estimate). Specify FORMAT explicitly to suppress. |

---

## Appendix D: Reserved Words

The following words are reserved in CAL/1. They cannot be used as unquoted identifiers even if not yet functional. This list consolidates reserved words from all sources.

### Active Keywords

```
RECALL, ASSEMBLE, WHERE, AND, OR, NOT, IN, BETWEEN, LIMIT, OFFSET,
ORDER, BY, ASC, DESC, WITH, EXPLAIN, SCOPE,
UNION, INTERSECT, EXCEPT,
SELECT, COUNT, FIRST, GROUP, SUBJECTS, OBJECTS, HASHES, PROJECT,
INCLUDE, EXCLUDE, IS, NULL, TRUE, FALSE,
EXISTS, HISTORY, DESCRIBE, BATCH, COALESCE,
ABOUT, RECENT, SINCE, LIKE, MY, CONTRADICTIONS, AS,
FOR, FROM, BUDGET, PRIORITY, FORMAT,
LET, THREAD, DIFF,
ADD, SUPERSEDE, REVERT, SET, REASON, BECAUSE, REHYDRATE, REPORT,
FORGET, PURGE, SUBJECT, OLDER, THAN, TYPE,
GRANT, REVOKE, ON, TO, SHOW, GRANTS, PRINCIPAL,
APPROVE, REJECT, APPLY, ROLLBACK, RUN, LOOP, FULL,
STREAM, TEMPLATE, DEFINE, UNDEFINE, EXTENDS, DROP, QUERY, QUERIES, DESCRIPTION,
HEADER, ELEMENT, ELEMENT_SUMMARY, ELEMENT_OMIT, SOURCE_BREAK, FOOTER,
PREFERENCE, KNOWLEDGE, PERMISSION, INTERACTION, AGENCY, LIFECYCLE, OBSERVATION,
CAL, OF
```

### Future-Reserved Words

```
FIND, RELATE, TIMELINE, TRACE, GRAPH, ANNOTATE,
MATCHING, SIMILAR, NEAR, TAGGED, USER,
VIA, DEPTH, TOP, UNTIL, LAST, HAVING,
DIVERSITY, MMR, THRESHOLD, RERANK, PROVENANCE,
SUPERSEDED, EXPLANATION, SCORE_BREAKDOWN,
CONSISTENCY, EVENTUAL, BOUNDED, LINEARIZABLE,
CACHE, PIN, UNPIN, MERGE, LANG,
CHUNK, PAUSE, RESUME, CANCEL,
ROLE                                        -- DEFINE ROLE, deferred (§3.2)
```

---

## Appendix E: Queryable Fields Reference

### Common Fields (All Grain Types)

| Field | Type | Operators | Sortable | Projectable | Groupable |
|-------|------|-----------|----------|-------------|-----------|
| `query` | String | `=` | No | No | No |
| `subject` | String | `=`, `!=`, `IN` | Yes | Yes | Yes |
| `relation` | String | `=`, `!=`, `IN`, `IS` | Yes | Yes | Yes |
| `object` | String | `=`, `!=`, `IN` | Yes | Yes | Yes |
| `user_id` | String | `=`, `!=` | No | Yes | Yes |
| `namespace` | String | `=` | No | No | No |
| `confidence` | Number | `=`, `!=`, `>=`, `<=`, `>`, `<` | Yes | Yes | No |
| `importance` | Number | `=`, `!=`, `>=`, `<=`, `>`, `<` | Yes | Yes | No |
| `score` | Number | `>=`, `>` | Yes | Yes | No |
| `tags` | Array | `INCLUDE`, `EXCLUDE` | No | Yes | No |
| `type` | GrainType | `=` | No | Yes | Yes |
| `time` | Temporal | `=`, `BETWEEN` | Yes | Yes | No |
| `hash` | Hash | `=` | No | Yes | No |
| `contradicted` | Boolean | `=` | No | Yes | No |
| `verification_status` | String | `=` | Yes | Yes | Yes |
| `source_type` | String | `=` | Yes | Yes | Yes |
| `recall_priority` | String | `=` | No | No | No |
| `epistemic_status` | String | `=` | No | No | No |

### Grain-Type-Specific Fields

| Grain Type | Field | Type | Operators |
|-----------|-------|------|-----------|
| Event | `role` | String | `=`, `!=` |
| Event | `session_id` | String | `=` |
| Event | `parent_message_id` | String | `=` |
| Event | `model_id` | String | `=`, `!=` |
| Event | `content` | String | `=` |
| State | `context` | String | `=`, `!=` |
| State | `plan` | String | `=` |
| Workflow | `node` | String | `=` |
| Workflow | `node` | String | `=` |
| Workflow | `binding` | String | `=` |
| Tool | `tool_name` | String | `=`, `!=`, `IN` |
| Tool | `tool_phase` | String | `=` |
| Tool | `is_error` | Boolean | `=` |
| Tool | `tool_call_id` | String | `=` |
| Observation | `observer_id` | String | `=`, `!=` |
| Observation | `observer_type` | String | `=`, `!=` |
| Goal | `goal_state` | String | `=`, `!=` |
| Goal | `assigned_agent` | String | `=`, `!=` |
| Goal | `deadline` | Temporal | `=`, `BETWEEN` |
| Goal | `depends_on` | String | `=`, `IN` |
| Reasoning | `reasoning_type` | String | `=` |
| Reasoning | `premises` | String | `=` |
| Reasoning | `conclusion` | String | `=`, `!=` |
| Consensus | `threshold` | Number | `=`, `>=`, `<=`, `>`, `<` |
| Consensus | `agreement_count` | Number | `=`, `>=`, `<=`, `>`, `<` |
| Consensus | `participating_observers` | Array | `INCLUDE` |
| Consent | `consent_action` | String | `=` |
| Consent | `purpose` | String | `=`, `!=` |
| Consent | `grantor_did` | String | `=` |
| Consent | `grantee_did` | String | `=` |
| Consent | `scope` | String | `=` |
| Consent | `expires_at` | Temporal | `=`, `BETWEEN` |
| Skill | `name` | String | `=`, `!=`, `IN` |
| Skill | `version` | String | `=`, `IN` |
| Skill | `domain` | String | `=`, `IN` |
| Skill | `holder_did` | String | `=` |
| Skill | `proficiency` | Number | `=`, `>=`, `<=`, `>`, `<` |
| Skill | `transferable` | Boolean | `=` |
| Skill | `practice_count` | Number | `=`, `>=`, `<=`, `>`, `<` |
| Skill | `last_practiced_at` | Temporal | `=`, `BETWEEN` |
| Recommendation | `target_ref` | String | `=`, `!=`, `IN` |
| Recommendation | `analyzer` | String | `=`, `IN` |
| Recommendation | `severity` | String | `=`, `IN` |
| Recommendation | `dedup_key` | String | `=` |
| Recommendation | `rec_status` | String | `=`, `IN` |

> **Note:** Section 6.3 is the normative source for these field sets; this appendix is a consolidated index of it. When a grain type is added, both MUST be updated together.

### Domain-Prefixed Fields

| Domain | Fields |
|--------|--------|
| `hc:` | `patient_id`, `encounter_id`, `provider_id`, `condition_code`, `phi_category` |
| `legal:` | `case_id`, `jurisdiction`, `privilege_status`, `retention_category` |
| `fin:` | `account_id`, `transaction_id`, `risk_category`, `compliance_flag` |
| `rob:` | `device_id`, `coordinate_frame`, `safety_zone` |
| `sci:` | `experiment_id`, `dataset_id`, `methodology`, `reproducibility_status` |
| `con:` | `session_context`, `interaction_channel` |
| `int:` | `source_system`, `correlation_id`, `sync_status` |

---

## Appendix F: Version History

| Version | Date | Change |
|---------|------|--------|
| 1.3-draft | 2026-08-10 | **The pillar changes, on purpose and in the open.** 1.0--1.2 promised "non-destructive by grammar; no unsafe mode." The promise was true and had a hidden cost: real deployments still needed erasure -- GDPR, retention -- so destruction happened anyway, outside the language, ungoverned by any spec. Exiling destruction never prevented it; it only prevented *specifying* it. 1.3 brings it inside, where grammar bounds its shape (whole grains by hash, identity, or age -- never a predicate, never a key, no history rewrite, no `UNFORGET`), grants gate it per principal, and the audit trail sees it (a mandatory in-memory audit Observation per Tier-2 execution). Adds the four-tier model (Core/Evolve/Destroy/Control; tiers gate operations, not portability -- §2.2), Tier 2 statements `FORGET <hash>`/`FORGET SUBJECT`/`PURGE OLDER THAN` (§8.14), Tier 3 DCL `GRANT`/`REVOKE`/`SHOW GRANTS`/`DESCRIBE PRINCIPAL` over in-memory grant grains (§8.15, OMS §12.6), Tier 3 governance statements `APPROVE`/`REJECT`/`APPLY`/`ROLLBACK`/`RUN LOOP` + `DESCRIBE loop`-family (§8.16), principal-bound sessions and the credential/policy split (§18.4), authorization error codes CAL-E121--E126, and the `cal_tiers` conformance declaration. `DEFINE ROLE` reserved, deferred. The anonymization batch (2026-08-16) adds `WITH anonymize("<level>")` as a strengthen-only recall/per-source option (§8.1.2, §8.2.1), the Tier-3 `REHYDRATE` statement with its stated classification obligation (§8.17, CAL-E127), pairing OMS §10.5's pseudonymized-egress model. The trigger batch (2026-08-19) adds `RECALL triggers` and removes the `ON "..."` workflow clause with OMS's `Workflow.trigger`. The saved-query batch (2026-08-19) adds §8.18 saved queries -- `DEFINE QUERY`/`RUN`/`DROP QUERY`/`DESCRIBE QUERIES`, Tier-0-only bodies validated at definition time, definitions as host metadata (OMS §28.4.1) rather than grains, error codes CAL-E130--E137 -- and admits `DROP` to the grammar with a closed two-target set that reaches no grain (§2.4). Every 1.0--1.2 document remains valid Tier-0/1 CAL; a session without grants is exactly the language those releases promised -- the floor, not the ceiling. |
| 1.2 | 2026-08-03 | As part of the OMS v1.5 release, adds the **Recommendation** grain type (`0x0C`) to the closed query set: `RECALL recommendations` with a type-specific field set (`target_ref`, `analyzer`, `severity`, `dedup_key`, `rec_status`), `<recommendation>` content projection, TOON columns, `recommendation_field` grammar production, and the type in `grain_type_plural`/`grain_type_singular`, `DESCRIBE`, and the JSON `valid_values` enum. Recommendation is **query-only** — it is engine-emitted and lifecycle-gated, so it is deliberately absent from the CAL-addable set (`ADD`/`SUPERSEDE SET` never create or transition a recommendation). Also adds the **`ELEMENT` shorthand** for custom templates (Section 10.6.1): `DEFINE TEMPLATE <name> AS "<text>"` and `FORMAT TEMPLATE "<text>"`, both defined by equivalence to an `ELEMENT` section, plus the `template_shorthand` production and `"TEMPLATE" , string_literal` in `format_spec`. This also corrects two 1.1 defects in the same examples (Sections 10.1.1, 14.2.1, 27): the `TEMPLATE "..."` form they use was not admitted by the Section 4 grammar, and they interpolated bare `{{subject}}`/`{{object}}` rather than the `grain.` namespace that Section 10.5 defines and Section 10.8 closes. Backward-compatible. |
| 1.1 | 2026-03-05 | Multi-format output: FORMAT and AS clauses accept bracketed format lists (`FORMAT [markdown, json]`). Single query execution produces multiple renderings. New Section 10.1.1 (syntax and rules), Section 14.2.1 (multi-format response shape), error code CAL-E110. Backward-compatible — single-format syntax unchanged. As part of the OMS v1.4 release, also adds the Skill grain type (`0x0B`) to the closed query set (`RECALL skills` / `ADD skill`, type-specific field set, `<skill>` projection, TOON columns, `mg:capable_of` in the `AGENCY` shortcut) and corrects `belief`/`action` literals the rename left behind (`grain_type_plural`, `RECALL MY`, TOON tables, `DESCRIBE` listing, `CAL-E051`). |
| 1.0 | 2026-03-03 | Initial CAL specification. 12-variant statement model. Tier 0 (RECALL, ASSEMBLE, SetOp, EXISTS, HISTORY, EXPLAIN, DESCRIBE, BATCH, COALESCE) + Tier 1 (ADD, SUPERSEDE, REVERT). ASSEMBLE with budget, priority, format, streaming. Semantic shortcuts (ABOUT, RECENT, SINCE, LIKE, MY, CONTRADICTIONS, BETWEEN). LET bindings. Custom FORMAT templates (Mustache-subset). Grain-type-specific queryable fields for all 10 OMS types. mg: relation vocabulary with category shortcuts. Domain profile querying. Dual wire format (text/cal + application/json+cal). Internationalization (Unicode NFC, cross-lingual search, bidi safety). Streaming protocol (SSE, NDJSON, WebSocket). THREAD shorthand. HISTORY AS OF and DIFF. Non-destructive safety model. Content Projection Model with flat semantic output (Section 10.3-10.4). PROJECT clause for custom field surfacing. Per-grain-type content projection rules with humanize() and time humanization. ELEMENT/ELEMENT_SUMMARY/SOURCE_BREAK template sections for flat semantic rendering. TOON (Token-Oriented Object Notation) format support — `toon` as a first-class FORMAT/AS preset (Section 10.9): tabular CSV rendering for uniform RECALL results, grouped-section rendering for ASSEMBLE results, per-grain-type column sets at each disclosure level, PROJECT integration, STREAM compatibility, auto-TOON budget-pressure hint (CAL-W005). |

---

**Document Status:** This is the CAL (Context Assembly Language) Specification v1.3, **draft for comment**. It defines a deterministic, LLM-native context assembly, evolution, destruction, and control language for OMS-compliant memory databases — append-only by construction for evolution, authorization-gated for destruction and control. CAL is part of the Open Memory Specification (OMS) v1.6 draft — see [SPECIFICATION.md](./SPECIFICATION.md).

**Last Updated:** 2026-08-10
**License:** This specification is offered under the Open Web Foundation Final Specification Agreement (OWFa 1.0)
**Copyright:** Public Domain (CC0 1.0 Universal)
