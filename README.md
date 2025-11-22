# Hybrid Support Bot (Advanced RAG) – BMW X5 Owner’s Manual

It implements a Retrieval-Augmented Generation (RAG) support assistant over the **BMW X5 Owner’s Manual (2025)** using:

- ✅ Local LLM (via **Ollama** – `llama3`)
- ✅ Local embeddings (`nomic-embed-text`)
- ✅ Vector database (**ChromaDB**)
- ✅ Metadata-aware retrieval (chapter + page)
- ✅ Latency logging (retrieval vs generation)
- ✅ “I don’t know based on the manual.” fallback to avoid hallucinations

---

## 📚 Problem Statement (Option 1 – “The Hybrid” Support Bot)

Goal:  
Build a support bot that can accurately answer user questions using a  (BMW X5 Owner’s Manual) with:

- Structured ingestion pipeline (one-time)
- Hybrid search: **metadata (chapter) filter + vector similarity**
- Local model only (no external APIs)
- Tracking retrieval latency & model generation latency
- Graceful handling of unknown questions

 What the Bot Can Do

Given a question like:

- “How do I open the tailgate?”
- “How do I activate parking assist?”
- “How do I turn on automatic headlights?”
- “How do I adjust the climate control?”
- “What does the tire pressure warning mean?”

The bot will:

1. Use the question to **guess the relevant chapter** (e.g., “Getting in”, “Driving”, “Climate control”).
2. Run **vector search in Chroma**, filtered by that chapter.
3. Build a context prompt from the top chunks.
4. Ask a **local LLM (`llama3` via Ollama)** to answer.
5. Return:
   - The answer
   - The chunks used (with chapter + page)
   - Retrieval latency (ms)
   - Generation latency (ms)
   - Which chapter filter was applied

If the answer is not in the manual, it responds:

> `"I don't know based on the manual."`

---

## 🏗 Tech Stack

- **Language**: Python 3.10
- **Backend**: FastAPI + Uvicorn
- **LLM runtime**: [Ollama](https://ollama.com/)  
  - `llama3` – for answer generation  
  - `nomic-embed-text` – for text embeddings
- **Vector DB**: ChromaDB (persistent mode)
- **PDF parsing**: `pdfplumber`
- **HTTP client**: `httpx`

---

## 📁 Project Structure

```text
support_bot_rag/
│
├── main.py                 # FastAPI app (query endpoint)
├── config.py               # All configuration (paths, models, thresholds)
├── requirements.txt
│
├── data/
│   └── 2025-bmw-x5-owners-manual.pdf   # PDF knowledge base
│
├── enbeddings/             # ChromaDB persistent storage (created after ingestion)
│
├── utils/
│   ├── __init__.py
│   ├── ollama_client.py    # Calls Ollama for embeddings + generation
│   └── text_chunker.py     # Merges line-level text into chunks
│
├── ingestion/
│   ├── __init__.py
│   ├── parse_pdf.py        # Parses PDF and detects chapter headings
│   └── build_index.py      # End-to-end ingestion: PDF → chunks → embeddings → Chroma
│
└── api/
    ├── __init__.py
    ├── schemas.py          # Pydantic models for request/response
    └── retrieval.py        # Hybrid retrieval + RAG answer pipeline
⚙️ Setup Instructions
1. Clone / download the project
cd C:\Users\ASUS\OneDrive\Desktop
# or wherever you keep the project
cd support_bot_rag

2. Python virtual environment (recommended)
python -m venv venv
venv\Scripts\activate

3. Install dependencies
pip install --default-timeout=200 -r requirements.txt

4. Install & prepare Ollama

Download Ollama from: https://ollama.com/download
 and install.

Then in a terminal:

ollama pull llama3
ollama pull nomic-embed-text


Check installed models:

ollama list

🧩 Ingestion Pipeline (One-Time Step)

This step:

Loads the BMW X5 manual from data/2025-bmw-x5-owners-manual.pdf

Extracts text page by page

Detects chapter/section headings (e.g., “Getting in”, “On the road”, “Parking”)

Converts lines into chunks (800 characters with overlap)

Sends chunks to Ollama (nomic-embed-text) to get embeddings

Stores chunks + embeddings + metadata in ChromaDB



After this, the enbeddings/ folder will contain the Chroma DB.

🚀 Running the API

Start the FastAPI app:

venv\Scripts\activate
uvicorn main:app --port 8001


You should see:

INFO:     Uvicorn running on http://127.0.0.1:8001

💬 Querying the Support Bot


http://127.0.0.1:8001/docs


Click POST /query

Click “Try it out”

Use a JSON body like:

{
  "question": "How do I open the tailgate?"
}


Click Execute

You’ll receive a response like:

{
  "answer": "...",
  "sources": [
    {
      "text": "...",
      "chapter": "Getting in",
      "page": 52,
      "score": 0.93
    }
  ],
  "retrieval_latency_ms": 45.7,
  "generation_latency_ms": 812.3,
  "applied_chapter_filter": "Getting in"
}


In the console running uvicorn, you’ll also see logs:

======= QUERY LOG =======
Q: How do I open the tailgate?
Chapter filter: Getting in
Retrieval (ms): 45.7
Generation (ms): 812.3
- Getting in p.52 score=0.93
=========================


These logs provide evidence of metadata filtering and performance tracking (important for the challenge).

🧪 Example Questions to Try

How do I fold the rear seats?

How do I activate cruise control?

How do I enable child safety locks?

How do I turn on automatic high beam?

What does the tire pressure warning mean?

And also test the “I don’t know” case:

What is the capital of France?

Expected answer:

"I don't know based on the manual."

🧱 Design Highlights

Separation of Concerns

ingestion/ handles one-time PDF parsing and indexing.

main.py + api/ handle runtime querying.

Metadata-Aware Retrieval

Each chunk stores chapter + page + source.

Query text is used to guess a chapter, and Chroma is filtered on that chapter before vector search.

Local-Only Inference

Both embeddings and generation use Ollama models.

No external cloud APIs are required.

Latency Tracking

Retrieval time (Chroma) and generation time (Ollama) are measured separately and returned in the API response.

Hallucination Control

System prompt enforces:
"If the answer is not clearly contained in the context, reply exactly: 'I don't know based on the manual.'".

Similarity threshold (MIN_SIMILARITY_THRESHOLD) provides an additional guard.

🧭 Possible Improvements

Smarter chapter detection using PDF font/size/style

LLM-based router to select chapter instead of simple keyword matching

Simple web frontend (React/HTML) on top of the FastAPI backend

Caching frequent queries


Better error handling around Ollama / Chroma failures
