---
description: Implement a task via a routed model — you spec and verify, the routed model builds
---

The brief: $ARGUMENTS

Read the Claude Maestro skill (`{baseDir}/SKILL.md`) and run its IMPLEMENT
flow: write a self-contained spec (you make every design call), delegate with
the compound delegate-and-verify command, judge the evidence, report in 5
lines or fewer. If the user named a model, route to that model. If they
signalled an effort, pass it through; otherwise judge effort per the skill's
EFFORT rules (fallback `MODEL_ROUTER_EFFORT`).