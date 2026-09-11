# Standard structure & aggregation (Haro Docs reference)
> **STOP — READ THIS FILE FULLY BEFORE ACTING.** This file is the normative reference for structure and aggregation (read when init/generate need it). Do not act, answer, edit, or call tools from memory: read every step below first. If in doubt at any point, re-read. The reference always wins over memory.
> **Ground rules (apply to every action in this file):** read `docroot` from `.haro-docs/project-profile.yaml` before operating — never guess it. Respect `language.response` (conversation) and `language.documentation` (doc content); if either is missing, ask the user first. Knowledge in `.haro-docs/knowledge/` is ground truth over scanned defaults (see `references/knowledge.md`).

## 7. Aggregation Matrix

| Document | Assembled from |
|----------|----------------------------|
| **BRD** | 01-overview + 02-business |
| **PRD** | 01-overview + 03-features + 02-business + 07-quality |
| **SAD** | 04-architecture + 05-security + 06-implementation + 07-quality + 08-operations |
| **FSD** | 03-features + 04-architecture |
| **SRD** | 05-security + 03-features |
| **Proposal** | 01-overview + 02-business |
| **Test Plan** | 07-quality + 03-features + 05-security |
| **Runbook** | 08-operations + 06-implementation + 04-architecture |
| **User/Admin Guide** | 09-guides + 03-features + 01-overview |

## 8. Standard structure (single)

Single structure for all projects:

``
docs/                        # or .haro-docs/docs/ depending on user-chosen doc-root
├── 00-common/               # 01-conventions.md, 02-references.md, 03-abbreviations.md, 04-glossary.md, 05-traceability.md (01 manual, 02-05 auto)
├── 01-overview/             # 01-problem-statement.md, 02-vision.md, 03-goals.md, 04-scope.md, 05-stakeholders.md, 06-constraints.md, 07-roadmap.md (Version|Goal|Target|Status)
├── 02-business/             # 01-value-proposition.md, 02-market-analysis.md, 03-business-model.md, 04-pricing.md, 05-sla.md, 06-risk-legal.md, use-cases/, change-requests/CR-*.md
├── 03-features/             # 01-feature-catalog.md, 04-dependencies.md, features/<feature>/ 01-overview.md, 02-user-stories.md, 03-acceptance-criteria.md
├── 04-architecture/         # 01-system-overview.md, 02-components.md, 03-data-flow.md, 04-api-spec.md, 05-data-model.md, 06-tech-stack.md, 07-decisions-adr.md
├── 05-security/             # 01-threat-model.md, 02-defense.md, 03-compliance.md, 04-privacy.md, 05-authz.md
├── 06-implementation/       # 01-deployment.md, 02-configuration.md, 03-integration.md, 04-migration.md, 05-development-guide.md
├── 07-quality/              # 01-test-strategy.md, 02-test-plan.md, 03-test-cases.md, 04-uat.md, 05-quality-metrics.md
├── 08-operations/           # 01-runbook.md, 02-incident-response.md, 03-dr-bcp.md, 04-monitoring.md, 05-support.md
├── 09-guides/               # 01-user-guide.md, 02-admin-guide.md, 03-training.md, 04-faq.md, 05-onboarding.md
├── 10-deliverables/         # 01-BRD.md, 02-PRD.md, 03-SAD.md, 04-FSD.md, 05-SRD.md, 06-Proposal.md, 07-TestPlan.md, 08-Runbook.md, 09-UserAdminGuide.md (placeholders with distinct headings + Ref links, not identical)
└── 99-assets/               # Images, diagrams, templates
``

`init` creates all folders/files above; each `10-deliverables/*.md` is a placeholder with its own headings plus `Ref: ../01-overview/...` links — not identical templates. In `00-common`, `01-conventions.md` is decided during `init` (§5.2 step 3b, 2 layers: fixed Part A + project-specific Part B, MD-only single source) while `02-references, 03-abbreviations, 04-glossary, 05-traceability` are living references auto-populated by `generate` from placeholders marked `auto-populated by generate — do not edit manually`; `05-traceability.md` format is a table `deliverable | source blocks | block status | knowledge refs`. `01-overview` is the starting point for writing (not `00-common`).

### Distinguishing `03-features` from `02-business/use-cases/`

These two folders cause the most confusion — distinguish by audience and detail level:

| | `03-features` | `02-business/use-cases/` |
|---|---------------|-------------------------------|
| Content | User stories + acceptance criteria + technical references | End-to-end interaction flows: main path, all alternative/exception paths |
| Audience | Developers, testers | Customers, BAs, test case writers |
| Language | Technical (schema/endpoint references allowed) | Pure business language, no technical detail |
| SSOT rule | Stories cross-link to use cases for context | Use cases contain NO technical detail |

One use case is typically decomposed into multiple user stories; each story links back to its source use case instead of retelling the flow.

Folders not yet needed may stay with their `README.md` and a short `> Out of scope for this project` note.

