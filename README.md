# RAG Document Chat

A local Retrieval-Augmented Generation (RAG) application that lets you upload a document and ask questions about it. Built with LangChain, ChromaDB, Groq, and Streamlit.

---

## How It Works

1. You upload a PDF, PPTX, or TXT file
2. The file is extracted, split into chunks, embedded, and stored in a local ChromaDB vector store
3. When you ask a question, the most relevant chunks are retrieved and passed to the LLM along with your question
4. The LLM answers strictly from the retrieved context — no hallucination from outside knowledge

Duplicate uploads are detected via SHA-256 hashing — re-uploading the same file is a no-op.

---

## Tech Stack

| Layer | Technology |
|---|---|
| UI | [Streamlit](https://streamlit.io) |
| LLM | [Groq API](https://console.groq.com) — `llama-3.3-70b-versatile` |
| Orchestration | [LangChain](https://python.langchain.com) (LCEL chain) |
| Embeddings | [HuggingFace](https://huggingface.co) — `all-MiniLM-L6-v2` (runs locally) |
| Vector Store | [ChromaDB](https://www.trychroma.com) (persisted on disk) |
| PDF Extraction | PyMuPDF (`pymupdf`) |
| PPTX Extraction | `unstructured[pptx]` |
| Config | Pydantic Settings + `.env` |
| Security | Custom prompt injection detector |

---

## Project Structure

```
rag_gemini/
├── streamlit_app.py          # UI — all rendering lives here
├── streamlit_backend.py      # Orchestration — upload and Q&A pipelines
├── backend/
│   ├── config.py             # All settings, driven by .env
│   ├── logging_config.py     # Rotating file logger setup
│   ├── core/
│   │   ├── chain.py          # LCEL RAG chain assembly
│   │   ├── prompts.py        # System + user prompt templates
│   │   └── retrieval.py      # format_docs helper
│   ├── services/
│   │   ├── llm.py            # Cached ChatGroq instance
│   │   ├── embeddings.py     # Cached HuggingFace embedding model
│   │   ├── vectordb.py       # ChromaDB store, retriever, duplicate detection
│   │   ├── chunker.py        # RecursiveCharacterTextSplitter wrapper
│   │   └── extractor.py      # PDF / PPTX / TXT loaders
│   └── security/
│       └── injection_detector.py  # Prompt injection guard
├── chroma_db/                # Auto-created — persisted vector store (gitignored)
├── logs/                     # Auto-created — rotating log files (gitignored)
├── .env                      # Your secrets and overrides (gitignored)
├── .env.example              # Template — copy this to .env
├── .streamlit/
│   └── config.toml           # Streamlit config (file watcher disabled)
└── requirements.txt
```

---

## Prerequisites

- Python **3.10 or higher**
- A free [Groq API key](https://console.groq.com) (takes 30 seconds to get)
- Git (optional)

---

## Setup & Installation

### 1. Clone or download the project

```bash
git clone <your-repo-url>
cd rag_gemini
```

### 2. Create a virtual environment

```bash
# Windows
python -m venv .venv
.venv\Scripts\activate

# macOS / Linux
python -m venv .venv
source .venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

> First install takes a few minutes — it downloads the `all-MiniLM-L6-v2` embedding model (~90 MB) on first run.

### 4. Configure environment variables

Copy the example env file and fill in your Groq API key:

```bash
cp .env.example .env
```

Open `.env` and set your key:

```env
GROQ_API_KEY=your_groq_api_key_here
```

Everything else has sensible defaults and works out of the box.

### 5. Run the app

```bash
streamlit run streamlit_app.py
```

The app opens automatically at `http://localhost:8501`.

---

## Environment Variables

All variables are optional except `GROQ_API_KEY`. Copy `.env.example` to `.env` and override only what you need.

### LLM

| Variable | Default | Description |
|---|---|---|
| `GROQ_API_KEY` | *(required)* | Your Groq API key from [console.groq.com](https://console.groq.com) |
| `LLM_MODEL` | `llama-3.3-70b-versatile` | Groq model to use |
| `LLM_TEMPERATURE` | `0.2` | Lower = more factual, higher = more creative |
| `LLM_MAX_TOKENS` | `1024` | Max tokens in the LLM response |
| `NOT_IN_DOCUMENT_PHRASE` | `Not in document.` | Fallback when no relevant chunks are found |

### Embeddings

| Variable | Default | Description |
|---|---|---|
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | HuggingFace sentence-transformer model |

### Vector Database

| Variable | Default | Description |
|---|---|---|
| `CHROMA_PATH` | `./chroma_db` | Where ChromaDB persists on disk |
| `CHROMA_COLLECTION_NAME` | `documents` | ChromaDB collection name |
| `VECTOR_DISTANCE_SPACE` | `cosine` | Distance metric (`cosine` or `l2`) |
| `SIMILARITY_SCORE_THRESHOLD` | `0.3` | Minimum similarity score to include a chunk (0–1) |

### Document Processing

| Variable | Default | Description |
|---|---|---|
| `CHUNK_SIZE` | `500` | Characters per chunk |
| `CHUNK_OVERLAP` | `100` | Overlap between adjacent chunks |
| `TOP_K` | `3` | Number of chunks retrieved per query |
| `MAX_FILE_SIZE_MB` | `20` | Max upload size in MB |
| `SUPPORTED_FILE_EXTENSIONS` | `.pdf,.pptx,.txt` | Comma-separated allowed extensions |

### Logging

| Variable | Default | Description |
|---|---|---|
| `LOG_DIR` | `logs` | Directory for log files |
| `LOG_FILE` | `app.log` | Log filename |
| `LOG_LEVEL` | `INFO` | Log level (`DEBUG`, `INFO`, `WARNING`, `ERROR`) |
| `LOG_TO_CONSOLE` | `false` | Set `true` to also print logs to terminal |

---

## Usage

1. **Upload a document** — Click "Choose a file" in the sidebar, select a PDF, PPTX, or TXT file, then click "Upload & Index"
2. **Wait for indexing** — The file is extracted, chunked, embedded, and stored (usually 5–15 seconds depending on file size)
3. **Ask questions** — Type your question in the chat input at the bottom of the page
4. **Re-upload safely** — Uploading the same file again is detected automatically and skipped

---

## Architecture

```
User
 │
 ▼
streamlit_app.py          (UI — upload widget, chat display)
 │
 ▼
streamlit_backend.py      (Orchestration)
 ├── process_upload()
 │    ├── extractor.py    (PDF/PPTX/TXT → LangChain Documents)
 │    ├── chunker.py      (Documents → chunks)
 │    └── vectordb.py     (chunks → ChromaDB, duplicate detection)
 │
 └── ask_question()
      ├── injection_detector.py   (security guard)
      ├── vectordb.py             (similarity retrieval)
      ├── chain.py                (LCEL chain: context + question → LLM)
      └── llm.py                  (cached ChatGroq)
```

### RAG Pipeline (per query)

```
Query
  │
  ├─► Injection detector — block malicious input
  │
  ├─► ChromaDB retriever — find top-K similar chunks (filtered by document_id)
  │
  ├─► format_docs() — join chunks into context string
  │
  ├─► LCEL chain:
  │     { context, question }
  │       → ChatPromptTemplate
  │       → ChatGroq (llama-3.3-70b-versatile)
  │       → StrOutputParser
  │
  └─► Answer string returned to UI
```

---

## Notes

- **ChromaDB is local** — all vectors are stored in `./chroma_db` on your machine. Nothing is sent to a remote vector database.
- **Embeddings run locally** — `all-MiniLM-L6-v2` runs on your CPU via `sentence-transformers`. No API calls for embeddings.
- **Only the LLM call hits an external API** — that's the Groq API for inference.
- **File watcher is disabled** — set in `.streamlit/config.toml` to suppress noisy `transformers` module inspection logs in the terminal. Auto-reload on file save is off; restart Streamlit manually after code changes.
- **Logs** rotate automatically at 10 MB, keeping the last 5 files, stored in `./logs/app.log`.

---

## Troubleshooting

**`GROQ_API_KEY not set` error in the UI**
Add your key to `.env` and restart Streamlit.

**`No text could be extracted` on upload**
The file may be a scanned image-only PDF with no embedded text. Try a different file.

**Answers seem unrelated to the document**
Lower `SIMILARITY_SCORE_THRESHOLD` in `.env` (e.g. `0.2`) to retrieve more chunks, or increase `TOP_K` (e.g. `5`).

**Slow first upload**
The embedding model (`all-MiniLM-L6-v2`) is downloaded from HuggingFace on first use and cached locally. Subsequent runs are fast.

**Terminal still noisy after setup**
Make sure `.streamlit/config.toml` exists with:
```toml
[server]
fileWatcherType = "none"
```
