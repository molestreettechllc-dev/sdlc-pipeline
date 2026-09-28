# Running Foreman outside Claude Code

**The short, honest version:** the *content* here — six role prompts, a
state-machine spec, a JSON schema for run state — is plain markdown and
JSON, not Claude-Code-specific in what it says. But the thing that makes a
run *autonomous* (one command kicks off PM → architect → UX → engineer →
reviewer → QA → PM again, with no human re-invoking each step) is Claude
Code's own plugin system: its skills, its subagent dispatch (the `Agent`
tool), and its `/slash-command` mechanism. No other harness we know of has
an exact equivalent of all three, so outside Claude Code, **you — or a
script you write — play the part of the orchestrator.** Each stage still
runs as a normal single-agent session; you're the one deciding "what's
next" and starting it, instead of the plugin deciding for you.

That's still genuinely useful — you get six carefully-written role prompts
and a state machine that's been exercised on real multi-milestone builds —
just not a one-command install.

## What's portable, file by file

| File | What it is | How to use it elsewhere |
|---|---|---|
| [`agents/pm.md`](../agents/pm.md), [`architect.md`](../agents/architect.md), [`ux_designer.md`](../agents/ux_designer.md), [`senior_engineer.md`](../agents/senior_engineer.md), [`code_reviewer.md`](../agents/code_reviewer.md), [`qa_engineer.md`](../agents/qa_engineer.md) | One role's full system prompt + stage instructions | Paste the body (skip the Claude-Code-specific YAML frontmatter) as your other tool's system/instructions for that one session. |
| [`skills/orchestrate/SKILL.md`](../skills/orchestrate/SKILL.md) | The whole state machine: stage order, gates, milestone loop, replans, escalations | Your reference for *when* to run which role and how to read/update `state.json` between stages — you execute this logic yourself, turn by turn. |
| `.sdlc/runs/<run-id>/state.json` + `artifacts/` | The run's entire memory | Same schema, same files, whatever harness wrote them — this is what makes resuming across tools/sessions possible at all. |

## OpenAI Codex CLI

Codex supports project-level `AGENTS.md` instructions and custom prompts
(markdown files Codex turns into slash commands — check `codex --help` or
Codex's own docs for the current directory and placeholder syntax, it has
moved between versions). There's no built-in equivalent of Claude Code's
subagent dispatch, so this is a manual loop:

1. For each role, copy that role's `agents/<role>.md` body (drop the
   frontmatter) into Codex's custom-prompts location as its own file —
   e.g. a prompt named `sdlc-pm`, `sdlc-architect`, and so on for all six.
2. Keep `.sdlc/runs/<run-id>/state.json` and `artifacts/` in your repo,
   same layout as above — this is what lets Codex sessions hand off to
   each other (and to a Claude Code session, if you switch tools mid-run).
3. To run one stage: open a Codex session in the project, invoke that
   stage's role prompt, and tell it plainly which run to act on and what
   to read first, e.g.:

   > Use the `sdlc-architect` prompt. Run id `2026-01-15-checkout`. Read
   > `.sdlc/runs/2026-01-15-checkout/state.json` and
   > `artifacts/01_sharpen_prd.html`, then do the `write_plan` stage per
   > `skills/orchestrate/SKILL.md`'s "Running a role stage" section.
   > Write the result to `artifacts/02_write_plan.md`, and update
   > `state.json` yourself: mark `write_plan` `done` with that artifact
   > path.

4. Read the result yourself, and when it's a gate stage (`gate_plan`,
   `gate_designs`, `gate_rollout`, `gate_flag`, `gate_next_phase`), you
   *are* the CEO here in the most literal sense — decide Approve / Request
   changes / Simplify / Reject yourself, per the same section of
   `SKILL.md`, and start the next Codex session accordingly.
5. For the milestone loop specifically, this means one Codex session per
   `implement`, one per `code_review`, one per `qa_tirekick`, run in that
   order by hand, exactly as `SKILL.md`'s "Milestone loop" section
   describes the automated version doing it.

A short wrapper script that reads `state.json`, figures out the next
pending stage, and shells out to `codex exec <prompt> "<context>"` would
close most of this gap if you want to script it — that's a genuinely good
first PR if you build one (see [`CONTRIBUTING.md`](../CONTRIBUTING.md)).

## Any other agent (Cursor, Aider, Cline, Windsurf, a plain chat session, …)

Same recipe, less tooling to lean on:

1. Pick the stage you want to run next (consult `SKILL.md`'s stage catalog
   and whatever `state.json` currently shows as `pending`).
2. Give your tool that role's `agents/<role>.md` body as its instructions
   for this session — as a system prompt, a pinned message, whatever your
   tool calls "give me a persona for this conversation."
3. Tell it explicitly which artifacts to read first and where to write its
   output — these files don't rely on any tool-specific mechanism, they're
   just paths in your repo.
4. Update `state.json` yourself afterward (or have the session do it, if
   it can write files) to mark the stage `done` with the artifact's path,
   so the next session — in this tool or a different one — can resume
   correctly.
5. At each gate, you decide the outcome yourself and note it in
   `state.json`'s `replan_history` or move the next stage to `pending`,
   matching what `SKILL.md` says the automated version would do.

The state file format is deliberately boring JSON for exactly this reason
— it doesn't assume anything about what wrote it or what reads it next.

## If you build real automation for this

If you write a wrapper — for Codex, for a generic CLI loop, for anything
else — that plays the orchestrator role well enough that a run genuinely
completes unattended, that's exactly the kind of PR described in
[`CONTRIBUTING.md`](../CONTRIBUTING.md). Cross-harness parity is a real
gap, not a "nice to have."
