# Data Pipeline — Books to Relational Catalog

This module implements a raw-to-relational catalog pipeline:

**scrape → clean → convert → normalize → load into SQLite → query with SQL → reproduce with pandas**

## Data source

The scraper uses the public practice site:

`https://books.toscrape.com/`

It scrapes the first **5 paginated All Products pages** and follows each book's detail page to obtain its category. The target is therefore at least 100 raw book rows and at least 3 categories.

No login, API key, or paid service is required.

## Fixed currency conversion

The project-required artificial baseline is:

**1 GBP = 105.50 INR**

`price_inr` is calculated only as:

`price_inr = price_gbp * 105.50`

This is a fixed project constant, not a live or historical exchange rate. No currency API is used.

## Schema

The SQLite database contains two normalized tables:

### categories

- `category_id INTEGER PRIMARY KEY`
- `category_name TEXT UNIQUE NOT NULL`

### books

- `book_id INTEGER PRIMARY KEY`
- `title TEXT NOT NULL`
- `price_gbp REAL NOT NULL`
- `price_inr REAL NOT NULL`
- `rating INTEGER NOT NULL`
- `in_stock INTEGER NOT NULL`
- `category_id INTEGER NOT NULL REFERENCES categories(category_id)`
- `source_url TEXT`

The category name is stored once in `categories`, while `books.category_id` is the foreign key.

## Cleaning decisions

- Currency symbols are removed from the price and the result is converted to `float`.
- Star-rating words `One` through `Five` are mapped to integers `1` through `5`.
- Availability text containing `In stock` becomes `True`; text containing `Out of stock` becomes `False`.
- Numeric parse failures for `price_gbp` and `rating` are filled using the column median.
- Missing/unparseable availability, category, or title is dropped because these fields cannot be meaningfully median-imputed.
- Ratings are rounded, converted to integer, and constrained to the valid range 1–5.
- Duplicate title/category rows are removed.
- SQLite stores booleans as `0/1` integers, as is standard for SQLite.

## Install

From the repository root:

```bash
cd data_pipeline
python -m venv .venv
```

Windows:

```bash
.venv\Scripts\activate
```

macOS/Linux:

```bash
source .venv/bin/activate
```

Install dependencies:

```bash
pip install -r requirements.txt
```

## Run

```bash
python scrape_pipeline.py
```

The script runs end to end and creates:

```text
data_pipeline/
├── scrape_pipeline.py
├── requirements.txt
├── README.md
├── books_catalog.sqlite
└── outputs/
    ├── raw_books.csv
    ├── clean_books.csv
    ├── queries.sql
    ├── query_outputs.txt
    └── join_comparison.txt
```

The SQLite database is regenerated from scratch each time.

## SQL requirements demonstrated

The pipeline executes six queries:

1. `SELECT` + `WHERE`
2. `ORDER BY`
3. `LIMIT`
4. `DISTINCT`
5. `BETWEEN`
6. `JOIN`

The SQL text and printed results are saved under `outputs/`.

## SQL vs pandas JOIN

The SQL JOIN combines `books` and `categories` using:

```sql
books.category_id = categories.category_id
```

The same relationship is reproduced in memory with:

```python
books_df.merge(categories_df, on="category_id", how="inner")
```

The script normalizes the column order/types and asserts that the two outputs are equivalent. The side-by-side results and the final `Equivalent: True` check are saved to:

`outputs/join_comparison.txt`

## Git workflow requirement

The overall repository must show this history at least once:

```bash
git checkout main
git pull

git checkout -b feature/data-pipeline

git add data_pipeline/
git commit -m "Add data pipeline scraping and cleaning"

git add data_pipeline/
git commit -m "Add SQLite queries and pandas validation"

git checkout main
git merge feature/data-pipeline
```

The assignment checks that a feature branch was created, had at least two commits, and was merged back into `main`.





# Module 3 — Support Assistant

This module implements the supplied offline-first Zepto support-assistant specification.

## Architecture

```text
8 exact policy documents
        |
        v
 ingestion/chunking
        |
        v
 all-MiniLM-L6-v2
 local embeddings
        |
        v
 ChromaDB: zepto_policies
        |
        v
 LangGraph StateGraph
        |
        +--> classify_intent
        |       |
        |       +--> policy_question --> retrieve_and_answer --> response
        |       |
        |       +--> general_question --> direct_answer ----------^
        |
        v
 Pydantic JSON
        |
        v
 FastAPI POST /ask
```

**Ingestion:** `build_vector_store()` reads `docs/doc_01.txt` through `doc_08.txt`, using one document-sized chunk per file.

**Embedding:** `LocalEmbedder` uses the local `all-MiniLM-L6-v2` Sentence Transformers model.

**Retrieval:** ChromaDB collection `zepto_policies` stores embeddings and documents. `retrieve()` embeds each incoming query and retrieves the top 3 by cosine similarity.

**Generation:** `retrieve_and_answer` generates the required deterministic mock answer from the top chunk for policy questions. `direct_answer` produces the deterministic general-question response.

Only generation branches on `MOCK_LLM`. Classification is always the specified keyword heuristic and retrieval always uses the real local embedding + ChromaDB path. With `MOCK_LLM=1` or unset, no LLM network call is made. With `MOCK_LLM=0`, the optional provider hook uses the structured prompt and validates raw JSON with up to two additional corrective retries.

## Structured prompt

`prompt_template.py` contains the actual role/context/task/format/length prompt, an explicit negative constraint not to use information outside supplied context, and a few-shot example.

## Install

```bash
cd support_assistant
python -m venv .venv
```

Windows:
```cmd
.venv\Scripts\activate
```

macOS/Linux:
```bash
source .venv/bin/activate
```

```bash
pip install -r requirements.txt
```

## Build and test

```bash
python build_index.py
python test_offline.py
```

The test checks that all 8 documents are indexed and exercises one retrieval query and one general query.

## FastAPI

```bash
uvicorn main:app --host 127.0.0.1 --port 7860
```

Policy example:

```bash
curl -X POST http://127.0.0.1:7860/ask \
  -H "Content-Type: application/json" \
  -d "{\"query\":\"How much is priority delivery?\"}"
```

General example:

```bash
curl -X POST http://127.0.0.1:7860/ask \
  -H "Content-Type: application/json" \
  -d "{\"query\":\"What is the capital of France?\"}"
```

The general mock response is deterministic:

```json
{"answer":"I can only answer questions about Zepto policies right now.","sources":[],"confidence":1.0}
```

The policy response's exact top-3 source ordering should be copied from the local run because it depends on the installed local embedding/Chroma versions; this README intentionally does not fabricate a retrieval transcript.

## Docker

```bash
docker build -t zepto-support .
docker run --rm -p 7860:7860 -e MOCK_LLM=1 zepto-support
```

The service then listens on `http://localhost:7860`.

## Optional real LLM

The required grading path does not need a real LLM. If `MOCK_LLM=0`, the optional implementation expects `groq` and `GROQ_API_KEY`:

```bash
pip install groq
export MOCK_LLM=0
export GROQ_API_KEY="your-key"
```

Never hardcode or commit the key.
