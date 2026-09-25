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
