# CONTRIBUTING.md

Workflow for a team where multiple people are each driving their own AI coding
agent (Claude Code, Antigravity, etc.) against the same repo. The goal: no
repeated work, no incompatible modules, a traceable history of what changed
and why.

## 1. Claim before you build

- All work starts as a GitHub Issue on the Projects board (`Backlog` → `In
  Progress` → `Review` → `Done`).
- Self-assign an issue and move it to `In Progress` **before** starting an
  agent session on it. This is the actual mechanism that stops two people's
  agents from building the same thing in parallel — the board, not the agent,
  is the source of truth for "who's doing what right now."
- Before kicking off a long agent run, post a one-line note in the team
  channel: what you're about to have your agent do, and which files it'll
  touch. Cheap insurance against overlap the board doesn't catch in real time.

## 2. Branching

- `main` is protected — no direct pushes.
- One branch per issue: `feat/<module>-<short-desc>` or `fix/<module>-<short-desc>`.
- If you're running Claude Code locally and want to parallelize your own
  agent sessions, use `git worktree` so each session gets an isolated working
  directory instead of fighting over the same checkout.

## 3. Pull requests

- Keep PRs small and scoped to one issue — large PRs are where redundant or
  incompatible work hides until it's expensive to unwind.
- CI (lint + tests) must pass before merge.
- At least one reviewer approves — ideally the owner of an adjacent module if
  the PR touches a shared interface (see `CLAUDE.md` for the ownership map).
- If the PR came out of an Antigravity session, attach the agent's
  Implementation Plan or walkthrough Artifact to the PR description — it's a
  faster review aid than reading the raw diff cold.

## 4. Interfaces change together, not silently

- Any change to a shared schema in `interfaces/` needs sign-off from an owner
  on each side of that interface before merge, not just the person who wrote it.
- If your task requires changing another module's interface, open the issue
  first and tag that module's owner — don't let an agent "helpfully" adjust
  someone else's contract to make its own code pass.

## 5. Commits and changelog

- Conventional Commits: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`, `chore:`.
- Update `CHANGELOG.md` as part of the PR, not after the fact — one line,
  plain language, what changed and why.

## 6. Bugs

- GitHub Issues, bug template, labeled by module.
- Reproduction steps required before anyone (human or agent) attempts a fix.
- Link the fixing PR back to the bug issue so the trail is complete.

## Module ownership

See the table in `CLAUDE.md` — same map, same rule: don't edit outside your
lane without opening an issue first.
