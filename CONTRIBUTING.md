# Contributing to the Open Memory Specification

Thank you for your interest in contributing to OMS. This document explains how to participate in the development of the specification.

---

## Table of Contents

- [Ways to Contribute](#ways-to-contribute)
- [Before You Start](#before-you-start)
- [Proposing Changes](#proposing-changes)
- [Writing Style](#writing-style)
- [Editorial Standards](#editorial-standards)
- [Normative Language](#normative-language)
- [Review Process](#review-process)
- [Security Issues](#security-issues)
- [Code of Conduct](#code-of-conduct)

---

## Ways to Contribute

- **Bug reports** — Errors, ambiguities, or contradictions in the specification
- **Clarifications** — Improving precision of existing normative language
- **New features** — Proposing additions to the format or semantics
- **Test vectors** — Adding or correcting test vectors
- **Implementations** — Reference implementations (listed separately once contributed)
- **Translations** — Translations of the specification into other languages

---

## Before You Start

1. Search [existing issues](../../issues) to avoid duplicates.
2. For significant changes, open an issue first to discuss the approach before writing text.
3. Read the full specification to understand the existing design and conventions.

---

## Proposing Changes

### For small fixes (typos, clarifications)

Open a pull request directly with:
- A clear title describing the change
- A short explanation of why the change improves the spec

### For significant changes (new fields, new types, semantic changes)

1. Open an issue with the label `proposal` describing:
   - The problem being solved
   - The proposed solution
   - Alternatives considered
   - Impact on existing implementations
2. Wait for discussion and rough consensus before drafting spec text.
3. Open a PR referencing the issue once consensus is reached.

### Pull Request Requirements

- [ ] Changes are in `oms-specification.md`
- [ ] New normative requirements use correct RFC 2119 language (MUST/SHOULD/MAY)
- [ ] Test vectors are updated if the binary format changes
- [ ] Conformance level impact is noted if applicable
- [ ] The change does not break backwards compatibility without a version bump discussion

---

## Writing Style

- Use clear, precise technical English
- Prefer short sentences over long ones
- Define terms in the Terminology section before using them
- Use tables for structured data (field definitions, type enumerations)
- Use code blocks for byte layouts, hex values, and examples

---

## Editorial Standards

- Section numbering must remain stable across minor revisions
- New sections go at the end unless they fit naturally within an existing section
- Deprecated features are marked `[DEPRECATED]`, never silently removed
- Every normative change to the binary format requires a new test vector

---

## Normative Language

Per [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174), use uppercase keywords precisely:

| Keyword | Use when... |
|---------|-------------|
| **MUST** / **SHALL** | Required for conformance |
| **MUST NOT** / **SHALL NOT** | Prohibited |
| **SHOULD** / **RECOMMENDED** | Best practice, deviations allowed with good reason |
| **SHOULD NOT** / **NOT RECOMMENDED** | Not advised, but not prohibited |
| **MAY** / **OPTIONAL** | Permitted but not required |

Do not use these words in lowercase for normative intent.

---

## Review Process

1. A maintainer will review your PR within 14 days.
2. Feedback will be given as review comments.
3. Significant changes require a 7-day comment period after the last revision before merging.
4. A PR needs at least one maintainer approval to merge.
5. Purely editorial fixes (grammar, formatting) may be merged without a waiting period.

---

## Security Issues

Do not open public issues for security vulnerabilities in the specification's cryptographic design or privacy mechanisms.

Report security concerns privately by opening a [GitHub Security Advisory](../../security/advisories/new) or emailing the maintainers directly (see the spec's Security Considerations section for contact guidance).

---

## Code of Conduct

All contributors are expected to follow the [Code of Conduct](./CODE_OF_CONDUCT.md). Be respectful and constructive.
