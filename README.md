# Knowledge Atlas – Multimodal Knowledge Workspace

A **local-first, Gemini-powered knowledge system** for querying, analyzing, and synthesizing insights across heterogeneous data sources.

## What This Demonstrates

This project is designed to showcase senior-level AI engineering:

- **System design**: Multi-component architecture with clear separation of concerns
- **Agent orchestration**: ReAct-style planning, tool execution, and self-repair
- **Multi-modal pipelines**: Unified interface across text, tables, and images
- **Evaluation-driven development**: Quantified metrics for RAG and SQL accuracy
- **Production patterns**: Structured logging, tracing, error handling

This is a working product, not a proof-of-concept.

---

## Core Capabilities

| Data Type | Approach | Example Query |
|-----------|----------|---------------|
| **Documents** (PDF, DOCX, TXT) | Chunked RAG with page-level citations | "What does the Q3 report say about margins?" |
| **Tables** (CSV, Excel) | Text-to-SQL via DuckDB | "What's the total revenue by region?" |
| **Images** (PNG, JPG) | Vision model analysis | "What trend does this chart show?" |

The system handles **mixed queries** that span multiple modalities:
> "Compare the revenue numbers in the spreadsheet with the projections mentioned in the strategy PDF."

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Frontend                                │
│         React + TypeScript • Chat UI • Citation Display         │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Backend API                              │
│                     FastAPI + Pydantic                          │
├─────────────────────────────────────────────────────────────────┤
│                        Agent Layer                              │
│         Planner → Executor → Synthesizer (ReAct loop)          │
├──────────────┬──────────────┬──────────────┬────────────────────┤
│  search_docs │ execute_sql  │analyze_image │    (more tools)    │
├──────────────┴──────────────┴──────────────┴────────────────────┤
│                         Storage                                 │
│    ChromaDB (vectors) • DuckDB (OLAP) • SQLite (metadata)      │
└─────────────────────────────────────────────────────────────────┘
```

### Why These Technology Choices

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Vector store | ChromaDB | Persistent, embedded, good Python API. FAISS is faster but lacks persistence out of the box. |
| OLAP | DuckDB | Embedded analytical DB, reads CSV/Parquet natively, SQL dialect is Postgres-compatible. |
| Metadata | SQLite | Zero-config, file-based, sufficient for single-user local app. |
| LLM | Gemini Pro/Flash | Strong reasoning, native vision, competitive pricing. Flash for lightweight routing. |
| Embeddings | Gemini Embeddings | Consistent with LLM provider, good quality. |

---

## Notebook Abstraction

A **Notebook** is the unit of isolation. Each notebook is a self-contained workspace with its own documents, tables, and vector index.

```
data/
└── notebooks/
    └── {notebook_id}/
        ├── raw/                 # Original uploaded files
        ├── metadata.json        # Notebook config and file registry
        ├── analytics.duckdb     # OLAP tables from CSV/Excel
        ├── vectors/             # ChromaDB persistent storage
        └── traces/              # Agent execution traces
```

---

## Agent Design

The agent executes an explicit **ReAct-style loop**:

1. **Plan**: Analyze query, determine required tools, generate execution plan
2. **Execute**: Run tools sequentially, collect observations
3. **Repair**: On tool failure, feed error to LLM, retry with corrected approach
4. **Synthesize**: Combine observations into final answer with citations

Every execution is traced and persisted for debugging and observability.

### Tool Interface

All tools implement a standard interface:

```python
class Tool(ABC):
    name: str
    description: str
    parameters_schema: dict  # JSON Schema
    
    async def execute(self, params: dict, context: ToolContext) -> ToolResult
```

**Initial tools:**
- `search_docs` – Vector similarity search over document chunks
- `describe_tables` – List OLAP tables and their schemas
- `generate_sql` – Convert natural language to SQL
- `execute_sql` – Run SQL against DuckDB
- `analyze_image` – Send image to Gemini Vision with question

---

## Success Metrics

The system includes an evaluation framework with quantified thresholds:

### RAG Quality
| Metric | Target | Measurement |
|--------|--------|-------------|
| Answer similarity | > 0.7 | Cosine similarity between generated and expected answer embeddings |
| Citation recall | > 80% | Percentage of expected source citations present in response |

### SQL Accuracy
| Metric | Target | Measurement |
|--------|--------|-------------|
| Execution success | > 90% | SQL executes without syntax/runtime errors |
| Result accuracy | > 85% | Query results match expected values |

Eval suites run against curated test datasets. Results are tracked over time to detect regressions.

---

## Repository Structure

```
backend/
├── app/
│   ├── api/           # FastAPI route handlers
│   ├── agent/         # Planner, executor, synthesizer
│   ├── tools/         # Tool implementations
│   ├── ingestion/     # Document loaders, chunkers
│   ├── storage/       # Vector store, OLAP, metadata
│   ├── services/      # LLM, embedding, vision wrappers
│   ├── schemas/       # Pydantic models
│   └── main.py
├── tests/
│   ├── fixtures/      # Test documents, CSVs, images
│   └── eval/          # Evaluation datasets
└── pyproject.toml

frontend/
├── src/
│   ├── components/    # Chat, Citations, FileUpload
│   ├── pages/         # NotebookList, NotebookDetail
│   └── services/      # API client
└── package.json

scripts/
├── test.sh            # Run all tests
├── eval_rag.py        # RAG evaluation suite
└── eval_sql.py        # SQL evaluation suite

data/                  # Runtime data (gitignored)
```

---

## Getting Started

```bash
# Clone and setup backend
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -e .
cp .env.example .env  # Add GEMINI_API_KEY

# Run backend
uvicorn app.main:app --reload

# In another terminal, setup and run frontend
cd frontend
npm install
npm run dev
```

Then:
1. Open `http://localhost:5173`
2. Create a notebook
3. Upload documents (PDF, TXT, CSV)
4. Ask questions

---

## Non-Goals (Current Scope)

- Cloud/SaaS deployment
- Multi-user collaboration
- Authentication / RBAC
- Video/audio transcription (deferred to future scope)

---

## Future Scope

- **Video ingestion**: ASR with Whisper, timestamped transcript RAG
- **Conversation memory**: Multi-turn context within chat sessions
- **Semantic caching**: Cache similar queries to reduce LLM calls
- **Deployment**: Docker compose, cloud deployment options

---

## License

MIT
