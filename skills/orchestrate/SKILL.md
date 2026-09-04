---
name: sdlc-orchestrate
description: Drives one turn of an sdlc-pipeline run -- reads run state, executes the next pending stage via the matching subagent, stops at gates, and applies replans. Invoked by the /sdlc-pipeline command; not meant to be invoked directly by name.
---

# SDLC Orchestrate

You are driving one turn of an SDLC pipeline run. A "turn" starts from
`/sdlc-pipeline start|resume|replan` and ends when you hit a gate, hit a
rework-cap escalation, or the run completes -- never leave the run mid-way
through a stage without writing state, since the next turn may be a new
session with no memory of this one.

## Run state

Path: `<project-root>/.sdlc/runs/<run-id>/state.json`. Artifacts live in
`<project-root>/.sdlc/runs/<run-id>/artifacts/`, named
`NN_<stage-id>.md` in execution order.

Schema:

```json
{
  "run_id": "2026-09-03-checkout-flow",
  "project_phase": 1,
  "parent_run_id": null,
  "source": {"type": "prd_file", "value": "path/to/prd.md", "checkout_path": null},
  "created_at": "2026-09-03T12:00:00Z",
  "status": "in_progress",
  "roster": [],
  "stage_plan": [
    {"id": "sharpen_prd", "role": "pm", "status": "pending", "artifact": null},
    {"id": "write_plan", "role": "architect", "status": "pending", "artifact": null},
    {"id": "gate_plan", "type": "gate", "status": "pending"},
    {"id": "spec_ui", "role": "ux_designer", "status": "pending", "artifact": null},
    {"id": "gate_designs", "type": "gate", "status": "pending"},
    {"id": "milestones", "type": "milestone_loop", "status": "pending", "items": []},
    {"id": "gate_rollout", "type": "gate", "status": "pending"},
    {"id": "ready_to_test_report", "role": "pm", "status": "pending", "artifact": null},
    {"id": "gate_flag", "type": "gate", "status": "pending"},
    {"id": "gate_next_phase", "type": "gate", "status": "pending"}
  ],
  "max_rework_rounds": 3,
  "replan_history": []
}
```

`source.type` is `"prd_file"` (value is a path, `checkout_path` stays
`null`) or `"repo"` (value is the git URL or local path the CEO gave,
`checkout_path` is where the orchestrator resolved it to on disk -- see
"`start`" below).

Each milestone item, once populated (see the milestone loop below), looks
like:

```json
{
  "name": "Add checkout API endpoint",
  "status": "pending",
  "implement": {"status": "pending", "rework_count": 0},
  "code_review": {"status": "pending", "round": 0, "max_rounds": 2},
  "qa_tirekick": {"status": "pending"}
}
```

`status` on any stage/item is one of: `pending`, `in_progress`, `done`,
`blocked` (rework cap hit, waiting on the CEO), `skipped` (dropped by
intake or a replan).

## Stage catalog

The `stage_plan` above is the full catalog with every stage present. The
`sharpen_prd` stage (run by the `pm` agent) decides which stages/roles are
actually needed for this PRD and prunes the plan accordingly -- mark
dropped stages `skipped` rather than deleting them, so `replan_history`
and post-hoc review can see what was considered and cut.

Gates fire at the macro level only: `gate_plan`, `gate_designs`,
`gate_rollout`, `gate_flag`, `gate_next_phase`. Milestones cycle under
`gate_rollout` without their own CEO gate -- do not add one.
