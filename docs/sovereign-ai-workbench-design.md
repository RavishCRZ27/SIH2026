# Sovereign On-Premise Agentic AI Workbench — System Design

SIH 2026 · Confidential industrial knowledge work, fully local, no external calls.

---

## 1. Architecture Layers

```
┌─────────────────────────────────────────────────────────────┐
│  CLIENT           Web UI (chat + task dashboard)             │
├─────────────────────────────────────────────────────────────┤
│  GATEWAY          Auth / RBAC / request logging              │
├─────────────────────────────────────────────────────────────┤
│  ORCHESTRATION    Agent loop: plan → route → call tool →     │
│                    observe → repeat → deliver                │
├─────────────────────────────────────────────────────────────┤
│  MODEL ROUTER     Task classifier → model registry lookup    │
├──────────────┬──────────────────────────────────────────────┤
│  MODEL POOL   │  TOOL LAYER                                  │
│  (vLLM/Ollama)│  sandbox exec · file I/O · doc gen ·         │
│  general/coder│  OCR+vision · RAG retriever                  │
│  /vision LLMs │                                              │
├──────────────┴──────────────────────────────────────────────┤
│  DATA LAYER       Vector DB · object store · audit log DB    │
├─────────────────────────────────────────────────────────────┤
│  ISOLATION        No default route, egress monitor, air-gap  │
│                    enforcement at network + container level  │
└─────────────────────────────────────────────────────────────┘
```

See the companion `.mermaid` diagram for the wired-up component view.

---

## 2. Component Responsibilities

| Layer | Component | Responsibility |
|---|---|---|
| Client | Web UI | Chat interface + a task/job dashboard showing agent steps, tool calls, and generated artifacts as they're produced (not just a final chat bubble) |
| Gateway | Auth/RBAC | Per-user/role access to knowledge base scopes and tool permissions (e.g. who can trigger code execution) |
| Orchestration | Agent Orchestrator | Owns the plan → act → observe loop; decides when to call a tool vs. a model vs. stop |
| Orchestration | Model Router | Classifies each sub-task (code / summarize-document / vision / general-reasoning) and picks the right model from the registry |
| Model Pool | Serving runtime | Hosts multiple open-weight models concurrently, hot-swappable via config |
| Tool Layer | Sandbox executor | Runs generated code in an isolated container, returns stdout/stderr/artifacts |
| Tool Layer | Doc generator | Produces real Word/PPT/Excel deliverables from structured agent output |
| Tool Layer | OCR/Vision preprocessor | Converts scanned PDFs/photos/drawings into text + structured findings before they hit the LLM |
| Tool Layer | RAG retriever | Hybrid (keyword + dense) search over the org's SOPs/manuals/correspondence |
| Data | Vector DB, object store, audit DB | Embeddings, source documents, and a full log of every action taken (needed for both trust and your "no external calls" proof) |
| Isolation | Egress monitor | Continuously watches for outbound connections; this is your live demo proof, not just a claim |

---

## 3. Model Routing Design

Keep this dead simple for the demo — a classifier plus a static registry, not a learned router.

**Target hardware — three honest tiers, not one aspirational number:**

| Tier | Hardware | Engine | Models resident |
|---|---|---|---|
| Internal round (now) | 6GB laptop GPU | Ollama (llama.cpp, GGUF, Q4) | One at a time, swapped on demand |
| Grand Finale target | 24GB-class GPU (e.g. RTX 4090/3090) | vLLM | Two to three, concurrent |
| Production | Org GPU server | vLLM, multi-GPU | Larger checkpoints, on-demand load-evict |

The same router and registry work at every tier — only the config file changes which checkpoint each task type points to.

**Model registry (config-driven, so adding a model = editing this file):**

```yaml
models:
  # --- Internal-round tier: 6GB laptop GPU, one model resident at a time ---
  general_reasoning:
    engine: ollama
    checkpoint: qwen2.5:3b-instruct-q4_K_M
    resident: on-demand
  coder:
    engine: ollama
    checkpoint: qwen2.5-coder:3b-instruct-q4_K_M
    resident: on-demand
  # No dedicated VLM at this tier — PaddleOCR (CPU) extracts text/tables,
  # general_reasoning model reasons over the extracted text instead.

  # --- Grand Finale tier: 24GB GPU, concurrent resident ---
  general_reasoning_finale:
    engine: vllm
    checkpoint: Qwen3-8B-Instruct
    quantization: awq-int4
  coder_finale:
    engine: vllm
    checkpoint: Qwen2.5-Coder-7B-Instruct
    quantization: awq-int4
  vision_finale:
    engine: vllm
    checkpoint: Qwen2.5-VL-7B-Instruct
    quantization: awq-int4

  # --- Production tier: org GPU server, on-demand load-evict ---
  general_reasoning_large:
    checkpoint: Qwen3.6-27B-Instruct
    resident: on-demand
  coder_large:
    checkpoint: Qwen3-Coder-30B
    resident: on-demand
  vision_large:
    checkpoint: Gemma-4-VL-26B   # or a larger Qwen-VL variant
    resident: on-demand
```

**Routing logic (rules first, LLM fallback only for ambiguous cases):**

```python
def route(task):
    if task.has_image or task.file_type in {"pdf_scanned", "jpg", "png"}:
        return "vision"
    if task.intent in {"write_code", "debug", "run_script"}:
        return "coder"
    if task.intent == "ambiguous":
        return classify_with_small_llm(task)   # cheap arbitration step
    return "general_reasoning"
```

This is enough to satisfy "auto-selection across at least two task types" convincingly, and it's honest about what open models can reliably do today — don't over-engineer a learned router you won't have time to validate.

---

## 4. Agent Orchestration Design

Use a ReAct-style loop (LangGraph, or hand-rolled — hand-rolled is easier to explain to judges line by line):

```
1. Receive task → decompose into steps (planner call to general_reasoning model)
2. For each step:
     a. route() → pick model
     b. if step needs a tool: call tool, capture result
     c. feed result back into context
     d. check: is task complete? if not, loop
3. On completion: assemble deliverable via doc generator
4. Log full trace to audit DB
```

**Tool interface** — give every tool the same schema so the orchestrator treats them uniformly:

```json
{
  "name": "ocr_extract",
  "description": "Extract text and layout from a scanned document",
  "parameters": {"file_path": "string"},
  "returns": {"text": "string", "confidence": "float", "regions": "array"}
}
```

---

## 5. Tech Stack

| Concern | Choice | Why |
|---|---|---|
| Model serving | vLLM (primary), Ollama (fallback/simplicity) | Multi-model concurrent serving, OpenAI-compatible API |
| Agent framework | LangGraph or custom ReAct loop | Explicit state machine, easy to demo/explain |
| Sandbox | Docker + resource limits, or Jupyter kernel gateway | Isolation without heavy infra |
| OCR | PaddleOCR (or Tesseract) | Strong on printed text and tables |
| Vision/document understanding | Qwen2.5-VL / Gemma-4 VL | Handles drawings, layout, degraded scans better than OCR alone |
| Embeddings | bge-m3 (local) | Strong open multilingual embedding model, no external API |
| Vector DB | Qdrant or Chroma | Simple to self-host, good hybrid search support |
| Retrieval | Hybrid BM25 + dense (you've done this shape before in CivicSync) | Better recall on SOP/manual jargon than dense-only |
| Relational/audit store | PostgreSQL | Job history, tool-call audit trail, RBAC |
| Backend API | FastAPI | Matches your existing stack, async-friendly |
| Doc generation | python-docx, python-pptx, openpyxl | Real deliverables, not just markdown |
| Frontend | React + a simple job/step visualizer | Show the agent's plan and tool calls live, not just final output |
| Code editor | Monaco Editor (the VS Code editor component, embedded in the React frontend) | Syntax highlighting + built-in diff view for showing what the agent generated/changed, without the overhead of a full IDE |
| Sandbox shell | bash | LLMs generate bash-native syntax by default; every Docker base image ships it; no interactive-shell quirks when scripts run non-interactively from the agent loop |
| Network isolation | iptables/firewall rules + Docker network with no default route | Enforces air-gap, not just claims it |
| Network proof | Custom middleware logging every attempted connection + iftop/Wireshark on screen during demo | This is your credibility moment — make it visible, not a slide |

---

## 6. Repository Structure

```
sovereign-ai-workbench/
├── README.md
├── docker-compose.yml              # spins up all services + models
├── models/
│   └── registry.yaml                # model configs (add new models here)
├── gateway/
│   ├── main.py                      # FastAPI entrypoint, auth/RBAC
│   └── middleware/egress_guard.py   # logs/blocks any outbound call
├── orchestrator/
│   ├── agent_loop.py                # plan → act → observe loop
│   ├── router.py                    # task classifier + registry lookup
│   └── planner.py                   # task decomposition prompts
├── tools/
│   ├── sandbox_exec.py
│   ├── doc_generator.py             # docx/pptx/xlsx builders
│   ├── ocr_vision.py
│   └── rag_retriever.py
├── data/
│   ├── ingestion/                   # SOP/manual loaders, chunkers
│   ├── vector_store/                # Qdrant client wrapper
│   └── audit_log/                   # Postgres models + queries
├── frontend/
│   └── src/                         # React chat + job dashboard
├── scripts/
│   ├── pull_models.sh               # one-time model download (pre-air-gap)
│   ├── verify_airgap.sh             # network monitor for the demo
│   └── seed_demo_data.py            # sample SOPs, scanned reports for judges
└── demo/
    ├── scanned_inspection_report.pdf
    ├── sample_codegen_task.md
    └── expected_outputs/
```

---

## 7. End-to-End Demo Flow (scanned inspection report → approval note)

1. User uploads scanned inspection report via Web UI
2. Router detects file type → sends to `vision` model path
3. OCR/vision tool extracts text + flags key findings (defects, measurements, recommendations)
4. Orchestrator calls `general_reasoning` model with extracted findings + RAG context pulled from relevant SOPs (e.g. inspection thresholds, approval templates)
5. Model drafts the approval note content, structured as sections
6. `doc_generator` tool renders it as an actual `.docx` with your org's template
7. Every step above is written to the audit log; the dashboard shows the plan and each tool call live
8. Egress monitor on screen shows zero outbound connections throughout

---

## 8. Suggested Build Order (fits your ~3-month window before finale)

1. **Weeks 1–3:** Model serving + router + RAG retriever (you have direct experience with this shape from CivicSync)
2. **Weeks 4–6:** Agent loop + doc generator, get the inspection-report → approval-note flow working end-to-end
3. **Weeks 7–8:** Sandbox code execution flow (second task type, proves auto-routing)
4. **Weeks 9–10:** OCR/vision hardening, network isolation + egress monitor
5. **Weeks 11–12:** Polish dashboard, prep demo script, stress-test the "no external calls" proof live

Build the RAG + routing + one full agentic flow first — that's your safety-net demo even if the more ambitious pieces (handwriting OCR, full sandbox reliability) don't fully land.

---

## 9. Why This Isn't Just "opencode + Ollama"

Judges will ask this. Be direct about it rather than dodging.

**Real overlap:** the coding-sandbox flow alone is not a differentiator — opencode (or Aider/similar) pointed at an Ollama-served model already gives a capable local coding agent with tool use and file edits. Don't lead the pitch with that piece.

**Where the actual differentiation is:**

- **Domain, not deployment.** opencode's world is a code repo and a terminal; the target users here (refinery engineers, PSU admin staff, inspectors) never touch a CLI. The scanned-report → approval-note flow, OCR/vision ingestion, and Word/PPT/Excel output are outside opencode's scope entirely — a different product surface, not a better version of the same one.
- **Task-aware routing across heterogeneous domains as an architectural feature.** Ollama serves one model per session, user-selected. This system's router picks vision vs. coder vs. general-reasoning per sub-task, automatically, within a single agent run.
- **Organizational grounding.** Neither tool has a first-class concept of "this org's SOPs/manuals" as a knowledge base tied to an audit log and RBAC.
- **Multi-user, single-server deployment.** opencode + Ollama is single-user/single-machine. This is a client-server product meant to serve many employees off one org GPU server with per-user access control.
- **Compliance-grade air-gap proof.** Both can run offline, but neither ships an audit trail or egress-monitoring layer a security officer would actually sign off on. That governance layer is the non-trivial engineering no assembly of existing tools provides for free.

**Pitch framing:** the value isn't "a better coding agent" — it's the integration and governance layer that turns a pile of open-weight models into something a regulated industrial org can deploy and trust.
