# OpenSana — Agents Guide

**Version:** 1.0  
**Status:** Draft  
**License:** Apache 2.0

---

## 1. What is an OpenSana Agent?

An OpenSana agent is a lightweight, stateless automation unit that:

1. Reads its behavioural rules from a `SKILL.md` file on the filesystem.
2. Accepts a user query or a trigger event (file drop, cron job, webhook).
3. Calls one or more tools (RAG retrieval, LLM inference, academic API, code execution).
4. Writes a structured Markdown artefact back to the research context folder.

Agents require no GUI, no persistent process, and no proprietary framework. Any Python script that reads a `SKILL.md` and calls an LLM endpoint qualifies as an agent.

---

## 2. Agent Execution Modes

### 2.1 Agent Mode (multi-document synthesis)

Powered by the **NotebookLM Community API** when enabled, or **RAGFlow + local LLM** in air-gap mode.

```
python scripts/agent.py \
  --skill   skills/lit_review.md \
  --folder  library/my-project \
  --query   "Summarise the evidence for compound X in glioblastoma"
```

Behaviour:

- Loads `SKILL.md` rules from `skills/lit_review.md`.
- Sends query to NotebookLM with the folder's notebook corpus ID.
- Receives answer with **passage-level citations**.
- Formats output according to rules (citation style, word limit, export format).
- Saves artefact to `artifacts/my-project/<timestamp>_lit_review.md`.

### 2.2 Ask Mode (single-folder RAG)

Powered by **Khoj** semantic search + local LLM.

```
python scripts/ask.py \
  --folder library/my-project \
  --query  "What sample size was used in the RCT by Smith et al. 2023?"
```

Behaviour:

- Performs vector similarity search across the folder-scoped Chroma/Qdrant index.
- Passes top-k chunks to the local LLM.
- Returns answer with **chunk-level source references**.
- Optionally appends result to `chats/my-project/session.md`.

### 2.3 Headless Batch Mode

Run agents without any user interaction, suitable for scheduled pipelines:

```bash
# Nightly summary agent triggered by cron
0 2 * * * python /opt/opensana/scripts/agent.py \
  --skill  skills/daily_summary.md \
  --folder library/my-project \
  --query  "What new papers were ingested today? Summarise findings." \
  >> logs/cron.log 2>&1
```

---

## 3. Orchestration

Agents are orchestrated via **LangChain tool chains** or bare Python, depending on complexity.

### 3.1 Simple (bare Python)

```python
from opensana.skill import load_skill
from opensana.llm   import query_llm
from opensana.rag   import retrieve_chunks

skill   = load_skill("skills/summarise.md")
chunks  = retrieve_chunks(folder="library/my-project", query=user_query, top_k=10)
answer  = query_llm(system_prompt=skill.system_prompt, context=chunks, query=user_query)
```

### 3.2 Compound (LangChain)

```python
from langchain.agents import initialize_agent
from opensana.tools   import RAGTool, AcademicSearchTool, NotebookLMTool

tools = [
    RAGTool(folder="library/my-project"),
    AcademicSearchTool(sources=["pubmed", "openalex"]),
    NotebookLMTool(notebook_id=".notebook_id"),
]
agent = initialize_agent(tools, llm, agent="zero-shot-react-description")
agent.run(user_query)
```

---

## 4. Academic Search Connectors

All connectors are implemented as LangChain tools and standalone Python clients. They are disabled by default and enabled per-folder in the `SKILL.md`.

### 4.1 General Academic

| Connector | Python Client | Free Tier |
|---|---|---|
| **OpenAlex** | `pyalex` | Unlimited (polite pool) |
| **Semantic Scholar** | `semanticscholar` | 100 req/5 min unauthenticated |
| **arXiv** | `arxiv` | Unlimited |
| **JSTOR** | REST API | Registered researcher access |

Example — arXiv search tool:

```python
import arxiv

def search_arxiv(query: str, max_results: int = 10) -> list[dict]:
    client  = arxiv.Client()
    results = client.results(arxiv.Search(query=query, max_results=max_results))
    return [{"title": r.title, "abstract": r.summary, "url": r.pdf_url} for r in results]
```

### 4.2 Life-Science Connectors

| Connector | Endpoint | Data |
|---|---|---|
| **PubMed E-utilities** | `https://eutils.ncbi.nlm.nih.gov/entrez/eutils/` | MEDLINE records, abstracts, MeSH |
| **bioRxiv/medRxiv** | `https://api.biorxiv.org/` | Preprint metadata, full text |
| **ClinicalTrials.gov** | `https://clinicaltrials.gov/api/v2/` | Trial records, results, arms |
| **ChEMBL REST** | `https://www.ebi.ac.uk/chembl/api/data/` | Compounds, targets, bioactivity |
| **RCSB PDB** | `https://data.rcsb.org/` | Protein structures, sequences |
| **Open Targets** | `https://api.platform.opentargets.org/api/v4/graphql` | Drug–target evidence |

Example — PubMed search tool:

```python
from Bio import Entrez

Entrez.email = "researcher@example.com"

def search_pubmed(query: str, max_results: int = 20) -> list[dict]:
    handle  = Entrez.esearch(db="pubmed", term=query, retmax=max_results)
    ids     = Entrez.read(handle)["IdList"]
    handle  = Entrez.efetch(db="pubmed", id=ids, rettype="abstract", retmode="text")
    return handle.read()
```

---

## 5. Agent Output Artefacts

Every agent run produces a structured Markdown artefact in `/artifacts/<project-folder>/`:

```markdown
---
agent:    lit_review
skill:    skills/lit_review.md
query:    "Summarise evidence for compound X in glioblastoma"
model:    llama3.3:70b
date:     2026-06-08T09:00:00Z
sources:  [doc1.pdf §3.2, doc7.pdf §5.1, doc12.pdf abstract]
---

## Summary

Compound X demonstrated a statistically significant reduction in tumour volume
in three Phase II RCTs (Smith 2022 [1], Patel 2023 [2], Lee 2024 [3]).

## Citations

[1] Smith J et al. *Lancet Oncol.* 2022;23(4):451–462. (doc1.pdf §3.2)
[2] Patel R et al. *J Neurooncol.* 2023;161:77–88. (doc7.pdf §5.1)
[3] Lee S et al. *Neuro Oncol.* 2024;26:200–211. (doc12.pdf abstract)
```

---

## 6. Hallucination Mitigation

OpenSana applies grounded-only generation by default:

| Mechanism | How It Works |
|---|---|
| **NotebookLM grounding parameter** | `grounded_only=true` in API call — model refuses to answer from outside the corpus |
| **RAGFlow chunk attribution** | Every chunk carries source document and page number; LLM is instructed to cite only provided chunks |
| **SKILL.md rule injection** | `SKILL.md` can include `RULE: Only answer from the provided context. Say "I don't know" if the answer is not in the corpus.` |
| **Confidence threshold** | RAG retrieval score threshold configurable; low-confidence chunks excluded from context |

---

## 7. Inter-Agent Communication

Agents communicate by writing and reading Markdown files — no message broker required:

```
/library/my-project/
├── .notebook_id           ← corpus ID written by ingest agent
├── SKILL.md               ← rules read by all agents in this context
├── queue/
│   └── tasks.md           ← pending task list (append-only)
└── /artifacts/
    ├── 20260608_090000_lit_review.md
    └── 20260608_093000_daily_summary.md
```

A supervisor agent can read `tasks.md`, dispatch child agents, and mark tasks complete — all through filesystem operations.

---

## 8. Adding a New Agent

1. Create a new `SKILL.md` in `/skills/` (see [SKILL.md](SKILL.md) for format).
2. Add any required tool imports to `scripts/agent.py` or create a new script.
3. Test locally:
   ```bash
   python scripts/agent.py --skill skills/my_new_agent.md --folder library/test --query "test query"
   ```
4. Add a slash-command alias to `/skills/index.md` if the agent should be invokable from the chat UI.

No framework changes, no configuration files, no restarts required.
