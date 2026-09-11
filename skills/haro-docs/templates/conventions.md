# Conventions

> Decided during `/haro-docs init` (§5.2 step 3b). Part A is fixed by the skill —
> do not edit. Part B is project-specific — edit via `/haro-docs config conventions`.
> This file is the single source of truth for conventions (MD-only, no YAML mirror).
> Status: RELEASED from creation.

## A. Fixed by skill (do not edit)

- **File naming:** kebab-case with numeric prefix indicating reading order (e.g. `01-problem-statement.md`).
- **Single Source of Truth:** each piece of content is written once in one file only; aggregated documents under `10-deliverables/` only assemble content, never duplicate it.
- **Status lifecycle:** every doc file carries `status:` in frontmatter (`DRAFT | UPDATING | RELEASED`) with a mirror in `.haro-docs/status/`. `RELEASED` files are referenceable and are never edited without an explicit update confirmation.
- **Language:** conversation replies use `language.response`, doc content uses `language.documentation` from `.haro-docs/project-profile.yaml` (`en` | `vi` | `vi-en`).
- **Images/diagrams:** stored in `99-assets/`, referenced via relative paths; no inline base64.
- **Auto files:** `02-references.md`, `03-abbreviations.md`, `04-glossary.md`, `05-traceability.md` are auto-populated by `generate` — do not edit manually.

## B. Project-specific (decided at init, editable via `/haro-docs config conventions`)

| # | Item | Value |
|---|------|-------|
| 1 | Diagram tool | {{diagram_tool}} <!-- mermaid \| plantuml \| drawio + image fallback --> |
| 2 | API spec format | {{api_format}} <!-- openapi-yaml \| md-table \| both --> |
| 3 | Tone & depth | {{tone}} <!-- high-level \| balanced \| deep-dive, synced with audience.technical_depth --> |
| 4 | RELEASED approver | {{approver}} <!-- single name/role, or per-domain approvers --> |
| 5 | Locked terms | {{terms}} <!-- product/brand terms with fixed spelling, or (none) --> |
| 6 | Priority deliverables | {{deliverables}} <!-- e.g. BRD, SAD — determines which conventions get detailed first, or (all) --> |
