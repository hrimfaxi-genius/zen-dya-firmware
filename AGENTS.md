# AGENTS.md

This file is read automatically by Codex CLI (and similar coding agents) when
working in this repository. Human-facing project docs are in `README.md`;
read that first for build/test/coding-standard details.

## Division of labor in this project

- **Claude** (in chat / Cowork) does investigation, architecture decisions,
  and writes design docs under `docs/design/*.md`.
- **Codex** (this agent, run from the terminal) implements against those
  design docs.

## How to work from a design doc

1. Read the target file under `docs/design/`. Its "Owner decisions (do not
   re-litigate)" section (or equivalent) is authoritative — implement it as
   written rather than re-deriving requirements from scratch.
2. If something in the doc turns out to be wrong or unworkable once you're in
   the code, don't silently override it: note the deviation and why, either
   as a comment at the top of the doc or by reporting it back, so the design
   can be corrected upstream instead of drifting undocumented.
3. Follow the existing repo conventions:
   - ZMK module changes: prefer touching the ZMK module first; the Web UI
     (`web/`) can be understood later if the task also needs it.
   - Always run tests and lint after changes (`python -m unittest` for the
     ZMK module, `cd web && npm test` / `npm run lint` for the Web UI).
   - Write tests for new behavior, not just the implementation.
4. When implementation is complete and tests pass, update the design doc's
   `Status:` line (e.g. `design` → `implemented`) so it's clear at a glance
   what's still pending versus done.

## Small, well-specified tasks

For small fixes that don't need a design doc (typos, small Kconfig tweaks,
CI fixes), just implement directly — no need to manufacture a doc for it.

## See also

- `.github/agents/maintainer.agent.md` and
  `.github/agents/new-feature-developer.agent.md` — role-specific guidance
  for GitHub Copilot's coding agent; the same principles apply here.
