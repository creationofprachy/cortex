Cortex
A local-first semantic search and knowledge graph engine for your own documents.
Cortex turns a folder of notes, papers, and PDFs into something you can search by meaning
and browse as a map — entirely on your own machine, with no external AI API, no API key,
and no data ever leaving your computer.
Problem Statement
Keyword search (`Ctrl+F`, `grep`, most note-taking apps) only finds documents that contain
the exact words you typed. If you search "speeding up database queries" it won't surface a
note titled "Database Indexing Fundamentals" — even though that's exactly what you meant.
On top of that, as a personal document collection grows, it becomes impossible to remember
which notes are actually related to each other.
Solution
Cortex builds a local semantic embedding index (TF-IDF + Latent Semantic Analysis) over
your documents so that search matches concepts, not just words — and it automatically
constructs a knowledge graph linking documents that are semantically related, so you can
see the shape of your own knowledge base at a glance.
Everything runs offline after installation: no calls to OpenAI, Hugging Face, or any other
external AI service. The "embedding model" is fit directly on your corpus using scikit-learn,
so it needs no GPU, no model download, and no internet connection.
Key Features
Semantic search — finds conceptually related content even with no shared keywords.
Automatic knowledge graph — documents are linked when their content is similar; view
and explore it as an interactive force-directed graph in the browser.
Multi-format ingestion — plain text, Markdown, and PDF, via drag-and-drop upload.
Fully local — no external API calls, no API keys, no network dependency at runtime.
Clean REST API — every feature is available over HTTP (FastAPI + OpenAPI docs at `/docs`).
Persistent index — the fitted model is cached to disk and reloaded on restart.
Screenshots / Demo
> Run the app locally (see below) and open `http://localhost:8000` — the **Search**, **Knowledge
> Graph**, and **Library** tabs are all in the single-page UI described under Example Usage.
```
┌────────────────────────────────────────────────────────────┐
│  Cortex                Search | Knowledge Graph | Library  │
├────────────────────────────────────────────────────────────┤
│  [ speeding up database queries                  ] [Search]│
│                                                              │
│  Database Indexing Fundamentals                             │
│  database_indexing.txt · chunk #0 · similarity 99.9%        │
│  "An index is a data structure that improves the speed..."  │
└────────────────────────────────────────────────────────────┘
```
Architecture
```
                ┌──────────────┐
   .txt/.md/.pdf│              │  1. extract + chunk text
  ─────────────▶│  Ingestion   │──────────────┐
                │              │              ▼
                └──────────────┘        ┌─────────────┐
                                        │   SQLite    │  documents + chunks
                                        └─────────────┘
                                              │
                                              ▼
                                  ┌─────────────────────────┐
                                  │  TF-IDF + Truncated SVD │  2. (re)fit on full corpus
                                  │   (Semantic Index)      │
                                  └─────────────────────────┘
                                         │            │
                             cosine similarity   cosine similarity
                                         │            │
                                         ▼            ▼
                                 ┌───────────┐  ┌──────────────┐
                                 │  Search   │  │  Knowledge   │
                                 │   API     │  │  Graph (nx)  │
                                 └───────────┘  └──────────────┘
                                         │            │
                                         └─────┬──────┘
                                               ▼
                                     FastAPI REST endpoints
                                               │
                                               ▼
                                  Static HTML/CSS/JS frontend
                                  (search UI + vis-network graph)
```
The index is rebuilt in-process whenever a document is added or removed and cached to disk
(`data/index/`), so a restart doesn't require re-processing the whole corpus unless it changed.
Tech Stack
Layer	Choice	Why
Backend	FastAPI	async-ready, typed, auto-generates OpenAPI docs
Storage	SQLite	zero-config, file-based, plenty for a personal tool
Embeddings	scikit-learn (TF-IDF + Truncated SVD)	fully local, no model download, fast to fit
Graph	NetworkX	standard graph library, easy JSON export
PDF parsing	pypdf	pure-Python, no system dependency
Frontend	vanilla HTML/CSS/JS + vis-network (CDN)	no build step, keeps the project easy to run
Tests	pytest + FastAPI `TestClient`	unit + full API integration coverage
Installation
Prerequisites
Python 3.10+
1. Clone and enter the project
```bash
git clone <YOUR_GITHUB_REPOSITORY_URL>
cd cortex
```
2. Create and activate a virtual environment
Windows (PowerShell):
```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```
macOS / Linux:
```bash
python3 -m venv .venv
source .venv/bin/activate
```
3. Install dependencies
```bash
pip install -r requirements.txt
```
4. Configuration
Cortex works with zero configuration. If you want to customize data locations or tuning
parameters, copy `.env.example` to `.env` and adjust:
```bash
cp .env.example .env   # optional
```
How to Run
```bash
uvicorn app.main:app --reload
```
Then open http://localhost:8000 in your browser.
Optionally, seed a few sample documents first so there's something to search immediately:
```bash
python scripts/seed_sample_data.py
```
Example Usage
Via the web UI: open the app, go to the Library tab, upload a `.txt`, `.md`, or `.pdf`
file, then switch to Search and type a concept (not necessarily an exact phrase from the
document). Switch to Knowledge Graph to see how your documents relate to each other.
Via the REST API:
```bash
# Upload a document
curl -F "file=@notes/neural_networks.txt" http://localhost:8000/api/ingest

# Semantic search
curl "http://localhost:8000/api/search?q=how+do+deep+networks+learn"

# Fetch the document knowledge graph
curl http://localhost:8000/api/graph

# List all indexed documents
curl http://localhost:8000/api/documents
```
Interactive API docs (Swagger UI) are available at `http://localhost:8000/docs`.
Project Structure
```
cortex/
├── README.md
├── requirements.txt
├── .gitignore
├── LICENSE
├── .env.example
│
├── app/
│   ├── main.py         # FastAPI app and route handlers
│   ├── config.py        # environment-driven configuration
│   ├── database.py      # SQLite schema and queries
│   ├── ingestion.py      # text extraction + chunking
│   ├── embeddings.py    # TF-IDF + Truncated SVD semantic index
│   ├── search.py         # query-time cosine-similarity search
│   ├── graph.py          # document-level knowledge graph construction
│   ├── models.py         # Pydantic request/response schemas
│   └── static/            # single-page frontend (HTML/CSS/JS)
│
├── tests/
│   ├── conftest.py        # isolated per-test data directories
│   ├── test_ingestion.py
│   ├── test_search.py
│   ├── test_graph.py
│   └── test_api.py       # end-to-end API tests
│
├── scripts/
│   └── seed_sample_data.py
│
└── data/                  # created at runtime (gitignored except structure)
    ├── documents/
    └── index/
```
Testing
```bash
pytest
```
19 tests cover chunking edge cases, semantic ranking correctness, knowledge-graph
construction, and full API round-trips (ingest → search → graph → delete), each running
against an isolated temporary data directory.
Future Improvements
Incremental index updates instead of a full refit on every ingest (fine at personal-corpus
scale, but would matter for thousands of documents).
Optional pluggable embedding backends (e.g. a local sentence-transformers model) behind the
same `SemanticIndex` interface, for users willing to trade setup complexity for accuracy.
Multi-user support with per-user document scoping and authentication.
Highlighted query-term spans directly in the graph view's node tooltips.
Limitations
Ranking quality depends on corpus size: TF-IDF + SVD needs a reasonable number of documents
to build a meaningful latent semantic space; with only 1–2 documents, search degrades toward
keyword matching.
No OCR: scanned/image-only PDFs will fail ingestion with "no extractable text."
Single-process, single-user design — not intended for concurrent multi-user deployment as-is.
License
MIT — see LICENSE.
Author
Built as a portfolio project demonstrating full-stack ML engineering: API design, local NLP
pipelines, graph construction, and frontend integration without relying on external AI
services.
