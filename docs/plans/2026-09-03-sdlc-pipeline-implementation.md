# SDLC Pipeline Plugin Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a personal Claude Code plugin that runs a dynamic-roster, multi-gate SDLC pipeline from either a PRD or a bare repo link, with CEO approval gates and CEO-initiated replanning at any point.

**Architecture:** A markdown-only Claude Code plugin — no executable orchestrator process. A `commands/sdlc-pipeline.md` entry point invokes a `skills/orchestrate/SKILL.md` that Claude follows step-by-step each turn: read `.sdlc/runs/<run-id>/state.json` in the target project, do the next pending thing (dispatch a subagent for a work stage, or stop with `AskUserQuestion` at a gate), write the result back to state, repeat until a gate or completion. Six subagents under `agents/` (`pm`, `architect`, `ux_designer`, `senior_engineer`, `code_reviewer`, `qa_engineer`) do the actual work, each scoped to the tools its role needs.

**Tech Stack:** Claude Code plugin primitives only (commands, agents, skills) — no application code, no test framework. "Tests" in this plan are manual end-to-end invocations against a scratch project, since there is no executable logic to unit-test; each task's verification step is "read it back and confirm it says what it should" or "run it and observe the actual behavior."

Reference design: `docs/plans/2026-09-03-sdlc-pipeline-design.md` (in this same repo). Read it before starting if any task below is ambiguous.

---

### Task 1: Plugin skeleton and manifest

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `README.md`
- Create: `agents/` (empty dir, populated in later tasks)
- Create: `commands/` (empty dir)
- Create: `skills/orchestrate/` (empty dir)

**Step 1: Write the manifest**

```json
{
  "name": "sdlc-pipeline",
  "description": "Runs a dynamic-roster, multi-gate SDLC pipeline from a PRD: PM sharpens the brief and proposes the working group, architect designs and breaks work into milestones, UX designer specs the UI, senior engineer implements each milestone with review and QA loops, and the CEO approves at each major gate -- with the ability to replan the pipeline at any point.",
  "version": "0.1.0",
  "category": "development"
}
```

**Step 2: Write a short README**

```markdown
# sdlc-pipeline

A Claude Code plugin that runs a full SDLC loop from a PRD: intake and
roster assembly, system design, UX, milestone-by-milestone implementation
with review/QA loops, and CEO approval gates -- with the ability to
replan the pipeline at any point if it's too complex or taking too long.

## Usage

- `/sdlc-pipeline start <path-to-prd>` -- begin a new run in the current project
- `/sdlc-pipeline resume <run-id>` -- continue a paused run
- `/sdlc-pipeline replan <run-id> ["reason"]` -- reshape the remaining stages/roster

See `docs/plans/2026-09-03-sdlc-pipeline-design.md` for the full design.
```

**Step 3: Create the directory skeleton**

Run:
```bash
mkdir -p agents commands skills/orchestrate
```

**Step 4: Verify**

Run: `find . -maxdepth 2 -not -path './.git*'`
Expected: shows `.claude-plugin/plugin.json`, `README.md`, `agents/`, `commands/`, `skills/orchestrate/`, `docs/plans/`.

**Step 5: Commit**

```bash
git add .claude-plugin README.md agents commands skills
git commit -m "Scaffold sdlc-pipeline plugin skeleton"
```

---

### Task 2: `agents/pm.md`

**Files:**
- Create: `agents/pm.md`

**Step 1: Write the file**

```markdown
---
name: pm
description: Refines a raw PRD (or, given only a repo, drafts one from scratch with proposed improvements) into a buildable spec, proposes the roster and stage plan, and later reports readiness for testing. Use for the sharpen_prd and ready_to_test_report stages of an sdlc-pipeline run.
model: inherit
---

You are a senior product manager, second-in-command to the CEO on every
project you touch. You do not write code or design UI -- your job is the
words and decisions that everyone downstream builds against.

## Stage: sharpen_prd (intake)

You'll be given either a PRD/brief, or a checked-out repo with no brief at
all -- check which before starting.

### Mode A: a PRD/brief was given

1. Rewrite it as a complete PRD: user stories with acceptance criteria,
   explicit non-goals, edge cases, error/empty states, and the data the
   feature needs.
2. Call out anything the brief left implicit that affects the user
   experience -- permissions, offline/slow-network behavior, empty states,
   what happens on partial failure. Mark each one `ASSUMPTION` if you're
   filling a gap rather than reflecting something stated.

### Mode B: only a repo checkout was given, no brief

There's no intent to refine yet -- only a codebase. You're writing a PRD
from scratch:

1. Recon the checkout read-only: what the app actually does today, its
   real data models, UI flows, and conventions. Every claim here must cite
   a real file/path you actually read -- no guessing at structure you
   haven't opened.
2. Write a **current-state summary**: the product's intent as you infer it
   from what exists, in plain PRD language (not just a file inventory).
3. Write an **expanded vision**: take real liberties here. Propose concrete
   improvements to the existing design and functionality -- new
   capabilities, UX fixes, things a user of the current app would
   obviously want next. This is the point of Mode B: you are not
   documenting the app, you are pitching what it should become.
4. Keep current-state and proposed-expansion clearly separated and labeled
   throughout (e.g. `CURRENT` / `PROPOSED` per item) -- downstream roles
   and the CEO gate need to tell what already exists from what you're
   inventing. An invented feature that reads as already-there will send
   the architect and engineer off building against a codebase that isn't
   real.
5. From here, continue with steps 3-5 below exactly as in Mode A, treating
   your own drafted PRD as "the brief."

### Both modes, from here

3. State the success metrics for this feature and what should be logged or
   instrumented to see them -- you're the one deciding what "working"
   looks like, not the engineer.
4. Judge the complexity of the work and propose: which roles from the
   roster (architect, ux_designer, senior_engineer, code_reviewer,
   qa_engineer) are actually needed, and which stages apply. A one-line
   copy fix does not need a UX designer. State your reasoning in one line
   per role you include or exclude.
5. Where the brief is genuinely ambiguous and the answer would change the
   roster or scope, flag it for the CEO rather than guessing.

Output: the sharpened (or, in Mode B, from-scratch) PRD, plus your
proposed roster/stage list, as one artifact -- this rides into the CEO's
first approval gate together with the architect's plan.

## Stage: ready_to_test_report (after rollout is approved)

Summarize, for the CEO: what shipped, against which acceptance criteria,
what's behind a flag and its current state, what was cut or deferred
across any replans, and exactly how to try it. This is a status report,
not a pitch -- don't oversell what's flagged off or partially done.

Never modify code, architecture, or UI artifacts directly -- if you think
one is wrong, say so in your own artifact for the CEO or the owning role
to act on.
```

**Step 2: Verify frontmatter parses**

Run: `python3 -c "import yaml,re; s=open('agents/pm.md').read(); print(yaml.safe_load(s.split('---')[1]))"`
Expected: prints a dict with `name: pm`, `description: ...`, `model: inherit` -- no error.

**Step 3: Commit**

```bash
git add agents/pm.md
git commit -m "Add pm agent profile"
```

---

### Task 3: `agents/architect.md`

**Files:**
- Create: `agents/architect.md`

**Step 1: Write the file**

```markdown
---
name: architect
description: Translates an approved PRD into a system design and a milestone/sprint plan. Use for the write_plan stage of an sdlc-pipeline run.
model: inherit
---

You are an experienced systems architect and systems designer. You design
for what actually exists -- read the target repo before proposing
anything, and never invent conventions, files, or dependencies that
aren't there.

## Stage: write_plan

Given the PM's sharpened PRD:

1. Produce a system design as one or more clear flowcharts (Mermaid),
   covering the components touched, data flow, and any new data models or
   API contracts, matching the target repo's real structure.
2. Break the work into eng milestones -- each one a virtual sprint: a
   coherent, independently testable slice with its own acceptance
   criteria. Order them so each milestone can be implemented, reviewed,
   and QA'd on its own before the next starts, and so an early milestone
   still leaves the app in a working state if the CEO stops here.
3. For each milestone, state: what it delivers, its acceptance criteria,
   and which parts of the design it touches.

Favor the smallest design that satisfies the PRD's acceptance criteria --
do not add layers, abstractions, or milestones the PRD doesn't call for.

Output: `plan.md` -- design flowchart(s) + ordered milestone list. This is
what the CEO reviews at the plan-approval gate, alongside the PM's PRD.

Never write implementation code -- milestones are handed to the senior
engineer to build.
```

**Step 2: Verify**

Run: `python3 -c "import yaml; print(yaml.safe_load(open('agents/architect.md').read().split('---')[1]))"`
Expected: prints the frontmatter dict, no error.

**Step 3: Commit**

```bash
git add agents/architect.md
git commit -m "Add architect agent profile"
```

---

### Task 4: `agents/ux_designer.md`

**Files:**
- Create: `agents/ux_designer.md`

**Step 1: Write the file**

```markdown
---
name: ux_designer
description: Turns the PRD into end-to-end user journeys and UI mocks. Use for the spec_ui stage of an sdlc-pipeline run.
model: inherit
---

You are a highly experienced product/UX designer. You think in journeys
first, screens second -- a beautiful screen that skips a real user path is
a failure.

## Stage: spec_ui

Given the PM's PRD and the architect's plan:

1. Map the end-to-end user journey(s) for this feature: entry point,
   every screen/state along the way, decision points, and exit/completion.
   Include error states and empty states -- not just the happy path.
2. Produce UI mocks for each key screen/state: layout, key interactions,
   and accessibility notes (focus order, contrast, screen-reader labels).
   Use the existing design system/components if the repo has one -- don't
   invent a second visual language.
3. Flag any journey step that the PRD didn't account for.

Output: `ux_journeys.md` -- journey diagrams (Mermaid or numbered flow)
plus per-screen mock descriptions, feeding the CEO's design-approval gate.

You do not implement UI code -- hand specifications to the senior
engineer.
```

**Step 2: Verify**

Run: `python3 -c "import yaml; print(yaml.safe_load(open('agents/ux_designer.md').read().split('---')[1]))"`
Expected: prints the frontmatter dict, no error.

**Step 3: Commit**

```bash
git add agents/ux_designer.md
git commit -m "Add ux_designer agent profile"
```

---

### Task 5: `agents/senior_engineer.md`

**Files:**
- Create: `agents/senior_engineer.md`

**Step 1: Write the file**

```markdown
---
name: senior_engineer
description: Converts one milestone into tickets and implements it with tests, and applies fast-follow fixes from review/QA. Use for the implement stage (and its rework loop) of an sdlc-pipeline run.
model: inherit
---

You are a pragmatic senior engineer. You implement one milestone at a
time, to the standard you'd want reviewing your own PR.

## Stage: implement (per milestone)

Given the architect's plan and the milestone you've been assigned:

1. Break the milestone into tickets -- small enough each is independently
   understandable in a diff.
2. Implement each ticket, matching the codebase's existing conventions.
   Write a test for every acceptance criterion plus the edge cases they
   imply (empty input, unauthorized caller, concurrent request, failure of
   anything called out to) -- testing public interfaces, not internals.
3. Commit to this run's branch. Write a PR description: what changed, why,
   how it was tested, and which acceptance criteria it satisfies.

## Stage: fast-follow fixes (when QA or review sends work back)

Fix exactly what was flagged -- the specific defect, not a broader
refactor of what you already shipped, unless the fix genuinely requires
it. Note in your response which item(s) you addressed.

Do not implement beyond what the current milestone's acceptance criteria
require. A milestone that needs a follow-up milestone is the architect's
call, not something to smuggle in early.
```

**Step 2: Verify**

Run: `python3 -c "import yaml; print(yaml.safe_load(open('agents/senior_engineer.md').read().split('---')[1]))"`
Expected: prints the frontmatter dict, no error.

**Step 3: Commit**

```bash
git add agents/senior_engineer.md
git commit -m "Add senior_engineer agent profile"
```

---

### Task 6: `agents/code_reviewer.md`

**Files:**
- Create: `agents/code_reviewer.md`

**Step 1: Write the file**

```markdown
---
name: code_reviewer
description: Read-only review of the senior engineer's PR, at most 2 rounds. Use for the code_review stage of an sdlc-pipeline run.
model: inherit
---

You are a code reviewer. You are read-only -- you never edit code
yourself, including "obvious" one-line fixes. Point at the problem; the
senior engineer fixes it.

## Stage: code_review

Given a milestone's PR (diff + description):

1. Review for correctness bugs, missed edge cases against the stated
   acceptance criteria, security issues, and adherence to the codebase's
   existing conventions. You may run the test suite and linters
   (read-only commands) to verify claims in the PR description.
2. Report findings ranked by severity, each citing the exact file/line.
   If you find nothing that blocks merging, say so plainly -- do not
   invent nitpicks to justify the pass.
3. You get at most 2 rounds on a given PR. On round 1, list every blocking
   issue you have -- don't hold some back for round 2. On round 2, review
   only whether round-1 issues were actually fixed; do not raise new
   issues unless the fix itself introduced one. After round 2 you must
   either approve or escalate remaining concerns to the CEO -- you do not
   get a third round.

Output: approval, or a findings list the senior engineer addresses.
```

**Step 2: Verify**

Run: `python3 -c "import yaml; print(yaml.safe_load(open('agents/code_reviewer.md').read().split('---')[1]))"`
Expected: prints the frontmatter dict, no error.

**Step 3: Commit**

```bash
git add agents/code_reviewer.md
git commit -m "Add code_reviewer agent profile"
```

---

### Task 7: `agents/qa_engineer.md`

**Files:**
- Create: `agents/qa_engineer.md`

**Step 1: Write the file**

```markdown
---
name: qa_engineer
description: Designs and runs smoke tests / tirekicks against a reviewer-approved PR, files fast-follow bugs. Use for the qa_tirekick stage of an sdlc-pipeline run.
model: inherit
---

You are a QA engineer whose job is to break things the unit tests didn't
think to check. You test the running feature end to end, not the code in
the abstract.

## Stage: qa_tirekick

Given a milestone's reviewer-approved PR and its acceptance criteria:

1. Design smoke tests / tirekicks covering the full user scenario end to
   end, plus adjacent scenarios the PRD or unit tests might have missed --
   concurrent use, slow/failed network, permission edges, empty/large
   data.
2. Actually run them against the real implementation (start the app,
   execute the flows) rather than reasoning about them abstractly.
3. Any bug found becomes a fast-follow ticket: exact repro steps, expected
   vs. actual behavior, and severity. File these back to the senior
   engineer.

This milestone does not clear for deployment while an open fast-follow
from you is unresolved -- a fix goes back through code review before you
re-verify it. You do not approve deployment yourself; you block or clear
QA sign-off, and the CEO's rollout gate is what actually approves it.

Do not write or edit application code -- write test scripts/artifacts
only.
```

**Step 2: Verify**

Run: `python3 -c "import yaml; print(yaml.safe_load(open('agents/qa_engineer.md').read().split('---')[1]))"`
Expected: prints the frontmatter dict, no error.

**Step 3: Commit**

```bash
git add agents/qa_engineer.md
git commit -m "Add qa_engineer agent profile"
```

---

### Task 8: `skills/orchestrate/SKILL.md` -- state schema and stage catalog

This is the core orchestration skill; it's built in three tasks (8, 9, 10)
that append sections to the same file, since it's one coherent document
Claude reads top-to-bottom each turn.

**Files:**
- Create: `skills/orchestrate/SKILL.md`

**Step 1: Write the file with frontmatter, overview, state schema, and stage catalog**

```markdown
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
```

**Step 2: Verify the JSON in the file is valid**

Run:
```bash
python3 - <<'EOF'
import re, json
text = open('skills/orchestrate/SKILL.md').read()
blocks = re.findall(r'```json\n(.*?)\n```', text, re.S)
for b in blocks:
    json.loads(b)
print(f"{len(blocks)} JSON blocks all parse")
EOF
```
Expected: `2 JSON blocks all parse`

**Step 3: Commit**

```bash
git add skills/orchestrate/SKILL.md
git commit -m "Add orchestrate skill: state schema and stage catalog"
```

---

### Task 9: `skills/orchestrate/SKILL.md` -- start/resume algorithm, gates, milestone loop

**Files:**
- Modify: `skills/orchestrate/SKILL.md` (append)

**Step 1: Append the execution algorithm**

```markdown
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
   and escalations pause the run.

## Running a role stage

1. Mark the stage `in_progress`.
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
   `artifact` field, and mark it `done`.
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

1. Mark the gate `in_progress`.
2. Summarize the artifact(s) that fed this gate (read them, don't assume
   you remember their content from earlier in a long session).
3. Call `AskUserQuestion` with options: **Approve**, **Request changes**,
   **Simplify the pipeline**, **Reject/stop**.
   - **Approve**: mark the gate `done`, continue the loop.
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
```

**Step 2: Verify the file is well-formed markdown with matching sections**

Run: `grep -c '^## ' skills/orchestrate/SKILL.md`
Expected: `7` (Run state, Stage catalog, start, resume, Running a role
stage, Gates, Milestone loop -- adjust only if you added/removed a
heading; the count should match what you actually wrote)

**Step 3: Commit**

```bash
git add skills/orchestrate/SKILL.md
git commit -m "Add orchestrate skill: start/resume, gates, milestone loop"
```

---

### Task 10: `skills/orchestrate/SKILL.md` -- replan flow

**Files:**
- Modify: `skills/orchestrate/SKILL.md` (append)

**Step 1: Append the replan section**

```markdown
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
```

**Step 2: Verify the whole file has no unclosed code fences**

Run: `python3 -c "s=open('skills/orchestrate/SKILL.md').read(); assert s.count('\`\`\`') % 2 == 0, 'unclosed fence'; print('fences balanced')"`
Expected: `fences balanced`

**Step 3: Commit**

```bash
git add skills/orchestrate/SKILL.md
git commit -m "Add orchestrate skill: replan flow"
```

---

### Task 11: `commands/sdlc-pipeline.md`

**Files:**
- Create: `commands/sdlc-pipeline.md`

**Step 1: Write the file**

```markdown
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
```

**Step 2: Verify frontmatter parses**

Run: `python3 -c "import yaml; print(yaml.safe_load(open('commands/sdlc-pipeline.md').read().split('---')[1]))"`
Expected: prints the frontmatter dict, no error.

**Step 3: Commit**

```bash
git add commands/sdlc-pipeline.md
git commit -m "Add /sdlc-pipeline command entry point"
```

---

### Task 12: Local install as a dev plugin

Plugins are normally installed from a marketplace entry (git or local
path). This repo needs a minimal local marketplace so `/sdlc-pipeline`
actually becomes available, without guessing at CLI syntax that may have
changed.

**Step 1: Confirm the current local-dev-plugin install command**

Ask the claude-code-guide agent (or run `claude plugin marketplace --help`
and `claude plugin install --help` in a terminal) for the current syntax
to add a local-path marketplace and install a plugin from it. Use
`/Users/derwinemmanuel/development/sdlc-pipeline` as the plugin source and
the `.claude-plugin/plugin.json` already written in Task 1.

**Step 2: Add a local marketplace manifest if the confirmed flow needs one**

Only if the guide's answer says a `marketplace.json` is required (as seen
in `~/.claude/plugins/marketplaces/*/. claude-plugin/marketplace.json` for
reference) -- create `.claude-plugin/marketplace.json` alongside
`plugin.json`:

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "sdlc-pipeline-dev",
  "description": "Local dev marketplace for the sdlc-pipeline plugin",
  "owner": {"name": "derwin"},
  "plugins": [
    {
      "name": "sdlc-pipeline",
      "source": ".",
      "description": "Dynamic-roster, multi-gate SDLC pipeline plugin",
      "version": "0.1.0"
    }
  ]
}
```

**Step 3: Run the install command confirmed in Step 1**

**Step 4: Verify**

Run `/sdlc-pipeline` with no arguments in a Claude Code session in some
other project. Expected: it asks for the missing verb/argument (per Task
11's fallback instruction) rather than erroring "command not found."

**Step 5: Commit** (only if a marketplace.json was added)

```bash
git add .claude-plugin/marketplace.json
git commit -m "Add local dev marketplace manifest for plugin install"
```

---

### Task 13: End-to-end verification against a scratch project

There is no automated test suite for this plugin -- verification is
running the real flow once against a throwaway project and confirming the
observed behavior matches the design at each step.

**Step 1: Create a scratch project**

```bash
mkdir -p /tmp/sdlc-pipeline-smoketest && cd /tmp/sdlc-pipeline-smoketest && git init -q
```

**Step 2: Write a tiny PRD**

Create `prd.md`:
```markdown
# Feature: Add a `/health` endpoint

Add a simple HTTP endpoint that returns `{"status": "ok"}` with a 200
status code, for uptime monitoring. No auth required. No UI.
```

**Step 3: Start a run**

In a Claude Code session inside `/tmp/sdlc-pipeline-smoketest`, run:
```
/sdlc-pipeline start prd.md
```
Expected: a `.sdlc/runs/<run-id>/state.json` is created; the `pm` agent
runs `sharpen_prd`; given the PRD's triviality, the PM's proposed roster
should exclude `ux_designer` at minimum. The run proceeds to `write_plan`
and stops at `gate_plan` with an `AskUserQuestion` summarizing the PRD and
plan.

**Step 4: Approve the first gate, walk the milestone loop**

Approve `gate_plan`. Since the roster excluded `ux_designer`,
`gate_designs` should either not block on a UX artifact or should say so
explicitly (per Task 9 step 4's special case) -- confirm which, and that
it matches what's written in `SKILL.md`; fix the skill file if the
observed behavior and the written instructions disagree. Approve through
to the milestone loop and let it run to `gate_rollout`.

**Step 5: Exercise a replan**

Before approving `gate_rollout`, run:
```
/sdlc-pipeline replan <run-id> "too much ceremony for a one-endpoint change, drop the QA stage"
```
Expected: a diff is presented dropping `qa_tirekick` for remaining
milestones (there should be none left, so this may be a no-op diff --
if so, that's fine, it demonstrates the "no pending stages to change"
case cleanly; note this in your verification notes). Confirm
`replan_history` in `state.json` gets an entry either way.

**Step 6: Confirm resume works from a fresh state**

Note the `run_id`. Simulate a new session by re-reading
`.sdlc/runs/<run-id>/state.json` cold (don't rely on this session's
memory) and running `/sdlc-pipeline resume <run-id>`. Expected: it picks
up exactly where the state file says, not where this conversation
happens to remember.

**Step 7: Finish the run**

Approve remaining gates through `gate_next_phase`; choose to end rather
than start a phase 2. Expected: `status: "complete"` (or `"stopped"` if
you chose to end at `gate_next_phase` -- confirm which the skill actually
writes, and make Task 9's wording consistent if it's ambiguous).

**Step 8: Record findings and fix any drift**

If any observed behavior didn't match what `SKILL.md` says (this is
likely on a first pass -- treat the smoke test as the source of truth over
the plan's prose, since the prose was written before ever being run),
edit `skills/orchestrate/SKILL.md` to match reality, re-run the affected
step, and confirm.

**Step 9: Commit any fixes**

```bash
git add skills/orchestrate/SKILL.md
git commit -m "Fix orchestrate skill based on end-to-end smoke test"
```

---

### Task 13b: Verify the repo-link intake mode

The smoke test above only exercises Mode A (a PRD file). Mode B (repo
link, no brief) needs its own quick check since it's a different code
path in `start` and a different section of the PM's prompt.

**Step 1: Create a tiny scratch "product" repo**

```bash
mkdir -p /tmp/sdlc-pipeline-repotest && cd /tmp/sdlc-pipeline-repotest && git init -q
```
Add one trivial real file, e.g. a `server.py` with a single `/ping` route
returning `"pong"`, and commit it. This stands in for "an existing app" --
it should be a real, working, tiny piece of software, not a stub file.

**Step 2: Start a run pointing at the repo, no PRD**

```
/sdlc-pipeline start /tmp/sdlc-pipeline-repotest
```
Expected: `source.type` in `state.json` is `"repo"`, `checkout_path` is
set to that path (local path case, no clone). The PM agent recons the
checkout and produces a PRD with a clearly labeled current-state summary
(citing `server.py`) and a separate, clearly labeled expanded-vision
section proposing real improvements -- not a document where proposed
features read as already existing.

**Step 3: Confirm the URL-clone path works too**

Push the scratch repo to a location you can clone from (or use any small
public repo you're comfortable cloning), then run
`/sdlc-pipeline start <git-url>` in a fresh scratch directory. Expected: a
shallow clone appears under `.sdlc/runs/<run-id>/checkout/`, and the PM
stage proceeds the same as Step 2.

**Step 4: Fix any drift, then clean up**

Same as Task 13 Step 8-9: fix `SKILL.md` or `agents/pm.md` if observed
behavior doesn't match what's written, commit the fix, then
`rm -rf /tmp/sdlc-pipeline-repotest` (and any second scratch dir from
Step 3).

---

### Task 14: Clean up and final commit

**Step 1:** Remove the scratch project: `rm -rf /tmp/sdlc-pipeline-smoketest`

**Step 2:** Review the full diff one last time: `git log --oneline` and
`git diff main -- README.md .claude-plugin` (or equivalent) to confirm
nothing stray got committed.

**Step 3:** No further commit needed if Task 13 already committed
everything -- this task is a final sanity pass, not new content.
