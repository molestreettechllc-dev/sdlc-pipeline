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
`NN_<stage-id>.md` in execution order -- except `sharpen_prd`, `write_plan`,
`spec_ui`, and `ready_to_test_report`, which are always `.html`: anything
the CEO is asked to approve is a presentation, not a markdown file
rendered flat. See "Presenting an approval artifact" below.

Schema:

```json
{
  "run_id": "2026-09-03-checkout-flow",
  "project_phase": 1,
  "parent_run_id": null,
  "source": {"type": "prd_file", "value": "path/to/prd.md", "checkout_path": null},
  "created_at": "2026-09-03T12:00:00Z",
  "status": "in_progress",
  "ui_mode": false,
  "pending_gate": null,
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

`ui_mode` (set by `start --with-ui`, default `false`) and `pending_gate`
(default `null`) together let an external dashboard drive this run instead
of a live terminal -- see "Posing a question to the CEO" below for what
they mean and how `resume` handles them.

Each milestone item, once populated (see the milestone loop below), looks
like:

```json
{
  "name": "Add checkout API endpoint",
  "status": "pending",
  "implement": {"status": "pending", "rework_count": 0},
  "code_review": {"status": "pending", "round": 0, "max_rounds": 2},
  "qa_tirekick": {"status": "pending"},
  "artifacts": []
}
```

`artifacts` accumulates paths as the milestone's sub-stages run (see
"Milestone loop" below) -- every dispatch writes one, same as a role
stage, so nothing a gate needs to summarize is ever un-persisted.

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

**`gate_flag` is skipped by default.** While a project is in active
development (no first production deploy to real users yet), feature-flag
gates are noise: at `start`, mark `gate_flag` `skipped`. It runs only when
the CEO asks for it -- in the `start` command's arguments or words, at any
gate, or via a replan that re-adds it (a previously skipped stage can be
re-added; see Replan). The PM's `ready_to_test_report` still states plainly
whether anything is behind a flag, but no approval question follows it, so
the run goes from `ready_to_test_report` straight to `gate_next_phase`.
When a phase-2+ run is created from `gate_next_phase`, carry the skip
over unless the CEO says otherwise.

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
   4, `roster` empty, every stage `pending` except `gate_flag` (`skipped`, see "Stage catalog"),
   `status: "in_progress"`,
   `ui_mode: true` if `--with-ui` was given (default `false`),
   `pending_gate: null`.
6. Continue as `resume <run-id>` below, in the same turn.

## `resume <run-id> [--answer "<text>"]`

1. Load `.sdlc/runs/<run-id>/state.json`. If it doesn't exist, stop and
   tell the CEO the run-id wasn't found -- do not guess a path.
2. If `pending_gate` is set (only possible when `ui_mode: true`), handle
   it before anything else -- see "Posing a question to the CEO" for the
   exact resolution logic. `--answer` not given: re-print
   `pending_gate.summary` and stop, nothing else to do. `--answer` given
   and it resolves: clear `pending_gate`, apply that option's branch logic,
   and fall through to step 3 in the same turn. `--answer` given but
   unresolved: leave `pending_gate` as is, print what couldn't be matched,
   and stop.
3. Find the first stage in `stage_plan` (top to bottom, expanding
   `milestone_loop` items in order) whose status is `pending`, `in_progress`,
   or `blocked`. If none, mark `status: "complete"` in state.json, report a
   summary of the whole run, and stop.
4. Dispatch on that stage's type:
   - **role stage** (`sharpen_prd`, `write_plan`, `spec_ui`,
     `ready_to_test_report`): see "Running a role stage" below.
   - **gate**: see "Gates" below.
   - **milestone_loop**: see "Milestone loop" below.
5. After handling one stage, loop back to step 3 -- keep going in the same
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

A gate or escalation still prints its `▶`/pre-question line, but posing
the question itself is the stop (see "Posing a question to the CEO" next)
-- don't print a redundant "waiting" line after it.

## Posing a question to the CEO

Every gate, rework-cap escalation, and replan confirmation below poses a
question with a fixed set of options plus, sometimes, free text. What the
options are and what each one does is unchanged either way -- this section
covers only *how* the question reaches the CEO, which depends on
`ui_mode`.

**`ui_mode: false` (default -- interactive terminal use):** call
`AskUserQuestion` with the given options exactly as written at each call
site. This is a live conversation: ask, get the answer, continue in the
same turn.

**`ui_mode: true` (`--with-ui`, driven by an external dashboard):** there
is no live terminal to answer into -- the process running this turn is not
the process that will supply the answer. Instead of calling
`AskUserQuestion`:

1. Write a `pending_gate` object to `state.json`:
   ```json
   {"id": "<gate/escalation/replan id>", "summary": ["line", "line", ...],
    "options": ["Option A", "Option B", ...]}
   ```
   `summary` is the context you'd otherwise only speak aloud before
   `AskUserQuestion` -- write it down in full, since the dashboard has no
   other way to show the CEO what's being asked.
2. Mark whatever stage/item this question blocks on `"blocked"`.
3. Print the `▶ ... -- awaiting CEO review` line, then end the turn
   immediately: no further tool calls, no summary line after. The next
   turn (a `resume` with an answer) is what handles the reply.

**Resolving a `pending_gate` on `resume` (`ui_mode: true` only):** covered
in "`resume <run-id>`" step 2 above -- summarized here for completeness.
Given `--answer "<text>"`, interpret it as the CEO's answer in their own
words, exactly as you'd interpret a live reply to `AskUserQuestion`: map it
to the closest matching option in `pending_gate.options`, and treat any
remaining text as the free-form detail that option's own branch logic
expects (a change request's specifics, a rejection's reason, a replan
direction, manual-round guidance, whatever applies at that call site). If
it genuinely doesn't map to any option, don't guess -- leave `pending_gate`
untouched, print what couldn't be resolved, and stop so the CEO can
re-answer. Once resolved, clear `pending_gate` to `null` and apply that
option's branch logic exactly as documented at its own gate/escalation/
replan section below.

## Presenting an approval artifact

`sharpen_prd`, `write_plan`, `spec_ui`, and `ready_to_test_report` produce
an `.html` file, not markdown -- the whole point is that the CEO looks at
something, not just reads a gate's text summary. Whenever one of these
feeds a gate (in "Gates" below, and at `ready_to_test_report`'s own
handoff into `gate_flag`, when that gate is enabled), actually present it, in addition to the usual
text summary:

- **If the `Artifact` tool is available in this session** (it publishes
  to a hosted, shareable link): publish the HTML file with it and share
  the resulting link -- this is the preferred path, since the CEO can
  click straight into a rendered page.
- **If it isn't available** (a plain terminal/CLI session): tell the CEO
  the exact file path and that it's an HTML file to open in a browser
  (e.g. `open <path>` on macOS, or the equivalent for their platform) --
  don't just paste its contents as text, that defeats the point of having
  built a presentation.

Either way, still write the text summary the gate/report calls for --
`ui_mode: true` has no browser session watching, only `pending_gate.summary`,
so the artifact link/path belongs in that summary too, not instead of it.

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
   `checkout_path` to recon. Tell every subagent the exact artifact path
   to write to, per "Run state" above (`.html` for `sharpen_prd`,
   `write_plan`, `spec_ui`, `ready_to_test_report`; `.md` otherwise).
3. On completion, record that path in the stage's `artifact` field, mark
   it `done`, and print the `✓ ... -- <one-line result>` line. If this
   stage's artifact is one of the four `.html` ones, present it now per
   "Presenting an approval artifact" above -- don't wait until the gate
   that reviews it to surface the link, so the CEO has time to open it
   before the question arrives.
4. Special case, `sharpen_prd` only: the PM's artifact includes a proposed
   roster and stage list. `architect` and `senior_engineer` are always in
   the roster regardless of what the PM proposes -- `write_plan` is what
   produces the milestone list the whole `milestone_loop` depends on, and
   no code gets written without an engineer, so neither is optional. If
   the PM's proposal omits either, keep it in the roster anyway and note
   the override plainly in the gate summary (e.g. "PM proposed skipping
   the architect; kept because milestones require one"). `ux_designer`,
   `code_reviewer`, and `qa_engineer` are genuinely optional -- set
   `roster` to `["architect", "senior_engineer"]` plus whichever of those
   three the PM included. For every stage in `stage_plan` whose `role` is
   not in the final roster (and is not itself a gate or the milestone
   loop), mark it `skipped` instead of `pending`. If the PM proposed
   dropping the UX stage, designer role never gets dispatched and
   `gate_designs` still fires but reviews only whatever artifacts remain
   relevant -- state that plainly in the gate summary rather than
   silently skipping the gate too.

## Gates

A gate always stops the turn once reached -- never dispatch a subagent and
a gate resolution in the same loop iteration without the CEO's answer in
between.

1. Mark the gate `in_progress`, print `▶ <gate_id> -- awaiting CEO review`.
2. Summarize the artifact(s) that fed this gate (read them, don't assume
   you remember their content from earlier in a long session). If any of
   them is one of the `.html` presentation artifacts and it wasn't already
   surfaced when its stage finished (e.g. this is a fresh `resume` in a
   new session), present it now per "Presenting an approval artifact"
   before posing the question.
3. Pose this to the CEO (see "Posing a question to the CEO") with options:
   **Approve**, **Request changes**, **Simplify the pipeline**,
   **Reject/stop**.
   - **Approve**: mark the gate `done`, print `✓ <gate_id> -- approved`,
     continue the loop.
   - **Request changes**: what needs to change is either the CEO's live
     follow-up reply (`ui_mode: false`) or the free text after "Request
     changes" in `--answer` (`ui_mode: true`) -- either way, mark the
     stage(s) that produced the reviewed artifact(s) back to `pending`,
     re-dispatch that role with the CEO's feedback included in the prompt,
     then re-present this same gate once the new artifact is ready (still
     in this turn; `ui_mode: true` re-presenting means writing a fresh
     `pending_gate` and ending the turn again, same as the first time).
   - **Simplify the pipeline**: run the Replan flow (below) with this
     gate's context as the implicit reason if the CEO doesn't give one,
     then continue the loop under the revised plan. Do not re-ask this
     gate afterward unless the replan changed what feeds it.
   - **Reject/stop**: mark `status: "stopped"` in state.json with the
     CEO's stated reason, report where things were left, and end the turn.

### Special case: `gate_next_phase`

This gate doesn't fit the four generic options above -- there's no
artifact to approve or send back for changes, and "Approve" alone doesn't
say whether the CEO wants a phase 2. Pose a dedicated question instead:
**Start next phase**, **End the run**.

- **Start next phase**: the phase-2 source (a new PRD path or repo link,
  same as `start` accepts) is either the CEO's live follow-up reply
  (`ui_mode: false`) or the free text after "Start next phase" in
  `--answer` (`ui_mode: true`). Generate a new `run_id`, create
  its `.sdlc/runs/<new-run-id>/` with a fresh `state.json` (full stage
  catalog reset to `pending`, empty `roster`, `pending_gate: null`,
  `ui_mode` copied from this run -- a UI-driven run's next phase is still
  UI-driven), set its `parent_run_id` to this run's `run_id` and
  `project_phase` to this run's `project_phase +
  1`. Mark this run's `gate_next_phase` `done` and this run's `status`
  `"complete"`. Continue as `resume <new-run-id>` in the same turn --
  phase 2 starts immediately, it does not wait for a separate `start`
  call.
- **End the run**: mark `gate_next_phase` `done` and `status: "complete"`.
  Report the full run summary and stop.

## Milestone loop

Print the `▶`/`✓` lines from "Progress logging" above for every sub-stage
and round below -- this loop is the part of a run most likely to chain
several autonomous steps in a row, so it's the part where live logging
matters most. Every dispatch below also writes its output to
`.sdlc/runs/<run-id>/artifacts/milestone-<index>_<substage>_round<N>.md`
(1-based milestone index, `<N>` the round for `implement`/`code_review`,
omitted for `qa_tirekick` since it doesn't repeat within one pass) and
appends that path to the item's `artifacts` list -- the same rule role
stages follow, so `gate_rollout` always has something real to read and
summarize.

1. If `items` is empty, populate it from the architect's milestone list
   (one item per milestone, in the architect's stated order), each with
   an `implement` sub-status block always, a `code_review` block only if
   `code_reviewer` is in `roster` (otherwise omit it -- there is no round
   cap or review to run), and a `qa_tirekick` block only if `qa_engineer`
   is in `roster`. A milestone item with neither sub-block completes as
   soon as `implement` finishes -- there is nothing else to gate it on.
2. Work items in order. For the first item not `done`, run its sub-loop:
   - **implement**: dispatch `senior_engineer` with the milestone spec (and
     any open fast-follow tickets from a prior QA round, if this is a
     rework pass). On completion, mark `implement.status: "done"` for this
     pass. If this item has a `code_review` block, set its status to
     `"pending"` and continue to code_review below; otherwise, if it has a
     `qa_tirekick` block, set that to `"pending"` instead; otherwise mark
     the whole milestone item `done` and move to the next item (step 2).
   - **code_review** (only if this item has this block): dispatch
     `code_reviewer` with the current diff. Increment `code_review.round`.
     - No blocking findings: mark `code_review.status: "done"`. If this
       item has a `qa_tirekick` block, set it to `"pending"`; otherwise
       mark the milestone item `done` and move to the next item.
     - Blocking findings, `round < max_rounds` (2): set `implement.status:
       "pending"`, increment `implement.rework_count`, loop back to
       implement with the findings as the fast-follow input.
     - Blocking findings, `round == max_rounds`: this is a rework-cap
       escalation (see below) -- do not loop again on your own.
   - **qa_tirekick** (only if this item has this block): dispatch
     `qa_engineer` against the reviewed diff.
     - No bugs found: mark `qa_tirekick.status: "done"` and the whole
       milestone item `done`; move to the next item (step 2).
     - Bugs found, `implement.rework_count < max_rework_rounds` (run-level,
       default 3): file the fast-follow, increment
       `implement.rework_count`, and if this item has a `code_review`
       block reset its `round` to 0 (the fix gets a fresh review, not a
       continuation of the old count). Either way, loop back to implement
       with the fast-follow ticket as input -- the fix goes through
       implement first, then code_review (if present) before qa_tirekick
       re-verifies.
     - Bugs found, `implement.rework_count == max_rework_rounds`: rework-cap
       escalation.
3. Once every item is `done`, mark the `milestone_loop` stage `done` and
   continue the outer loop.

## Rework-cap escalation

Stop the turn (do not auto-continue). Mark the relevant sub-stage
`blocked` first, so a fresh `resume` lands back on this same question no
matter how the turn actually ended. Summarize what's stuck and why
(which cap was hit, what the remaining findings/bugs are),
then pose this to the CEO (see "Posing a question to the CEO") with:
**Approve anyway** (mark the blocking sub-stage `done` despite open
findings -- record this in state.json), **One more manual round** (CEO
gives specific direction -- their live follow-up reply, or the free text
after "One more manual round" in `--answer` -- one extra round runs
outside the normal cap), **Stop the run** (mark `status: "stopped"`).

A manual round runs exactly like an ordinary rework pass (dispatch
`senior_engineer` with the CEO's direction as input, then back through
`code_review`/`qa_tirekick` as normal for this item) but does not touch
`round` or `rework_count` -- it's outside the cap, not a reset of it. If it
still fails, you're back at this same escalation: summarize what the
manual round changed and didn't fix, and ask again. There is no third
option beyond these three; a manual round can be requested more than
once, each one a fresh CEO decision, not an automatic retry.

## Replan

Triggered two ways: a gate's "Simplify the pipeline" option, or standalone
via `/sdlc-pipeline replan <run-id> ["reason"]`. Same flow either way.

1. Load `state.json`. Identify every stage/milestone item whose status is
   not `done` -- a replan can only change the future, never rewrite what
   already happened.
2. If no reason was given, pose the CEO a question (see "Posing a question
   to the CEO") asking what's driving the change (too complex, taking too
   long, scope changed, etc.) before proposing anything -- this one has no
   fixed menu, so in `ui_mode: true` write `pending_gate` with `options`
   as a single placeholder like `["Provide a reason"]` and treat the whole
   `--answer` text as the reason, rather than trying to match it against a
   choice.
3. Propose a revised `stage_plan`/`roster`/`max_rework_rounds`/milestone
   items. Allowed edits: drop a pending stage (mark `skipped`), re-add a
   previously skipped one, swap a role, collapse remaining stages or
   milestones into fewer, shrink `max_rework_rounds`. Present it as a diff
   against the current plan, with one line per change on what it trades
   away (e.g. "dropping qa_tirekick for milestone 3 means only code review
   catches issues before rollout").
4. Pose this to the CEO (see "Posing a question to the CEO") with:
   **Apply**, **Adjust further**, **Cancel**.
   - **Adjust further**: go back to step 3 with the CEO's refinement --
     their live follow-up reply, or the free text after "Adjust further"
     in `--answer`.
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
