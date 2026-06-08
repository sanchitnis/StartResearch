# OpenSana — Functional Specification

**Version:** 1.0  
**Status:** Draft  
**License:** Apache 2.0

---

## 1. Purpose and Scope

OpenSana is a self-hosted, open-source scientific research AI platform. This specification defines the functional requirements, user stories, and acceptance criteria for the platform's initial release (v1.0).

The platform targets individual researchers, small labs, and academic institutions that require:

- Full data sovereignty (no documents sent to external clouds).
- Multi-document synthesis with passage-level citations.
- Flexible, model-agnostic AI (Bring Your Own Model).
- Programmatic agentic automation via lightweight Markdown skill files.

---

## 2. Definitions

| Term | Definition |
|---|---|
| **Corpus** | A collection of documents uploaded to a single research context (folder). |
| **Research Context** | A filesystem folder containing documents, a `SKILL.md`, and a vector store index. |
| **SKILL.md** | A plain Markdown file that encodes prompt rules and slash-command behaviour for a research context. |
| **Notebook** | A NotebookLM Community API notebook object; maps 1-to-1 with a research context. |
| **Agent Mode** | Multi-document synthesis via NotebookLM API + SKILL.md orchestration. |
| **Ask Mode** | Single-document or folder-scoped RAG query via Khoj. |
| **Air-Gap Mode** | Full offline operation; NotebookLM API and all external connectors disabled. |

---

## 3. Functional Requirements

### 3.1 Document Ingestion and Parsing

| ID | Requirement | Priority |
|---|---|---|
| ING-01 | The system shall accept PDF, DOCX, PPTX, EPUB, TXT, and MD files. | Must |
| ING-02 | The system shall accept MP3 and MP4 audio/video files and transcribe them locally via Whisper. | Must |
| ING-03 | The system shall accept YouTube URLs and transcribe the video locally via Whisper. | Should |
| ING-04 | Scanned PDFs and handwritten notes shall be OCR-processed via Tesseract 5 and PaddleOCR. | Must |
| ING-05 | Figures, tables, UMAP plots, and gel images shall be extracted via Unstructured.io. | Should |
| ING-06 | Documents shall be indexed in the local vector store (Chroma or Qdrant) upon ingestion. | Must |
| ING-07 | Documents shall be uploaded to a NotebookLM notebook via the Community API upon ingestion. | Must |
| ING-08 | Upload capacity shall be bounded only by host storage and VRAM, not by an artificial document limit. | Must |

### 3.2 Knowledge Organisation

| ID | Requirement | Priority |
|---|---|---|
| ORG-01 | Each research context shall be a filesystem folder containing documents and a `SKILL.md`. | Must |
| ORG-02 | Each folder shall have its own scoped vector store index. | Must |
| ORG-03 | A `SKILL.md` file shall encode per-folder prompt rules (citation style, summary length, language, etc.). | Must |
| ORG-04 | The platform shall support Obsidian and Logseq as Markdown note-taking workspaces. | Should |
| ORG-05 | LaTeX math blocks (`$$ ... $$`) shall be rendered natively in Obsidian, Logseq, and Jupyter outputs. | Should |
| ORG-06 | Zotero with Better BibTeX shall auto-export `.bib` files to `/library` for agent ingestion. | Could |

### 3.3 AI Research Agent Capabilities

| ID | Requirement | Priority |
|---|---|---|
| AGT-01 | Agent Mode shall query the full uploaded corpus via the NotebookLM Community API. | Must |
| AGT-02 | Agent Mode shall return passage-level citations for every claim. | Must |
| AGT-03 | The grounded-only mode shall constrain responses exclusively to the uploaded corpus. | Must |
| AGT-04 | The platform shall support external academic search via OpenAlex, Semantic Scholar, arXiv, and JSTOR APIs. | Should |
| AGT-05 | The platform shall support life-science connectors: PubMed, bioRxiv/medRxiv, ClinicalTrials.gov, ChEMBL, RCSB PDB, Open Targets. | Should |
| AGT-06 | SKILL.md agents shall be invocable via Bash/Python scripts without a GUI. | Must |
| AGT-07 | Agent outputs shall be written back to the research context folder as Markdown artefacts. | Must |
| AGT-08 | The platform shall support context windows of 128 K+ tokens with compatible local models (e.g. Llama 3.3, Qwen 2.5). | Should |

### 3.4 Chat Interaction

| ID | Requirement | Priority |
|---|---|---|
| CHT-01 | The platform shall provide Ask Mode: single-document RAG via Khoj. | Must |
| CHT-02 | The platform shall provide Agent Mode: multi-document synthesis via NotebookLM API + SKILL.md. | Must |
| CHT-03 | Slash-commands shall map to named SKILL.md files in the `/skills` folder. | Must |
| CHT-04 | Chat history shall be stored as Markdown files in `/chats` and be searchable via grep or Obsidian full-text search. | Must |
| CHT-05 | The platform shall provide a web-based chat interface via Open WebUI. | Must |
| CHT-06 | A headless CLI mode shall be available for automated agent runs. | Must |

### 3.5 Outputs and Export

| ID | Requirement | Priority |
|---|---|---|
| OUT-01 | Agent-generated scripts shall execute locally in Jupyter Notebook or Marimo; outputs saved to `/artifacts`. | Must |
| OUT-02 | The platform shall export to: Markdown, `.docx`, `.xlsx`, LaTeX blocks, Mermaid mindmaps, BibTeX. | Must |
| OUT-03 | Audio overview generation shall be available via the NotebookLM Community API (multi-speaker podcast). | Should |
| OUT-04 | A local TTS fallback (Kokoro) shall be available when the NotebookLM API is unavailable. | Could |
| OUT-05 | Real-time collaborative Markdown editing shall be available via HedgeDoc or CodiMD. | Should |

### 3.6 Deployment and Operations

| ID | Requirement | Priority |
|---|---|---|
| DEP-01 | The entire stack shall be deployable via a single `docker compose up -d` command. | Must |
| DEP-02 | Minimum hardware requirements: 16 GB RAM, 8-core CPU, 50 GB SSD. | Must |
| DEP-03 | GPU acceleration (CUDA) shall be optional; the platform shall run on CPU-only hardware. | Must |
| DEP-04 | Full air-gap mode shall be supported by disabling NotebookLM and all external connectors. | Must |
| DEP-05 | The platform shall emit zero third-party telemetry. | Must |
| DEP-06 | Optional Grafana dashboard shall display folder activity, query logs, and agent run stats using local telemetry only. | Could |

---

## 4. Non-Functional Requirements

| Category | Requirement |
|---|---|
| **Privacy** | All documents processed locally; no data sent externally unless the user explicitly enables external connectors. |
| **Portability** | Runs on Linux, macOS, and Windows (via Docker); bare-metal Python installation also supported. |
| **Extensibility** | New agent skills are added by dropping a `.md` file into `/skills`; no code changes required. |
| **Resilience** | If the NotebookLM API becomes unavailable, the platform falls back to RAGFlow + Ollama automatically. |
| **Observability** | All agent runs produce structured Markdown logs in `/artifacts`; optional Grafana integration for dashboards. |
| **License compliance** | All bundled components are Apache 2.0, MIT, or equivalent open-source licenses. |

---

## 5. User Stories

### Researcher (primary persona)

- As a researcher, I want to upload 50+ PDFs and ask cross-paper questions with citations, so I can synthesise literature without reading every paper manually.
- As a researcher, I want my documents to stay on my own server, so I comply with my institution's data governance policy.
- As a researcher, I want to define custom prompt rules per project folder, so the AI always cites in Vancouver style for my clinical project and APA for my social science project.
- As a researcher, I want to transcribe a conference recording and include it in my literature corpus, so I can query spoken content alongside papers.

### Lab IT administrator (secondary persona)

- As an IT admin, I want to deploy the full stack with one command, so I can onboard a new research group in under an hour.
- As an IT admin, I want a fully offline mode, so I can run the platform in a classified network environment.
- As an IT admin, I want optional Grafana monitoring, so I can track resource usage and query volumes without installing third-party agents.

---

## 6. Out of Scope (v1.0)

- Multi-tenant user authentication and role-based access control (planned for v2.0).
- Fine-tuning or training of local models.
- Integration with commercial reference management tools (e.g. EndNote, RefWorks).
- Mobile application.

---

## 7. Acceptance Criteria

A release is considered complete when:

1. All **Must** requirements in Section 3 pass integration tests.
2. `docker compose up -d` brings up all services on a clean Ubuntu 22.04 machine meeting minimum hardware spec.
3. A 20-document corpus can be ingested, queried in Agent Mode, and returns passage-level citations.
4. Air-gap mode operates without any outbound network requests (verified via network capture).
5. A SKILL.md slash-command successfully invokes a custom agent and writes a Markdown artefact to `/artifacts`.
