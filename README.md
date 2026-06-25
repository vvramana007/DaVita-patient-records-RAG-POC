# DaVita IKC — Clinical Patient-Records RAG

An end-to-end **Retrieval-Augmented Generation (RAG)** system for clinical documents in the
kidney-care (dialysis / CKD) domain. It implements the full pipeline:

**ingest → chunk → index → retrieve → cite**

plus the safeguards that make RAG trustworthy in healthcare: **grounded answers with inline
citations**, a **refusal guardrail** (never invents a lab value or dose), **per-patient data
isolation**, and a **live chat UI**.

## Stack

- **Sentence-Transformers** (local `all-MiniLM-L6-v2`, 384-dim) for embeddings — runs offline, no API calls
- **ChromaDB** vector store (cosine, HNSW) with metadata filtering for per-patient scoping
- **Google Gemini** (`gemini-2.5-flash`) for the final grounded answer
- **pypdf** / **reportlab** for document generation and parsing
- **Gradio** chat interface

## Pipeline

1. **Configure Gemini** — supply a `GEMINI_API_KEY`.
2. **Generate synthetic patient records** — HIPAA-safe stand-ins for real clinic documents, written to PDFs.
3. **Ingest + structure-aware chunking** — extract text per page (keeping source + page for citations), split by clinical section (note / labs / meds), and cap chunk size with overlap. Every chunk carries metadata (`patient_mrn`, `section`, `source`, `page`).
4. **Embed locally + build the vector store** — encode chunks with the local model and store them in a ChromaDB collection.
5. **Retrieve + grounded answer** — nearest-neighbor search (optionally filtered to one patient), then Gemini answers using *only* the retrieved context, citing every claim as `[filename, p.PAGE]` and refusing when the answer isn't present.
6. **Chat UI (Gradio)** — pick a patient, ask a question, or try the examples. One example deliberately asks for a value that isn't in the records to show the refusal guardrail in action.

## Running

The project is a single Jupyter notebook: `DaVita_Final_Demo.ipynb`. It is designed to run in
Google Colab (or any Jupyter environment).

1. Open the notebook in Colab or Jupyter.
2. Run the first cell to install dependencies:

   ```
   google-generativeai pypdf reportlab chromadb gradio pysqlite3-binary sentence-transformers
   ```

3. Provide your Gemini API key when prompted (in Colab, store it as the secret `GEMINI_API_KEY`).
4. Run the cells top to bottom and open the Gradio share link from the final cell.

## Notes

- All patient data is **synthetic** — generated at runtime purely to exercise the pipeline.
- A `pysqlite3` shim is applied before importing ChromaDB to satisfy its SQLite version requirement.
- Your API key is entered at runtime and is **not** stored in the repository.
