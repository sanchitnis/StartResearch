# OpenSana — Skill Plugin Format

**Version:** 1.0  
**Status:** Draft  
**License:** Apache 2.0

---

## 1. What is a SKILL.md?

A `SKILL.md` is a plain Markdown file that encodes the behavioural rules for an OpenSana agent. It is:

- **Zero-dependency** — a `.md` text file; no imports, no binary, no package manager.
- **Portable** — paste into any LLM session (Claude, ChatGPT, Ollama, Khoj) and it works immediately.
- **Scoped** — placed in a research context folder to apply rules to everything inside that folder, or in `/skills/` for global slash-command invocation.
- **Composable** — skills can `@include` other skills to build compound behaviours.

---

## 2. File Locations

| Location | Scope | Invocation |
|---|---|---|
| `/library/<project-folder>/SKILL.md` | Folder-scoped — auto-loaded for every agent run in that folder | Automatic |
| `/skills/<name>.md` | Global — invokable from any folder | `/skill <name>` slash-command or `--skill skills/<name>.md` CLI flag |

---

## 3. SKILL.md Structure

A `SKILL.md` consists of one or more named sections separated by level-2 headings (`##`).

### Minimal example

```markdown
## SYSTEM

You are a scientific research assistant. Answer only from the provided documents.
Say "I don't know" if the answer is not in the corpus.
Cite every claim with author, year, and source document.
```

### Full example

```markdown
---
skill_name:   literature_review
version:      1.0
author:       research-team
description:  Systematic literature review assistant for oncology projects
---

## SYSTEM

You are a systematic review assistant specialising in oncology.
Answer only from the provided documents (grounded mode).
If a claim cannot be supported by the corpus, respond: "Not found in corpus."

## RULES

- CITATION_STYLE: Vancouver
- SUMMARY_LENGTH: 300 words maximum per answer
- LANGUAGE: English (British spelling)
- OUTPUT_FORMAT: Markdown with a ## Summary section and a ## References section
- CONFIDENCE: If retrieval score < 0.6, prefix answer with ⚠️ LOW CONFIDENCE

## SLASH_COMMANDS

| Command | Behaviour |
|---|---|
| `/summarise` | Produce a 200-word summary of the most relevant passages |
| `/compare`   | Compare findings across documents on a given topic |
| `/gaps`      | Identify research gaps not addressed in the corpus |
| `/bibtex`    | Export all cited sources as BibTeX entries |
| `/podcast`   | Generate an audio overview script (NotebookLM style) |

## TOOLS

- rag_retrieval: enabled
- notebooklm: enabled
- pubmed_search: enabled
- openalex_search: enabled
- code_execution: disabled

## OUTPUT

- write_artefact: true
- artefact_path:  artifacts/{project}/{timestamp}_{skill_name}.md
- export_formats: [markdown, docx, bibtex]
```

---

## 4. Section Reference

### `## SYSTEM`

The system prompt injected at the top of every LLM call. Keep it concise (under 500 tokens). This is the most important section.

### `## RULES`

Key–value pairs that configure agent behaviour. Recognised keys:

| Key | Values | Default |
|---|---|---|
| `CITATION_STYLE` | `Vancouver`, `APA`, `Harvard`, `Chicago`, `MLA`, `None` | `APA` |
| `SUMMARY_LENGTH` | Integer (words) | `500` |
| `LANGUAGE` | Any IETF language tag (e.g. `en-GB`, `fr-FR`, `de-DE`) | `en` |
| `OUTPUT_FORMAT` | `Markdown`, `Plain`, `LaTeX`, `JSON` | `Markdown` |
| `CONFIDENCE` | `strict` (refuse low-confidence answers), `warn` (prefix with ⚠️), `silent` | `warn` |
| `GROUNDED_ONLY` | `true` / `false` | `true` |

### `## SLASH_COMMANDS`

Maps `/command` strings to agent behaviours. Commands are available in the chat UI (Open WebUI, Khoj Web) and via the `--command` CLI flag.

### `## TOOLS`

Declares which tools the agent may use. Setting a tool to `disabled` prevents it from being called even if globally available.

Recognised tool keys: `rag_retrieval`, `notebooklm`, `pubmed_search`, `openalex_search`, `arxiv_search`, `chembl_search`, `pdb_search`, `clinical_trials_search`, `code_execution`, `audio_transcription`.

### `## OUTPUT`

Controls where and how artefacts are written.

| Key | Description |
|---|---|
| `write_artefact` | `true` / `false` — whether to save output to disk |
| `artefact_path` | Path template; supports `{project}`, `{timestamp}`, `{skill_name}` |
| `export_formats` | List of formats: `markdown`, `docx`, `xlsx`, `bibtex`, `latex`, `mermaid` |

### `## @include`

Compose skills from reusable fragments:

```markdown
## @include

- skills/base_system.md
- skills/vancouver_citations.md
```

Included files are merged in order; later sections override earlier ones.

---

## 5. Slash-Command Invocation

### From the chat UI (Open WebUI / Khoj Web)

Type a slash-command in the chat input:

```
/summarise What are the main findings about compound X?
```

The platform resolves the command against the active folder's `SKILL.md` and the global `/skills/index.md`.

### From the CLI

```bash
python scripts/agent.py \
  --skill   skills/literature_review.md \
  --command summarise \
  --folder  library/my-project \
  --query   "What are the main findings about compound X?"
```

### From a script

```python
from opensana.skill import load_skill, run_command

skill  = load_skill("skills/literature_review.md")
result = run_command(skill, command="summarise", query="Main findings on compound X?",
                     folder="library/my-project")
print(result.markdown)
```

---

## 6. Per-Folder vs. Global Skills

### Per-folder `SKILL.md`

Place `SKILL.md` directly in a research context folder:

```
library/
└── oncology-project/
    ├── SKILL.md          ← auto-loaded for all agent runs in this folder
    ├── paper1.pdf
    └── paper2.pdf
```

The folder-level `SKILL.md` takes priority over global skills on every key it defines.

### Global skills in `/skills/`

```
skills/
├── index.md              ← lists all available slash-commands
├── literature_review.md
├── daily_summary.md
├── bibtex_export.md
└── podcast_script.md
```

`index.md` maps slash-command names to skill files so the chat UI can autocomplete them.

---

## 7. Example Skills

### `skills/summarise.md` — Quick summary

```markdown
## SYSTEM

You are a concise scientific summariser.
Summarise the key findings in the provided documents in plain language.
Use no more than 200 words. Cite every claim with [Author Year].

## RULES

- CITATION_STYLE: APA
- SUMMARY_LENGTH: 200
- GROUNDED_ONLY: true
- OUTPUT_FORMAT: Markdown

## OUTPUT

- write_artefact: true
- artefact_path:  artifacts/{project}/{timestamp}_summary.md
```

### `skills/vancouver_clinical.md` — Clinical review (Vancouver citations)

```markdown
---
skill_name: vancouver_clinical
description: Clinical literature review with Vancouver-style citations
---

## SYSTEM

You are a clinical research assistant. Answer only from the provided documents.
Format every citation in Vancouver style: [1] Author A, Author B. Title. Journal. Year;Vol(Issue):Pages.
Structure responses with: Background · Findings · Clinical Implications · References.

## RULES

- CITATION_STYLE: Vancouver
- LANGUAGE: en-GB
- GROUNDED_ONLY: true
- OUTPUT_FORMAT: Markdown
```

### `skills/bibtex_export.md` — Export references

```markdown
## SYSTEM

You are a reference management assistant.
Extract all citations from the provided documents and format them as BibTeX entries.
Output only valid BibTeX; do not include any explanatory text.

## RULES

- OUTPUT_FORMAT: Plain
- GROUNDED_ONLY: true

## OUTPUT

- write_artefact: true
- artefact_path:  artifacts/{project}/{timestamp}_references.bib
- export_formats: [bibtex]
```

---

## 8. Best Practices

1. **Keep `## SYSTEM` under 500 tokens.** Longer system prompts consume context and may degrade response quality.
2. **Set `GROUNDED_ONLY: true` by default.** Disable it only when you explicitly want the model to reason beyond the corpus.
3. **Use per-folder `SKILL.md` for project-specific rules** (citation style, domain vocabulary) and global `/skills/` for reusable tasks (summarise, export, search).
4. **Version your skills with `version:` in the front-matter.** This makes it easy to roll back if a skill change produces unexpected results.
5. **Test skills with a small query before running batch jobs.** Use `--dry-run` to preview the prompt without calling the LLM.
6. **Avoid hardcoding absolute paths** in `artefact_path`. Use template variables (`{project}`, `{timestamp}`) so skills are portable across machines.
7. **Document slash-commands in `/skills/index.md`** so team members can discover available commands without reading individual skill files.
