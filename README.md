# 📚 Board Meeting RAG

A four-notebook RAG pipeline that turns a pile of board meeting documents (PDF + DOCX) into a queryable corpus. Parses, chunks, indexes with Databricks Vector Search, summarizes per quarter with Claude, and answers natural-language questions with inline citations grounded in the source material.

The example corpus shipped with this repo is **publicly published board meeting materials from US public pension funds** (SFERS, SWIB, VPIC, and others — see the entity registry in `01_Ingest.ipynb`). These institutions release their board packets publicly, which makes them an ideal demo corpus: real, unstructured, mixed PDF/DOCX, and reproducible without any private data.

The pattern adapts to any rollup's internal document pile (board minutes, diligence files, customer service transcripts, technician notes, contracts) by swapping the source Volume and the entity registry.

## 🧩 Pipeline

```
   PDFs / DOCX in Volume                    Q&A with citations
            │                                       ▲
            ▼                                       │
  ┌─────────────────┐    ┌──────────────────┐    ┌──────────┐
  │ 01_Ingest       │ →  │ 02_Build_Index   │ →  │ 04_Query │
  │ parse + chunk   │    │ vector index     │    │ retrieve │
  │ → Delta table   │    │ over chunks      │    │ + answer │
  └─────────────────┘    └──────────────────┘    └──────────┘
            │
            ▼
  ┌─────────────────────────────┐
  │ 03_Summarize_And_Post       │
  │ per-entity quarterly digest │
  │ + audit table               │
  └─────────────────────────────┘
```

| notebook | what it does |
|---|---|
| `01_Ingest.ipynb` | Walks the raw Volume, parses PDFs (one chunk per page) and DOCX (one chunk per `Month YYYY` heading), MERGEs into the chunks Delta table on a deterministic `chunk_id`. Idempotent. |
| `02_Build_Index.ipynb` | Creates (or syncs) a Delta Sync Vector Search index over `content`. Uses server-side embeddings via `databricks-gte-large-en` so we never compute or store vectors locally. |
| `03_Summarize_And_Post.ipynb` | Pulls every chunk for a given quarter, groups by entity, and asks Claude Sonnet 4.6 for a neutral 4–7 bullet summary per entity. Writes Markdown to a Volume + appends to a Delta audit table. |
| `04_Query.ipynb` | Natural-language Q&A. Retrieves the top-K chunks (with optional quarter / entity filters) and asks Claude Sonnet 4.6 to answer using only those chunks, with citations like `[SMIB p14]` or `[CRPTF March 2026]`. |

## 🧱 Stack

- **Databricks Serverless Notebook Compute** (or any cluster with the Foundation Model API enabled)
- **PySpark + Delta Lake** for the chunks table and audit table
- **Databricks Vector Search** with Delta Sync indexes
- **`databricks-gte-large-en`** for server-side embeddings
- **`databricks-claude-sonnet-4-6`** for summaries and Q&A (swap for any served chat model)
- **`pypdf` + `python-docx`** for parsing
- **Unity Catalog Volumes** for the raw document store

No external API calls. All inference and embedding stay in the workspace.

## 📋 Unity Catalog layout

The notebooks use `acme_holdings.<schema>.<table>` as a placeholder namespace. Swap to your own catalog when adapting.

### Tables

**`acme_holdings.documents.board_meetings_chunks`** — one row per chunk

| column | type | notes |
|---|---|---|
| `chunk_id` | string | deterministic: `<filestem>:<page>:<ordinal>:<sha1[:10]>` |
| `source_file` | string | filename of the original document |
| `entity` | string | short code from the entity registry (e.g., `SFERS`) |
| `meeting_date` | date | parsed from filename or DOCX heading; nullable |
| `quarter` | string | `YYYYQn` |
| `doc_type` | string | `pdf` or `docx` |
| `page_number` | int | nullable for DOCX |
| `section` | string | DOCX section heading (`October 2025`, etc.); nullable for PDF |
| `content` | string | the chunk text — what gets embedded |
| `ingested_at` | timestamp | |

**`acme_holdings.documents.board_meeting_summaries`** — audit log for `03_Summarize_And_Post`

| column | type |
|---|---|
| `quarter` | string |
| `generated_at` | timestamp |
| `entity_summaries` | string (JSON) |
| `linkedin_post` | string (nullable) |
| `model_post` | string (nullable) |
| `model_summary` | string |

### Vector index

**`acme_holdings.embeddings.board_meetings_index`** — Delta Sync index, primary key `chunk_id`, embedding source `content`, served by endpoint `acme_rag_endpoint`.

### Volume

```
/Volumes/acme_holdings/documents/board_meetings_raw/
    2026Q1/
        SFERS_Retirement_Board_2026-03-11.pdf
        SWIB_Board_Meeting_2026-03-18.pdf
        VPIC_Board_Meeting_2026-03-24.pdf
        ...
    2026Q2/
        ...
    _output/
        summary_2026Q1.md
        summary_2026Q2.md
```

The folder name (`YYYYQn`) is read by the ingest notebook to assign a quarter.

## 🚀 Quick start

1. **Create the Unity Catalog scaffolding**:
   ```sql
   CREATE CATALOG IF NOT EXISTS acme_holdings;
   CREATE SCHEMA  IF NOT EXISTS acme_holdings.documents;
   CREATE SCHEMA  IF NOT EXISTS acme_holdings.embeddings;
   CREATE VOLUME  IF NOT EXISTS acme_holdings.documents.board_meetings_raw;

   CREATE TABLE IF NOT EXISTS acme_holdings.documents.board_meetings_chunks (
     chunk_id     STRING NOT NULL,
     source_file  STRING NOT NULL,
     entity       STRING,
     meeting_date DATE,
     quarter      STRING,
     doc_type     STRING,
     page_number  INT,
     section      STRING,
     content      STRING NOT NULL,
     ingested_at  TIMESTAMP
   ) USING DELTA TBLPROPERTIES (delta.enableChangeDataFeed = true);
   ```
   `delta.enableChangeDataFeed = true` is required for Delta Sync vector indexes.

2. **Drop documents** under `/Volumes/acme_holdings/documents/board_meetings_raw/<quarter>/`. Filenames should start with the entity code (e.g., `SFERS_Board_2026-03-11.pdf`); the ingest notebook reads the entity from there.

3. **Run the notebooks in order**:
   - `01_Ingest` (chunks ~hundreds of pages in a minute or two)
   - `02_Build_Index` (first-time index creation takes 10–15 min for the endpoint, plus a few minutes for the initial snapshot)
   - `03_Summarize_And_Post` (set the `quarter` widget, run all)
   - `04_Query` (set the `question` / `quarter` / `entity` widgets, run all)

4. To swap models, change `SUMMARY_MODEL` in `03` and `ANSWER_MODEL` in `04` to any Databricks-served chat endpoint.

## 🎛️ Configuration knobs

| where | knob | what it does |
|---|---|---|
| `01_Ingest`, Cell 2 | `ENTITY_REGISTRY` | Add new entities by adding a code, full name, and filename regex. |
| `02_Build_Index`, Cell 2 | `EMBEDDING_MODEL` | Default `databricks-gte-large-en`. Swap for any served embedding model with the same primary-key contract. |
| `03_Summarize_And_Post`, Cell 2 | `quarter` widget | Drives which quarter gets summarized. |
| `03_Summarize_And_Post`, Cell 6 | summary system prompt | Tone and rules for the per-entity summary. |
| `04_Query`, Cell 2 | `question` / `quarter` / `entity` / `num_results` widgets | All four are interactive widgets. |
| `04_Query`, Cell 6 | `ANSWER_SYSTEM` | Citation format and grounding rules. |

## 🧠 Why this exists

Rollups accumulate unstructured text faster than humans can read it: board minutes, M&A decks, technician handoff notes, customer call transcripts, contracts. Almost none of it is queryable. Keyword search fails on intent ("show me where AI was discussed last quarter" doesn't lend itself to grep). RAG is the standard remedy, and on Databricks the standard remedy is now four short notebooks. This repo is one worked example.

## ⚠️ Honest limits

- **Citations are a UX guarantee, not a factual one.** The model is told to ground in retrieved chunks and cite. It usually does. Verify before quoting.
- **PDF text extraction is imperfect.** Tables, scanned pages, and image-heavy slides extract poorly. Treat extraction quality as a known unknown and spot-check.
- **Page-level chunks are coarse.** A 2000-word board page can dilute the signal in a single chunk. For dense corpora, sub-page chunking (e.g., 500-token windows with 50-token overlap) often retrieves better. The trade-off is page-level chunks make citations cleaner.
- **Embeddings are English-only here.** `gte-large-en` is the default. Swap if your corpus is multilingual.
- **The summary step pulls every chunk for the quarter.** That's fine for a few hundred pages; for thousands, switch to a map-reduce summarization pattern.

## 📁 Repo layout

```
board_meeting_rag/
├── README.md
├── 01_Ingest.ipynb
├── 02_Build_Index.ipynb
├── 03_Summarize_And_Post.ipynb
└── 04_Query.ipynb
```

## 📜 License

MIT.
