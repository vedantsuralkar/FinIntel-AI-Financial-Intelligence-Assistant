# FinSight

FinSight is a production-style Financial Document Intelligence MVP. It will let users upload financial PDFs, index them with a lightweight RAG pipeline, and ask questions with source citations.

## Phase 1 Scope

- PDF upload
- Text extraction with PyMuPDF
- Intelligent chunking
- Embedding generation with Sentence Transformers
- Chroma vector storage
- Semantic retrieval
- Groq grounded answer generation
- Citation-aware answers
- Retrieved evidence and similarity scores for debugging

## Project Structure

```text
FinSight/
+-- app/
|   +-- api/              # FastAPI app, routes, and schemas
|   +-- frontend/         # Streamlit MVP frontend
|   +-- rag/              # Embeddings, vector store, retrieval, generation, citations
|   +-- ingestion/        # PDF parsing and chunking
|   +-- evaluation/       # RAG evaluation utilities, added later
|   +-- observability/    # Logging and tracing helpers
|   +-- utils/            # Shared config and utilities
+-- data/
|   +-- uploads/          # Local uploaded PDFs
|   +-- chroma/           # Local Chroma persistence
+-- docs/                 # Architecture notes
+-- notebooks/            # Scratch exploration only
+-- tests/                # Unit and integration tests
+-- requirements.txt
+-- docker-compose.yml
+-- .env.example
```

## Ingestion Pipeline

Phase 1A turns uploaded PDFs into citation-ready chunks. The API stores the PDF in `data/uploads`, extracts text page by page with PyMuPDF, skips empty pages, detects likely section headings, keeps table-like rows together where possible, and creates overlap-aware chunks.

The ingestion modules are separated by responsibility:

- `app/ingestion/loader.py`: opens PDFs with PyMuPDF and returns non-empty page objects.
- `app/ingestion/chunker.py`: converts page text into section-aware, overlap-aware chunks.
- `app/ingestion/models.py`: defines typed page, chunk, and ingestion result models.
- `app/ingestion/pipeline.py`: orchestrates loading and chunking for the API.

Each chunk preserves:

- `chunk_id`: deterministic identifier for downstream vector records.
- `source_filename`: original PDF filename.
- `page_number`: 1-based page number for citations.
- `section_title`: detected heading when available.
- `text`: chunk text that will be embedded in the next phase.
- `metadata`: source path, chunk index, character count, and block types.

Citation tracking matters because financial answers must be auditable. When retrieval returns a chunk, the system can show the page and section that supported the answer instead of producing an unsupported summary.

## Embeddings And Vector Storage

Phase 1B turns chunks into searchable semantic memory.

- `app/rag/embeddings.py`: loads a shared CPU-friendly Sentence Transformers model, `BAAI/bge-small-en-v1.5`, and exposes `embed_documents()` plus `embed_query()`.
- `app/rag/vector_store.py`: wraps a persistent ChromaDB collection stored under `data/chroma`.
- `app/rag/retriever.py`: embeds user questions, queries Chroma, and returns structured retrieval results.

Upload flow:

```text
PDF -> page extraction -> citation-aware chunks -> BGE embeddings -> Chroma upsert
```

Each Chroma record stores:

- chunk text as the retrievable document
- embedding vector from `BAAI/bge-small-en-v1.5`
- metadata: `source`, `page`, `section`, `chunk_id`, and selected ingestion metadata

Retrieval remains independent from generation. This makes it possible to test and tune search quality without invoking an LLM.

## Grounded Answer Generation

Phase 1C adds citation-aware answer synthesis with Groq.

Generation flow:

```text
question -> semantic retrieval -> bounded context builder -> Groq LLM -> answer + citations + retrieved evidence
```

The generation modules are separated by responsibility:

- `app/rag/prompts.py`: system and answer prompts with grounding rules.
- `app/rag/context_builder.py`: converts retrieved chunks into ordered, citation-labeled context blocks.
- `app/rag/generator.py`: centralized Groq API integration with timeout handling and graceful failures.
- `app/rag/citation.py`: extracts and deduplicates citation metadata.
- `app/rag/pipeline.py`: orchestrates retrieval, context assembly, generation, and citation attachment.

The `/api/v1/qa` endpoint now returns:

- generated answer
- citations with source filename, page number, and section title
- retrieved chunks
- similarity scores
- generation model and prompt/response sizes

## Hallucination Mitigation

FinSight reduces hallucination risk by keeping retrieval and generation decoupled and forcing generation to use only retrieved context. The prompt explicitly tells the model to avoid inventing facts and to admit when context is insufficient. The context builder also preserves source filename, page number, and section title near each chunk, so generated answers remain traceable to evidence.

If no chunks are retrieved, the pipeline returns an insufficient-context answer instead of calling the LLM.

## Citation Architecture

Each chunk carries citation metadata from ingestion into Chroma. Retrieval returns that metadata with the text and similarity score. The citation layer deduplicates repeated references by source filename, page number, and section title, then attaches the final citation list to the generated answer.

This design keeps citations tied to retrieved evidence rather than to unsupported model output.

## Local Setup

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
Copy-Item .env.example .env
```

Add your Groq key to `.env`:

```text
GROQ_API_KEY=your_key_here
GENERATION_MODEL=llama-3.1-8b-instant
```

## Run the API

```powershell
python -m uvicorn app.api.main:app --reload --host 127.0.0.1 --port 8000
```

Health check:

```text
http://127.0.0.1:8000/api/v1/health
```

## Run Streamlit

```powershell
python -m streamlit run app/frontend/streamlit_app.py
```

## Current Status

The upload route stores PDFs locally, runs ingestion, embeds chunks, and upserts them into Chroma. The `/api/v1/qa` endpoint performs semantic retrieval, builds grounded context, calls Groq for answer generation, and returns citations plus the retrieved chunks and similarity scores used as evidence.

## Phase 2A: Evaluation, Hallucination Detection, and RAG Observability

Phase 2A transforms FinSight from "RAG that works" to "RAG that can be measured and trusted."

### Evaluation Architecture

Evaluation is completely decoupled from the RAG pipeline. The system measures retrieval quality, answer faithfulness, and detects unsupported claims through four key components:

**Metrics Module (`app/evaluation/metrics.py`)**
- Uses RAGAS metrics for standardized RAG evaluation:
  - `faithfulness`: answer grounded in retrieved context [0, 1]
  - `answer_relevancy`: answer addresses the question [0, 1]
  - `context_precision`: retrieved context focuses on answering the question [0, 1]
  - `context_recall`: retrieved context contains information needed [0, 1]
- Adds custom metrics:
  - `retrieval_precision`: fraction of chunks that are relevant
  - `retrieval_ndcg`: NDCG@k ranking quality of semantic search
- All metrics fail gracefully with heuristic fallbacks if RAGAS unavailable
- Metrics computed independently so one failure doesn't cascade

**Hallucination Detection (`app/evaluation/hallucination.py`)**
- Pure heuristic-based detection (no LLM calls, deterministic)
- Approach: Split answer into sentences, extract entities/tokens, check against context
- Sentence-level granularity: pinpoints exact unsupported claims
- Confidence scoring [0, 1]: distinguishes high/medium/low confidence issues
- Key strategies:
  - Named entity matching (numbers, percentages, proper nouns)
  - Key token overlap with context
  - Entity extraction using regex patterns for financial amounts/years
  - Support threshold tuning for sensitivity control

**Evaluator (`app/evaluation/evaluator.py`)**
- Orchestrates metric computation + hallucination detection
- Takes immutable `RagPipelineResult` as input
- Returns comprehensive `EvaluationResult` with all metrics + hallucinations
- Batch evaluation support for benchmarking
- Structured logging: logs faithfulness, hallucination count, retrieval stats

**Benchmark Runner (`app/evaluation/benchmark.py`)**
- Executes RAG pipeline + evaluation on datasets
- Aggregates metrics: computes averages across queries
- Generates `BenchmarkReport` with summary statistics
- Report comparison: detects regressions (>5% faithfulness drop or 10% hallucination increase)
- Saves results to JSON for tracking performance over time

### Why This Approach?

**Why not pure LLM self-evaluation?**
- Groq API adds latency and cost for every evaluation
- Less reproducible (model updates change results)
- Heuristic hallucination detection is faster, deterministic, transparent
- Can validate claims using text matching + entity extraction

**Why separate evaluation from pipeline?**
- Pipeline remains unchanged and focused
- Evaluation can be tested independently
- Easy to add/modify metrics later
- Evaluation logic is reusable across pipelines

**Why heuristic hallucination detection?**
- Catches most unsupported numeric claims, names, dates
- No external API calls = fast and reproducible
- Confidence scoring allows filtering false positives
- Sentence-level precision helps target improvements

### Using Evaluation in FinSight

**API Endpoint: `POST /api/v1/evaluate`**

Request:
```json
{
  "question": "What is the revenue?",
  "expected_keywords": ["revenue", "million"],
  "top_k": 5
}
```

Response:
```json
{
  "question": "What is the revenue?",
  "answer": "The revenue was $100 million in 2023.",
  "citations": [...],
  "metrics": {
    "faithfulness": 0.92,
    "answer_relevancy": 0.88,
    "context_precision": 0.85,
    "context_recall": 0.80,
    "retrieval_precision": 0.90,
    "retrieval_ndcg": 0.88
  },
  "hallucination_warnings": [
    {
      "sentence": "The company was founded in 1800.",
      "confidence": 0.85,
      "explanation": "Founded year not mentioned in context"
    }
  ],
  "hallucination_rate": 0.33,
  "retrieval_stats": {
    "chunk_count": 5,
    "avg_similarity_score": 0.85,
    "evaluation_latency_ms": 245
  }
}
```

**Streamlit UI: Evaluation Section**

Checkbox to enable `Enable evaluation metrics`. When checked, displays:
- Four metric gauges: Faithfulness, Answer Relevancy, Context Precision, Context Recall
- Retrieval metrics: Retrieval Precision, NDCG@5, Chunks Retrieved
- Hallucination rate percentage
- Individual hallucinations with confidence color-coding (🔴 HIGH, 🟡 MEDIUM, 🟢 LOW)
- Export evaluation as JSON button

**Programmatic Evaluation**

```python
from app.evaluation import RagEvaluator, load_benchmark_dataset
from app.rag.pipeline import RagPipeline

pipeline = RagPipeline()
evaluator = RagEvaluator()

# Single evaluation
rag_result = pipeline.answer("What is revenue?")
eval_result = evaluator.evaluate(
    rag_result=rag_result,
    expected_keywords=["revenue", "million"],
)
print(f"Faithfulness: {eval_result.metrics.faithfulness:.2%}")
print(f"Hallucinations: {len(eval_result.hallucinations)}")

# Benchmark evaluation
from app.evaluation import BenchmarkRunner

runner = BenchmarkRunner(pipeline, evaluator)
dataset = load_benchmark_dataset("tests/evaluation_queries.json")
report = runner.run_benchmark(dataset, dataset_name="financial_qa")
print(f"Avg Faithfulness: {report.avg_faithfulness:.2%}")
print(f"Hallucination Rate: {report.hallucination_rate:.2%}")

# Save for tracking
runner.save_results(report, "benchmark_results_2024-01-15.json")
```

### Benchmark Dataset Format

File: `tests/evaluation_queries.json`

```json
[
  {
    "question": "What is the operating revenue for 2023?",
    "expected_answer": "The operating revenue for 2023 was $150 million.",
    "expected_keywords": ["operating revenue", "2023", "150 million"],
    "document_source": "financial_report_2023.pdf",
    "ground_truth_chunks": ["revenue_section_chunk_1", "revenue_section_chunk_2"]
  }
]
```

### Observability and Logging

All evaluation events logged with structured fields:

```python
# Logged when evaluation completes
logger.info("evaluation.completed", extra={
    "faithfulness": 0.92,
    "answer_relevancy": 0.88,
    "hallucination_count": 1,
    "hallucination_rate": 0.33,
    "chunk_count": 5,
})

# Logged when hallucinations detected
logger.warning("evaluation.hallucinations_detected", extra={
    "count": 2,
    "avg_confidence": 0.82,
    "sentences": ["Unsupported claim 1", "Unsupported claim 2"],
})
```

### Key Files

| Module | File | Responsibility |
|--------|------|-----------------|
| Models | `app/evaluation/models.py` | EvaluationResult, MetricsResult, HallucinationIssue |
| Metrics | `app/evaluation/metrics.py` | RAGAS integration + custom metrics |
| Hallucination | `app/evaluation/hallucination.py` | Heuristic-based unsupported claim detection |
| Evaluator | `app/evaluation/evaluator.py` | Orchestration layer |
| Benchmark | `app/evaluation/benchmark.py` | Benchmark execution + aggregation |
| Datasets | `app/evaluation/datasets.py` | Dataset loading + validation |
| API | `app/api/routes.py` | POST /api/v1/evaluate endpoint |
| Tests | `tests/test_evaluation.py` | Evaluator unit tests |
| Tests | `tests/test_hallucination.py` | Hallucination detection tests |
| Tests | `tests/test_benchmark.py` | Benchmark runner tests |
| Benchmark Data | `tests/evaluation_queries.json` | Sample Q&A pairs for evaluation |

### Testing Evaluation

All evaluation components have unit tests with no external API calls:

```powershell
pytest tests/test_evaluation.py       # Evaluator tests
pytest tests/test_hallucination.py    # Hallucination detection
pytest tests/test_benchmark.py        # Benchmark runner
```

### Next Steps

FinSight now transitions from "RAG that works" to "RAG that can be measured and trusted." Future phases can:
- Train domain-specific hallucination detectors
- Integrate online evaluation metrics
- Add active learning for improving retrieval quality
- Build dashboards for evaluation metrics over time
