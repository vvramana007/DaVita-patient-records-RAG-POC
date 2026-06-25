# DaVita Patient-Records RAG — Proof of Concept

A working Retrieval-Augmented Generation system over **dialysis / CKD patient
records**, built to mirror the architecture in the DaVita AI Consultant role:
hybrid retrieval, a multi-agent pipeline, grounded answers with citations, a
FastAPI backend + web UI, and a RAGAS-ready evaluation harness.

> **All patient data is 100% synthetic.** No real PHI is used. The 15 records are
> randomly generated dialysis charts (demographics, labs, session notes, meds,
> care plans).

---

## Why this matches the brief

| JD requirement | How the POC demonstrates it |
|---|---|
| **RAG pipeline** | End-to-end: ingest → chunk → embed → hybrid retrieve → synthesize with citations |
| **Multi-agent architecture** | Router → Retriever → Synthesis → Safety agents (`app/agents.py`), drop-in replaceable with LangGraph nodes |
| **Advanced chunking** | Section-aware + recursive chunking with overlap (`app/chunking.py`) |
| **Advanced retrieval** | Hybrid dense + BM25 search, Reciprocal Rank Fusion, query routing, optional cross-encoder re-ranking (`app/retriever.py`) |
| **Vector DB management** | Chroma backend with automatic numpy fallback; swap to pgvector/Pinecone in prod (`app/vectorstore.py`) |
| **Evaluation (RAGAS)** | Retrieval/grounding metrics with no API key + a RAGAS script (`eval/`) |
| **Backend + UI** | FastAPI API and a single-page chat UI (`app/server.py`) |
| **GCP / Vertex AI** | LLM layer is provider-pluggable (Gemini/Vertex, OpenAI, or offline extractive) (`app/llm.py`) |

---

## Two ways to explore this repo

**1. Notebooks (quickest)** — self-contained Colab notebooks that build the whole pipeline end to end:
- **`DaVita_RAG_Final.ipynb`** — recommended walkthrough: ingest → chunk → local embeddings → ChromaDB → grounded, cited answers, plus a Gradio chat UI.
- `DaVita_RAG_Demo.ipynb` / `DaVita_RAG_Demo_v2_VectorDB_UI.ipynb` — earlier, progressively built versions.

**2. Full application** — the production-shaped FastAPI + multi-agent service described below (`app/`).

A deployable **Hugging Face Space** version lives in `hf_space/` (Gradio UI, local Sentence-Transformers
embeddings, Gemini for answer generation). See **`DaVita_RAG_architecture.svg`** for the pipeline diagram
and **`DaVita_RAG_interview_notes.md`** for design rationale and likely interview Q&A.

---

## Architecture

```
                 ┌──────────────────────────────────────────────┐
  Patient .md    │                INGESTION                      │
  records  ─────▶│  section-aware + recursive chunking           │
                 │  → embeddings → Chroma (vectors) + BM25 index │
                 └──────────────────────────────────────────────┘
                                      │
  User question ──▶  RouterAgent  ──▶ RetrieverAgent ──▶ SynthesisAgent ──▶ SafetyAgent ──▶ Answer
                     (intent +         (hybrid search      (grounded LLM      (PHI / scope     (+ citations,
                      patient filter)   + RRF + rerank)     answer + cites)    guard)           agent trace)
```

- **RouterAgent** — classifies the query (patient-specific / cohort / general) and
  extracts an MRN or patient name so retrieval can be metadata-filtered to one chart.
- **RetrieverAgent** — runs dense + sparse search in parallel, fuses with RRF, and
  optionally re-ranks with a cross-encoder.
- **SynthesisAgent** — produces a grounded answer where every claim carries a
  `[MRN | Section]` citation. Uses OpenAI / Gemini if a key is set, else a
  zero-hallucination extractive mode.
- **SafetyAgent** — blocks prompt-injection / PII-exfiltration attempts and appends
  a clinical-use disclaimer.

---

## Quick start

```bash
cd davita-rag-poc
python -m venv .venv && source .venv/bin/activate      # optional
pip install -r requirements.txt

# 1. (already generated; re-run to regenerate) synthetic records
python data/generate_patients.py

# 2. build the index (embeddings + vector store + BM25)
python -m app.ingest

# 3. run the app
uvicorn app.server:app --reload
# open http://localhost:8000
```

The POC runs **with zero API keys** out of the box (offline embeddings + extractive
answers), so it always demos. Flip on the production-grade components with env vars:

```bash
export EMBED_PROVIDER=sentence-transformers   # real semantic embeddings (default)
export VECTOR_BACKEND=chroma                   # real vector DB (default)
export USE_RERANKER=true                        # cross-encoder re-ranking

# choose an LLM for fluent synthesis:
export LLM_PROVIDER=openai && export OPENAI_API_KEY=sk-...
#   or
export LLM_PROVIDER=gemini && export GOOGLE_API_KEY=...     # Vertex/Gemini path
```

> Note: the bundled index was built with the offline `hash` embedder so it runs
> anywhere. For the live demo, set `EMBED_PROVIDER=sentence-transformers` and
> re-run `python -m app.ingest` to get full semantic retrieval quality.

---

## Evaluation

```bash
# Retrieval + grounding metrics — no API key required:
python -m eval.evaluate

# RAGAS metrics (faithfulness, answer relevancy, context precision/recall):
pip install ragas datasets
export OPENAI_API_KEY=... LLM_PROVIDER=openai
python -m eval.ragas_eval
```

`eval/evaluate.py` reports Context Precision, Hit Rate@k, MRR, and **Citation
Faithfulness** (every citation in the answer must map to a retrieved chunk — a
direct hallucination guard). Ground truth is derived from the source records, so
it's always correct.

---

## Try these in the UI

- "What is the dialysis modality and vascular access for patient DV-1000?"
- "Summarize the most recent labs and care plan for patient DV-1003."
- "Which patients had intradialytic hypotension during a session?"
- "What medications is the patient with MRN DV-1005 taking and why?"

---

## Path to production (talking points for the call)

- **Security / HIPAA**: deploy in a BAA-covered VPC; encrypt at rest + in transit;
  de-identify or tokenize PHI; role-based access; full audit logging of every
  query and retrieved chunk; private model endpoints (no data leaves the boundary).
- **Scale**: move vectors to **pgvector on Cloud SQL** or a managed vector DB;
  host embeddings + LLM on **Vertex AI**; containerize (Docker) + orchestrate (GKE);
  Terraform IaC; CI/CD.
- **Quality**: continuous RAGAS eval in CI, golden QA sets per clinical domain,
  human-in-the-loop review, hallucination + citation-coverage gates before release.
- **Multi-agent expansion**: add specialized agents (labs trend analysis, med
  interaction check, care-gap detection) coordinated via LangGraph.

---

## Project layout

```
davita-rag-poc/
├── DaVita_RAG_Final.ipynb        # recommended end-to-end notebook (+ Gradio UI)
├── DaVita_RAG_Demo*.ipynb         # earlier notebook versions
├── DaVita_RAG_architecture.svg    # pipeline diagram
├── DaVita_RAG_interview_notes.md  # design rationale + interview Q&A
├── hf_space/                      # Hugging Face Space (Gradio) deployment
│   ├── app.py
│   ├── requirements.txt
│   └── README.md
├── app/
│   ├── chunking.py      # section-aware + recursive chunking
│   ├── embeddings.py    # sentence-transformers | offline hash fallback
│   ├── vectorstore.py   # Chroma | numpy fallback
│   ├── retriever.py     # hybrid (vector + BM25) + RRF + reranker
│   ├── llm.py           # OpenAI | Gemini | extractive
│   ├── agents.py        # Router / Retriever / Synthesis / Safety
│   ├── pipeline.py      # loads a query-ready pipeline
│   ├── ingest.py        # build the index
│   └── server.py        # FastAPI + single-page UI
├── data/
│   ├── generate_patients.py   # synthetic record generator
│   └── patients/              # 15 synthetic records (.md + .json)
├── eval/
│   ├── evaluate.py      # retrieval + grounding metrics (no key)
│   └── ragas_eval.py    # RAGAS metrics (needs LLM key)
└── requirements.txt
```
