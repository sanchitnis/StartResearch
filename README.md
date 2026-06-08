# OpenSana — Open-Source Scientific Research AI Platform

> **Self-hosted · Privacy-first · BYOM (Bring Your Own Model)**

OpenSana is an open-source scientific research AI platform that gives researchers full control over their data, models, and workflows. Built on a composable stack of open-source tools, it provides multi-document synthesis, passage-level citations, agentic automation, and seamless knowledge organisation — all without sending a single document to a third-party cloud.

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

---

## ✨ Key Features

| Feature | Details |
|---|---|
| **Multi-document Q&A** | NotebookLM Community API — upload entire corpora, query with passage-level citations |
| **Local LLM backbone** | Ollama · Llama.cpp · OpenRouter · HuggingFace endpoints (plug in any model) |
| **Deep-layout RAG** | RAGFlow parses figures, tables, and complex PDFs; Khoj adds multi-source semantic search |
| **Agentic skills** | Plain Markdown `SKILL.md` files — no framework lock-in, paste into any LLM session |
| **Audio/video ingestion** | Whisper (local) transcribes MP3, MP4, and YouTube URLs to searchable text |
| **Full air-gap mode** | Disable external APIs; run entirely on RAGFlow + Ollama + Chroma + Khoj |
| **100 % data autonomy** | Documents never leave the host; zero third-party telemetry |

---

## 📚 Documentation

| Document | Purpose |
|---|---|
| [SPEC.md](SPEC.md) | Functional specification — features, requirements, and acceptance criteria |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System architecture — components, data flows, and deployment topology |
| [AGENTS.md](AGENTS.md) | Agent capabilities — orchestration, academic connectors, and agentic execution |
| [SKILL.md](SKILL.md) | Skill plugin format — how to write, organise, and invoke `SKILL.md` files |

---

## 🚀 Quick Start

### Prerequisites

- Docker & Docker Compose ≥ 2.20
- 16 GB RAM · 8-core CPU · 50 GB free SSD
- (Optional) NVIDIA GPU with CUDA for Whisper acceleration and local LLM inference

### 1 — Clone and configure

```bash
git clone https://github.com/sanchitnis/StartResearch.git
cd StartResearch
cp .env.example .env          # edit API keys / model endpoints as needed
```

### 2 — Start all services

```bash
docker compose up -d
```

Services started:

| Service | URL | Purpose |
|---|---|---|
| Open WebUI | http://localhost:3000 | Chat interface for Ollama models |
| RAGFlow | http://localhost:8000 | Deep-layout document RAG |
| Khoj | http://localhost:42110 | Multi-source semantic search |
| Chroma | http://localhost:8001 | Local vector store |
| HedgeDoc | http://localhost:3001 | Collaborative Markdown editor |

### 3 — Upload documents

Drop files into a research context folder and run the ingestion script:

```bash
python scripts/ingest.py --folder ./library/my-project
```

Supported formats: `PDF · DOCX · PPTX · EPUB · TXT · MD · MP3 · MP4 · YouTube URL`

### 4 — Query your corpus

```bash
# Ask Mode — single-document RAG via Khoj
python scripts/ask.py "What is the mechanism of action of compound X?"

# Agent Mode — multi-document synthesis via NotebookLM API + SKILL.md
python scripts/agent.py --skill skills/summarise.md --query "Summarise findings across all uploaded papers"
```

---

## 🗂 Repository Layout

```
StartResearch/
├── README.md            ← this file
├── SPEC.md              ← functional specification
├── ARCHITECTURE.md      ← system architecture
├── AGENTS.md            ← agent capabilities guide
├── SKILL.md             ← skill plugin format guide
├── docker-compose.yml   ← single-file container stack
├── .env.example         ← environment variable template
├── scripts/             ← ingestion, query, and agent runner scripts
├── skills/              ← named SKILL.md agent definitions
├── library/             ← research context folders (git-ignored)
├── chats/               ← chat history as Markdown files (git-ignored)
└── artifacts/           ← agent-generated outputs (git-ignored)
```

---

## 🔒 Privacy & Air-Gap Mode

OpenSana is designed for researchers who handle sensitive data. Every component can run fully offline:

1. Set `NOTEBOOKLM_ENABLED=false` in `.env`
2. Point the LLM backbone to a local Ollama instance
3. Disable `OPENROUTER_ENABLED` and all external academic search connectors

In air-gap mode the only network calls are within your own LAN.

---

## 🤝 Contributing

Contributions are welcome! Please read the [contributing guide](CONTRIBUTING.md) and open a pull request.

---

## 📄 License

Apache 2.0 — see [LICENSE](LICENSE) for details. All core components are open-source or open-weight.