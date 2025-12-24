# Local NotebookLM – Multimodal Knowledge Workspace (Gemini-powered)

## Overview

This project is a **local-first, Gemini-powered NotebookLM-style system** that allows knowledge workers to query, analyze, and synthesize insights across **heterogeneous data sources**:

- Structured data (CSV, XLS, DuckDB tables) via **OLAP + Text-to-SQL**
- Unstructured text (PDFs, DOCX, Markdown, plain text) via **advanced RAG**
- Images via **OCR + Vision models**
- Video via **ASR + timestamped transcript RAG**

The system is designed as a **true agentic application**, not a single-shot LLM wrapper. It uses an explicit **ReAct-style agent loop** to:
- Plan tool usage
- Route queries across modalities
- Execute, observe, repair, and synthesize answers
- Provide citations and provenance

This is intentionally scoped as a **developer-grade side project** that demonstrates modern AI engineering patterns end-to-end.

---

## Core Goals

- Notebook-based workspace abstraction (like NotebookLM)
- Local-first storage of user data
- Explicit separation of:
  - Structured analytics (OLAP)
  - Unstructured retrieval (RAG)
  - Multimodal understanding (image/video)
- Clear, inspectable agent loop with tool calls
- Incremental build-out with working software at every phase

---

## Non-Goals (Initial Versions)

- Full cloud SaaS deployment
- Real-time collaboration
- Enterprise auth / RBAC
- Perfect schema inference or automated data modeling

---

## High-Level Architecture

### Logical Components

1. **Frontend**
   - Notebook selection
   - File ingestion (local)
   - Chat interface
   - Answer rendering with citations
   - Optional data preview tabs (tables, transcripts)

2. **Backend API**
   - FastAPI-based service
   - Agent loop orchestrator
   - Tool registry and executors
   - Ingestion pipelines
   - Query routing

3. **Agent Layer**
   - Gemini-powered planner
   - Tool execution loop (ReAct-style)
   - Self-correction and retry
   - Final synthesis with provenance

4. **Storage**
   - Metadata DB (SQLite or Postgres)
   - OLAP DB (DuckDB)
   - Vector store (FAISS / Chroma)
   - Raw local files

---

## Technology Stack (Recommended)

### Backend
- Python 3.11+
- FastAPI
- Pydantic
- DuckDB
- FAISS or Chroma
- SQLite (metadata)
- ffmpeg (video/audio)
- Whisper (local ASR, optional)

### Frontend
- React + TypeScript
- Vite
- Simple component library (MUI / Radix optional)

### AI / ML
- Gemini Pro (planning, synthesis, text-to-SQL)
- Gemini Flash (routing, lightweight steps)
- Gemini Embeddings
- Gemini Vision (image understanding)

---

## Notebook Abstraction

A **Notebook** is the unit of isolation and context.

Each notebook contains:
- Raw files
- Parsed documents
- OLAP tables
- Vector index
- Metadata

Suggested on-disk layout:

data/
notebooks/
<notebook_id>/
raw/
metadata.db
analytics.duckdb
vectors/

---

## Query Capabilities

The system supports:

- “What does this document say about X?”
- “Compare revenue by region over time”
- “What was decided in the meeting at 14:30 in the video?”
- “Show me a chart of customer growth and explain anomalies”
- Mixed queries spanning **documents + numbers + transcripts**

---

## Agent Design (Conceptual)

The agent executes an explicit loop:

1. Inspect notebook context
2. Classify query intent:
   - structured
   - unstructured
   - multimodal
   - mixed
3. Generate a plan (tool calls)
4. Execute tools step-by-step
5. Repair on failure
6. Synthesize final answer with citations

This loop is observable and debuggable.

---

## Tooling Primitives (Conceptual)

- `GetNotebookContext`
- `DescribeSchema`
- `ExecuteSQL`
- `RAGSearch`
- `SummarizeChunks`
- `RunOCR`
- `TranscribeVideo`

Each tool is deterministic and testable.

---

## Repository Structure (Proposed)

backend/
app/
api/
agent/
tools/
ingestion/
storage/
schemas/
main.py

frontend/
src/
components/
pages/
services/

data/
scripts/
PLAN.md
README.md

---

## Getting Started (Developer)

1. Clone repo
2. Install backend dependencies
3. Run backend (`uvicorn`)
4. Run frontend (`npm run dev`)
5. Create a notebook
6. Upload documents
7. Ask questions

Details are intentionally deferred to implementation phases.

---

## Evaluation Criteria

- Agent plans are multi-step and visible
- Text-to-SQL works on non-trivial queries
- RAG answers cite sources correctly
- Video answers include timestamps
- System remains usable with large documents

---

## License

MIT (recommended for a side project)
