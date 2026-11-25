# AGENTS.md — The documentation standard

This file defines the document tiers, placement rules, and writing rules for `docs/`. It follows the [dsh documentation standard](../../Harness/reference-repos/deepseek-harness/docs/AGENTS.md) with one deliberate divergence recorded below: this repository is single-language.

## Language policy

All original documentation in this repository is English. There are no `.zh.md` counterparts, no `.i18n.yaml` consistency records, and no translation-pairing gate. The two process manuals at the repository root ([feature-workflow.md](../feature-workflow.md), [feature-workflow-pipeline.md](../feature-workflow-pipeline.md)) predate this policy and stay in Chinese. Mirrored files under `.agents/` and `.github/` are verbatim copies of dsh originals and keep the original's form; feature-workflow.md §11 owns the mirror inventory.

## The tier taxonomy: one home per fact

| Tier | Job | Does NOT belong there |
|---|---|---|
| Root `AGENTS.md` | Standing orders, one to three lines each, linking their home. Currently a placeholder; filled when the project foundation lands. | Stories, procedures, restated facts |
| [feature-workflow.md](../feature-workflow.md) + [feature-workflow-pipeline.md](../feature-workflow-pipeline.md) | The process home: Stage 0–9 flow, gates, wiring checklist | Product docs, decision rationale |
| [.github/AGENTS.md](../.github/AGENTS.md) | Stage 7 merge-chain authority: `Policy` workflows, fixed check names, `MERGE_POLICY_*` modes | Anything outside GitHub mechanisms |
| [Agent Notes](../.agents/notes/README.md) | Decision records: the why, what was given up, required verification; `implemented/` is present-tense shipped reality | Migration plans, spec-speak after shipping |
| `.agents/skills/` | Skill mirrors: reusable workflow manuals | Product and runtime contracts |
| [glossary.md](glossary.md) | The English vocabulary of the process | Procedure, rationale |
| [postmortem/](postmortem/README.md) | Incident write-ups — the only tier where failure narrative belongs | Deliberate decisions (→ Agent Notes) |
| `cookbook/` | Verified how-to procedures, one recipe per file; empty until real procedures exist | Design rationale (→ Agent Notes) |
| `defensive-patterns/` | Review-enforcement patterns; empty until review traffic exists | Incident narrative (→ postmortem/) |
| `upgrade-guide/` | One migration guide per released version; activates with the first release | Unreleased changes |
| `architecture.md`, `testing.md`, `development.md` | Ordered map, test strategy, and contributor setup of the app source; empty placeholders until the source tree exists (the daily flow lives in feature-workflow.md) | — |

Placement: bugs → [postmortem/](postmortem/README.md); rationale → [Agent Notes](../.agents/notes/README.md); procedure → the process manuals or a skill; standing orders → root `AGENTS.md`; vocabulary → [glossary.md](glossary.md).

## Writing rules

- **Document current state.** History lives in commits, Agent Notes, and postmortems; no "implemented!" or "future: …" annotations.
- **One home per fact.** Search a distinctive phrase before adding a rule; link instead of restating.
- **Classify each document as tutorial or reference.** Tutorials follow an ordered path; references define a lookup scope.
- **One physical line per paragraph.** Use editor soft-wrap.
- **Direct naming.** Name the exact check, file, or mechanism; no metaphor.
- Links are relative Markdown and every link target exists.

No word-count gate is installed; keep root `AGENTS.md` a router anyway. A router that grows into a manual is a bug.

## The slop checklist

Hunt these in any document: duplicated rules (one home, link the rest); implementation-status annotations (status rots); hand-restated inventories when the owning file is authoritative; reasoning transcripts (keep the contract, delete the derivation); paragraph walls; spec-speak in `implemented/` Agent Notes.
