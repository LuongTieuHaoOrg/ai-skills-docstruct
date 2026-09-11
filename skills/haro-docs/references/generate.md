# Generate (Haro Docs reference)
> **STOP — READ THIS FILE FULLY BEFORE ACTING.** This file is the normative workflow for `/haro-docs generate`. Do not act, answer, edit, or call tools from memory: read every step below first. If in doubt at any point, re-read. The reference always wins over memory.
> **Ground rules (apply to every action in this file):** read `docroot` from `.haro-docs/project-profile.yaml` before operating — never guess it. Respect `language.response` (conversation) and `language.documentation` (doc content); if either is missing, ask the user first. Knowledge in `.haro-docs/knowledge/` is ground truth over scanned defaults (see `references/knowledge.md`).
> Content writing in this file must follow `references/authoring.md`.

## 6. Command `/haro-docs generate` / `generate <file>` — Build Docs in Order (Role-Adaptive)

Build documentation flexibly based on current state + your role. Two modes:

- `/haro-docs generate` — propose 2–3 next files to build, based on reality, not rigid order. Skips all `00-common` files — `01-conventions.md` is decided during `init` (§5.2 step 3b) and `02-references, 03-abbreviations, 04-glossary, 05-traceability` are auto-populated; start writing at `01-overview`.
- `/haro-docs generate <file>` — focus on a specific file (e.g. `02-business/01-value-prop.md` or `10-deliverables/01-BRD.md`) to create or adjust it. If the file is `RELEASED`, warn and ask to update.

### Ordered index

Canonical order is `00-common → 01-overview → 02-business → 03-features → 04-architecture → 05-security → 06-implementation → 07-quality → 08-operations → 09-guides → 10-deliverables → 99-assets` as defined in `schema.yaml` and references/structure.md (§8). Within each folder, files are ordered by numeric prefix. This order is the **reference**, not a rigid gate: use it to understand what is prerequisite for what, but do not block flexibly. If all files are `RELEASED`, reply `All done — every file is RELEASED.`.

### Elicitation storage

Interim Q&A is stored in `.haro-docs/elicitation/<sanitized-path>.md` (e.g. `02-business-01-value-prop.md`) to avoid context overload. It is read on demand during `generate` and not loaded by default. After the target file is completed, ask the user whether to keep or delete the elicitation file.

### Status

Each doc file carries `status:` in frontmatter and a mirror in `.haro-docs/status/<path>`:

- `DRAFT` — brand-new file, first version.
- `UPDATING` — has reference content and is being edited.
- `RELEASED` — completed, may be referenced by later files, not edited. If `generate <file>` targets a `RELEASED` file, warn `File is RELEASED` and ask `Update? (yes/no)`. On `yes`, set status to `UPDATING` before editing.

No backward compatibility: every creation or adjustment is treated as the **first version** per references/authoring.md (§9).

### Workflow

1. **Selective RAG** — read `knowledge/_index.md` (if exists), pick rows whose domain/tags match the target file (`business` for BRD/Proposal, `technical`+`architecture` for SAD/FSD, `team` for people), load only those knowledge files.
2. **Ask who you are & propose next file based on reality:**
   - Ask first: `Who are you in this project? What is your role? (customer / sales / BA / dev / PM) — what do you know best about?` Save answer to `knowledge/team-role.md` if not yet stored (or confirm `Still <role>?`), and also to `elicitation/<target>.md`.
    - Scan the current docs reality: which files are missing/empty (`DRAFT`), which are `UPDATING`, which are `RELEASED`, which `change-requests/CR-*.md` are new. Use the canonical order as reference to understand prerequisites (e.g. `01-overview` and `02-business` are prerequisites for `04-architecture`), but **do not enforce rigidly** — understand flexibly what is actually needed. Skip the whole `00-common` folder (`01-conventions.md` is decided at init, `02-05` are auto); start writing at `01-overview`, not `00-common`.
   - If `<file>` was given: treat it as the user's preference, but if it depends on an unfinished prerequisite (e.g. targeting `04-architecture` while `01-overview/03-goals.md` is still empty), explain why the prerequisite matters and ask `Do you want to continue with this file or switch to the prerequisite?`.
   - If no `<file>`: propose **2–3 candidates** for the next file, each with a one-line reason (e.g. `1. 01-overview/01-problem-statement.md — foundational purpose is still empty; 2. 02-business/01-value-prop.md — business value needed before architecture`). Let the user pick. If the user picks none, they can specify another file.
   - Once a target is chosen/confirmed, read its existing content if any (`UPDATING` case) and nearby files in the same folder + glossary for context. Note current status (`DRAFT` if new, `UPDATING` if exists).
3. **Role-adaptive elicitation (flexible, not rigid):**
   - Generate **~5 tailored questions** based on `role + file domain + current reality` (e.g. for `02-business/*` with customer → market/pain/SLA/metrics; for `04-architecture/*` with customer → flows/business rules only, with dev → endpoints/schemas/NFR/diagrams). **Never ask a customer about code**, but otherwise adapt to what is actually missing — do not apply a hard rule like "purpose before technical" to every case.
   - Append Q&A to the elicitation file.
   - Analyze: check which required sections of the target file (per schema) are still missing. If incomplete, ask another 3–5 follow-up questions, append to elicitation file, repeat until sufficient.
4. **Propose outline** — list sections for the target file, annotating source (which block/folder and which knowledge files). Wait for outline approval.
5. **Write** — create/update the target file in its canonical folder (kebab-case + numeric prefix, references/authoring.md (§9)) using current-state style (references/authoring.md (§9), no backward). Follow `00-common/01-conventions.md` Part B (diagram tool, API spec format, tone & depth, locked terms); if conventions lack guidance for this file, use the sensible default and note it in one line without editing conventions. Update `status:` frontmatter to `DRAFT` or `UPDATING` as appropriate. If knowledge facts were introduced, also update `knowledge/` if needed. Record `related_paths` if any.
6. **Auto-update 00-common** — immediately after writing, scan the new content for glossary terms, abbreviations, and references not yet in `00-common/`. Auto-append them to the corresponding file without requiring another command:
   - New term → append to `00-common/04-glossary.md` with one-line definition inferred from context; if `vi-en` mode, add English gloss.
   - New abbreviation → append to `00-common/03-abbreviations.md`.
   - New external reference (link, doc, standard) → append to `00-common/02-references.md`.
   No confirmation needed; just note `Auto-updated 00-common: +2 terms, +1 abbreviation.` in the reply. Deduplicate before appending and respect `status: RELEASED` — still auto-append even to RELEASED `00-common` files (they are living references).
7. **Elicitation cleanup** — ask `Keep elicitation file for reference or delete? (keep / delete)`. Act accordingly.
8. **Confirm & mark status** — ask `Mark this file as? (DRAFT / UPDATING / RELEASED)`. Default is `RELEASED` if the user says the file is done. Update frontmatter and `.haro-docs/status/<path>` accordingly. `RELEASED` files become referenceable by later `generate` runs.
9. **Next-step popup** — after marking status, always ask what to do next (picker tool when available, otherwise numbered list). Propose 2–3 concrete candidates based on the fresh reality scan (prerequisites + gaps), each with a one-line reason, plus `review <just-finished file>` and `stop`. Example: `1. generate 02-business/01-value-prop.md — business value needed before architecture; 2. review 01-overview/01-problem-statement.md — just RELEASED, worth a critic pass; 3. stop`. Wait for the pick; do NOT auto-run.

### Examples

``
/haro-docs generate
/haro-docs generate 02-business/01-value-prop.md
/haro-docs generate 10-deliverables/01-BRD.md
``

