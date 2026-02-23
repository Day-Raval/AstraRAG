# AstraRAG — Agentic RAG Chatbot

AstraRAG is an **agentic Retrieval-Augmented Generation (RAG)** chatbot that answers questions using a local document knowledge base.
It combines:

- **CrewAI** for agent/task orchestration,
- **LlamaIndex + ChromaDB** for retrieval,
- **FastAPI** for the backend API,
- **Streamlit** for the chat UI,
- **Groq-hosted LLMs** for response generation.

The repository already includes sample Biology PDFs in `docs_dir/` that can be ingested into a vector store.

---

## Architecture Overview

1. **Document ingestion** (`src/rag_doc_ingestion/ingest_docs.py`)
   - Loads PDFs from `DOCUMENTS_DIR`
   - Splits documents into chunks
   - Embeds chunks with Hugging Face embeddings
   - Stores vectors in a persistent Chroma collection

2. **Backend API** (`src/backend_src/main.py` + `src/backend_src/api/chat.py`)
   - Exposes `POST /chat/answer`
   - Accepts full chat history
   - Sends query + prior context to CrewAI pipeline

3. **Agentic QA flow** (`src/agents_src/*`)
   - A Crew with one QA agent and one QA task
   - Uses `rag_query_tool` to retrieve supporting context from vector store
   - Returns structured output: `answer`, `sources`, `tool_used`, and `rationale`

4. **Frontend UI** (`src/frontend_src/app.py`)
   - Streamlit chat app
   - Displays assistant answer, source files, and rationale/tool details

---

## Project Structure

```text
AstraRAG/
├── docs_dir/                         # Source PDFs to index
├── src/
│   ├── rag_doc_ingestion/            # One-time ingestion pipeline
│   ├── backend_src/                  # FastAPI app
│   ├── frontend_src/                 # Streamlit app
│   └── agents_src/                   # CrewAI agents, tasks, tools, and LLM config
├── env_template.txt                  # Sample environment variables
├── requirements.txt                  # Python dependencies
├── start.sh                          # Ingest + backend + frontend launcher
└── Dockerfile                        # Containerized deployment
```

---

## Prerequisites

- Python **3.11+**
- A valid **Groq API key**

---

## Environment Configuration

1. Create an environment file:

```bash
cp env_template.txt .env
```

2. Update `.env` values:

```env
GROQ_API_KEY="your_groq_api_key"
DOCUMENTS_DIR="/absolute/or/project-relative/path/to/docs_dir"
VECTOR_STORE_DIR="/absolute/or/project-relative/path/to/doc_vector_store"
COLLECTION_NAME="document_collection"
MODEL_NAME="llama-3.3-70b-versatile"
MODEL_TEMPERATURE=0.0
CHAT_ENDPOINT_URL="http://localhost:8000/chat/answer"
```

> Tip: If you run from the project root, `DOCUMENTS_DIR="docs_dir"` and `VECTOR_STORE_DIR="doc_vector_store"` are convenient defaults.

---

## Local Setup & Run

### 1) Install dependencies

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 2) Ingest documents (one-time or when docs change)

```bash
python -m src.rag_doc_ingestion.ingest_docs
```

### 3) Start backend API

```bash
uvicorn src.backend_src.main:app --host 0.0.0.0 --port 8000
```

### 4) Start frontend (new terminal)

```bash
streamlit run src/frontend_src/app.py --server.port 8501 --server.address 0.0.0.0
```

### 5) Open the app

- Streamlit UI: http://localhost:8501
- FastAPI endpoint: http://localhost:8000/chat/answer

---

## One-Command Startup

The included script runs ingestion, then backend + frontend:

```bash
chmod +x start.sh
./start.sh
```

---

## Docker Deployment

### Build image

```bash
docker build -t astrarag-chatbot:latest .
```

### Run container

```bash
docker run -p 8000:8000 -p 8501:8501 -e GROQ_API_KEY=your_groq_api_key astrarag-chatbot:latest
```

### Detached mode

```bash
docker run -d -p 8000:8000 -p 8501:8501 -e GROQ_API_KEY=your_groq_api_key astrarag-chatbot:latest
```

---

## API Contract

### `POST /chat/answer`

Request body:

```json
{
  "chat_history": [
    {"role": "user", "content": "What is evolution?"},
    {"role": "assistant", "content": "..."},
    {"role": "user", "content": "Explain adaptive radiation."}
  ]
}
```

Example response:

```json
{
  "answer": "...",
  "sources": ["6. Evolution.pdf"],
  "tool_used": "rag_query_tool",
  "rationale": "..."
}
```

---

## Troubleshooting

- **`GROQ_API_KEY` missing / auth failures**
  - Ensure `.env` exists and key is valid.
- **No sources returned**
  - Re-run ingestion and verify `DOCUMENTS_DIR` and `VECTOR_STORE_DIR` paths.
- **Frontend cannot reach backend**
  - Confirm `CHAT_ENDPOINT_URL` points to the running backend.
- **Slow first run**
  - Initial embedding model download can take time.

---

## Useful Commands

```bash
# Ingestion only
python -m src.rag_doc_ingestion.ingest_docs

# Quick crew test
python -m src.agents_src.check_crew

# Backend
python -m src.backend_src.main

# Frontend
streamlit run src/frontend_src/app.py
```

---

## License

Add your preferred license information here.
