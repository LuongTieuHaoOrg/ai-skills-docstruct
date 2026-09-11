# Authoring rules (Haro Docs reference)
> **STOP — READ THIS FILE FULLY BEFORE ACTING.** This file is the normative rules for writing any doc content. Do not act, answer, edit, or call tools from memory: read every step below first. If in doubt at any point, re-read. The reference always wins over memory.
> **Ground rules (apply to every action in this file):** read `docroot` from `.haro-docs/project-profile.yaml` before operating — never guess it. Respect `language.response` (conversation) and `language.documentation` (doc content); if either is missing, ask the user first. Knowledge in `.haro-docs/knowledge/` is ground truth over scanned defaults .

## 9. Authoring Rules

1. **Single Source of Truth:** each piece of content is written once in one file only; aggregated documents only assemble content, never duplicate it.
2. **Language configuration** — read from `project-profile.yaml` (`language.response`, `language.documentation`):
   - **Conversation replies** (clarifying questions, elicitation Q&A, outline proposals...) use `language.response`.
   - **Documentation content** uses `language.documentation`, defined as:

     | Mode | Definition | Example |
     |------|------------|---------|
     | `en` | Pure English | — |
     | `vi` | Pure Vietnamese; English kept ONLY for proper nouns / product / technology names with no Vietnamese equivalent (code, Java, PostgreSQL, REST API...) | "Hệ thống chạy trên PostgreSQL" |
     | `vi-en` | Vietnamese with an English gloss in parentheses on FIRST use of each specialized term; register all glossed terms in `glossary.md` | "Cơ sở dữ liệu (database) lưu trữ hồ sơ" |

      The distinction between the last two: in `vi`, English appears because *no Vietnamese equivalent exists*; in `vi-en`, English glosses are used proactively to *teach terminology* so readers can research further.

      **`vi` in detail — pure Vietnamese.** English appears ONLY where Vietnamese has no word: proper nouns, product/technology names, code identifiers (PostgreSQL, REST API, `userId`...). Never translate a proper noun into Vietnamese, never add an English gloss in parentheses.
      - Right: "Hệ thống chạy trên PostgreSQL và expose REST API."
      - Wrong: "Cơ sở dữ liệu (database) lưu trữ hồ sơ." — parenthesized glosses belong to `vi-en`; in `vi` write just "Cơ sở dữ liệu".

      **`vi-en` in detail — Vietnamese that teaches terminology.** On the FIRST use of each specialized/domain term, add the English origin in parentheses and register the term in `glossary.md`; later occurrences use Vietnamese only.
      - First use: "Cơ sở dữ liệu (database) lưu trữ hồ sơ." Later: "Cơ sở dữ liệu được backup hằng ngày."
      - Gloss ONLY specialized/domain terms (database, deployment, stakeholder...). Do NOT gloss everyday words — "Hệ thống (system) chạy (run)" is noise. Do NOT gloss proper nouns — they stay bare as in `vi`.

      **How to choose:** readers work purely in Vietnamese → `vi`; readers will consult English sources later → `vi-en`, so the doc teaches each term at first contact.
   - **Fallback:** if either language setting is empty or missing, ASK the user to decide before running any command. Never assume a default.
3. **Write current state, not changes — no backward compatibility:** when creating or updating any block/document (new or adjustment), always treat it as the **first version**. Write the final content as if written from scratch today. A document describes how things ARE, never how they CHANGED. Do NOT keep backward compatibility: never mention backward, previous version, migration from old, or version history. Forbidden in document bodies: change-log phrasing such as "updated...", "added...", "removed...", "no longer applies...", "previously...", "backward compatible", "previous version", "migration". Do not embed version history, revision notes, or "what's new" sections anywhere — version control is handled by **git alone**.

   | Wrong (in body) | Right |
   |-----------------|-------|
   | "The payment feature has been added to the billing module." | "The billing module includes a payment feature..." |
   | "The legacy report section was removed in this version." | *(delete the section entirely, leave no trace)* |

4. **No duplication:** check the glossary before defining a new term.
5. **File naming:** kebab-case with numeric prefix indicating reading order — e.g. `01-problem-statement.md`.
6. **Images/diagrams:** store in `99-assets/`, reference via relative paths; no inline base64.
7. **Block lifecycle:** each block has status `draft → review → approved`, tracked in `.haro-docs/status/`; only `approved` blocks may be aggregated into deliverables without further review.
8. **Sub-READMEs:** every folder must have a `README.md` describing its scope and file list.

