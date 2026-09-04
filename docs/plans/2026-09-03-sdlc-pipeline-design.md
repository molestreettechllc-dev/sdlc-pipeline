# SDLC Pipeline Plugin — Design

## Problem

A single, fixed-roster, single-gate pipeline (the earlier `app-dev-pipeline`
skill draft) doesn't match how the CEO actually wants to run projects: the
working group and stage list should flex with the complexity of the PRD, the
CEO needs multiple approval gates along the way (not just one at the end),
and the CEO needs to be able to reshape the plan mid-run if it's gotten too
complex or is taking too long — without waiting for the next gate.

## Form factor

A full personal Claude Code plugin (`~/.claude/plugins/sdlc-pipeline/`),
usable from any project:

- `commands/sdlc-pipeline.md` — entry point (`start`, `resume`, `replan`)
- `agents/*.md` — one subagent per worker profile, own context/tools
- `skills/orchestrate/SKILL.md` — stage catalog, gate logic, rework rules

No daemon or scripting runtime exists between invocations, so every command
run re-derives "where am I" from a state file rather than assuming
continuity with a prior turn.

## Run state

`<project>/.sdlc/runs/<run-id>/state.json` — lives inside the target repo
(diffable, travels with the project), not in the plugin's own global dir.

```json
{
  "run_id": "...", "project_phase": 1, "parent_run_id": null,
  "source": {"type": "prd_file", "value": "path", "checkout_path": null},
  "created_at": "...",
  "roster": ["pm", "architect", "ux_designer", "senior_engineer",
             "code_reviewer", "qa_engineer"],
  "stage_plan": [
    {"id": "sharpen_prd", "role": "pm", "status": "done", "artifact": "..."},
    {"id": "write_plan", "role": "architect", "status": "done", "artifact": "..."},
    {"id": "gate_plan", "type": "gate", "status": "pending"},
    {"id": "spec_ui", "role": "ux_designer", "status": "pending"},
    {"id": "gate_designs", "type": "gate", "status": "pending"},
    {"id": "milestones", "type": "milestone_loop", "status": "pending",
     "items": [
       {"name": "...", "status": "pending",
        "implement": {"status": "pending", "rework_count": 0},
        "code_review": {"status": "pending", "round": 0, "max_rounds": 2},
        "qa_tirekick": {"status": "pending", "rework_count": 0}}
     ]},
    {"id": "gate_rollout", "type": "gate", "status": "pending"},
    {"id": "ready_to_test_report", "role": "pm", "status": "pending"},
    {"id": "gate_flag", "type": "gate", "status": "pending"},
    {"id": "gate_next_phase", "type": "gate", "status": "pending"}
  ],
  "max_rework_rounds": 3,
  "replan_history": []
}
```

Artifacts live in `.sdlc/runs/<run-id>/artifacts/`. Each "next phase" is a
new run with `parent_run_id` set, so phase 2's roster/stages can be simpler
than phase 1's without warping the schema.

## Stage catalog

```
sharpen_prd (pm: PRD + roster/stage proposal + success metrics)
  -> GATE_plan (reviews PM's PRD + architect's plan together)
write_plan (architect: design flowchart(s) + milestone/sprint list)
spec_ui (ux_designer: user journeys + UI mocks)
  -> GATE_designs
per milestone, in order:
  implement (senior_engineer)
  code_review (code_reviewer, hard cap 2 rounds, escalates to CEO past cap)
  qa_tirekick (qa_engineer; bug found -> fast-follow ticket -> back to
               senior_engineer -> code_review -> qa_tirekick, capped at
               max_rework_rounds, escalates to CEO past cap)
  -> next milestone, or fall through once all milestones are done
GATE_rollout (real PR/deploy approval)
ready_to_test_report (pm)
GATE_flag (feature flag on/off)
GATE_next_phase (CEO supplies next-phase PRD or ends the run)
```

Metrics-planning and dashboard-update (present in the original reference
diagram) are folded into the PM's `sharpen_prd` stage rather than kept as
separate stages/roles — no data-analyst role was defined, and the PM
already states success metrics as part of PRD refinement.

Gates fire at the macro level only. A 5-milestone feature does not produce
5 rollout approvals — milestones cycle underneath `GATE_rollout` and only
surface to the CEO if a rework loop hits its cap.

## Intake / dynamic roster

Folded into `sharpen_prd`, not a separate stage. The PM agent reads the raw
PRD and makes the judgment call on which of the 6 roles and which stages
apply (e.g. a copy-only change skips `ux_designer`, `qa_tirekick` might be
lightweight for a trivial change) — no fixed complexity tiers. The proposed
roster and stage list are part of the PM's artifact and ride into
`GATE_plan` alongside the architect's plan, rather than getting their own
gate.

### Input modes: PRD file or repo link

`start` accepts either a PRD/brief file, or a repo (a git URL, or a local
path) with no brief at all. The two feed the same `sharpen_prd` stage
differently:

- **PRD given:** as already described — refine the brief that's there.
- **Repo given, no PRD:** there's no intent to refine yet, only a codebase.
  Before dispatching the PM, the orchestrator resolves the repo to a local
  checkout — clone it (shallow) into
  `.sdlc/runs/<run-id>/checkout/` if it's a URL, or use the given path
  directly if it's already local — and records that path as
  `source.checkout_path`. The PM then does read-only recon of the checkout
  (structure, data models, existing UX flows, what the app currently does)
  and drafts a PRD from scratch: a summary of the product's current
  intent, plus an **expanded** vision that takes real liberties — proposed
  improvements to the existing design and functionality, not just a
  restatement of what's there. Every claim about *current* behavior must
  cite a real file/path in the checkout; every *proposed* addition must be
  clearly marked as new so the CEO gate isn't reviewing a document where
  invented features read as already-existing ones.

This still produces one artifact reviewed at `GATE_plan` — the CEO is
approving "build this expanded product," with the current-state summary as
grounding, not approving two separate documents.

`source.type` is `"prd_file"` or `"repo"`; `checkout_path` is set only for
the repo case and stays `null` for a PRD-file run with no repo context.

## Process manager and live logging

The orchestrate skill *is* the process manager — there's no separate
component. Its defining rule: **only a gate, a rework-cap escalation, or
run completion stops the turn.** Every ordinary stage transition — a role
finishing and handing its output to the next role, a code-review finding
routing back to the engineer, a QA bug becoming a fast-follow ticket —
happens automatically, in the same turn, with no CEO input. The CEO
supplies feedback only at the five gates and at replan; everywhere else,
subagents hand each other exactly what they need directly through the
state file and artifact paths.

Because a run can chain through many stages unattended, the chat must
narrate it live, not just report a summary once it stops. Before
dispatching any subagent, the orchestrator prints one line naming the
stage and role about to run; after it completes, one line with the
one-line result. Milestone loop iterations log every sub-stage and round
(implement → code_review round N → qa_tirekick, and any rework it
triggers), so a currently-open session reads as a live status feed of
who's doing what — "the coder just started milestone 2", not silence
until the next gate.

## Gates

Every `type: "gate"` stage stops the turn and calls `AskUserQuestion` with:
**Approve**, **Request changes** (redo the artifact), **Simplify the
pipeline** (enter replan flow), **Reject/stop**. Identical behavior whether
the gate is reached at the end of a `start` or is the first pending item
found on `resume`.

## Replan (CEO can reshape the graph at any point)

Two entry points:
- From any gate's "Simplify the pipeline" option.
- Standalone: `/sdlc-pipeline replan <run-id> ["reason"]` — works regardless
  of current stage status, since it doesn't wait for a gate to come around.
  This is the practical limit of "any point": Claude Code cannot interrupt
  itself mid-subagent-call within one turn, same as today's Ctrl+C.

A replan can only touch stages not yet `done` — it edits the future, not
the record of what happened. Allowed edits: drop/re-add a pending stage,
swap a role, collapse remaining stages, shrink `max_rework_rounds`.

Flow: read current state → propose a revised plan as a diff with one line
per cut on what it trades away → one confirmation (apply / adjust further /
cancel) → on confirm, write the new plan, append to `replan_history`
(timestamp, reason, diff), continue into the next stage in the same turn
unless that stage is itself a gate.

## Agent profiles

Six subagents, each single-purpose and scoped by tool access:

| Agent | Tools | Role |
|---|---|---|
| `pm` | Read, Grep, Glob, Write | PRD refinement, roster/stage proposal, success metrics, final readiness report |
| `architect` | Read, Grep, Glob, Write | System design flowchart(s), milestone/sprint breakdown |
| `ux_designer` | Read, Grep, Glob, Write | User journeys, UI mocks, accessibility notes |
| `senior_engineer` | Read, Grep, Glob, Write, Edit, Bash | Tickets + implementation + tests per milestone, fast-follow fixes |
| `code_reviewer` | Read, Grep, Glob, Bash | Read-only PR review, hard cap 2 rounds |
| `qa_engineer` | Read, Grep, Glob, Bash | End-to-end smoke tests / tirekicks, fast-follow bug tickets, QA sign-off (not deployment approval) |

Full system prompts for each are captured in the conversation that produced
this design and will be written verbatim into `agents/*.md` during
implementation.

## Out of scope (for this version)

- Auto-triggered replan nudges (e.g. the pipeline suggesting simplification
  on its own when a stage is running long) — replan is CEO-initiated only.
  Add if it turns out gates alone don't catch runaway complexity in
  practice.
- A dedicated data-analyst role / metrics-planning or dashboard-update
  stages — folded into the PM's PRD stage. Split out again if metrics work
  genuinely needs its own owner.
- Per-milestone CEO gates — gates stay macro-level. Revisit if a real run
  shows the CEO wanting visibility between milestones, not just at rollout.
