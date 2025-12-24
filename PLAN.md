# Project Plan – Local NotebookLM Clone

This plan follows a strict **Agile WBS (Work Breakdown Structure)**.
- **Epics (Phases)**: Major milestones delivering distinct value.
- **Features**: Specific functional capabilities.
- **Tickets**: Atomic, implementable tasks.
- **Exit Criteria**: strict definitions of done, including automated tests.

---

## Epic 0: Initialization & Scaffolding
**Goal**: Establish a robust "Factory" for development. We don't build the car yet; we build the assembly line.
**Timeline**: Week 1 (Days 1-3)

### Feature 0.1: Monorepo & Configuration
- **Ticket 0.1.1**: Initialize Git repo, `.gitignore` (Python/Node), and directory structure (`/backend`, `/frontend`, `/data`).
- **Ticket 0.1.2**: Backend Setup – Initialize Python environment (Poetry/venv), install FastAPI, Uvicorn. Create `GET /health` endpoint.
- **Ticket 0.1.3**: Frontend Setup – Initialize React + TypeScript + Vite. Ensure it builds and runs.
- **Ticket 0.1.4**: Shared Type Safety – Setup `mypy` (backend) and `eslint` (frontend).

### Feature 0.2: Automated Testing Harness
- **Ticket 0.2.1**: Configure `pytest` for backend. Write one dummy test to ensure discovery works.
- **Ticket 0.2.2**: Configure `vitest` for frontend. Write one dummy component test.
- **Ticket 0.2.3**: Create a `run_tests.sh` script in root that runs both suites and fails if either fails.

### Acceptance Criteria & Tests
- [ ] Command `sh run_tests.sh` executes successfully on a fresh clone.
- [ ] Backend is running on localhost:8000 returning `{"status": "ok"}`.
- [ ] Frontend is running on localhost:5173 rendering "NotebookLM Local".

---

## Epic 1: The "Walking Skeleton" (MVP)
**Goal**: End-to-end data flow. A user can create a notebook, upload a raw text file, and get an answer based on it.
**Timeline**: Week 1 (Days 4-7)

### Feature 1.1: Notebook Management (Backend & Storage)
- **Ticket 1.1.1**: Design `Notebook` model (Pydantic).
- **Ticket 1.1.2**: Implement File-System storage (Create folder `data/notebooks/{id}`).
- **Ticket 1.1.3**: API Endpoints: `POST /notebooks`, `GET /notebooks`, `GET /notebooks/{id}`.
- **Test**: Unit tests for Notebook CRUD operations.

### Feature 1.2: Simple Ingestion & Vector Store
- **Ticket 1.2.1**: Integrate `ChromaDB` (or FAISS).
- **Ticket 1.2.2**: Implement `TextLoader` for `.txt` and `.md` files.
- **Ticket 1.2.3**: Implement simple chunking (FixedSizeSplitter).
- **Ticket 1.2.4**: Create Ingestion Pipeline: Upload -> Chunk -> Embed (Gemini) -> Store.
- **Test**: Integration test: Upload a file containing specific string "YellowSubmarine", query vector store for it, assert retrieval.

### Feature 1.3: Basic Chat & RAG
- **Ticket 1.3.1**: Create `GeminiService` wrapper for generation.
- **Ticket 1.3.2**: Implement `/chat` endpoint receiving `query` and `notebook_id`.
- **Ticket 1.3.3**: Simple RAG Logic: Retrieve top-k chunks -> Stuff Prompt -> Generate.

### Feature 1.4: Frontend Integration
- **Ticket 1.4.1**: UI for "Create Notebook".
- **Ticket 1.4.2**: UI for "Upload File" within a notebook.
- **Ticket 1.4.3**: Simple Chat Interface (Input + Message History).

### Acceptance Criteria & Tests
- [ ] **Automated Test**: `test_end_to_end_rag.py` – Uploads a text file with a known secret fact. Asks question. Asserts answer contains the fact.
- [ ] UI allows creating a notebook and chatting with a `.txt` file.

---

## Epic 2: Unstructured Data Mastery
**Goal**: Handle real-world documents (PDF/Docs) with high fidelity and citations.
**Timeline**: Week 2

### Feature 2.1: Advanced Ingestion
- **Ticket 2.1.1**: Add `pypdf` or `pdfplumber` integration.
- **Ticket 2.1.2**: Add `python-docx` integration.
- **Ticket 2.1.3**: Metadata Extraction – Extract page numbers and filenames during parsing.

### Feature 2.2: Citation System
- **Ticket 2.2.1**: Update Vector Schema to store `source_id`, `start_index`, `page_number`.
- **Ticket 2.2.2**: Update Prompt to require `[Source: X, Page: Y]` citations in answers.
- **Ticket 2.2.3**: Frontend: Render citations as clickable tooltips or footnotes.

### Acceptance Criteria & Tests
- [ ] **Automated Test**: Upload a multi-page PDF. Ask about specific detail on Page 3. Assert answer mentions "Page 3".
- [ ] System handles >50 page PDF without crashing.

---

## Epic 3: Structured Data Agent (Text-to-SQL)
**Goal**: Enable analytics. The system can "reason" about numbers, not just retrieve text.
**Timeline**: Week 3

### Feature 3.1: OLAP Engine (DuckDB)
- **Ticket 3.1.1**: Integrate DuckDB. Create one db file per notebook (`analytics.duckdb`).
- **Ticket 3.1.2**: CSV/Excel Ingestion – Auto-create table from file upload.
- **Ticket 3.1.3**: Schema Introspection Tool – `get_table_schema(table_name)`.

### Feature 3.2: SQL Agent Logic
- **Ticket 3.2.1**: Create `GenerateSQL` prompt (Schema + Question -> SQL).
- **Ticket 3.2.2**: Implement `ExecuteSQL` tool.
- **Ticket 3.2.3**: Implement "Retry/Repair" loop: If SQL fails, feed error back to LLM to fix.

### Acceptance Criteria & Tests
- [ ] **Automated Test**: Upload `sales.csv`. Query "What is the total revenue?". Assert response contains correct sum.
- [ ] **Automated Test**: Test bad SQL generation triggers repair loop (mock failure).

---

## Epic 4: Multimodal Vision (Images)
**Goal**: "See" charts and diagrams.
**Timeline**: Week 4

### Feature 4.1: Image Pipeline
- **Ticket 4.1.1**: Backend upload support for PNG/JPG.
- **Ticket 4.1.2**: Multi-modal Vector Store support (storing image embeddings OR captioning images).
- **Ticket 4.1.3**: Implementation Choice: Use Gemini Vision to generate dense captions -> Embed Captions (Simpler for RAG).

### Feature 4.2: Vision RAG
- **Ticket 4.2.1**: When retrieving image chunks, pass original image to Gemini Vision for final answer synthesis.
- **Ticket 4.2.2**: UI: Display image thumbnails in chat citations.

### Acceptance Criteria & Tests
- [ ] **Automated Test**: Upload graph image. Ask "What is the trend?". Assert answer describes trend correctly.

---

## Epic 5: The Agent Loop (ReAct Refactor)
**Goal**: Move from hardcoded pipelines to a dynamic reasoning engine.
**Timeline**: Week 5

### Feature 5.1: Router & Planner
- **Ticket 5.1.1**: Implement `ClassifyIntent` step (Is this SQL? RAG? Mixed?).
- **Ticket 5.1.2**: Create structured `Plan` object (List of steps).

### Feature 5.2: Tool Execution Engine
- **Ticket 5.2.1**: Standardize Tool Interface (`name`, `description`, `args_schema`, `run()`).
- **Ticket 5.2.2**: Wrap previous features as tools: `search_docs`, `query_database`, `read_image`.
- **Ticket 5.2.3**: Implement Main Loop: `Plan -> Execute Step -> Observe -> Refine -> Final Answer`.

### Acceptance Criteria & Tests
- [ ] **Automated Test**: Complex Query "Compare the revenue from sales.csv with the projections in outlook.pdf".
- [ ] Logs show: Tool(SQL) -> Tool(RAG) -> Final Synthesis.

---

## Epic 6: Polish & Hardening
**Goal**: Production-ready stability and UX.
**Timeline**: Week 6

### Feature 6.1: UX Improvements
- **Ticket 6.1.1**: Streaming responses (Server-Sent Events).
- **Ticket 6.1.2**: Loading states and "Thought Process" visibility (Show tool calls in UI).

### Feature 6.2: Optimization
- **Ticket 6.2.1**: Caching tool outputs.
- **Ticket 6.2.2**: Optimizing chunk sizes and retrieval settings.

### Acceptance Criteria & Tests
- [ ] System handles parallel requests.
- [ ] UI is responsive.
