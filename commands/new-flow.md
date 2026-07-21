---
description: Scaffold a new Maestro flow — a reusable routed-model task (e.g. code-review, pr-review) with its own slash command
---

Flow to create: $ARGUMENTS

You are extending Claude Maestro with a new named flow. A flow is a reusable,
specialized variant of consult/critique/implement — e.g. "code-review",
"pr-review", "write-tests". Read the skill (`{baseDir}/SKILL.md`) and the flow
template (`{baseDir}/flows/_TEMPLATE.md`) first so the artifacts you write match
the house style.

## 1. Infer the flow's parameters from the description

From `$ARGUMENTS`, work out:

- **name** — a short kebab-case id (e.g. `code-review`, `pr-review`). Avoid
  collisions with existing commands (`consult`, `critique`, `implement`, `setup`,
  `new-flow`) and existing files in `{baseDir}/flows/`.
- **mode** — `read-only` if the flow only inspects/reports (reviews, audits,
  explanations); `write` if the routed model should edit files (generators,
  refactors, test-writers). When unsure, prefer `read-only` — it's the safer
  default and the user can regenerate as `write`.
- **inputs** — what the user will pass: file paths, a git range, a GitHub PR/issue
  number or URL, or free text. Note how to normalize it.
- **requires** — external tools the delegate must run itself (e.g. `gh` for a
  GitHub flow). Read-only mode still allows Bash, so `gh`/`git` work; just record
  the dependency so it's visible.
- **effort** — the default cognitive load (see the skill's EFFORT section). Reviews
  and design judgement lean `high`; mechanical flows lean `medium`.
- **model** — almost always OMIT this. A flow with no `model` inherits the user's
  default (`MODEL_ROUTER_MODEL`), which keeps it portable across machines and lets
  the user's own default (or a per-call choice) win. Pin a `model:` only for a
  flow-intrinsic reason — the flow genuinely runs better on one specific model (a
  capability or quality fit that's part of the flow's design), not because a caller
  mentioned one this once. A caller naming a model at call time still overrides it.
- **prompt shape** — the templated instruction the delegate receives, ending in an
  output cap. Keep the TOKEN RULES: the prompt passes paths/numbers and the
  delegate fetches contents on its own quota.

## 2. Confirm before writing

Show the user the inferred spec compactly — name, mode, inputs, requires, effort,
model (or "inherits default" when omitted), and the prompt shape — and the three
files you'll create/touch. Let them correct
anything, then proceed. (If they've said "just do it" / autonomy is clearly
wanted, write first and show the result.)

## 3. Write three things

**a) `{baseDir}/flows/<name>.md`** — follow `_TEMPLATE.md` exactly. It's a spec,
not a script: it never repeats the bash block, it just sets `mode`/`effort`/
inputs and the prompt shape. Keep it short.

**b) `{baseDir}/commands/<name>.md`** — a thin wrapper mirroring the existing ones
(`consult.md`, `critique.md`). Shape:

```markdown
---
description: <one line — what this flow does>
---

<Input label>: $ARGUMENTS

Read the Claude Maestro skill (`{baseDir}/SKILL.md`) and its
`{baseDir}/flows/<name>.md` flow, then run that flow on the input above. Pass
paths/numbers, not contents. If the user named a model, route to it; if they
signalled an effort, honor it, else use the flow's default. <Any flow-specific
note, e.g. "The delegate runs `gh` itself — don't fetch the PR into this session.">
```

**c) The SKILL.md index** — insert ONE bullet between the
`<!-- MAESTRO:FLOWS ... -->` and `<!-- /MAESTRO:FLOWS -->` markers in
`{baseDir}/SKILL.md`, formatted:

`- **<name>** — <when to use, one line> → `flows/<name>.md``

If the placeholder line `_None yet. …_` sits between the markers, replace it with
your bullet (don't leave the placeholder once a real flow exists). If bullets are
already there, add yours after the last one. Never disturb the marker comments —
they're the anchor the next `/maestro:new-flow` writes against. Keep the index and
each flow file's "When to use" wording consistent.

Idempotency: if a flow with this name already exists, don't blindly overwrite —
tell the user it exists and offer to update it in place instead.

## 4. Report

Tell the user, briefly: the flow is live now via natural language ("run the
<name> flow on …") and via `/maestro:<name>` **after the next session reload**
(new command files register on restart; the flow itself works immediately because
the skill reads `flows/` at runtime). Show the created paths.
