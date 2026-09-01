# Sovereign AI Workbench

**Your Data. Your Models. Your Control.**

A self-hosted, air-gapped agentic AI workbench for confidential industrial
knowledge work — built for Smart India Hackathon 2026.

| | |
|---|---|
| **Problem Statement** | SIH26117 — Sovereign On-Premise Agentic AI Workbench using Open-Weight Multimodal LLMs for Confidential Industrial Work |
| **Theme** | Smart Automation |
| **Category** | Software |
| **Team** | Metamorphosis |

## The Problem

Refineries, PSUs, defence-linked manufacturing units, and government offices
generate large volumes of sensitive knowledge work — approval notes, P&IDs,
scanned inspection reports, internal correspondence — that can't be sent to
cloud AI tools. Today, that work either happens manually or people quietly
paste confidential material into public tools anyway. This project is a
locally-run alternative: multiple open-weight models, task-aware routing,
agentic tool use, and multimodal document understanding, all running inside
the organization's own infrastructure with zero external calls.

## Key Capabilities

- **Code Execution** — write, run, and debug code in a secure sandbox
- **Document Generation** — Word, Excel, PPT, PDF from structured output
- **OCR & Vision** — extract text, tables, and inspect scanned drawings/images
- **RAG Search** — semantic + keyword search over organizational SOPs/manuals
- **Workflow Automation** — end-to-end job execution with full audit trail
- **Governance & Security** — RBAC, audit logs, egress monitoring, air-gap enforcement

## Architecture

Client → Gateway (Auth/RBAC) → Agent Orchestrator (plan → route → act →
observe → deliver) → Model Router → local Model Pool (vLLM/Ollama) + Tool
Layer (sandbox, doc generator, OCR/vision, RAG retriever) → Data Layer
(vector DB, object store, audit log) — entirely inside an air-gapped network
boundary with a live egress monitor as proof.

Full design rationale, tech stack, model registry, and end-to-end demo flow:
see [`docs/sovereign-ai-workbench-design.md`](docs/sovereign-ai-workbench-design.md).

## Tech Stack

Python · FastAPI · React · LangGraph · PostgreSQL · vLLM · Ollama · Qdrant ·
PaddleOCR · Docker

## Repo Structure

```
models/         Model registry + serving config
orchestrator/   Agent loop, planner, task router
gateway/        Auth/RBAC, request logging
tools/          Sandbox exec, doc generator, OCR/vision, RAG retriever
data/           Vector store, object store, audit log
frontend/       Chat + task dashboard
interfaces/     Shared contracts — read before building against another module
demo/           Sample data and demo script
docs/           Design doc and reference material
```

## Getting Started

Setup is not yet code-runnable end-to-end — see
[`SETUP.md`](SETUP.md) for the repo/CI setup sequence and per-agent
(Claude Code / Antigravity) configuration steps. Run instructions will be
added here as modules land.

## Contributing

Working in this repo with an AI coding agent (Claude Code, Antigravity, or
otherwise)? Read [`CONTRIBUTING.md`](CONTRIBUTING.md) for branching, PR, and
task-claiming conventions, and [`AGENTS.md`](AGENTS.md) /
[`CLAUDE.md`](CLAUDE.md) for the shared agent context and module ownership
map — both are loaded automatically by Antigravity and Claude Code
respectively at session start.
