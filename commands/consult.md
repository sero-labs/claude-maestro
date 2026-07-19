---
description: Get a second opinion from a routed model (GPT-5.6 Sol by default) without leaving this session
---

Question / topic: $ARGUMENTS

Read the Claude Maestro skill (`{baseDir}/SKILL.md`) and run its CONSULT
flow on the question above. If the user named a model in the arguments, route
to that model. If they signalled an effort — literally (`--effort low`) or in
plain words ("quick", "think hard") — pass it through; otherwise judge effort
from the question per the skill's EFFORT rules. Keep this session's own
commentary brief — the point is the other model's opinion, returned inline.