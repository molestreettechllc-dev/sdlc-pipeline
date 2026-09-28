# Contributing to Foreman

PRs are genuinely welcome — this started as a personal tool, not a
finished product, and the parts most likely to be wrong are exactly the
parts a second set of eyes (or a second team's real usage) catches fastest.

## Good first contributions

- **Run it on your own project and report what broke.** The single most
  useful contribution right now. Open an issue with the run's `state.json`
  and whichever artifact looked wrong.
- **Sharpen a role's prompt.** Each role is one file under
  [`agents/`](agents/) — if you've watched the PM, architect, reviewer, or
  QA agent make a bad call, the fix is almost always a clearer instruction
  in that file, not new code.
- **Tighten the orchestrator.** [`skills/orchestrate/SKILL.md`](skills/orchestrate/SKILL.md)
  is the whole state machine: stage sequencing, gates, the milestone loop,
  replans, escalations. Gaps here tend to show up as a run stalling or a
  gate asking the wrong question.
- **Docs.** [`docs/plans/`](docs/plans/) has the original design docs;
  [`docs/other-agents.md`](docs/other-agents.md) covers running this
  outside Claude Code. Both drift from the skill as it evolves — PRs that
  bring them back in sync are welcome even if they touch no behavior.

## Before you open a PR

- **Read the relevant `agents/*.md` or `SKILL.md` in full first.** These
  files are dense on purpose — every sentence is there because some run
  actually needed it. A change that looks like a simplification is often
  a regression that removes the instruction covering an edge case you
  haven't hit yet.
- **Test against a real run**, not just a read-through. Point
  `/sdlc-pipeline start` at a small real repo or brief and watch the stage
  your change touches actually execute. Paste the relevant bit of the
  resulting `state.json` or artifact into the PR description as evidence.
- **Keep the state schema stable** unless the PR is specifically about
  changing it. A lot of the value here is that `resume` works cold, in a
  brand-new session, months later — a schema change should say plainly
  what it does to a run's `state.json` that was created before the change.
- **Small, focused diffs.** One role's prompt, one section of the
  orchestrator, one doc — not a drive-by rewrite of everything you were
  looking at while you were in there.

## What this isn't looking for

- A rewrite into a different architecture (a custom server, a different
  state store, etc.) without discussing it in an issue first — the whole
  point is that this stays plain files on top of Claude Code's own
  primitives.
- Prompt changes that make a role more verbose without changing what it
  actually catches or decides. If you can't point to a run this would
  have improved, it's probably not worth the extra tokens every run pays.

## Getting help

Open an issue. If you're not sure whether something is a bug in a role's
judgment or a gap in the orchestrator's logic, say what you saw and let's
figure out which together.
