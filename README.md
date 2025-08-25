ArXiv Paper Curator — RAG Toolkit for AI Research Papers
=======================================================

[![Releases - Download](https://img.shields.io/badge/Releases-Download-blue?logo=github)](https://github.com/Khozanathan/arxiv-paper-curator/releases)

<p align="center">
  <img src="static/mother_of_ai_project_rag_architecture.gif" alt="RAG Architecture" width="780">
</p>

Build a production-ready Retrieval-Augmented Generation (RAG) pipeline for arXiv papers. This project teaches the full stack: ingestion, embeddings, vector search with OpenSearch, retrieval, and LLM generation via a FastAPI service and Docker Compose.

Table of contents
-----------------
- Features
- Architecture
- Quick start
- From source (developer)
- Core concepts
- API reference
- Command-line usage
- Indexing workflow
- Configuration
- Testing and CI
- Contributing
- License

Features
--------
- Ingest arXiv metadata and PDFs into a vector store.
- Convert text to embeddings and store vectors in OpenSearch.
- Retrieve relevant passages for prompts using dense retrieval.
- Serve a simple RAG API via FastAPI for chat and single-query flows.
- Local-first dev stack with Docker Compose for OpenSearch and app services.
- Scripts for batch indexing, incremental updates, and reindexing.

Architecture
------------
The pipeline splits work into clear components:
- Ingest: fetch metadata and PDF, extract text, chunk.
- Embed: create embedding vectors per chunk.
- Store: index vectors and metadata in OpenSearch.
- Retrieve: find top-N candidate chunks for a query.
- Generate: pass retrieved context to an LLM to produce the final answer.

Key pieces:
- FastAPI app for HTTP endpoints and background tasks.
- OpenSearch as vector store and full-text fallback.
- Embedding model (configurable) to encode text.
- Simple generator interface that calls an LLM provider.

Quick start
-----------
1) Download the release archive and run the bundle:
   - Visit the Releases page and download the latest release archive from:
     https://github.com/Khozanathan/arxiv-paper-curator/releases
   - Extract the archive and run the included install script:
     ./install.sh
   - The install script boots Docker Compose (OpenSearch + app), installs Python deps, and runs initial indexing.

2) Or start with Docker Compose:
   - docker compose up -d
   - ./scripts/bootstrap.sh

The release archive you download from the Releases page contains a runnable bundle with a ready docker-compose.yml and an install script. Download the release file and execute the included install.sh to bootstrap the stack.

Prerequisites
-------------
- Linux/macOS or WSL2
- Docker & Docker Compose
- Python 3.12+
- Git

From source (developer)
-----------------------
Clone and install:
- git clone https://github.com/Khozanathan/arxiv-paper-curator.git
- cd arxiv-paper-curator
- python -m venv .venv
- source .venv/bin/activate
- pip install -r requirements.txt

Run dev stack:
- docker compose up -d opensearch
- export OPENSEARCH_URL=http://localhost:9200
- uvicorn app.main:app --reload --port 8000

Project layout
--------------
- app/              FastAPI app and routers
- ingest/           scrapers, PDF processing, chunkers
- embeddings/       model wrappers, batching
- indexer/          OpenSearch index templates and helpers
- scripts/          bootstrap, maintenance, reindex
- tests/            unit and integration tests
- docker/           docker-compose files and configs
- static/           architecture images and demo assets

Core concepts
-------------
- Chunking: split long documents into overlapping passages (e.g., 512 tokens, stride 128).
- Embeddings: compute fixed-length vectors. Configurable model (open models supported).
- Vector index: OpenSearch k-NN index for dense retrieval.
- Retriever: fetch top-k passages and rank by hybrid score.
- Generator: feed retrieved contexts to an LLM with a template prompt.

API reference
-------------
Base URL: http://localhost:8000

Endpoints
- POST /ingest
  - Body: { "arxiv_id": "2201.00001" }
  - Action: fetch metadata + PDF, extract text, chunk, embed, index.

- GET /papers/{arxiv_id}
  - Return: metadata, indexed chunk count, index status.

- POST /query
  - Body: { "query": "How does method X scale?", "k": 5 }
  - Return: generated answer, source passages, metadata.

- GET /health
  - Return: service and index health.

Example curl
------------
Ingest a paper:
- curl -X POST -H "Content-Type: application/json" -d '{"arxiv_id":"2201.00001"}' http://localhost:8000/ingest

Query with RAG:
- curl -X POST -H "Content-Type: application/json" -d '{"query":"What is RAG?", "k":3}' http://localhost:8000/query

Command-line usage
------------------
Scripts:
- scripts/ingest_arxiv.py --id 2201.00001 --batch 4
- scripts/index_folder.py --path data/pdfs
- scripts/reindex.sh

Example CLI flow:
- python scripts/ingest_arxiv.py --id 2103.00001
- python scripts/query_local.py --query "transformer architecture" --k 5

Indexing workflow
-----------------
1) Fetch metadata: arXiv API for title, authors, abstract, categories.
2) Download PDF: store under data/pdfs.
3) Extract text: use pdfminer or similar.
4) Clean text: remove headers, fonts, duplicated pages.
5) Chunk text: fixed token-length windows with overlap.
6) Embed chunks: batch requests to embedding model.
7) Index: push vectors and chunk metadata to OpenSearch k-NN index.

OpenSearch index design
-----------------------
- Index name: arxiv-papers-v1
- Mapping:
  - text_chunks: keyword + vector field
  - metadata: title, authors, abstract, arxiv_id, url
- k-NN params: space = "cosinesimil", dimension = 1536 (configurable)

Configuration
-------------
Configuration lives in config/*.yaml and environment variables:
- OPENSEARCH_URL (default: http://localhost:9200)
- EMBEDDING_MODEL (default: local or remote model name)
- LLM_PROVIDER (e.g., openai, local-llm)
- INDEX_NAME (default: arxiv-papers-v1)
- CHUNK_SIZE and CHUNK_STRIDE for text splitting

Switch providers by setting LLM_PROVIDER and EMBEDDING_MODEL in .env.

RAG prompt templates
--------------------
The repo includes simple prompt templates:
- extractive_prompt.txt — for short answers using retrieved passages.
- summarize_prompt.txt — for long summaries across passages.
You can add templates under templates/ and reference them from the API.

Testing and CI
--------------
- Unit tests: pytest tests/unit
- Integration: pytest tests/integration (requires docker compose)
- Lint: pre-commit (black, ruff)

CI pipeline:
- Run unit tests on each PR.
- Start ephemeral OpenSearch for integration tests.
- Build and cache Python dependencies.

Troubleshoot
------------
- If indexing fails, check OpenSearch index health at OPENSEARCH_URL/_cat/indices.
- If embeddings error, verify EMBEDDING_MODEL and quota for remote providers.
- If API returns 502 from the generator, verify LLM_PROVIDER keys.

Contributing
------------
- Fork the repo, create a feature branch, open a PR.
- Run tests locally before PR.
- Keep commits small and focused.
- Add a short description and example when you add a new feature.

Releases
--------
Download the releases bundle and run the included install script:
https://github.com/Khozanathan/arxiv-paper-curator/releases

Visit the Releases page to get packaged installers, tarballs, and prebuilt assets. Each release includes a README with the exact asset name and instructions. If you prefer to build from source, follow the From source section above.

Security and credentials
------------------------
- Keep API keys and tokens out of source. Use .env for secrets.
- Rotate keys and limit scope for third-party LLM or embedding providers.

Examples and demos
------------------
- demo/notebooks contains example notebooks for:
  - end-to-end ingestion of a single arXiv paper
  - embedding visualization
  - interactive retrieval and generation

- static/ includes architecture GIFs and example outputs for the RAG workflow.

FAQ
---
Q: Can I run this without Docker?
A: Yes. Run OpenSearch as a managed service or a remote cluster and point OPENSEARCH_URL to it. Install Python deps and run uvicorn.

Q: Can I use a custom embedding model?
A: Yes. Set EMBEDDING_MODEL to your model endpoint or path. Update batch size in config.

Q: How do I scale for large corpora?
A: Use sharded OpenSearch indices, increase node count, and run batch embedding pipelines. Use incremental ingestion to avoid reprocessing.

License
-------
MIT License. See LICENSE file.

Contact
-------
Open an issue or PR on GitHub for bugs or feature requests. Use the repository issue tracker to propose changes.