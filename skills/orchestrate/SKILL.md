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

## `start <prd-path-or-repo-link>`

1. Determine the source type: if the argument looks like a git URL
   (`https://...`, `git@...`) or an existing local directory that isn't a
   single file, treat it as `type: "repo"`. Otherwise treat it as
   `type: "prd_file"`.
2. Generate `run_id` as `<YYYY-MM-DD>-<short-slug-of-source-name>` (repo
   name for a repo, filename for a PRD file). If that id already exists
   under `.sdlc/runs/`, append `-2`, `-3`, etc.
3. Create `.sdlc/runs/<run-id>/artifacts/`.
4. If `type: "repo"`:
   - A git URL: shallow-clone it into `.sdlc/runs/<run-id>/checkout/`
     (`git clone --depth 1 <url> .sdlc/runs/<run-id>/checkout`).
   - A local path: use it directly, read-only -- do not copy it. Set
     `checkout_path` to that path as given.
   - Set `source.checkout_path` accordingly.
5. Write `state.json` from the schema above: `source` set per steps 1 and
   4, `roster` empty, every stage `pending`, `status: "in_progress"`.
6. Continue as `resume <run-id>` below, in the same turn.

## `resume <run-id>`

1. Load `.sdlc/runs/<run-id>/state.json`. If it doesn't exist, stop and
   tell the CEO the run-id wasn't found -- do not guess a path.
2. Find the first stage in `stage_plan` (top to bottom, expanding
   `milestone_loop` items in order) whose status is `pending`, `in_progress`,
   or `blocked`. If none, mark `status: "complete"` in state.json, report a
   summary of the whole run, and stop.
3. Dispatch on that stage's type:
   - **role stage** (`sharpen_prd`, `write_plan`, `spec_ui`,
     `ready_to_test_report`): see "Running a role stage" below.
   - **gate**: see "Gates" below.
   - **milestone_loop**: see "Milestone loop" below.
4. After handling one stage, loop back to step 2 -- keep going in the same
   turn until you hit a gate, a `blocked` escalation, or completion. Do not
   stop after an ordinary role stage just because it finished; only gates
   and escalations pause the run. This is the core autonomy rule: subagents
   hand each other feedback (review findings, fast-follow tickets) directly
   through state and artifacts, with no CEO round-trip in between.

## Progress logging

You are a process manager, and the CEO is watching this turn happen live
-- print progress as you go, not just a summary once you stop. Every time
you're about to do something in the loop above, print one line first;
print one more when it finishes. Use this format:

```
▶ <stage_id> [role: <role>] -- starting
✓ <stage_id> -- <one-line result>
```

For the milestone loop, include the milestone name and, for code_review,
the round number, e.g.:

```
▶ milestone 2/4 "Add checkout API endpoint" > implement [role: senior_engineer] -- starting
✓ milestone 2/4 > implement -- 3 tickets, all tests passing
▶ milestone 2/4 > code_review round 1 [role: code_reviewer] -- starting
✓ milestone 2/4 > code_review round 1 -- 2 blocking findings, back to implement
▶ milestone 2/4 > implement (rework 1) [role: senior_engineer] -- starting
✓ milestone 2/4 > implement (rework 1) -- addressed both findings
▶ milestone 2/4 > code_review round 2 [role: code_reviewer] -- starting
✓ milestone 2/4 > code_review round 2 -- approved
▶ milestone 2/4 > qa_tirekick [role: qa_engineer] -- starting
✓ milestone 2/4 > qa_tirekick -- clean, milestone done
```

A gate or escalation still prints its `▶`/pre-question line, but the
`AskUserQuestion` itself is the stop -- don't print a redundant "waiting"
line after it.

## Running a role stage

1. Mark the stage `in_progress`, print the stage's `▶ ... -- starting` line.
2. Dispatch the matching subagent (same name as the stage's `role`) via the
   Agent tool. Give it: the stage instructions are already in its own
   agent file, so your dispatch prompt only needs to name which prior
   artifacts are relevant inputs (by path, under
   `.sdlc/runs/<run-id>/artifacts/`) -- the subagent reads them itself via
   Read. Do not paste artifact contents into the dispatch prompt. For
   `sharpen_prd` specifically, also tell the PM which mode applies:
   `source.type` and, for a PRD file, its path; for a repo, its
   `checkout_path` to recon.
3. On completion, write the subagent's output to
   `.sdlc/runs/<run-id>/artifacts/NN_<stage-id>.md` (NN = this stage's
   1-based position in execution order), record that path in the stage's
   `artifact` field, mark it `done`, and print the `✓ ... -- <one-line
   result>` line.
4. Special case, `sharpen_prd` only: the PM's artifact includes a proposed
   roster and stage list. Set `roster` in state.json to the PM's proposed
   roles. For every stage in `stage_plan` whose `role` is not in the new
   roster (and is not itself a gate or the milestone loop), mark it
   `skipped` instead of `pending`. If the PM proposed dropping the UX
   stage, designer role never gets dispatched and `gate_designs` still
   fires but reviews only whatever artifacts remain relevant -- state that
   plainly in the gate summary rather than silently skipping the gate too.

## Gates

A gate always stops the turn once reached -- never dispatch a subagent and
a gate resolution in the same loop iteration without the CEO's answer in
between.

1. Mark the gate `in_progress`, print `▶ <gate_id> -- awaiting CEO review`.
2. Summarize the artifact(s) that fed this gate (read them, don't assume
   you remember their content from earlier in a long session).
3. Call `AskUserQuestion` with options: **Approve**, **Request changes**,
   **Simplify the pipeline**, **Reject/stop**.
   - **Approve**: mark the gate `done`, print `✓ <gate_id> -- approved`,
     continue the loop.
   - **Request changes**: ask what needs to change, mark the stage(s) that
     produced the reviewed artifact(s) back to `pending`, re-dispatch that
     role with the CEO's feedback included in the prompt, then re-present
     this same gate once the new artifact is ready (still in this turn).
   - **Simplify the pipeline**: run the Replan flow (below) with this
     gate's context as the implicit reason if the CEO doesn't give one,
     then continue the loop under the revised plan. Do not re-ask this
     gate afterward unless the replan changed what feeds it.
   - **Reject/stop**: mark `status: "stopped"` in state.json with the
     CEO's stated reason, report where things were left, and end the turn.

## Milestone loop

Print the `▶`/`✓` lines from "Progress logging" above for every sub-stage
and round below -- this loop is the part of a run most likely to chain
several autonomous steps in a row, so it's the part where live logging
matters most.

1. If `items` is empty, populate it from the architect's milestone list
   (one item per milestone, in the architect's stated order), each with
   `implement`/`code_review`/`qa_tirekick` sub-status blocks as shown in
   the schema.
2. Work items in order. For the first item not `done`, run its sub-loop:
   - **implement**: dispatch `senior_engineer` with the milestone spec (and
     any open fast-follow tickets from a prior QA round, if this is a
     rework pass). On completion, mark `implement.status: "done"` for this
     pass and set `code_review.status: "pending"`.
   - **code_review**: dispatch `code_reviewer` with the current diff.
     Increment `code_review.round`.
     - No blocking findings: mark `code_review.status: "done"`, set
       `qa_tirekick.status: "pending"`.
     - Blocking findings, `round < max_rounds` (2): set `implement.status:
       "pending"`, increment `implement.rework_count`, loop back to
       implement with the findings as the fast-follow input.
     - Blocking findings, `round == max_rounds`: this is a rework-cap
       escalation (see below) -- do not loop again on your own.
   - **qa_tirekick**: dispatch `qa_engineer` against the reviewed diff.
     - No bugs found: mark `qa_tirekick.status: "done"` and the whole
       milestone item `done`; move to the next item (step 2).
     - Bugs found, `implement.rework_count < max_rework_rounds` (run-level,
       default 3): file the fast-follow, increment
       `implement.rework_count`, reset `code_review.round` to 0 (the fix
       gets a fresh review, not a continuation of the old count), loop
       back to implement with the fast-follow ticket as input.
     - Bugs found, `implement.rework_count == max_rework_rounds`: rework-cap
       escalation.
3. Once every item is `done`, mark the `milestone_loop` stage `done` and
   continue the outer loop.

## Rework-cap escalation

Stop the turn (do not auto-continue). Summarize what's stuck and why
(which cap was hit, what the remaining findings/bugs are), and call
`AskUserQuestion` with: **Approve anyway** (mark the blocking sub-stage
`done` despite open findings -- record this in state.json), **One more
manual round** (CEO gives specific direction, one extra round runs outside
the normal cap), **Stop the run** (mark `status: "stopped"`). Mark the
relevant sub-stage `blocked` before asking, so a fresh `resume` lands back
on this same question if the CEO doesn't answer in this turn.

## Replan

Triggered two ways: a gate's "Simplify the pipeline" option, or standalone
via `/sdlc-pipeline replan <run-id> ["reason"]`. Same flow either way.

1. Load `state.json`. Identify every stage/milestone item whose status is
   not `done` -- a replan can only change the future, never rewrite what
   already happened.
2. If no reason was given, ask the CEO what's driving the change (too
   complex, taking too long, scope changed, etc.) before proposing
   anything.
3. Propose a revised `stage_plan`/`roster`/`max_rework_rounds`/milestone
   items. Allowed edits: drop a pending stage (mark `skipped`), re-add a
   previously skipped one, swap a role, collapse remaining stages or
   milestones into fewer, shrink `max_rework_rounds`. Present it as a diff
   against the current plan, with one line per change on what it trades
   away (e.g. "dropping qa_tirekick for milestone 3 means only code review
   catches issues before rollout").
4. Call `AskUserQuestion` with: **Apply**, **Adjust further**, **Cancel**.
   - **Adjust further**: go back to step 3 with the CEO's refinement.
   - **Cancel**: leave `state.json` untouched, return to wherever
     execution was (if this came from a gate, re-present that gate).
   - **Apply**: write the revised plan to `state.json`, append one entry to
     `replan_history` (`{"timestamp", "reason", "diff"}`), then continue
     the main `resume` loop in the same turn -- unless the very next stage
     under the new plan is itself a gate, in which case that gate fires
     normally.

## `replan <run-id> ["reason"]`

Entry point for the standalone command. Load state, run the Replan flow
above with the given reason (or none, triggering step 2's question), then
continue the `resume` loop under the resulting plan in the same turn.
