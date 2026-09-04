# sdlc-pipeline

A Claude Code plugin that runs a full SDLC loop from a PRD or a bare repo
link: intake and roster assembly, system design, UX, milestone-by-milestone
implementation with review/QA loops, and CEO approval gates -- with the
ability to replan the pipeline at any point if it's too complex or taking
too long.

## Usage

- `/sdlc-pipeline start <path-to-prd>` -- begin a new run from a PRD/brief
- `/sdlc-pipeline start <repo-url-or-path>` -- begin a new run from an
  existing repo with no brief; the PM recons it and drafts a PRD from
  scratch, current-state summary plus a proposed expansion
- `/sdlc-pipeline resume <run-id>` -- continue a paused run
- `/sdlc-pipeline replan <run-id> ["reason"]` -- reshape the remaining
  stages/roster

See `docs/plans/2026-09-03-sdlc-pipeline-design.md` for the full design.
