# AARVA — AI-Based Adaptive Learning & Document Intelligence System

[![FastAPI](https://img.shields.io/badge/Backend-FastAPI-009688.svg?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![React](https://img.shields.io/badge/Frontend-React%2019%20%7C%20TypeScript-61DAFB.svg?logo=react&logoColor=black)](https://react.dev)
[![ChromaDB](https://img.shields.io/badge/Vector%20Store-ChromaDB-FF6600.svg)](https://www.trychroma.com)
[![Ollama](https://img.shields.io/badge/AI%20Engine-Qwen3%20%7C%20Llama%203%20%7C%20Mistral-blue.svg)](https://ollama.com)
[![SQLite / PostgreSQL](https://img.shields.io/badge/Database-PostgreSQL%20%2F%20SQLite-336791.svg?logo=postgresql&logoColor=white)](https://www.postgresql.org)

**AARVA** is an AI-driven learning and document intelligence platform that transforms uploaded textbooks, lecture notes, and documents into interactive learning experiences — with an AI tutor, multi-mode summarizer, adaptive quizzes, and a study task planner.

---

## 🌟 Key Features

### 1. Authentication & Account Management
- **Login**: Email and password login for students; JWT token issued on success and stored in `localStorage`.
- **Signup (3 steps)**: Personal info → Interests & preferences → Email OTP verification.
  - Learner types supported: `school`, `college`, `professional`, `independent`.
  - Captures: institution, department, year/semester/grade/job role, subjects, goals, and learning preferences (TTS, STT, font size, etc.).
- **Forgot Password**: Email OTP dispatch via SMTP → 4-digit code entry → new password set; auto-login on success.
- **Session Tracking**: Every login creates a `UserSession` row (JWT `jti`, IP, device, expiry). Logout revokes all active sessions for the user.

### 2. Multi-Format Document Ingestion
- **Supported formats**: PDF (`.pdf`), Word (`.docx`, `.doc`), Excel (`.xlsx`), plain text (`.txt`, `.md`, `.csv`, `.rst`), and images (`.png`, `.jpg`, `.jpeg`, `.bmp`, `.tiff`, `.webp`) via OCR (EasyOCR + pytesseract fallback).
- **Ingestion pipeline**: Parse raw file → Semantic paragraph chunking (120-char overlap) → Embed into ChromaDB → Store metadata & summary in PostgreSQL/SQLite.
- **File serving**: Raw uploaded files streamed back to the browser with the correct `Content-Type` for in-browser preview.
- **Deletion**: Removes the database record, all associated ChromaDB vector chunks, and the local file on disk.

### 3. Document Summarizer (5 Modes)
Pre-computed `summary_data` stored at upload time, or generated on-demand via LLM:
1. **Complete Overview** — high-level summary, key points, topics covered, chapters, definitions, and important notes.
2. **Chapter Breakdown** — per-chapter summary with key points and definitions.
3. **Core Concepts** — concept-focused AI generation.
4. **Page/Section** — page-level AI generation.
5. **Chat History** — stored conversation messages per textbook session (retrieved separately).

### 4. Hybrid RAG AI Tutor Chat
- **Hybrid Retrieval**: ChromaDB dense vector search (1024-dim cosine) + BM25 sparse lexical search, merged via **Reciprocal Rank Fusion (RRF)** (`k=60`, dense weight 0.65, sparse weight 0.35).
- **AI Engine**: Qwen3 8B (primary via Ollama) with automatic fallback to Llama 3, Mistral, or any available local model.
- **Multi-turn conversation**: Full conversation history passed on every request for context continuity.
- **Chat persistence**: Messages saved into `ChatSession` / `ChatMessage` rows in the database when `user_id` and `textbook_id` are provided.
- **Direct alias**: `POST /api/chat` mirrors `POST /api/summary/chat`.

### 5. Adaptive Quiz Engine
- **AI-generated questions**: Uses Hybrid RAG + LLM to generate questions directly from the uploaded document. Falls back to a curated static question bank when Ollama is unavailable.
- **Difficulty levels**: `easy`, `medium`, `hard`, or `mixed`.
- **Submission & scoring**: Correct count, percentage, XP award (50 XP base + bonus up to 30 XP for ≥ 75%), performance feedback message, and badge unlocks.
- **Badges unlocked on quiz**: `Quiz Ace` (≥ 90%), `Perfect Score` (100%), `Knowledge Seeker` (5+ quizzes completed).
- **Leaderboard**: Top 10 students by XP from `LearningProgress`; falls back to a static mock list if no data exists.

### 6. Student Dashboard
- **Overview cards**: Books uploaded count and points earned (books × 80).
- **Study Task Planner**: Add custom tasks with duration presets (15, 30, 45, 60 min). Tasks are persisted per user via `localStorage`.
- **Library view**: Grid of all uploaded books; open a book to enter the document viewer + AI tutor workspace.
- **Tests view**: Start quizzes against any uploaded book.
- **Profile view**: Displays name, email, and total book count.
- **Settings view**: Toggle learning preferences (TTS, STT, larger text, simplified explanations); saved via `PUT /api/student/preferences/{user_id}`.

### 7. Admin Portal
- **Metrics**: Total registered users and count of active sessions (login activity in the last 4 hours).
- **Student roster**: List all users, searchable by name/email and filterable by learner type.
- **Student detail view**: Full profile data and login activity log (last 10 entries).
- **Account control**: Toggle student account active/inactive status with a single click.
- **Admin access**: Admin accounts are seeded at startup; admin login is not accessible from the student portal.

---

## 🏗️ System Architecture

```mermaid
flowchart TD
    subgraph ClientLayer ["Frontend (React 19 + TypeScript + Vite — single App.tsx SPA)"]
        Landing["Landing Page (Hero, Features, About, Contact)"]
        Auth["Auth Page (Login / 3-Step Signup / Forgot Password)"]
        Dash["Dashboard Shell (Sidebar Nav)"]
        Upload["Upload Workspace (Document Viewer + AI Tutor Chat + Summarizer)"]
        Library["Library View (Book Grid)"]
        Tests["Tests View (Quiz Engine)"]
        ProfileSettings["Profile & Settings"]
    end

    subgraph APILayer ["FastAPI Gateway (:8000)"]
        AuthRoute["/api/auth"]
        AdminRoute["/api/admin"]
        TextbookRoute["/api/textbooks"]
        SummaryRoute["/api/summary"]
        ChatRoute["/api/chat"]
        QuizRoute["/api/quiz"]
        StudentRoute["/api/student"]
    end

    subgraph ServiceLayer ["AI & Processing Services"]
        DocParser["Document Parser (PDF, DOCX, DOC, XLSX, TXT, Images/OCR)"]
        Chunker["Semantic Paragraph Chunker (overlap 120 chars)"]
        HybridEngine["Hybrid Retrieval (ChromaDB Dense + BM25 + RRF k=60)"]
        LLMService["LLM Service (Qwen3 8B primary → Llama3/Mistral fallback via Ollama)"]
        AdaptiveEngine["Adaptive Service (insights & recommendations)"]
        EmailService["SMTP Email Service (OTP dispatch)"]
    end

    subgraph StorageLayer ["Persistence"]
        DB[("PostgreSQL / SQLite\nusers, profiles, interests, preferences\nlogin_activity, user_sessions\ntextbooks, quiz_results\nchat_sessions, chat_messages\nlearning_progress")]
        ChromaVDB[("ChromaDB Vector Store\n1024-dim cosine embeddings")]
        Disk["Local /uploads (raw files for streaming)"]
    end

    Auth --> AuthRoute
    Upload --> TextbookRoute
    Upload --> ChatRoute
    Upload --> SummaryRoute
    Tests --> QuizRoute
    Dash --> StudentRoute
    Auth --> AdminRoute

    AuthRoute --> EmailService
    AuthRoute --> DB
    AdminRoute --> DB
    StudentRoute --> DB
    QuizRoute --> AdaptiveEngine
    QuizRoute --> DB

    TextbookRoute --> DocParser
    DocParser --> Disk
    DocParser --> Chunker
    Chunker --> ChromaVDB
    TextbookRoute --> DB

    SummaryRoute --> LLMService
    ChatRoute --> HybridEngine
    HybridEngine --> ChromaVDB
    HybridEngine --> LLMService
```

---

## 🛠️ Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Frontend Framework** | React 19 + TypeScript | High-performance SPA with modern React hooks & state |
| **Build Tool** | Vite 8 + Tailwind CSS v4 | Fast HMR dev server, instant build, modern CSS utility engine |
| **Document Preview** | `pdfjs-dist`, `mammoth`, `xlsx` | In-browser client preview for PDF, Word (.docx), and Excel (.xlsx) |
| **Backend Framework** | FastAPI (Python 3.10+) | Async REST API with OpenAPI documentation (`/docs`) |
| **Database & ORM** | PostgreSQL / SQLite + SQLAlchemy 2.0 | Users, sessions, textbooks, quizzes, chat history |
| **Vector Database** | ChromaDB | Local persistent vector store with cosine distance |
| **Document Ingestion** | PyMuPDF, pdfplumber, python-docx, mammoth, openpyxl, EasyOCR, pytesseract | Text & tabular extraction across PDF, DOCX, XLSX, TXT, and images |
| **Hybrid Retrieval** | ChromaDB Dense + BM25 (rank-bm25) | Reciprocal Rank Fusion (RRF `k=60`) combining semantic & keyword search |
| **AI Inference** | Qwen3 8B (primary), Llama 3 / Mistral (fallbacks) via Ollama | Multi-mode summarization, adaptive quiz generation, grounded RAG tutor |
| **Auth & Security** | JWT (HS256) + Passlib (Bcrypt) + SMTP OTP | Token-based auth, session tracking, email OTP verification |

---

## 📁 Repository Structure

```
d:\Aarva\
├── frontend/                          # React 19 / TypeScript / Vite SPA
│   ├── public/                        # Static assets (logo, icons)
│   ├── src/
│   │   ├── App.tsx                    # Entire frontend — all pages and components
│   │   ├── index.css                  # Modern styles (glassmorphism, themes, layout)
│   │   └── main.tsx                   # React entry point
│   ├── package.json
│   └── vite.config.ts
├── backend/                           # FastAPI Python Application
│   ├── app.py                         # ASGI app, CORS middleware, startup seeder, /api/chat alias
│   ├── config.py                      # Settings loaded from .env
│   ├── models/models.py               # SQLAlchemy ORM models
│   ├── database/
│   │   ├── postgres.py                # DB engine, session factory, demo seed data
│   │   └── chroma.py                  # ChromaDB collection management & retrieval
│   ├── routes/
│   │   ├── auth.py                    # Signup, login, OTP send/verify, reset-password, sessions
│   │   ├── admin.py                   # Metrics, student roster, student detail, status toggle
│   │   ├── students.py                # Dashboard, XP award, preferences update
│   │   ├── textbooks.py               # Upload, file serve, delete
│   │   ├── summary.py                 # Summary modes, chat history, RAG chat
│   │   └── quiz.py                    # Generate questions, submit answers, leaderboard
│   ├── services/
│   │   ├── llama_service.py           # Ollama inference — RAG chat & summarization
│   │   ├── hybrid_retrieval.py        # RRF fusion of ChromaDB dense + BM25 sparse
│   │   ├── knowledge_base_service.py  # End-to-end ingestion pipeline coordinator
│   │   ├── document_parser.py         # Multi-format text extraction (PDF, DOCX, XLSX, TXT, OCR)
│   │   ├── chunking_service.py        # Semantic paragraph chunker
│   │   ├── quiz_service.py            # AI question generation + static fallback bank
│   │   ├── adaptive_service.py        # Quiz insight and recommendations logic
│   │   ├── auth_service.py            # JWT creation and bcrypt password hashing
│   │   └── email_service.py           # SMTP OTP dispatch and verification
│   └── requirements.txt
└── docs/
    ├── ARCHITECTURE.md                # System architecture, workflows & tech stack detailed notes
    ├── DB_SCHEMA.md                   # Database schema and ER diagram
    └── API_OVERVIEW.md                # REST API endpoint reference
```

---

## ⚙️ Setup & Configuration

### 1. Prerequisites
- **Node.js** v18+ and `npm`
- **Python** 3.10+
- **Ollama** for local LLM inference — run `ollama pull qwen3:8b` (or `ollama pull llama3`)

### 2. Backend Setup
```bash
cd backend

# Create and activate virtual environment
python -m venv venv
# Windows:
venv\Scripts\activate
# Linux/macOS:
# source venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

> Configure your `.env` file in `backend/` by referencing `backend/.env.example` for all configurable keys (database connection, JWT secret, Ollama base URL, SMTP mail configuration, ChromaDB storage path, etc.).

### 3. Frontend Setup
```bash
cd ../frontend
npm install
```

---

## 🚀 Running Locally

### 1. Start the FastAPI Backend
```bash
cd backend
python -m uvicorn app:app --port 8000 --reload
```
- **API**: `http://127.0.0.1:8000`
- **Swagger Docs**: `http://127.0.0.1:8000/docs`
- **Health Check**: `http://127.0.0.1:8000/api/health`

### 2. Start the React Frontend
```bash
cd frontend
npm run dev
```
Open `http://localhost:5173` in your browser.

---

## 🔑 Demo Accounts

Pre-seeded at startup via `database/postgres.py`:

| Role | Email | Password | Learner Type |
|---|---|---|---|
| **College Student** | `student@college.edu` | `student123` | College — Alex Morgan |
| **School Student** | `priya.patel@school.edu` | `student123` | School — Priya Patel |
| **Working Professional** | `rohan.iyer@techcorp.com` | `student123` | Professional — Rohan Iyer |
| **Independent Learner** | `elena.rostova@global.org` | `student123` | Independent — Elena Rostova |
| **Administrator** | `admin@aarva.edu` | `admin123` | Admin |

---

## 📚 Documentation Reference

- 📘 [Architecture Overview & Workflow Diagrams](file:///d:/Aarva/docs/ARCHITECTURE.md)
- 🗄️ [Database Schema & ER Diagram](file:///d:/Aarva/docs/DB_SCHEMA.md)
- 🔌 [API Endpoints Reference](file:///d:/Aarva/docs/API_OVERVIEW.md)

---

## 🎥 Project Demo Video

- **Watch the Video Walkthrough on Google Drive**: [AARVA System Demo Video](https://drive.google.com/file/d/1ybVgyU1iQzW7RVQTdciTqkYm3qatSfJi/view?usp=sharing)
