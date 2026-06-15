# OneLake → AI Search notebooks

End-to-end pipelines that index files from a Microsoft Fabric Lakehouse into Azure AI Search, with the Content Understanding skill for page-aware extraction.

## Setup

```powershell
# from the repo root
cd notebooks

# 1. create + activate a virtual environment
python -m venv .venv
.\.venv\Scripts\Activate.ps1

# 2. install dependencies
pip install --upgrade pip
pip install -r requirements.txt

# 3. register the venv as a Jupyter kernel (so the notebook can pick it)
python -m ipykernel install --user --name fti-onelake --display-name "Python (fti-onelake)"

# 4. copy and fill in env vars
Copy-Item .env.example .env
notepad .env

# 5. launch
jupyter lab    # or: code .  and open the .ipynb in VS Code
```

In the notebook, select the **Python (fti-onelake)** kernel.

## Notebooks

Three pipeline variants. They share env vars (see `.env.example`) and the venv setup above. Pick one based on whether you need image descriptions and whether you can use preview APIs.

| Notebook | API track | Image descriptions | When to use |
|---|---|---|---|
| `onelake_content_understanding_indexer.ipynb` | **GA** | ❌ none | Production; you only need text + page numbers + tables (Markdown). Images come through as figure regions with OCR'd text inside, no AI-generated description. |
| `onelake_content_understanding_genai_verbalization.ipynb` | **GA** | ✅ via GenAI Prompt skill | Production with multimodal RAG. Chains the GA `GenAIPromptSkill` over CU's `normalized_images` to verbalize each figure. Two index projections: one per text chunk, one per figure, both in the same index, distinguished by a `kind` field (`text` / `figure`). |
| `onelake_content_understanding_indexer_preview_verbalization.ipynb` | 🧪 **Preview** (`2026-05-01-preview`) | ✅ inline via CU | Demos and experimentation. Uses CU's built-in `modelName` / `modelDeployment` parameters to inline figure descriptions inside `text_sections[*].content` Markdown — single skill, single API call. Subject to preview SLA. |

## Verbalization: which path?

The two GA-vs-preview verbalization notebooks produce **functionally equivalent retrieval quality**. The difference is plumbing:

|  | GA chain (`..._genai_verbalization.ipynb`) | Preview inline (`..._preview_verbalization.ipynb`) |
|---|---|---|
| Skills in skillset | 4 (CU + GenAI Prompt + 2× embedding) | 2 (CU + 1× embedding) |
| Description location | Separate search docs with `kind = "figure"` | Inlined into chunk Markdown |
| API version | `2026-04-01` GA | `2026-05-01-preview` |
| Production support | ✅ | ❌ preview SLA |
| Custom prompt control | ✅ explicit `systemPrompt` / `userPrompt` | ❌ implicit |
| Pricing | CU + AOAI chat tokens (per figure) | CU pays for the chat call internally |
| Semantic chunking (token-based) | ❌ (fixed-size only on GA) | ✅ |

**Decision rule:** if image descriptions matter and you're shipping to prod, use the GA chain. If you want the simplest skillset and don't mind preview, use the preview inline. If you don't care about image descriptions at all, use the base GA notebook.

## Files

| File | Purpose |
|---|---|
| `onelake_content_understanding_indexer.ipynb` | GA pipeline, no image verbalization |
| `onelake_content_understanding_genai_verbalization.ipynb` | GA pipeline, image verbalization via GenAI Prompt skill chain |
| `onelake_content_understanding_indexer_preview_verbalization.ipynb` | Preview pipeline, image verbalization inlined by CU itself |
| `requirements.txt` | Python dependencies for the venv |
| `.env.example` | Template for the env vars the notebooks read. Copy to `.env` and fill in. |

## macOS / Linux equivalents

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python -m ipykernel install --user --name fti-onelake --display-name "Python (fti-onelake)"
cp .env.example .env
```

## Notes

- The `%pip install` cell in the notebook is redundant once the venv is set up — leave it for users running in Colab / Codespaces, or comment it out.
- `azure-search-documents` is pinned to a **beta** because GA does not yet expose the integrated-vectorization query types this notebook uses. Re-pin to GA once Microsoft promotes the feature.
