DaVita IKC — Clinical Patient-Records RAG
A working Retrieval-Augmented Generation (RAG) system that answers questions about patient records in the kidney-care (dialysis / CKD) domain. Every answer is grounded in the source charts, carries inline citations, and the assistant refuses when the information isn't in the records — so it never invents a lab value, dose, or diagnosis.
All data is synthetic. No real patient data (PHI) is used.
What it does
Ask a clinical question, optionally scoped to a single patient, and the system embeds the question, searches a vector index, retrieves the most relevant chart sections (filtered to the selected patient), and asks the LLM to answer using only those sources — citing each as [file, p.N]. If the answer isn't in the retrieved context, it declines rather than guessing.
Pipeline
ingest (PDF) → structure-aware chunking → local embeddings → ChromaDB vector store (metadata filtering for per-patient isolation) → retrieve → grounded answer with citations
Tech stack

Embeddings: Sentence-Transformers all-MiniLM-L6-v2 — runs locally, no API calls, rate limits, or timeouts
LLM (answer generation): Google Gemini gemini-2.5-flash
Vector database: ChromaDB (cosine similarity, HNSW index, native metadata filtering)
UI: Gradio chat interface
Ingestion: pypdf

Embeddings and generation are decoupled, so either can be swapped independently without touching the rest of the pipeline.
Production path (GCP)
Maps directly to a production deployment on Google Cloud: Vertex AI for the models, Document AI for robust ingestion of scanned/faxed documents, and Vertex AI Vector Search as the managed vector store — all inside a HIPAA-eligible project under a BAA, with PHI never leaving the boundary.
