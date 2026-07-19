# Meridian Financial Group

**Non-Human Identity Governance Architecture**

A case study in bringing 4 million ungoverned machine identities under continuous, evidence-producing control before a regulatory examination forces the question.

---

## The Problem

Meridian Financial Group, a $340 billion multinational financial institution, has more non-human identities than it can account for. Service accounts, API keys, trading bots, and AI agents outnumber its 28,000 employees by orders of magnitude. No formal ownership model exists. Credentials do not expire. When an examiner asks what has access to Meridian's systems and why, no one can answer completely.

This case study documents the architecture that closes that gap.

## Documents

| # | Document | Description |
|---|----------|--------------|
| 01 | [README](01-readme.md) | Case study overview |
| 02 | [Business Problem](02-business-problem.md) | Why 4 million identities went ungoverned |
| 03 | [Requirements](03-requirements.md) | Functional and non-functional requirements |
| 04 | [Constraints and Risks](04-constraints-and-risks.md) | What limits the design, and what could go wrong |
| 05 | [As-Is Architecture](05-as-is-architecture.md) | Current state: fragmented, ad hoc, unowned |
| 06 | [To-Be Architecture](06-to-be-architecture.md) | Target state: discovery, lifecycle automation, deterministic policy enforcement |
| 07 | [Business Case](07-business-case.md) | The financial and regulatory case for funding this now |
| 08 | [Governance](08-governance.md) | Ownership, review cadence, exception handling, metrics |
| 09 | [Roadmap](09-roadmap.md) | Three-phase, eighteen-month delivery plan |
| 10 | [Executive Summary](10-executive-summary.md) | The full case in one page |
| — | [Architecture Decision Records](adr/) | Three ADRs documenting the key design decisions |

## Architecture at a Glance

The design rests on three decisions, each documented as its own ADR:

- **Continuous automated discovery** over periodic manual review, since a population that grows by thousands per sprint cannot be governed by a quarterly sample
- **A deterministic policy engine** over LLM-only compliance assessment, since an examiner's question needs an answer that traces to a fired rule, not a probabilistic judgment
- **Automated credential lifecycle and expiration** over static, indefinite credentials, since a standing credential is a standing target regardless of intent

## About

**AI & Identity Security Architect @ Go Cloud Architects**

Contact: curtis@igasecurityconsulting.com
