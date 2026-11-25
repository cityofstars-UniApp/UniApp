# Post-mortems

Incident write-ups: a bug reached a place it shouldn't have (a real user, a merged PR, a release), and the interesting part is *why the process let it through*, not just the one-line fix.

A post-mortem is not an [Agent Note](../../.agents/notes/README.md) — an Agent Note records a deliberate decision and its rejected alternatives. A post-mortem is a backward-looking record of a failure: what broke, the mechanism, why every safety net missed it, and the concrete guardrails added so the same class of bug fails loudly next time.

Write one when a bug is **subtle** (the mechanism is non-obvious; a careful engineer would re-derive it the hard way), **systemic** (it escaped through a gap in tests, tooling, or conventions, not a one-off typo), and **costly to rediscover** (it cost real debugging time and would cost it again). Link the guardrails — tests, AGENTS.md rules, Agent Notes — the write-up motivated.

Every post-mortem opens with an **Executive summary**: one short paragraph a busy reader absorbs in thirty seconds — what broke, the root cause in plain terms, why it escaped, and the durable lesson — before the Summary / Timeline / Root cause / Guardrails sections that follow.

Records are numbered `NNNN-slug.md` in this directory. This repository has none yet; the first incident starts at `0001-<slug>.md`.
