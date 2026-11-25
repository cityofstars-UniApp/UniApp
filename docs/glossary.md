# Glossary

The English vocabulary of this repository's process. The manuals ([feature-workflow.md](../feature-workflow.md), [feature-workflow-pipeline.md](../feature-workflow-pipeline.md)) own the flow; this file names its terms in English so other documents use one word per concept. Mirrored mechanism files under `.github/` and `.agents/` are the authority for what each gate checks.

## The flow

- **Stage 0–9** — the feature flow's ten stations: 0 wiring, 1 issue intake, 2 board triage, 3 branch + decision note, 4 implementation + tests, 5 local gates, 6 pull request, 7 the gate wall, 8 merge + note transition, 9 lifetime maintenance ([pipeline map](../feature-workflow-pipeline.md)).
- **Dual track** — an Issue (work-ticket layer, GitHub-visible, on the Project board) and an Agent Note (decision layer, `.agents/notes/`) coexist on the same PR.
- **Mirror** — a verbatim copy of a dsh original kept for reference; the manuals link each mirror to its original under `../Harness/reference-repos/deepseek-harness/`. Design reference, not the running mechanism (feature-workflow.md §12 records the gaps).

## The gate wall (Stage 7)

- **Gate / check** — one automated merge condition published under a fixed check name; branch protection matches by name, so workflow and job names never change.
- **`Issue policy`** — a PR must link a valid Issue (`Fixes`/`Refs #N`) and satisfy the label discipline.
- **`Policy / Master Change`** — external action reports what changed on the base since the PR started.
- **`Policy / Pre-Merge Check`** — merge-tree the base into the head, then run the root pre-merge script.
- **`all checks passed`** — the CI matrix verdict.
- **Weighted approval** — review points accumulated per the approval policy; the author's own points are capped.
- **`Policy / Approve Check` / `Policy / Reviewer Check`** — approval sufficiency and per-file review coverage.
- **`Policy / Recheck`** — a push to master re-runs the head checks of every open PR.
- **`MERGE_POLICY_*` mode** — per-gate knob: `off` / `report` / `block`; missing or unknown counts as `report`. New gates go live as `report` first, then tighten to `block`.
- **Local gates** — lefthook hooks: pre-commit lint + whitespace, pre-push typecheck. Local fast, CI full.

## Issues and labels

- **Issue type** — Idea / Feature / Bug / Research / Task, attached by the template frontmatter.
- **`kind/*`** — PR-only, exactly one of six: feature, bug-fix, doc, testing, cleanup, dependency.
- **`area/*`** — PR labels for every touched surface; `p0-p3` at most one, equal to the highest Priority of the resolved Issues; `source/*` is Issue-only.
- **Project Status** — the board's seven states: Inbox / Backlog / Ready / In progress / In review / Done / No action. Priority is a board field, not a label.

## Decisions

- **Agent Note** — a decision record in `.agents/notes/`, path-encoded `{lifecycle}/{class}/yyyy-mm-dd-topic.md`. Lifecycle: `proposed/` (unbuilt), `implemented/` (shipped, kept present-tense current), `rejected/` (frozen verdict on the `Status:` line), `archived/` (sealed history). Class: feature / bug-fix / simplification / architecture / process / testing.
- **`## Alternatives considered`** — mandatory in every note: each genuine alternative and why it lost.
- **Note transition** — `proposed/` → `implemented/` in the merge commit: `Status: implemented`, `## Proposal` → `## Decision`, criteria + risks → `## Consequences`.
- **Skill** — a workflow manual in `.agents/skills/`; no workflow ever invokes one. Walls catch skeletons, manuals teach judgment.
- **Post-mortem** — an incident record, [postmortem/](postmortem/README.md); not a decision.
