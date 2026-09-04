---
description: Start, resume, or replan an SDLC pipeline run (PRD or repo link -> design -> milestones -> rollout, with CEO approval gates). Usage: /sdlc-pipeline start <prd-path-or-repo-link> | /sdlc-pipeline resume <run-id> | /sdlc-pipeline replan <run-id> ["reason"]
disable-model-invocation: true
---

Parse the arguments as `<verb> <rest>` where verb is `start`, `resume`, or
`replan`.

Invoke the sdlc-pipeline:sdlc-orchestrate skill and follow it exactly as
presented to you, running the section matching the verb:

- `start <prd-path-or-repo-link>` -> the "`start <prd-path-or-repo-link>`" section
- `resume <run-id>` -> the "`resume <run-id>`" section
- `replan <run-id> ["reason"]` -> the "`replan <run-id> [\"reason\"]`" section

If no verb is given, or the run-id/source is missing, ask the CEO for the
missing piece rather than guessing a run-id or defaulting to `start`.
