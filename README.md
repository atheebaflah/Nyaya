

## Overview

Nyaya addresses a critical gap in legal access in Sri Lanka — complex legal language and limited availability of structured legal information. The platform ingests, indexes, and retrieves Sri Lankan statutes, case law, and legal concepts, then uses a hybrid retrieval strategy to generate accurate, cited answers through a conversational interface.

This project was initiated as a university Software Development Group Project (SDGP, Group CS-108) and has since evolved into a fully production-deployed platform.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Frontend (Vercel)                    │
│                   Next.js + TypeScript + Tailwind           │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTPS / REST
┌──────────────────────────▼──────────────────────────────────┐
│                     Backend API (Railway)                   │
│                   FastAPI + Python (Async)                  │
│                                                             │
│   ┌─────────────────┐        ┌──────────────────────────┐   │
│   │  Hybrid Retrieval│       │     LLM Generation       │   │
│   │                 │       │                          │   │
│   │  ┌───────────┐  │       │   Google Gemini API      │   │
│   │  │  Qdrant   │  │       │   (Grounded Response)    │   │
│   │  │ (Semantic)│  │       └──────────────────────────┘   │
│   │  └───────────┘  │                                      │
│   │  ┌───────────┐  │                                      │
│   │  │   Neo4j   │  │                                      │
│   │  │  (Graph)  │  │                                      │
│   │  └───────────┘  │                                      │
│   └─────────────────┘                                      │
└─────────────────────────────────────────────────────────────┘
```

### Hybrid Retrieval Strategy

Nyaya uses two complementary retrieval engines that run in parallel and whose results are merged before being passed to the LLM:

| Engine | Database | What it finds |
|--------|----------|----------------|
| Semantic Search | Qdrant | Contextually similar passages via dense vector embeddings |
| Graph Traversal | Neo4j | Related statutes, legal concepts, and case references via relationship edges |

---

## Tech Stack

### Backend
- **FastAPI** — async Python web framework
- **Qdrant** — vector database for semantic search
- **Neo4j** — knowledge graph for legal entity relationships
- **Sentence Transformers** — local embedding model (dense vectors)
- **Google Gemini API** — LLM for response generation
- **Python async/await** — non-blocking ETL and retrieval pipeline

### Frontend
- **Next.js 14** (App Router)
- **TypeScript**
- **Tailwind CSS**
- **Vercel** — deployment and edge functions

### Infrastructure
- **Railway** — backend and database hosting
- **Vercel** — frontend hosting + CDN
- **Cloudflare** — DNS, WAF, and CDN layer
- **Docker** — containerised services for local development

---

## Features

- **Conversational legal Q&A** — Ask questions in plain English about Sri Lankan law
- **Hybrid RAG** — Semantic search + knowledge graph traversal for high-recall retrieval
- **Source citations** — Every answer is grounded and cites the relevant statute or case
- **Knowledge graph** — Legal concepts, statutes, and cases are linked as a graph, enabling multi-hop retrieval
- **Async pipeline** — Non-blocking ingestion and retrieval for production throughput
- **Responsive UI** — Works on desktop and mobile

---

## Project Structure

```
nyaya/
├── backend/
│   ├── app/
│   │   ├── api/            # FastAPI route handlers
│   │   ├── core/           # Config, settings, startup
│   │   ├── retrieval/      # Hybrid retrieval logic
│   │   │   ├── qdrant.py   # Vector search
│   │   │   ├── neo4j.py    # Graph traversal
│   │   │   └── hybrid.py   # Merge & rerank
│   │   ├── generation/     # LLM prompting and response assembly
│   │   ├── ingestion/      # Document parsing and indexing pipeline
│   │   └── models/         # Pydantic schemas
│   ├── Dockerfile
│   └── requirements.txt
│
├── frontend/
│   ├── app/                # Next.js App Router pages
│   ├── components/         # Reusable UI components
│   ├── lib/                # API client and utilities
│   └── public/
│
├── docker-compose.yml      # Local dev: Qdrant + Neo4j + API
└── README.md
```

---

## Getting Started

### Prerequisites

- Python 3.11+
- Node.js 18+
- Docker & Docker Compose
- Google Gemini API key
- Qdrant instance (local via Docker or Qdrant Cloud)
- Neo4j instance (local via Docker or Neo4j Aura)

### 1. Clone the repository

```bash
git clone https://github.com/atheebaflah/nyaya.git
cd nyaya
```

### 2. Start infrastructure (Qdrant + Neo4j)

```bash
docker-compose up -d
```

### 3. Backend setup

```bash
cd backend
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env      # Fill in your keys
uvicorn app.main:app --reload
```

### 4. Frontend setup

```bash
cd frontend
npm install
cp .env.local.example .env.local   # Fill in API base URL
npm run dev
```

The app will be available at `http://localhost:3000` and the API at `http://localhost:8000`.

---

## Environment Variables

### Backend (`.env`)

```env
# Google Gemini
GEMINI_API_KEY=your_gemini_api_key

# Qdrant
QDRANT_URL=http://localhost:6333
QDRANT_COLLECTION=nyaya_legal

# Neo4j
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=your_password

# App
ENVIRONMENT=development
```

### Frontend (`.env.local`)

```env
NEXT_PUBLIC_API_BASE_URL=http://localhost:8000
```

---

## Deployment

### Backend → Railway

1. Push the `backend/` directory to a Railway project.
2. Set all environment variables in Railway's dashboard.
3. Railway auto-detects the `Dockerfile` and builds on push.

### Frontend → Vercel

```bash
cd frontend
vercel deploy --prod
```

Set `NEXT_PUBLIC_API_BASE_URL` to your Railway backend URL in Vercel's environment settings.

### DNS → Cloudflare

Point your domain to Vercel's nameservers and enable the Cloudflare proxy for WAF and CDN.

---

## RAG Pipeline

### Ingestion

1. Legal documents (statutes, case law) are parsed and chunked.
2. Each chunk is embedded using **Sentence Transformers** and stored in **Qdrant**.
3. Legal entities and relationships (Acts → Sections → Cases) are extracted and stored as nodes/edges in **Neo4j**.

### Retrieval

On each user query:
1. The query is embedded and used to perform **dense vector search** in Qdrant (top-K chunks).
2. Key legal terms are extracted and used to run **graph traversal** in Neo4j (related statutes and cases).
3. Both result sets are **merged and deduplicated**, then passed as context to the LLM.

### Generation

The merged context and the user's question are formatted into a structured prompt and sent to **Google Gemini**. The response includes the answer and source references.

---

## API Reference

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/query` | Submit a legal question, returns answer + sources |
| `GET` | `/api/health` | Health check |
| `POST` | `/api/ingest` | Trigger document ingestion (admin) |

### Example Request

```bash
curl -X POST https://your-api.railway.app/api/query \
  -H "Content-Type: application/json" \
  -d '{"question": "What are the grounds for divorce under Sri Lankan law?"}'
```

### Example Response

```json
{
  "answer": "Under the Marriage Registration Ordinance (Cap. 111), a marriage may be dissolved on grounds including...",
  "sources": [
    { "title": "Marriage Registration Ordinance", "section": "Section 24", "relevance": 0.91 },
    { "title": "Kandyan Marriage Act", "section": "Section 19", "relevance": 0.87 }
  ]
}
```

---

## Nyaya Quiz

A built-in quiz module for testing and reinforcing legal knowledge. While the core platform answers legal questions, the Quiz feature lets users actively test themselves across topics and difficulty levels.

### Features

- Multiple quiz topics (JavaScript, Python, SQL, and legal concepts)
- Three difficulty levels per topic: Easy, Medium, Hard
- Instant feedback with explanations after each answer
- Score tracking and progress visualization
- User attempt history to review past performance
- Responsive UI built with Tailwind CSS

### Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend | Next.js 16, React 19, Tailwind CSS 4 |
| Backend | FastAPI, SQLAlchemy |
| Database | PostgreSQL via Supabase |

### Quiz Setup

#### Prerequisites

- Node.js 18+
- Python 3.8+
- Supabase account with a PostgreSQL database

#### Backend

```bash
cd backend
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

Create a `.env` file in the `backend/` directory:

```env
DATABASE_URL=postgresql://username:password@localhost:5432/nyaya_quiz
```

Initialize the database with tables and sample questions:

```bash
python init_db.py
```

Start the backend server:

```bash
uvicorn main:app --reload
# API runs at http://127.0.0.1:8000
```

#### Frontend

```bash
cd frontend
npm install
npm run dev
# App runs at http://localhost:3000
```

---

## Roadmap

- [ ] User authentication and saved query history
- [ ] Multi-language support (Sinhala, Tamil)
- [ ] Fine-tuned embedding model on Sri Lankan legal corpus
- [ ] Citation deep-links to source documents
- [ ] Admin dashboard for ingestion management
- [ ] Mobile app (React Native)

---

## Author

**Adheeb** — Computer Science Undergraduate, Informatics Institute of Technology (IIT), affiliated with the University of Westminster, UK.

- GitHub: [@atheebaflah](https://github.com/atheebaflah)
- Project: [Nyaya on GitHub](https://github.com/atheebaflah/nyaya)

---

> *Nyaya (न्याय) — Sanskrit/Pali for "justice" or "rule of law".*
