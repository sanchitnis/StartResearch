# OpenSana — System Architecture

**Version:** 1.0  
**Status:** Draft  
**License:** Apache 2.0

---

## 1. Overview

OpenSana is a layered, service-oriented platform composed exclusively of open-source components. All services run inside Docker containers on the researcher's own hardware. Data flows between components over a private Docker network; nothing leaves the host unless the user explicitly enables external connectors.

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Researcher's Machine                         │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────┐ │
│  │  Open WebUI  │  │  Khoj Web    │  │  CLI / Python Scripts      │ │
│  │  (chat UI)   │  │  (search UI) │  │  (headless agent runner)   │ │
│  └──────┬───────┘  └──────┬───────┘  └────────────┬───────────────┘ │
│         │                 │                        │                 │
│  ┌──────▼─────────────────▼────────────────────────▼──────────────┐ │
│  │                     Orchestration Layer                         │ │
│  │   LangChain tool chains · SKILL.md agent runner · REST hooks    │ │
│  └──────┬───────────────────────────────────┬───────────────────── ┘ │
│         │                                   │                        │
│  ┌──────▼──────────┐               ┌────────▼────────┐              │
│  │  RAGFlow        │               │  Khoj           │              │
│  │  (deep-layout   │               │  (multi-source  │              │
│  │   RAG engine)   │               │   semantic RAG) │              │
│  └──────┬──────────┘               └────────┬────────┘              │
│         │                                   │                        │
│  ┌──────▼───────────────────────────────────▼────────┐              │
│  │              Vector Store (Chroma / Qdrant)        │              │
│  └───────────────────────────────────────────────────┘              │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    LLM Backbone (BYOM)                       │    │
│  │   Ollama · Llama.cpp · OpenRouter endpoint · HuggingFace    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                     Filesystem (Host)                          │  │
│  │   /library  /skills  /chats  /artifacts  /notebooks           │  │
│  └───────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
            │  (optional — user-enabled)
            ▼
   ┌──────────────────────────────────────────────────┐
   │  External Connectors (all optional, all opt-in)  │
   │  NotebookLM Community API  ·  OpenAlex           │
   │  Semantic Scholar  ·  arXiv  ·  PubMed           │
   │  OpenRouter LLM routing                          │
   └──────────────────────────────────────────────────┘
```

---

## 2. Component Catalogue

### 2.1 Chat and Interaction Layer

| Component | Role | Port |
|---|---|---|
| **Open WebUI** | Web chat interface for Ollama-backed local models | 3000 |
| **Khoj Web** | Browser UI for semantic search across corpora | 42110 |
| **CLI runner** | Headless Python scripts; triggers SKILL.md agents, writes artefacts | — |

### 2.2 Orchestration Layer

| Component | Role |
|---|---|
| **LangChain** | Chains tools (vector search → LLM → output formatter); wraps academic API clients |
| **SKILL.md runner** | Bash/Python loader that reads a `SKILL.md` file and injects its rules into the active LLM session |
| **REST API hooks** | Lightweight FastAPI endpoints for inter-service calls and webhook triggers |

### 2.3 RAG Engines

| Component | Strengths | Use Case |
|---|---|---|
| **RAGFlow** | Deep layout analysis; understands multi-column PDFs, figures, tables | Primary ingestion and retrieval engine |
| **Khoj** | Multi-source parser; native Obsidian/Logseq plugin; fast semantic search | Ask Mode; Obsidian integration |

### 2.4 Document Parsing Pipeline

```
Raw file
   │
   ├─ PDF (scanned) ──► Tesseract 5 + PaddleOCR ──► text chunks
   ├─ PDF (digital) ──► RAGFlow layout parser ──────► text + figure metadata
   ├─ DOCX/PPTX/EPUB ► Unstructured.io ─────────────► text chunks
   ├─ MP3/MP4 ────────► Whisper (local) ─────────────► transcript text
   └─ YouTube URL ────► yt-dlp → Whisper ─────────────► transcript text
                                 │
                                 ▼
                      Chunking & Embedding
                      (sentence-transformers / Ollama embeddings)
                                 │
                        ┌────────┴──────────┐
                        ▼                   ▼
                  Chroma / Qdrant    NotebookLM API
                  (local index)      (optional cloud index)
```

### 2.5 Vector Store

| Option | Deployment | Best For |
|---|---|---|
| **Chroma** | Embedded Python process or Docker container | Single-researcher, simple setup |
| **Qdrant** | Docker container with persistent volume | Multi-user or high-volume corpora |

Both stores are scoped per research context folder. A folder switch reloads the corresponding index.

### 2.6 LLM Backbone (Bring Your Own Model)

OpenSana is model-agnostic. The LLM endpoint is configured via a single environment variable (`OPENSANA_LLM_BASE_URL`):

| Backend | How to Enable |
|---|---|
| **Ollama** | Set `LLM_BACKEND=ollama`; specify `OLLAMA_HOST` |
| **Llama.cpp** | Run `llama-server`; point `LLM_BASE_URL` to its OpenAI-compatible endpoint |
| **OpenRouter** | Set `LLM_BACKEND=openrouter`; provide `OPENROUTER_API_KEY` |
| **HuggingFace TGI/vLLM** | Set `LLM_BASE_URL` to the HuggingFace-compatible endpoint |

Recommended models (128 K+ context): `llama3.3:70b`, `qwen2.5:72b`, `mistral-large`.

### 2.7 External Connectors (all opt-in)

| Connector | Protocol | Data Returned |
|---|---|---|
| NotebookLM Community API | HTTPS REST | Multi-doc Q&A, passage citations, audio overview |
| OpenAlex | HTTPS REST | Open-access metadata, abstracts, citation graph |
| Semantic Scholar | HTTPS REST | Abstracts, citations, authors, TLDR summaries |
| arXiv API | HTTPS Atom/REST | Preprint metadata and PDF links |
| PubMed E-utilities | HTTPS REST | MEDLINE records, abstracts, MeSH terms |
| bioRxiv/medRxiv API | HTTPS REST | Preprint metadata and full text |
| ClinicalTrials.gov API | HTTPS REST | Trial metadata, results, interventions |
| ChEMBL REST | HTTPS REST | Compound, target, and bioactivity data |
| RCSB PDB API | HTTPS REST/GraphQL | Protein structure metadata |
| Open Targets GraphQL | HTTPS GraphQL | Drug–target evidence and disease associations |

---

## 3. Data Flow

### 3.1 Document Ingestion Flow

```
1. Researcher drops file into /library/<project-folder>/
2. ingest.py detects new file (inotify / polling)
3. File type router:
     • Audio/video → Whisper transcription → .txt saved alongside original
     • Scanned PDF → Tesseract/PaddleOCR → text layer injected
     • Other → pass-through
4. RAGFlow parses document layout → produces text chunks + figure metadata
5. Chunks embedded (sentence-transformers) → stored in Chroma/Qdrant (folder-scoped index)
6. If NOTEBOOKLM_ENABLED=true:
     → Document uploaded to NotebookLM notebook via Community API
     → Notebook corpus ID stored in /library/<project-folder>/.notebook_id
7. Ingestion summary written to /artifacts/<project-folder>/ingest_log.md
```

### 3.2 Agent Mode Query Flow

```
1. User invokes: python scripts/agent.py --skill skills/summarise.md --query "..."
2. SKILL.md loader reads rules from skills/summarise.md
3. Orchestrator builds prompt: system rules (from SKILL.md) + user query
4. If NotebookLM enabled:
     → Query sent to NotebookLM API with notebook corpus ID
     → Response includes answer + passage-level citations
5. If NotebookLM disabled (air-gap):
     → Query sent to RAGFlow/Khoj vector search
     → Retrieved chunks + query sent to local LLM
     → Response generated with chunk-level citations
6. Output formatted per SKILL.md rules (citation style, word limit, export format)
7. Artefact written to /artifacts/<project-folder>/<timestamp>_output.md
```

### 3.3 Ask Mode Query Flow

```
1. User submits query in Khoj Web or via CLI
2. Khoj performs semantic search across folder-scoped vector index
3. Top-k chunks retrieved and passed to local LLM (Ollama)
4. Response streamed back to UI or CLI
5. Response stored in /chats/<project-folder>/<timestamp>.md
```

---

## 4. Deployment Topology

### 4.1 Docker Compose Services

```yaml
services:
  ragflow:        # deep layout RAG engine
  khoj:           # multi-source semantic search
  chroma:         # local vector store (alt: qdrant)
  open-webui:     # Ollama chat frontend
  hedgedoc:       # collaborative Markdown editor
  grafana:        # optional monitoring dashboard
```

All services communicate over an internal Docker bridge network (`opensana_net`). No service port is exposed to the internet by default.

### 4.2 Minimum Hardware Requirements

| Resource | Minimum | Recommended |
|---|---|---|
| RAM | 16 GB | 32 GB |
| CPU | 8 cores | 16 cores |
| Storage | 50 GB SSD | 500 GB NVMe |
| GPU | None (CPU-only mode) | NVIDIA 24 GB VRAM (for local 70B models) |

### 4.3 Air-Gap Mode

Set the following in `.env` to disable all external network calls:

```env
NOTEBOOKLM_ENABLED=false
OPENROUTER_ENABLED=false
ACADEMIC_SEARCH_ENABLED=false
LLM_BACKEND=ollama
```

In this configuration all processing (OCR, transcription, embedding, inference) happens locally.

---

## 5. Security Considerations

| Concern | Mitigation |
|---|---|
| Data exfiltration | All external connectors are opt-in and disabled by default |
| API key exposure | Keys stored in `.env` (git-ignored); Docker secrets support planned for v2.0 |
| Telemetry | Zero outbound telemetry; Grafana dashboard is local-only |
| Inter-service trust | Services communicate over an isolated Docker bridge network; no public exposure |
| NotebookLM API availability | Platform degrades gracefully to RAGFlow + Ollama if the Community API becomes unavailable |

---

## 6. Technology Stack Summary

| Layer | Technology | License |
|---|---|---|
| Container runtime | Docker + Docker Compose | Apache 2.0 |
| RAG engine (primary) | RAGFlow | Apache 2.0 |
| RAG engine (secondary) | Khoj | AGPL 3.0 |
| Vector store | Chroma · Qdrant | Apache 2.0 |
| LLM serving | Ollama · Llama.cpp | MIT |
| OCR | Tesseract 5 · PaddleOCR | Apache 2.0 |
| Document parsing | Unstructured.io | Apache 2.0 |
| Audio transcription | Whisper (OpenAI OSS) | MIT |
| Orchestration | LangChain | MIT |
| Chat UI | Open WebUI | MIT |
| Collaborative editing | HedgeDoc | AGPL 3.0 |
| Code execution | Jupyter · Marimo | BSD · Apache 2.0 |
| Reference management | Zotero + Better BibTeX | AGPL 3.0 + MIT |
| Monitoring | Grafana | AGPL 3.0 |
| Notes workspace | Obsidian · Logseq | Proprietary (free) · AGPL 3.0 |
