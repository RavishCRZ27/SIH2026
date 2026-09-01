# AGENTS.md — Shared Agent Context

Every agent session (Claude Code, Antigravity, or otherwise) working in this repo
should read this file first. Antigravity loads `AGENTS.md` (this file)
automatically at the project root at session start. Claude Code loads
`CLAUDE.md` instead — keep that file identical to this one. If you edit shared
conventions here, mirror the change into `CLAUDE.md` in the same commit.

## Project

Sovereign On-Premise Agentic AI Workbench — SIH26117, Team Metamorphosis.
Full architecture, tech stack, and rationale: see `docs/sovereign-ai-workbench-design.md`
in this repo. Read that before touching anything — it explains *why* the layers
are split the way they are, not just what they're called.

## Module ownership — do not edit outside your lane without opening an issue first

| Owner | Path(s) | Responsibility |
|---|---|---|
| Ravish | `models/`, `orchestrator/`, `tools/rag_retriever.py` | Model registry, task routing, agent loop, RAG retrieval |
| Sachin | `gateway/`, `tools/sandbox_exec.py`, `data/` | Auth/RBAC, code sandbox, vector store + audit log |
| Harshit | `tools/ocr_vision.py`, `tools/doc_generator.py` | OCR/vision extraction, Word/PPT/Excel generation |
| Aditya | `frontend/`, `demo/` | Dashboard UI, demo data/script |

If a task requires touching a file outside your listed paths, open a GitHub issue
tagging that path's owner instead of editing it directly. This is the rule that
actually prevents two agents from silently duplicating or conflicting on the
same file.

## Contracts — the thing that keeps everyone's code compatible

All cross-module interfaces (tool schemas, request/response shapes) live in
`interfaces/`. This is the one directory everyone can read but only merges to
via a PR reviewed by at least one owner from each side of the interface.
Build against what's in `interfaces/`, not against another module's internal
implementation — if the internal implementation isn't done yet, mock the
interface and keep going.

## Conventions

- Python: PEP8, formatted with `black`, type-hinted where practical.
- Frontend: ESLint + Prettier defaults, functional components.
- Commit messages: Conventional Commits (`feat:`, `fix:`, `refactor:`, `docs:`, `test:`, `chore:`).
- Every module ships with unit tests against the shared interface before a PR is opened.

## Before starting any task

1. Check the GitHub Projects board — claim the issue, move it to "In Progress."
2. Confirm no one else has an open branch/PR touching the same path.
3. If your task changes a shared interface, flag it in the issue before writing code.

## Bugs

File as a GitHub Issue using the bug template, labeled with the affected module
(`models`, `orchestrator`, `gateway`, `tools-sandbox`, `tools-rag`, `data`,
`tools-ocr`, `tools-doc`, `frontend`). Don't fix a bug outside your module
silently — tag the owner even if you know the fix, so the change log stays
attributable.
