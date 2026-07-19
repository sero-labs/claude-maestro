---
description: Route a GitHub PR review to a frontier model — the delegate fetches the PR itself, then reports ranked issues
---

PR to review (a number or URL): $ARGUMENTS

Read the Claude Maestro skill (`{baseDir}/SKILL.md`) and its
`{baseDir}/flows/pr-review.md` flow, then run that flow on the PR above. Pass
only the PR number — the delegate runs `gh pr view` / `gh pr diff` itself, so
the diff never enters this session. If the user named a model, route to it; if
they signalled an effort, honor it, else use the flow's default (`high`).
Requires `gh` to be installed and authenticated.
