---
name: <kebab-flow-name>
mode: read-only          # read-only | write
effort: high             # default effort: low | medium | high | xhigh | max
inputs: <one line — what $ARGUMENTS carries: paths / a PR or issue number / a git range / free text>
requires: <optional — external tools the delegate must run, e.g. "gh (GitHub CLI, authenticated)". Omit if none.>
---

# Maestro flow: <name>

**When to use:** <one sentence naming the trigger. Mirror this in the SKILL.md index line so the two agree.>

**Normalize the input:** <how to turn $ARGUMENTS into what the prompt needs — e.g. "strip a PR URL down to the bare number"; "if no paths given, default to the current `git diff`". Keep the TOKEN RULES: pass paths/numbers, never file contents.>

**Prompt shape:** <the exact instruction handed to the delegate. Reference paths/numbers the delegate fetches ITSELF (it reads files / runs `gh` on its own quota). End with an output cap, e.g. "Ranked, most severe first. 40 lines or fewer.">

**After it returns:** <how the orchestrator relays the result. Default: relay inline, add your own take only where you'd reprioritize, keep the `session_id` for a `--resume` follow-up.>

<!--
  HOW A FLOW RUNS (do not copy this comment into a real flow):
  - This file is a SPEC, not a script. It never repeats the bash block.
  - read-only flows run through SKILL.md's CONSULT/CRITIQUE delegate call;
    write flows run through the IMPLEMENT call. `mode` above selects which.
  - `effort` is the default; a caller who signals effort still overrides it.
  - Keep the file short — it loads on demand, but brevity keeps it cheap.
-->
