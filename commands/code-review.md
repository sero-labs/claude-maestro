---
description: Route a code review to a frontier model — ranked issues with concrete fixes, no edits made
---

Target to review (paths, a git range, or blank for the current diff): $ARGUMENTS

Read the Claude Maestro skill (`{baseDir}/SKILL.md`) and its
`{baseDir}/flows/code-review.md` flow, then run that flow on the target above.
Pass paths/ranges, not diff contents — the delegate reads files and runs
`git diff` itself. If the user named a model, route to it; if they signalled an
effort, honor it, else use the flow's default (`high`).
