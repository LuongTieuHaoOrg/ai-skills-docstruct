# Knowledge memory (Haro Docs reference)
> **STOP — READ THIS FILE FULLY BEFORE ACTING.** This file is the normative workflow for `/haro-docs remember` / `/haro-docs knowledge`. Do not act, answer, edit, or call tools from memory: read every step below first. If in doubt at any point, re-read. The reference always wins over memory.
> **Ground rules (apply to every action in this file):** read `docroot` from `.haro-docs/project-profile.yaml` before operating — never guess it. Respect `language.response` (conversation) and `language.documentation` (doc content); if either is missing, ask the user first. Knowledge in `.haro-docs/knowledge/` is ground truth over scanned defaults.

## 1.1 Knowledge as Selective RAG (no vector DB)

Because vector search is not available, knowledge is organized for **filename + index** retrieval:

- Each fact is stored in a file named by domain: `business-*.md`, `technical-*.md`, `team-*.md`, `common-*.md`, `security-*.md`, etc. The agent chooses the name that best matches the content so it can be found without loading all memory.
- `.haro-docs/knowledge/_index.md` is the single lookup table: for each file, one row with `file | domain | one-line summary | tags`. Agents read **only `_index.md`** (small) to decide which files to load for the current task, then load only those files. Never load the entire `knowledge/` directory by default.
- Use `_index.md` before every `init` or `generate` turn that needs project context: read `_index.md`, pick rows whose domain/tags match the task (e.g. business for BRD, technical/architecture for SAD), load only those files. Knowledge overrides scanned defaults when they conflict.

## 4. Command `/haro-docs remember / knowledge` — Project Knowledge Memory (Selective RAG)

Knowledge files are project facts the agent must remember and follow. They are stored as small domain-scoped files so they can be loaded selectively without reading all memory.

### Storage

```
.haro-docs/knowledge/
├── _index.md          # file | domain | summary | tags — the only file read by default
├── business-*.md
├── technical-*.md
├── team-*.md
├── common-*.md
└── security-*.md
```

`_index.md` format (one row per knowledge file):

| file | domain | summary | tags |
|------|--------|---------|------|
| business-stid-team.md | business | STID is company, PM is Hao, members Hao/Vu/Dai | stid, team, pm |
| technical-stack.md | technical | Stack: Next.js + PostgreSQL | nextjs, postgres |

Domains: `business`, `technical`, `team`, `common`, `security`, `quality`, `operations`, `guides` — pick the closest. The agent chooses the filename that best matches the content (e.g. `business-stid-team.md` for team facts) so it can be found without loading all memory. Keep each file focused (1-3 facts); split a user message that contains multiple domains into multiple files.

### Selective loading (RAG without vector DB)

Before any `init` or `generate` turn that needs project context:

1. Read `knowledge/_index.md` if it exists (small, always).
2. Pick only rows whose `domain`/`tags`/`summary` match the current task (e.g. `business` for BRD/Proposal, `technical`+`architecture` for SAD/FSD, `team` for any task mentioning people).
3. Load only those matched knowledge files. Do NOT load the whole `knowledge/` directory.

Knowledge is ground truth: when a knowledge file conflicts with scanned code/README, prefer knowledge.

### Subcommands

| Subcommand | Behaviour |
|------------|-----------|
| `remember <free text>` | **Analyze → confirm → save.** Parse the free text, extract distinct facts, rewrite each into a canonical sentence (current-state style per references/authoring.md (§9), no history phrasing), choose a domain filename for each, and show a preview table `file | canonical content` + updated `_index` rows. Then ask `Confirm? (yes / edit <corrections>)`. Only on `yes` (or after applying `edit`) write the files and update `_index.md`. If the user says `edit`, re-render the preview with corrections and ask again. `knowledge <free text>` is an alias for this subcommand. No backward compatibility is kept. |
| `knowledge` | **List all knowledge** (no args). Read `_index.md` and render the table plus file counts. If `_index.md` missing but files exist, rebuild it from filenames. Read-only. |

Unknown subcommand for this group → `Unknown command 'X'. Valid: remember <free text>, knowledge.` then show this table. `help`/`--help`/`-h` also show it.

### Examples

```
/haro-docs remember STID is my company, I am PM, members are Hao (me), Vu (Backend) and Dai (Frontend)
/haro-docs knowledge
```

