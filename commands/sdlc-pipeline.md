---
description: Start, resume, or replan an SDLC pipeline run (PRD or repo link -> design -> milestones -> rollout, with CEO approval gates). Usage: /sdlc-pipeline start <prd-path-or-repo-link> [--with-ui] | /sdlc-pipeline resume <run-id> [--answer "<text>"] | /sdlc-pipeline replan <run-id> ["reason"]
disable-model-invocation: true
---

Parse the arguments as `<verb> <rest>` where verb is `start`, `resume`, or
`replan`. `--with-ui` (on `start` only) and `--answer "<text>"` (on
`resume` only) are optional flags within `<rest>`, not part of the
positional source/run-id.

Invoke the sdlc-pipeline:sdlc-orchestrate skill and follow it exactly as
presented to you, running the section matching the verb:

- `start <prd-path-or-repo-link> [--with-ui]` -> the "`start <prd-path-or-repo-link>`" section
- `resume <run-id> [--answer "<text>"]` -> the "`resume <run-id>`" section
- `replan <run-id> ["reason"]` -> the "`replan <run-id> [\"reason\"]`" section

If no verb is given, or the run-id/source is missing, ask the CEO for the
missing piece rather than guessing a run-id or defaulting to `start`.
