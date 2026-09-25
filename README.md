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


# Module 2 — Analytics Pipeline

This `/analytics` module follows the supplied assignment exactly: one Titanic dataset load, an offline CSV fallback, defensible cleaning, EDA/data story, train-only modeling preprocessing, three classifiers, imbalance comparison, Random Forest tuning with OOB, a fare regression side-task, and a reloadable complete pipeline artifact.

## Files

```text
analytics/
├── 01_eda.py
├── 02_modeling.py
├── requirements.txt
├── titanic.csv                  # generated/committed offline fallback
├── clean_titanic.csv            # generated cleaned working data
├── best_titanic_pipeline.joblib # generated complete pipeline
├── charts/
└── outputs/
```

## Important dataset-loading rule

`01_eda.py` contains the module's **only** `sns.load_dataset("titanic")` call. It immediately saves the returned raw DataFrame as:

```python
df.to_csv("titanic.csv", index=False)
```

`02_modeling.py` reads that committed CSV and never calls Seaborn's loader again.

If the first run cannot access the internet, `01_eda.py` requires an already committed `titanic.csv` fallback. This prevents the modeling stage from becoming a second independent network load.

## Install and run

From the repository root:

```bash
cd analytics
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

Install:

```bash
pip install -r requirements.txt
```

Run EDA first:

```bash
python 01_eda.py
```

Then run modeling:

```bash
python 02_modeling.py
```

## Cleaning decisions

The script calculates and saves exact missing percentages before cleaning in `outputs/missing_percentages.csv`.

The required threshold rule is:

- `<5%`: drop affected rows.
- `5%–30%`: impute.
- `>30%`: explicitly decide whether to drop the column or treat missingness as a category.

The implementation:
- drops rows missing `embarked` because it is below 5% missing;
- median-imputes `age` because it falls in the 5%–30% range;
- drops `deck` because its missingness is above 30% and direct imputation would be unreliable;
- treats `embark_town` as redundant with `embarked` and does not use it for modeling.

The exact percentages are generated from the actual loaded data rather than hard-coded.

## EDA requirements covered

`01_eda.py` produces:

- `df.info()`
- `df.describe()`
- `df.shape`
- missing percentages
- IQR outlier counts for age and fare
- fare mean, median, mode and skewness interpretation
- survival rates by sex
- survival rates by pclass
- survival rates by sex + pclass
- exactly the six-column correlation matrix required by the assignment
- correlation heatmap
- two strongest correlations by absolute off-diagonal coefficient
- age/fare z-score before/after check
- four distinct multivariate data-story charts, each described in `outputs/EDA_INTERPRETATIONS.md`

The correlation matrix excludes `adult_male` and `alone`.

## Modeling requirements covered

`02_modeling.py`:

1. Reads the same committed `titanic.csv`.
2. Cleans it consistently with the EDA stage.
3. Performs the stratified train/test split **before preprocessing**.
4. Uses a `ColumnTransformer` containing training-only imputation, one-hot encoding, and `StandardScaler`.
5. Trains Logistic Regression, Decision Tree, and Random Forest on the same split.
6. Reports accuracy, precision, recall, F1, confusion matrix, ROC and AUC.
7. Renders the Decision Tree with feature and class labels.
8. Compares baseline, `class_weight='balanced'`, and SMOTE.
9. Applies SMOTE only inside the training pipeline.
10. Runs `GridSearchCV` over Random Forest `n_estimators`, `max_depth`, and `max_features`.
11. Constructs the tuned Random Forest with `oob_score=True` and reports OOB score.
12. Runs multivariate linear regression for fare.
13. Reports MAE, RMSE, R² and Adjusted R².
14. Creates a residual plot and calculates a diagnostic heteroscedasticity flag based on the correlation between fitted values and absolute residuals.
15. Writes a final recommendation using the actual generated test metrics.
16. Saves the complete fitted preprocessing + estimator pipeline with `joblib.dump`.
17. Reloads the artifact and predicts directly from raw feature values.

## Outputs

Generated evidence includes:

```text
outputs/
├── missing_percentages.csv
├── survival_by_sex.csv
├── survival_by_pclass.csv
├── survival_by_sex_pclass.csv
├── correlation_matrix.csv
├── two_strongest_correlations.csv
├── standardization_check.csv
├── EDA_INTERPRETATIONS.md
├── classification_metrics.csv
├── imbalance_comparison.csv
├── random_forest_grid_search.csv
├── regression_metrics.csv
├── model_comparison.csv
├── artifact_check.txt
└── FINAL_RECOMMENDATION.md
```

## Model metric groups

Classification metrics and regression metrics are intentionally kept separate. Accuracy/precision/recall/F1/AUC measure classification behavior, while MAE/RMSE/R²/Adjusted R² measure the fare regression task. They are not treated as one common numerical scale.

## Git requirement

The assignment requires the repository's overall history to show a feature branch with at least two commits and a merge back to `main`.

Example:

```bash
git checkout main
git checkout -b feature/analytics-pipeline

git add analytics/
git commit -m "Add Titanic EDA and cleaning pipeline"

git add analytics/
git commit -m "Add modeling, tuning, regression and pipeline artifact"

git checkout main
git merge feature/analytics-pipeline
```



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
