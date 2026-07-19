---
description: Have a routed model critique files, a diff, or a design — ranked issues with concrete fixes
---

Target to critique: $ARGUMENTS

Read the Claude Maestro skill (`{baseDir}/SKILL.md`) and run its CRITIQUE
flow on the target above (pass paths, not contents). If the user named a
model, route to that model. If they signalled an effort (literally or in plain
words), pass it through; otherwise judge effort per the skill's EFFORT rules.
After the critique returns, add your own short take only where you disagree or
would reprioritize — then stop.