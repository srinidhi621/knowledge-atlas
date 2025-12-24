# Project Plan – Local NotebookLM Clone

This plan follows a strict **Agile WBS (Work Breakdown Structure)**.

- **Epics**: Major milestones delivering distinct, demonstrable value.
- **Features**: Specific functional capabilities within an epic.
- **Tickets**: Atomic, implementable tasks (1-4 hours of work).
- **Acceptance Criteria**: Quantified, automatable definitions of done.

## Guiding Principles

1. **Agent-First Architecture**: The agent loop is the spine. Tools are muscles attached to it. We build the spine first.
2. **Incremental Value**: Every epic produces a working, demonstrable system. No "big bang" integrations.
3. **Observable by Default**: Structured logging and tracing from Day 1. If you can't see it, you can't debug it.
4. **Tested at Every Layer**: Unit tests for logic, integration tests for pipelines, end-to-end tests for user flows.
5. **Evaluation-Driven**: We measure what matters. RAG quality and SQL accuracy have quantified metrics.

---

## Epic 0: Project Scaffolding

**Goal**: Establish the development environment, CI harness, and basic project structure. No features—just the "factory" that builds features.

**Deliverable**: A repo where `./scripts/test.sh` passes, backend serves health check, frontend renders.

---

### Feature 0.1: Repository Structure

**Ticket 0.1.1**: Initialize Git repository and directory structure
- Create `.gitignore` (Python, Node, IDE files, `.env`, `data/`)
- Create directory structure:
  ```
  backend/
    app/
      api/
      agent/
      tools/
      ingestion/
      storage/
      schemas/
      main.py
    tests/
    pyproject.toml
  frontend/
    src/
      components/
      pages/
      services/
    package.json
  scripts/
  data/
  ```
- **Exit Criteria**: `git status` shows clean repo with structure. No code yet.

**Ticket 0.1.2**: Backend environment setup
- Initialize Python 3.11+ environment with Poetry or `pyproject.toml`
- Install dependencies: `fastapi`, `uvicorn`, `pydantic`, `pytest`, `httpx`
- Create `backend/app/main.py` with `/health` endpoint returning `{"status": "ok"}`
- **Exit Criteria**: 
  - `cd backend && uvicorn app.main:app` starts server
  - `curl localhost:8000/health` returns `{"status": "ok"}`

**Ticket 0.1.3**: Frontend environment setup
- Initialize React + TypeScript + Vite project in `frontend/`
- Install dependencies: `react`, `react-dom`, `typescript`, `vite`
- Create minimal `App.tsx` rendering "Knowledge Atlas"
- **Exit Criteria**:
  - `cd frontend && npm run dev` starts dev server
  - Browser at `localhost:5173` shows "Knowledge Atlas"

**Ticket 0.1.4**: Type checking and linting
- Backend: Configure `mypy` with strict mode, `ruff` for linting
- Frontend: Configure `eslint` with TypeScript rules
- **Exit Criteria**:
  - `cd backend && mypy app/` passes with no errors
  - `cd frontend && npm run lint` passes with no errors

---

### Feature 0.2: Testing Harness

**Ticket 0.2.1**: Backend test configuration
- Configure `pytest` with `pytest-asyncio` for async tests
- Create `backend/tests/conftest.py` with test client fixture
- Create `backend/tests/test_health.py` with health endpoint test
- **Exit Criteria**: `cd backend && pytest` discovers and passes 1 test

**Ticket 0.2.2**: Frontend test configuration
- Configure `vitest` with React Testing Library
- Create `frontend/src/App.test.tsx` testing "Knowledge Atlas" renders
- **Exit Criteria**: `cd frontend && npm test` discovers and passes 1 test

**Ticket 0.2.3**: Unified test script
- Create `scripts/test.sh` that:
  - Runs backend tests
  - Runs frontend tests
  - Exits with failure if either fails
- **Exit Criteria**: `./scripts/test.sh` executes both test suites, returns 0 on success

---

### Feature 0.3: Logging and Configuration

**Ticket 0.3.1**: Structured logging setup
- Install `structlog` for backend
- Configure JSON logging with timestamp, level, message, and arbitrary context fields
- Create `backend/app/logging.py` with configured logger factory
- **Exit Criteria**: 
  - Log output is valid JSON
  - Logs include `timestamp`, `level`, `event` fields

**Ticket 0.3.2**: Configuration management
- Create `backend/app/config.py` using Pydantic Settings
- Support environment variables: `GEMINI_API_KEY`, `DATA_DIR`, `LOG_LEVEL`
- Create `.env.example` with placeholder values
- **Exit Criteria**:
  - App reads config from environment
  - Missing required config (like `GEMINI_API_KEY`) raises clear error at startup

---

### Epic 0 Acceptance Criteria

- [ ] `./scripts/test.sh` passes on fresh clone (after `pip install` and `npm install`)
- [ ] `curl localhost:8000/health` returns `{"status": "ok"}`
- [ ] `localhost:5173` renders "Knowledge Atlas"
- [ ] All linters pass with zero errors
- [ ] Logs are structured JSON

---

## Epic 1: Agent Skeleton with Minimal RAG

**Goal**: Build the core agent loop architecture with a single working tool. This establishes the pattern all future tools will follow.

**Deliverable**: User can create a notebook, upload a text file, and ask a question. The agent plans, executes the `search_docs` tool, and synthesizes an answer.

---

### Feature 1.1: Notebook Abstraction

**Ticket 1.1.1**: Notebook Pydantic models
- Create `backend/app/schemas/notebook.py`
- Define `NotebookCreate`, `Notebook`, `NotebookSummary` models
- Fields: `id` (UUID), `name`, `created_at`, `updated_at`, `file_count`
- **Exit Criteria**: Models serialize/deserialize correctly (unit test)

**Ticket 1.1.2**: Notebook filesystem storage
- Create `backend/app/storage/notebook_store.py`
- Implement `NotebookStore` class with methods:
  - `create(name: str) -> Notebook`
  - `get(notebook_id: str) -> Notebook | None`
  - `list() -> list[NotebookSummary]`
  - `delete(notebook_id: str) -> bool`
- Storage structure: `data/notebooks/{notebook_id}/` with `metadata.json`
- **Exit Criteria**: 
  - Unit test: Create notebook, retrieve it, list includes it, delete removes it
  - Filesystem contains expected structure after create

**Ticket 1.1.3**: Notebook API endpoints
- Create `backend/app/api/notebooks.py`
- Implement endpoints:
  - `POST /api/notebooks` → creates notebook, returns `Notebook`
  - `GET /api/notebooks` → returns `list[NotebookSummary]`
  - `GET /api/notebooks/{id}` → returns `Notebook` or 404
  - `DELETE /api/notebooks/{id}` → returns 204 or 404
- **Exit Criteria**:
  - Integration test: Full CRUD cycle via HTTP
  - 404 returned for non-existent notebook

**Depends on**: 1.1.1, 1.1.2

---

### Feature 1.2: Document Ingestion (Text Files)

**Ticket 1.2.1**: File upload endpoint
- Create `POST /api/notebooks/{id}/files` accepting multipart file upload
- Save raw file to `data/notebooks/{id}/raw/{filename}`
- Store file metadata in notebook's `metadata.json`
- **Exit Criteria**:
  - Integration test: Upload file, verify it exists on disk
  - Duplicate filename returns 409 Conflict

**Depends on**: 1.1.3

**Ticket 1.2.2**: Text file loader
- Create `backend/app/ingestion/loaders/text_loader.py`
- Implement `TextLoader` that reads `.txt` and `.md` files
- Return structured `Document` with `content`, `source_path`, `metadata`
- **Exit Criteria**: Unit test with sample files, correct content extraction

**Ticket 1.2.3**: Document chunker
- Create `backend/app/ingestion/chunker.py`
- Implement `Chunker` with configurable `chunk_size` (default 1000 chars) and `overlap` (default 200 chars)
- Each chunk includes: `content`, `chunk_index`, `start_char`, `end_char`, `source_path`
- **Exit Criteria**: 
  - Unit test: 2500 char document with 1000/200 settings produces 3 chunks
  - Chunks overlap correctly

**Ticket 1.2.4**: Gemini embedding service
- Create `backend/app/services/embedding_service.py`
- Implement `EmbeddingService` wrapping Gemini embedding API
- Method: `embed(texts: list[str]) -> list[list[float]]`
- Handle rate limiting with exponential backoff
- **Exit Criteria**:
  - Integration test: Embed 3 strings, receive 3 vectors of correct dimension
  - Mock test: Verify backoff behavior on 429 response

**Ticket 1.2.5**: Vector store integration
- Create `backend/app/storage/vector_store.py`
- Implement `VectorStore` using ChromaDB (persistent, per-notebook collection)
- Methods:
  - `add(chunks: list[Chunk], embeddings: list[list[float]])`
  - `search(query_embedding: list[float], k: int) -> list[ChunkResult]`
- Store path: `data/notebooks/{id}/vectors/`
- **Exit Criteria**:
  - Integration test: Add 10 chunks, search returns top-k ordered by similarity
  - Persistence test: Restart process, data still present

**Ticket 1.2.6**: Ingestion pipeline orchestrator
- Create `backend/app/ingestion/pipeline.py`
- Implement `IngestDocument(notebook_id, file_path)`:
  1. Load document (TextLoader)
  2. Chunk document (Chunker)
  3. Embed chunks (EmbeddingService)
  4. Store in vector store (VectorStore)
  5. Update notebook metadata with document info
- **Exit Criteria**:
  - Integration test: Ingest file, query vector store, retrieve relevant chunk

**Depends on**: 1.2.2, 1.2.3, 1.2.4, 1.2.5

**Ticket 1.2.7**: Trigger ingestion on file upload
- Modify file upload endpoint to call ingestion pipeline after saving
- Ingestion runs synchronously for MVP (async queue is future optimization)
- **Exit Criteria**:
  - Integration test: Upload file via API, immediately query vector store, get results

**Depends on**: 1.2.1, 1.2.6

---

### Feature 1.3: Tool Interface and Registry

**Ticket 1.3.1**: Tool base interface
- Create `backend/app/agent/tool_interface.py`
- Define `Tool` abstract base class:
  ```python
  class Tool(ABC):
      name: str
      description: str
      parameters_schema: dict  # JSON Schema
      
      @abstractmethod
      async def execute(self, params: dict, context: ToolContext) -> ToolResult
  ```
- Define `ToolContext` (contains `notebook_id`, `trace_id`, `logger`)
- Define `ToolResult` (contains `success`, `data`, `error`, `metadata`)
- **Exit Criteria**: Interface defined, importable, no implementation yet

**Ticket 1.3.2**: Tool registry
- Create `backend/app/agent/tool_registry.py`
- Implement `ToolRegistry`:
  - `register(tool: Tool)`
  - `get(name: str) -> Tool | None`
  - `list_tools() -> list[ToolDefinition]` (for LLM prompt)
- **Exit Criteria**: Unit test: Register tool, retrieve by name, list includes it

**Depends on**: 1.3.1

**Ticket 1.3.3**: Implement `search_docs` tool
- Create `backend/app/tools/search_docs.py`
- Implement `SearchDocsTool(Tool)`:
  - Parameters: `query: str`, `k: int = 5`
  - Execution: Embed query, search vector store, return chunks with metadata
- **Exit Criteria**:
  - Integration test: With ingested document, tool returns relevant chunks
  - Chunks include `source_path`, `chunk_index`, `content`, `score`

**Depends on**: 1.2.5, 1.3.1

---

### Feature 1.4: Gemini LLM Service

**Ticket 1.4.1**: Gemini generation service
- Create `backend/app/services/llm_service.py`
- Implement `LLMService` wrapping Gemini Pro API
- Methods:
  - `generate(prompt: str, system_prompt: str | None) -> str`
  - `generate_structured(prompt: str, schema: dict) -> dict` (JSON mode)
- Handle rate limiting, log token usage
- **Exit Criteria**:
  - Integration test: Generate response to simple prompt
  - Mock test: Verify retry on transient error

**Ticket 1.4.2**: Prompt templates
- Create `backend/app/agent/prompts.py`
- Define prompt templates as Python string templates:
  - `PLANNER_SYSTEM_PROMPT`: Instructions for planning tool usage
  - `PLANNER_USER_PROMPT`: Template for user query + notebook context
  - `SYNTHESIS_PROMPT`: Template for final answer from tool results
- **Exit Criteria**: Templates render without error with sample data

---

### Feature 1.5: Agent Loop (Core)

**Ticket 1.5.1**: Agent state models
- Create `backend/app/agent/models.py`
- Define:
  - `AgentPlan`: List of planned tool calls
  - `ToolCall`: Tool name + parameters
  - `ToolObservation`: Result of tool execution
  - `AgentTrace`: Full execution trace (plan, calls, observations, final answer)
- **Exit Criteria**: Models serialize to JSON for logging/storage

**Ticket 1.5.2**: Planner implementation
- Create `backend/app/agent/planner.py`
- Implement `Planner`:
  - Input: User query, notebook context (available tools, document list)
  - Output: `AgentPlan` with ordered tool calls
  - Uses `LLMService.generate_structured` with tool definitions
- **Exit Criteria**:
  - Unit test with mocked LLM: Returns valid plan structure
  - Plan includes tool names that exist in registry

**Depends on**: 1.3.2, 1.4.1, 1.4.2, 1.5.1

**Ticket 1.5.3**: Executor implementation
- Create `backend/app/agent/executor.py`
- Implement `Executor`:
  - Input: `AgentPlan`, `ToolRegistry`, `ToolContext`
  - Executes tools sequentially, collects observations
  - On tool failure: Log error, continue with partial results (no repair loop yet)
  - Output: List of `ToolObservation`
- **Exit Criteria**:
  - Unit test: Execute plan with mock tools, observations collected
  - Failed tool produces error observation, doesn't crash loop

**Depends on**: 1.3.2, 1.5.1

**Ticket 1.5.4**: Synthesizer implementation
- Create `backend/app/agent/synthesizer.py`
- Implement `Synthesizer`:
  - Input: Original query, list of `ToolObservation`
  - Output: Final answer string with citations
  - Uses `SYNTHESIS_PROMPT` template
- Citation format: `[Source: filename, Chunk: N]`
- **Exit Criteria**:
  - Unit test with mocked LLM: Returns answer string
  - Answer includes citation markers when sources present

**Depends on**: 1.4.1, 1.4.2, 1.5.1

**Ticket 1.5.5**: Agent orchestrator
- Create `backend/app/agent/orchestrator.py`
- Implement `Agent`:
  - `run(query: str, notebook_id: str) -> AgentResponse`
  - Orchestrates: Plan → Execute → Synthesize
  - Persists `AgentTrace` to `data/notebooks/{id}/traces/{trace_id}.json`
  - Logs each step with structured logging
- **Exit Criteria**:
  - Integration test: Full agent run with real ingested document
  - Trace file written to disk with complete execution record

**Depends on**: 1.5.2, 1.5.3, 1.5.4

---

### Feature 1.6: Chat API

**Ticket 1.6.1**: Chat endpoint
- Create `POST /api/notebooks/{id}/chat`
- Request body: `{"query": "string"}`
- Response: `{"answer": "string", "citations": [...], "trace_id": "uuid"}`
- Calls `Agent.run()`, returns synthesized response
- **Exit Criteria**:
  - Integration test: Upload doc, chat, receive answer with citations
  - Invalid notebook_id returns 404

**Depends on**: 1.5.5

**Ticket 1.6.2**: Trace retrieval endpoint
- Create `GET /api/notebooks/{id}/traces/{trace_id}`
- Returns full `AgentTrace` for debugging/observability
- **Exit Criteria**:
  - Integration test: Run chat, retrieve trace, verify structure

---

### Feature 1.7: Minimal Frontend

**Ticket 1.7.1**: API client service
- Create `frontend/src/services/api.ts`
- Implement typed API client:
  - `createNotebook(name: string): Promise<Notebook>`
  - `listNotebooks(): Promise<NotebookSummary[]>`
  - `uploadFile(notebookId: string, file: File): Promise<void>`
  - `chat(notebookId: string, query: string): Promise<ChatResponse>`
- **Exit Criteria**: Functions compile, types match backend schemas

**Ticket 1.7.2**: Notebook list page
- Create `frontend/src/pages/NotebookList.tsx`
- Display list of notebooks with name and file count
- "Create Notebook" button with name input modal
- Click notebook navigates to notebook detail
- **Exit Criteria**: Component renders, create button works (manual test)

**Ticket 1.7.3**: Notebook detail page
- Create `frontend/src/pages/NotebookDetail.tsx`
- Display notebook name and list of uploaded files
- File upload dropzone/button
- **Exit Criteria**: Component renders, file upload works (manual test)

**Ticket 1.7.4**: Chat interface
- Create `frontend/src/components/Chat.tsx`
- Message history display (user messages and assistant responses)
- Input field with send button
- Display citations inline or as footnotes
- Loading state while waiting for response
- **Exit Criteria**: 
  - Component renders message history
  - Send triggers API call, response displays

**Ticket 1.7.5**: Wire up routing
- Install `react-router-dom`
- Routes:
  - `/` → NotebookList
  - `/notebooks/:id` → NotebookDetail with Chat
- **Exit Criteria**: Navigation works between pages

**Depends on**: 1.7.1, 1.7.2, 1.7.3, 1.7.4

---

### Epic 1 Acceptance Criteria

- [ ] **E2E Test** (`test_epic1_e2e.py`):
  1. Create notebook via API
  2. Upload `fixtures/sample.txt` containing "The secret code is ATLAS-7429"
  3. POST `/chat` with query "What is the secret code?"
  4. Assert response contains "ATLAS-7429"
  5. Assert response contains citation referencing `sample.txt`
  6. Assert trace file exists with plan containing `search_docs` tool

- [ ] Agent trace shows: Plan(search_docs) → Execute(search_docs) → Synthesize

- [ ] Frontend allows: Create notebook → Upload file → Ask question → See answer

---

## Epic 2: Robust Document Handling

**Goal**: Handle real-world documents (PDF, DOCX) with proper metadata extraction and page-level citations.

**Deliverable**: User can upload PDFs and Word docs, ask questions, and receive answers citing specific pages.

---

### Feature 2.1: PDF Ingestion

**Ticket 2.1.1**: PDF loader implementation
- Create `backend/app/ingestion/loaders/pdf_loader.py`
- Use `pdfplumber` for text extraction
- Extract per-page text with page numbers in metadata
- Handle multi-column layouts (best effort)
- **Exit Criteria**:
  - Unit test: Load `fixtures/multipage.pdf`, extract text from all pages
  - Page numbers correctly associated with content

**Ticket 2.1.2**: Loader registry
- Create `backend/app/ingestion/loaders/registry.py`
- Map file extensions to loader classes: `.txt`→TextLoader, `.md`→TextLoader, `.pdf`→PDFLoader
- Method: `get_loader(file_path: str) -> Loader`
- **Exit Criteria**: Unit test: Correct loader returned for each extension

**Depends on**: 1.2.2, 2.1.1

**Ticket 2.1.3**: Update chunker for page-aware chunking
- Modify `Chunker` to preserve page boundaries in metadata
- Each chunk includes `page_numbers: list[int]` (may span pages)
- **Exit Criteria**:
  - Unit test: Chunk spanning pages 2-3 has `page_numbers: [2, 3]`

**Depends on**: 1.2.3

**Ticket 2.1.4**: Update ingestion pipeline for PDF
- Modify pipeline to use loader registry
- Verify PDF files ingest correctly
- **Exit Criteria**:
  - Integration test: Ingest PDF, search returns chunks with page numbers

**Depends on**: 2.1.2, 2.1.3

---

### Feature 2.2: DOCX Ingestion

**Ticket 2.2.1**: DOCX loader implementation
- Create `backend/app/ingestion/loaders/docx_loader.py`
- Use `python-docx` for text extraction
- Extract paragraph-level text, track approximate page numbers (if possible) or section numbers
- **Exit Criteria**:
  - Unit test: Load `fixtures/sample.docx`, extract all paragraphs

**Ticket 2.2.2**: Register DOCX loader
- Add `.docx` mapping to loader registry
- **Exit Criteria**: DOCX files route to correct loader

**Depends on**: 2.1.2, 2.2.1

---

### Feature 2.3: Enhanced Citations

**Ticket 2.3.1**: Update vector store schema
- Add fields: `source_id`, `page_numbers`, `section` to chunk storage
- Ensure searchable and retrievable
- **Exit Criteria**: Integration test: Store chunk with page info, retrieve includes it

**Ticket 2.3.2**: Update synthesis prompt for page citations
- Modify `SYNTHESIS_PROMPT` to instruct: "Cite sources as [Source: filename, Page: X]"
- **Exit Criteria**: Mocked synthesis includes page-level citations

**Ticket 2.3.3**: Parse citations in response
- Create `backend/app/agent/citation_parser.py`
- Extract citations from answer text, return structured `Citation` objects
- **Exit Criteria**: Unit test: Parse "[Source: doc.pdf, Page: 3]" → Citation object

**Ticket 2.3.4**: Update chat response schema
- Add `citations: list[Citation]` to chat response
- Each citation: `source`, `page`, `chunk_index`, `text_snippet`
- **Exit Criteria**: API returns structured citations alongside answer

**Depends on**: 2.3.3

---

### Feature 2.4: Frontend Citation Display

**Ticket 2.4.1**: Citation tooltip component
- Create `frontend/src/components/Citation.tsx`
- Render citation as clickable reference
- Tooltip shows source snippet on hover
- **Exit Criteria**: Component renders, hover shows tooltip

**Ticket 2.4.2**: Integrate citations in chat
- Update Chat component to render inline citation markers
- Link citations to tooltip component
- **Exit Criteria**: Chat answers show clickable citation markers

**Depends on**: 2.4.1

---

### Epic 2 Acceptance Criteria

- [ ] **E2E Test** (`test_epic2_pdf.py`):
  1. Create notebook
  2. Upload `fixtures/multipage.pdf` (10 pages, distinct content per page)
  3. Query "What is discussed on page 7?"
  4. Assert answer references content from page 7
  5. Assert citation includes `Page: 7`

- [ ] **E2E Test** (`test_epic2_docx.py`):
  1. Upload DOCX file
  2. Query about specific content
  3. Assert correct retrieval

- [ ] **Load Test**: Upload 100-page PDF. Ingestion completes within 180 seconds. Query returns response within 10 seconds.

---

## Epic 3: Structured Data Agent (Text-to-SQL)

**Goal**: Enable analytics queries over tabular data. The agent can "reason" about numbers using SQL.

**Deliverable**: User can upload CSV/Excel files, ask analytical questions, and receive computed answers.

---

### Feature 3.1: OLAP Engine Setup

**Ticket 3.1.1**: DuckDB integration
- Create `backend/app/storage/olap_store.py`
- Implement `OLAPStore`:
  - One DuckDB database per notebook: `data/notebooks/{id}/analytics.duckdb`
  - Methods: `execute(sql: str) -> DataFrame`, `get_tables() -> list[str]`
- **Exit Criteria**:
  - Unit test: Create table, insert data, query returns correct results
  - Database persists across process restarts

**Ticket 3.1.2**: CSV ingestion to DuckDB
- Create `backend/app/ingestion/loaders/csv_loader.py`
- Auto-infer schema from CSV headers and data types
- Create table in DuckDB with sanitized table name (from filename)
- **Exit Criteria**:
  - Integration test: Upload `sales.csv`, table `sales` exists in DuckDB
  - Column types correctly inferred (string, int, float, date)

**Ticket 3.1.3**: Excel ingestion to DuckDB
- Create `backend/app/ingestion/loaders/excel_loader.py`
- Use `openpyxl` for `.xlsx` files
- Handle multiple sheets: create table per sheet (e.g., `filename_sheet1`)
- **Exit Criteria**:
  - Integration test: Upload multi-sheet Excel, all sheets become tables

**Ticket 3.1.4**: Update ingestion pipeline for tabular data
- Detect tabular files (`.csv`, `.xlsx`) and route to OLAP ingestion
- Store metadata about tables in notebook metadata
- **Exit Criteria**: Upload CSV via API, table queryable in DuckDB

**Depends on**: 3.1.1, 3.1.2, 3.1.3

---

### Feature 3.2: Schema Introspection Tool

**Ticket 3.2.1**: Implement `describe_tables` tool
- Create `backend/app/tools/describe_tables.py`
- Returns list of tables with their schemas (column names, types)
- Parameters: none (uses notebook context)
- **Exit Criteria**:
  - Integration test: With uploaded CSV, tool returns correct schema

**Depends on**: 3.1.1

**Ticket 3.2.2**: Register tool in registry
- Add to tool registry
- Update planner prompt to include schema introspection capability
- **Exit Criteria**: Planner can plan to use `describe_tables`

**Depends on**: 1.3.2, 3.2.1

---

### Feature 3.3: SQL Execution Tool

**Ticket 3.3.1**: Implement `execute_sql` tool
- Create `backend/app/tools/execute_sql.py`
- Parameters: `sql: str`
- Executes SQL against notebook's DuckDB
- Returns: Result rows (limited to 100) + row count + column names
- **Security**: Validate SQL is SELECT only (no DDL/DML)
- **Exit Criteria**:
  - Integration test: Valid SELECT returns data
  - Integration test: INSERT/DELETE rejected with error

**Depends on**: 3.1.1

**Ticket 3.3.2**: SQL error handling
- On SQL syntax error: Return error message suitable for LLM repair
- Include: Error type, message, problematic SQL
- **Exit Criteria**: Malformed SQL returns structured error, not crash

---

### Feature 3.4: Text-to-SQL Generation

**Ticket 3.4.1**: Text-to-SQL prompt template
- Create `SQL_GENERATION_PROMPT` in prompts.py
- Include: Table schemas, column descriptions, SQL dialect notes (DuckDB)
- Few-shot examples for common patterns
- **Exit Criteria**: Template renders with sample schema

**Ticket 3.4.2**: Implement `generate_sql` tool
- Create `backend/app/tools/generate_sql.py`
- Parameters: `question: str`, `table_schemas: list[TableSchema]`
- Uses LLM to generate SQL query
- Returns: Generated SQL string
- **Exit Criteria**:
  - Integration test: "What is total revenue?" generates valid SQL with SUM

**Depends on**: 1.4.1, 3.4.1

**Ticket 3.4.3**: SQL repair loop
- If `execute_sql` fails, agent should:
  1. Feed error back to LLM
  2. Request corrected SQL
  3. Retry (max 2 attempts)
- Implement in executor as repair strategy for SQL tool
- **Exit Criteria**:
  - Integration test: Intentionally generate bad SQL (mock), verify retry occurs
  - Trace shows repair attempt

**Depends on**: 3.3.1, 3.4.2

---

### Feature 3.5: SQL Agent Integration

**Ticket 3.5.1**: Update planner for SQL queries
- Planner should recognize analytical queries and plan:
  1. `describe_tables` (if schema unknown)
  2. `generate_sql`
  3. `execute_sql`
  4. Synthesize answer
- **Exit Criteria**: Analytical query produces correct plan

**Ticket 3.5.2**: Update synthesizer for tabular results
- Synthesizer should format SQL results naturally
- Include the query that was run for transparency
- **Exit Criteria**: Answer includes natural language interpretation of data

---

### Epic 3 Acceptance Criteria

- [ ] **E2E Test** (`test_epic3_sql.py`):
  1. Create notebook
  2. Upload `fixtures/sales.csv` with columns: `region`, `product`, `revenue`, `date`
  3. Query "What is the total revenue by region?"
  4. Assert answer contains correct sums (pre-computed expected values)
  5. Assert trace shows: describe_tables → generate_sql → execute_sql

- [ ] **E2E Test** (`test_epic3_repair.py`):
  1. Upload CSV with unusual column names
  2. Query that will likely cause initial SQL error
  3. Assert repair loop triggered (check trace)
  4. Assert final answer still correct

- [ ] **SQL Injection Test**: Query containing `; DROP TABLE` does not execute destructive command

---

## Epic 4: Multimodal Vision (Images)

**Goal**: The system can "see" uploaded images and answer questions about them.

**Deliverable**: User can upload images (charts, diagrams, screenshots), and the agent uses vision to answer.

---

### Feature 4.1: Image Ingestion

**Ticket 4.1.1**: Image upload support
- Update file upload endpoint to accept `.png`, `.jpg`, `.jpeg`, `.webp`
- Store images in `data/notebooks/{id}/raw/`
- Store image metadata (dimensions, format) in notebook metadata
- **Exit Criteria**: Integration test: Upload image, file exists on disk

**Ticket 4.1.2**: Image captioning service
- Create `backend/app/services/vision_service.py`
- Use Gemini Vision to generate dense captions for images
- Method: `caption(image_path: str) -> str`
- Caption should describe: content, text visible, data if chart/graph
- **Exit Criteria**:
  - Integration test: Caption `fixtures/chart.png`, caption mentions "chart" or "graph"

**Ticket 4.1.3**: Image to vector store
- On image upload:
  1. Generate caption via vision service
  2. Create chunk with caption text + image path in metadata
  3. Embed and store in vector store
- **Exit Criteria**: Search for content in image returns the image chunk

**Depends on**: 4.1.1, 4.1.2

---

### Feature 4.2: Vision RAG

**Ticket 4.2.1**: Implement `analyze_image` tool
- Create `backend/app/tools/analyze_image.py`
- Parameters: `image_path: str`, `question: str`
- Sends image + question to Gemini Vision
- Returns: Detailed answer about the image
- **Exit Criteria**:
  - Integration test: Ask "What trend does this show?" about chart, get trend description

**Ticket 4.2.2**: Update retrieval for images
- When `search_docs` returns image chunks, include image path in results
- Planner should recognize when to call `analyze_image` for follow-up
- **Exit Criteria**: Query about chart triggers both search and image analysis

**Ticket 4.2.3**: Multi-step vision workflow
- For image queries, agent should:
  1. `search_docs` to find relevant images
  2. `analyze_image` on retrieved images
  3. Synthesize answer
- **Exit Criteria**: Trace shows multi-tool usage for image query

---

### Feature 4.3: Frontend Image Display

**Ticket 4.3.1**: Image thumbnail in citations
- When citation references an image, show thumbnail
- Click to expand full image
- **Exit Criteria**: Image citations show visual preview

**Ticket 4.3.2**: Image list in notebook view
- Display uploaded images as gallery in notebook detail
- **Exit Criteria**: Images visible in notebook view

---

### Epic 4 Acceptance Criteria

- [ ] **E2E Test** (`test_epic4_vision.py`):
  1. Create notebook
  2. Upload `fixtures/revenue_chart.png` (bar chart showing revenue by quarter)
  3. Query "What quarter had the highest revenue?"
  4. Assert answer correctly identifies the quarter (based on known fixture data)
  5. Assert citation references the image

- [ ] **E2E Test** (`test_epic4_mixed.py`):
  1. Upload both PDF and image
  2. Query that requires both sources
  3. Assert answer synthesizes from both

---

## Epic 5: Evaluation Framework

**Goal**: Measure and demonstrate system quality with quantified metrics.

**Deliverable**: Eval suite that measures RAG and SQL accuracy, with metrics tracked over time.

---

### Feature 5.1: RAG Evaluation

**Ticket 5.1.1**: Create RAG eval dataset
- Create `backend/tests/eval/rag_dataset.json`
- Structure: 
  ```json
  [
    {
      "documents": ["path/to/doc.txt"],
      "query": "What is X?",
      "expected_answer": "X is ...",
      "required_citations": ["doc.txt"]
    }
  ]
  ```
- Include 15-20 diverse test cases
- **Exit Criteria**: Dataset file exists, valid JSON

**Ticket 5.1.2**: RAG answer scoring
- Create `backend/app/eval/rag_eval.py`
- Implement scoring:
  - `answer_similarity`: Semantic similarity to expected (Gemini embedding cosine)
  - `citation_recall`: % of required citations present
- **Exit Criteria**: Unit test: Score sample answer, get numeric scores

**Ticket 5.1.3**: RAG eval runner
- Create `scripts/eval_rag.py`
- For each test case:
  1. Create temp notebook
  2. Ingest documents
  3. Run query
  4. Score answer
- Output: JSON report with per-case scores and aggregates
- **Exit Criteria**: Script runs, produces report

**Depends on**: 5.1.1, 5.1.2

**Ticket 5.1.4**: RAG quality threshold test
- Create pytest test that runs eval and asserts:
  - Average answer_similarity > 0.7
  - Average citation_recall > 0.8
- **Exit Criteria**: Test passes (may require tuning prompts)

**Depends on**: 5.1.3

---

### Feature 5.2: SQL Evaluation

**Ticket 5.2.1**: Create SQL eval dataset
- Create `backend/tests/eval/sql_dataset.json`
- Structure:
  ```json
  [
    {
      "csv_file": "path/to/data.csv",
      "query": "What is the average X?",
      "expected_sql_result": 42.5,
      "expected_sql_pattern": "AVG\\(.*\\)"
    }
  ]
  ```
- Include 10-15 test cases with varying complexity
- **Exit Criteria**: Dataset file exists, valid JSON

**Ticket 5.2.2**: SQL execution accuracy scorer
- Create `backend/app/eval/sql_eval.py`
- Implement scoring:
  - `execution_success`: SQL executed without error
  - `result_accuracy`: Result matches expected (within tolerance for floats)
- **Exit Criteria**: Unit test: Score sample SQL result

**Ticket 5.2.3**: SQL eval runner
- Create `scripts/eval_sql.py`
- Similar structure to RAG eval
- Output: JSON report with execution accuracy %
- **Exit Criteria**: Script runs, produces report

**Depends on**: 5.2.1, 5.2.2

**Ticket 5.2.4**: SQL quality threshold test
- Assert execution_success > 90%
- Assert result_accuracy > 85%
- **Exit Criteria**: Test passes

---

### Feature 5.3: Metrics Dashboard (Optional)

**Ticket 5.3.1**: Metrics storage
- Store eval results in `data/eval_history.json`
- Track: timestamp, commit hash, RAG scores, SQL scores
- **Exit Criteria**: Eval runs append to history file

**Ticket 5.3.2**: Simple metrics viewer
- Create endpoint `GET /api/eval/history`
- Returns historical eval scores
- (Frontend display optional—JSON is enough for demo)
- **Exit Criteria**: Endpoint returns historical data

---

### Epic 5 Acceptance Criteria

- [ ] `scripts/eval_rag.py` runs and produces JSON report
- [ ] `scripts/eval_sql.py` runs and produces JSON report
- [ ] RAG eval passes threshold (similarity > 0.7, citation_recall > 0.8)
- [ ] SQL eval passes threshold (execution > 90%, accuracy > 85%)
- [ ] Eval history tracked across runs

---

## Epic 6: Observability and Polish

**Goal**: Production-grade observability, streaming responses, and UX refinements.

**Deliverable**: System is debuggable, responsive, and demo-ready.

---

### Feature 6.1: Enhanced Tracing

**Ticket 6.1.1**: Trace context propagation
- Generate `trace_id` at request start
- Pass through all components (agent, tools, services)
- Include in all log entries
- **Exit Criteria**: Single trace_id appears in all logs for one request

**Ticket 6.1.2**: Timing instrumentation
- Log duration of: LLM calls, embeddings, SQL execution, tool execution
- Add timing to trace records
- **Exit Criteria**: Trace includes timing breakdown

**Ticket 6.1.3**: Trace explorer endpoint
- `GET /api/notebooks/{id}/traces` - list recent traces
- `GET /api/notebooks/{id}/traces/{trace_id}` - full trace detail
- **Exit Criteria**: API returns trace data

---

### Feature 6.2: Streaming Responses

**Ticket 6.2.1**: Backend SSE support
- Modify `/chat` endpoint to support `Accept: text/event-stream`
- Stream: planning status, tool executions, final answer tokens
- **Exit Criteria**:
  - Integration test: SSE connection receives multiple events
  - Final answer streams token-by-token

**Ticket 6.2.2**: Frontend streaming integration
- Update Chat component to consume SSE
- Show "Thinking..." with tool execution steps
- Stream answer text as it arrives
- **Exit Criteria**: UI shows progressive updates during query

---

### Feature 6.3: Thought Process Visibility

**Ticket 6.3.1**: Expandable trace view in chat
- After answer, show "View thought process" toggle
- Expand to show: Plan, Tool calls with inputs/outputs, Timing
- **Exit Criteria**: Toggle works, trace readable

---

### Feature 6.4: Error Handling & Edge Cases

**Ticket 6.4.1**: Graceful LLM failures
- If Gemini API fails, return user-friendly error
- Retry transient errors (429, 503) with backoff
- **Exit Criteria**: Simulated API failure returns graceful message, not 500

**Ticket 6.4.2**: Large document handling
- Test and tune for 200+ page PDFs
- If too large, chunk ingestion into batches
- **Exit Criteria**: 200-page PDF ingests successfully (may take time)

**Ticket 6.4.3**: Empty/invalid file handling
- Empty file upload returns 400 with clear message
- Corrupt PDF returns 400 with clear message
- **Exit Criteria**: Integration tests for edge cases

---

### Feature 6.5: UI Polish

**Ticket 6.5.1**: Loading states
- Show spinner during file upload
- Show progress during ingestion (if possible)
- Disable chat input while processing
- **Exit Criteria**: No broken states during async operations

**Ticket 6.5.2**: Responsive layout
- Chat interface works on tablet-width screens
- Notebook list scrolls properly with many notebooks
- **Exit Criteria**: Manual test at 768px width

**Ticket 6.5.3**: Error toast notifications
- Display API errors as toast messages
- Auto-dismiss after 5 seconds
- **Exit Criteria**: Errors visible, not silent failures

---

### Epic 6 Acceptance Criteria

- [ ] Query with streaming shows progressive updates in UI
- [ ] "View thought process" shows complete agent trace
- [ ] All logs for one request share same trace_id
- [ ] 200-page PDF ingests without OOM or timeout
- [ ] API errors display user-friendly messages

---

## Future Scope (Deferred)

The following are explicitly out of scope for initial implementation:

- **Video/Audio Ingestion**: ASR with Whisper, timestamped transcript RAG. Significant complexity (ffmpeg, alignment, streaming) for marginal portfolio value.
- **Real-time Collaboration**: WebSocket sync, conflict resolution.
- **Multi-user / Auth**: User accounts, RBAC, tenant isolation.
- **Cloud Deployment**: Docker, Kubernetes, managed services integration.
- **Advanced Caching**: Redis, semantic cache for similar queries.
- **Conversation Memory**: Multi-turn context within a chat session.

---

## Appendix: Test Fixtures Required

Create `backend/tests/fixtures/`:

- `sample.txt` - Simple text file with known content
- `multipage.pdf` - 10-page PDF with distinct content per page
- `sample.docx` - Word document with paragraphs
- `sales.csv` - Sales data with region, product, revenue, date columns
- `revenue_chart.png` - Bar chart with clear trend
- `large_document.pdf` - 100+ page PDF for load testing

---

## Appendix: Definition of Done (Global)

Every ticket is complete when:

1. Code is written and follows project conventions
2. Unit tests pass (if logic exists)
3. Integration tests pass (if external dependencies)
4. Linters pass (`mypy`, `ruff`, `eslint`)
5. Code is reviewed (self-review acceptable for solo project)
6. Changes are committed with descriptive message

Every epic is complete when:

1. All tickets in all features are done
2. Epic acceptance criteria tests pass
3. System is demonstrable end-to-end for the epic's scope
