# SML (Semantic Markup Language) Specification v1.0

**Status:** Standards Track | **Date:** 2026-03-03 | **Version:** 1.0 | **Classification:** Experimental
**Part of:** [Open Memory Specification (OMS) v1.3](./SPECIFICATION.md)

---

## Table of Contents

- [Abstract](#abstract)
1. [What is SML?](#1-what-is-sml)
2. [Structural Rules](#2-structural-rules)
3. [Comprehensive Example — All 10 Grain Types](#3-comprehensive-example--all-10-grain-types)
4. [Progressive Disclosure](#4-progressive-disclosure)
- [Appendix A: Relationship to CAL](#appendix-a-relationship-to-cal)

---

## Abstract

**SML (Semantic Markup Language)** is a flat, tag-based markup format optimized for LLM context consumption. It is the default output format for CAL `ASSEMBLE` statements and can be consumed directly by any LLM without an XML processor.

SML uses tag syntax as a **semantic signalling device** — the tag name tells the LLM what kind of information follows (using OMS grain types as tag names), the attributes carry lightweight decision metadata, and the element text is natural language content.

SML is part of the [Open Memory Specification (OMS) v1.3](./SPECIFICATION.md) family. The `FORMAT` system that produces SML output is defined in the [Context Assembly Language (CAL) specification](./CONTEXT-ASSEMBLY-LANGUAGE-CAL-SPECIFICATION.md).

---

## 1. What is SML?

**SML (Semantic Markup Language)** is a flat, tag-based markup format defined by CAL specifically for LLM context consumption. It is not XML. SML uses the tag syntax as a **semantic signalling device** — the tag name tells the LLM what kind of information follows; the attributes carry lightweight decision metadata; the element text is natural language content. SML does not require, and is not validated by, an XML processor.

| Property | SML | Standard XML |
|----------|-----|--------------|
| Tag names | Grain type identifiers (`belief`, `goal`, `event`, …) | Arbitrary element names |
| Text content | Natural language prose, not decomposed triples | Structured data |
| Attributes | Human-readable metadata hints (`confidence`, `state`, `time`) | Machine-parseable key-value pairs |
| Nesting | **Flat only** — no nested elements (one `<context>` envelope only) | Arbitrary depth |
| Document declaration | None | `<?xml …?>`, DTD, prologue |
| Parser requirement | None — consumed directly by an LLM | Conforming XML processor |
| Schema validation | Not applicable | DTD / XSD validation |
| Escape sequences | Not required | `&amp;`, `&lt;`, etc. required |

SML is the default format for the `structured` preset (alias `sml`) in CAL's [FORMAT system](./CONTEXT-ASSEMBLY-LANGUAGE-CAL-SPECIFICATION.md#10-format-system).

---

## 2. Structural Rules

The formatted output uses a **flat, semantic structure**:

1. **Tag names ARE the grain type.** No generic `<grain>` wrapper — use `<belief>`, `<goal>`, `<event>`, etc.
2. **No group wrappers.** No `<source>`, `<beliefs>`, or other container elements. Whitespace separates grain groups.
3. **Storage internals never appear.** No hashes, counts, namespaces, or OMS-internal fields.
4. **Text content reads as natural language.** The element text is the projected content, not decomposed triples.
5. **One envelope element.** The `<context>` wrapper carries only the `intent` attribute. It is the sole container.

---

## 3. Comprehensive Example — All 10 Grain Types

The following example shows a single assembled context covering every grain type. The scenario: an agent helping alice prepare her Q1 engineering review.

```sml
<context intent="helping alice prepare her Q1 engineering review">

  <belief subject="alice" confidence="0.95">prefers dark mode in all tools</belief>
  <belief subject="alice" confidence="0.88">requires keyboard shortcuts for productivity</belief>
  <belief subject="alice" confidence="0.82">works best in deep-focus blocks of 90 minutes</belief>

  <goal subject="alice" state="active" deadline="2026-03-15">complete Q1 engineering review presentation</goal>
  <goal subject="alice" state="active">reduce P0 incident rate by 20% in Q2</goal>

  <event role="user" time="10m ago">Can you help me pull together the Q1 metrics?</event>
  <event role="assistant" time="10m ago">Sure — retrieving deployment counts, incident data, and velocity now.</event>
  <event role="user" time="8m ago">Focus on the reliability numbers first.</event>

  <action tool="query_metrics" phase="completed">retrieved 47 deployments and 3 P0 incidents for Q1 2026</action>
  <action tool="search_docs" phase="completed">found Q1 review template in confluence/engineering/reviews</action>

  <observation observer="system">alice opened incident-dashboard at 09:14 UTC</observation>
  <observation observer="system" source="calendar">Q1 review presentation scheduled for 2026-03-15 14:00 UTC</observation>

  <reasoning type="deductive">alice is prioritising reliability given 3 P0 incidents; lead with incident reduction narrative</reasoning>
  <reasoning type="abductive">low velocity in week 8 likely caused by the infra migration; flag as contextual outlier</reasoning>

  <state context="q1_review_prep">outlining slides: 1. headline metrics  2. incident retrospective  3. velocity trend  4. Q2 goals</state>

  <workflow trigger="review_prep_requested">1. retrieve Q1 metrics  2. identify narrative arc  3. draft slide outline  4. populate data  5. send for review by 2026-03-14</workflow>

  <consensus threshold="3" count="4">Q1 deployment frequency improved 18% over Q4 2025</consensus>

  <consent action="granted" grantor="alice" grantee="agent">access engineering metrics dashboards for review preparation</consent>

</context>
```

Each element maps directly to one grain type. The LLM reads the tag to understand the epistemic status of the content — a `<belief>` carries a known confidence, a `<reasoning>` signals inference, a `<consent>` signals an explicit permission grant — without needing to parse attribute schemas or understand OMS internals.

---

## 4. Progressive Disclosure

Progressive disclosure controls **metadata density on a flat structure**, not nesting depth. The element shape stays the same across all levels — only the number of attributes changes.

| Level | Example |
|-------|---------|
| `summary` | `<belief subject="alice">prefers dark mode</belief>` |
| `standard` | `<belief subject="alice" confidence="0.92">prefers dark mode</belief>` |
| `full` | `<belief subject="alice" confidence="0.92" source="explicit" observed="2d ago">prefers dark mode</belief>` |

---

## Appendix A: Relationship to CAL

SML is produced by the CAL `ASSEMBLE` statement's `FORMAT` system. The CAL specification defines:

- The `structured` / `sml` preset that selects SML output
- The Content Projection Model (CAL §10.3) that maps grain fields to SML elements
- Template variables and custom templates for SML output
- Progressive disclosure levels (`summary`, `standard`, `full`)

See the [CAL specification](./CONTEXT-ASSEMBLY-LANGUAGE-CAL-SPECIFICATION.md) for the complete FORMAT system, template engine, and ASSEMBLE statement semantics.
