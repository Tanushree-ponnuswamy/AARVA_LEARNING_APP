# Architecture Overview – AARVA System

**AARVA** (AI-Based Adaptive Learning & Document Intelligence System) is an enterprise-grade, modular educational platform engineered for personalized learning across school students, college undergraduates, working professionals, and lifelong learners.

---

## 1. System Architecture Diagram

```mermaid
flowchart TB
    subgraph Client ["Frontend Client (React 18 + TypeScript + Vite SPA)"]
        UI["Web UI Components & Themes (Dark / Light)"]
        DocViewer["Multi-Format Document Viewer (PDF / DOCX / XLSX / TXT)"]
        VoiceSTT["Speech-to-Text (STT) & Audio TTS (Web Speech API)"]
        LangContext["i18n Multilingual Context (6 Languages)"]
        TaskPlanner["Interactive Study Task Planner (Per-User Isolation)"]
        QuizModule["3-Tier Adaptive Quiz & AI Discussion Drawer"]
        ApiClient["Axios / Fetch API Gateway Client"]
    end

    subgraph BackendGateway ["FastAPI Application Gateway (:8000)"]
        CORSMiddleware["CORS & Request Middleware"]
        AuthRouter["/api/auth (JWT, OTP, Sessions, Reset Password)"]
        StudentRouter["/api/students (Profiles, Dashboards, Tasks, XP)"]
        TextbookRouter["/api/textbooks (Multi-Format Upload, Stream, Delete)"]
        SummaryRouter["/api/summary (5-Mode Hierarchical Summarizer)"]
        ChatRouter["/api/chat (Hybrid RAG AI Tutor)"]
        QuizRouter["/api/quiz (Adaptive Quiz Evaluator & Leaderboard)"]
        AdminRouter["/api/admin (Platform Telemetry & User Mgmt)"]
    end

    subgraph AIServices ["AI Intelligence & Retrieval Tier"]
        DocParser["Multi-Format Document Parser\n(pypdf, python-docx, openpyxl)"]
        SemanticChunker["Semantic Paragraph Chunker (Overlap & Boundary Detection)"]
        BM25Engine["Sparse BM25 Lexical Search Index"]
        ChromaService["ChromaDB Dense Embedding Engine (1024-dim Cosine)"]
        RRFCombiner["Reciprocal Rank Fusion (RRF) Hybrid Merger (k=60)"]
        LLMService["AI Engine (Qwen3 8B Primary, Llama 3 & Mistral Fallbacks)"]
        AdaptiveEngine["Adaptive Engine (Difficulty Calibration & Remediation)"]
    end

    subgraph DataPersistence ["Storage & Persistence Tier"]
        PostgresDB[("PostgreSQL / SQLite Database\n- Users & Credentials (Bcrypt)\n- Student Profiles & Personas\n- Learning Preferences & UI Settings\n- Progress, Streaks & XP Levels\n- Active User Sessions (token_jti)\n- Quiz Submissions & Leaderboards\n- Chat Sessions & Threaded Messages")]
        ChromaStore[("ChromaDB Vector Store\n- Semantic Text Chunk Embeddings\n- Document & Chapter Metadata\n- Cosine Distance Collections")]
        DocStorage["Local Storage (/uploads)\n- Preserved Raw Documents for Streaming"]
    end

    %% Client to Backend Gateway
    UI --> ApiClient
    DocViewer --> ApiClient
    VoiceSTT --> ApiClient
    LangContext --> UI
    TaskPlanner --> ApiClient
    QuizModule --> ApiClient
    ApiClient --> CORSMiddleware
    
    CORSMiddleware --> AuthRouter
    CORSMiddleware --> StudentRouter
    CORSMiddleware --> TextbookRouter
    CORSMiddleware --> SummaryRouter
    CORSMiddleware --> ChatRouter
    CORSMiddleware --> QuizRouter
    CORSMiddleware --> AdminRouter

    %% Gateway to Services
    TextbookRouter --> DocParser
    DocParser --> DocStorage
    DocParser --> SemanticChunker
    SemanticChunker --> ChromaService
    SemanticChunker --> BM25Engine

    ChatRouter --> RRFCombiner
    RRFCombiner --> ChromaService
    RRFCombiner --> BM25Engine
    RRFCombiner --> LLMService

    SummaryRouter --> LLMService
    QuizRouter --> AdaptiveEngine
    AdaptiveEngine --> LLMService

    %% Services to Persistence
    ChromaService --> ChromaStore
    AuthRouter --> PostgresDB
    AdminRouter --> PostgresDB
    StudentRouter --> PostgresDB
    QuizRouter --> PostgresDB
    TextbookRouter --> PostgresDB
    ChatRouter --> PostgresDB
```

---

## 2. Key Architectural Layers

### A. Client Layer (Frontend SPA)
- **Framework**: React 18, TypeScript, Vite.
- **State Management & Contexts**:
  - `AuthContext`: Role-based authentication (`student` / `admin`), JWT token persistence in `localStorage`, user session sync, and persona demo switching.
  - `ThemeContext`: Dynamic light and dark mode toggling, 6-language internationalization (English, Hindi, Tamil, Telugu, Spanish, French), and Web Speech API synthesis/recognition.
- **Component Architecture**:
  - `UniversalSignup.tsx`: 4-step dynamic adaptive onboarding stepper with persona-tailored inputs (School, College, Professional, Independent).
  - `StudentPortal.tsx`: Dual-view learning space coordinator featuring:
    - Left side: Full Document & Spreadsheet Viewer (`iframe` / PDF preview, DOCX, XLSX rendering) with tab toggle to AI Tutor Chat.
    - Right side: Hierarchical Document Summarizer (Complete Book, Chapter Breakdown, Core Concepts, Glossary, Important Notes) with Markdown & Print export.
  - `AIChat.tsx`: Real-time RAG-powered tutor with grounded chapter and page citations, audio speech playback, and suggested follow-ups.
  - `QuizModule.tsx`: 3-tier adaptive practice quizzes (Basic, Medium, Advanced) with instant pedagogical answers, celebratory confetti, and wrong-answer AI Discussion drawer.
  - `ProgressView.tsx`: Daily streak counter (🔥), concept mastery meters, and XP level progression.
  - `AdminPortal.tsx`: System telemetry dashboard, student inspection modal, and account suspension controls.

### B. Gateway & API Layer (FastAPI Backend)
- **FastAPI**: Asynchronous Python ASGI web server providing interactive OpenAPI (Swagger) documentation and typed Pydantic validation.
- **Security & Session Management**:
  - Passwords hashed with `bcrypt` (12 rounds).
  - Stateless JSON Web Tokens (`HS256`) paired with database-backed `user_sessions` tracking for multi-device visibility and instant session revocation.
  - Email verification codes and password recovery dispatched via SMTP (`services/email_service.py`).

### C. Multi-Format Ingestion & Hybrid RRF Retrieval Pipeline
- **Document Parsing**: Ingests PDFs (`pypdf`), Word documents (`python-docx`), Excel spreadsheets (`openpyxl`), and plain text (`.txt`).
- **Semantic Paragraph Chunker**: Splits documents into cohesive chunks respecting paragraph boundaries, headers, and bullet structures with configurable overlaps (`CHUNK_OVERLAP=120`).
- **Hybrid Retrieval (Dense + BM25 + RRF)**:
  - Dense similarity search via ChromaDB vector embeddings (1024-dim cosine distance).
  - Sparse lexical search via BM25 to catch exact terminology, acronyms, formulas, and section identifiers.
  - **Reciprocal Rank Fusion (RRF)**:
    $$\text{RRF Score}(d) = w_{\text{dense}} \cdot \frac{1}{k + \text{rank}_{\text{dense}}(d)} + w_{\text{sparse}} \cdot \frac{1}{k + \text{rank}_{\text{sparse}}(d)}$$
    Where $k = 60$, $w_{\text{dense}} = 0.65$, and $w_{\text{sparse}} = 0.35$.
- **Primary AI Engine (`Qwen3 8B` / `Qwen2.5`)**:
  - Serves as the primary inference model for chapter summarization, key concept extraction, adaptive question generation, and grounded RAG tutor chat.
  - Automated fallback to `Llama 3`, `Mistral`, or local CPU/GPU profiles via Ollama.

---

## 3. End-to-End Workflow Diagrams

### A. Hybrid RAG AI Tutor Chat Pipeline

```mermaid
sequenceDiagram
    autonumber
    actor Student as Student (Web / Mobile)
    participant ChatUI as AIChat Component
    participant API as FastAPI (/api/chat)
    participant RRF as Hybrid RRF Engine
    participant Chroma as ChromaDB (Dense)
    participant BM25 as BM25 (Sparse)
    participant LLM as Qwen / Llama AI Service
    participant DB as PostgreSQL Database

    Student->>ChatUI: Ask question (Voice input or typed text)
    ChatUI->>API: POST /api/chat { textbook_id, query, language, history }
    API->>RRF: Request hybrid retrieval for query
    par Parallel Search
        RRF->>Chroma: Dense embedding similarity query (Top-K)
        RRF->>BM25: Sparse keyword lexical search (Top-K)
    end
    Chroma-->>RRF: Dense ranked candidates
    BM25-->>RRF: Sparse ranked candidates
    RRF->>RRF: Merge ranks using Reciprocal Rank Fusion (k=60)
    RRF-->>API: Top ranked context chunks with chapter & page metadata
    API->>LLM: Synthesize grounded response with context & student persona
    LLM-->>API: Response { answer, citations, suggested_followups }
    API->>DB: Persist message into ChatMessage table
    API-->>ChatUI: JSON response { answer, citations, followups }
    ChatUI-->>Student: Display formatted answer + Trigger TTS Voice Readout
```

### B. 3-Tier Adaptive Quiz Submission & Remediation

```mermaid
sequenceDiagram
    autonumber
    actor Student as Student
    participant QuizUI as QuizModule Component
    participant Drawer as AI Discussion Drawer
    participant API as FastAPI (/api/quiz/submit)
    participant DB as PostgreSQL Database

    Student->>QuizUI: Submits completed 3-tier adaptive quiz
    QuizUI->>API: POST /api/quiz/submit { textbook_id, answers, topic }
    API->>DB: Save QuizResult record
    API->>DB: Update LearningProgress (XP += 40, Streak += 1, Mastery %)
    DB-->>API: Return updated learning progress
    API-->>QuizUI: Return { score, percentage, xp_awarded, streak, remediation }
    QuizUI-->>Student: Display score card, celebratory confetti, and wrong answer review
    
    opt Discuss Wrong Answer
        Student->>Drawer: Click "Discuss with AI Tutor"
        Drawer->>API: POST /api/chat { query: "Why is option C correct for question 2?" }
        API-->>Drawer: Detailed step-by-step remediation explanation
        Drawer-->>Student: Render pedagogical remediation and memory tips
    end
```

### C. Multi-Format Document Ingestion & Vector Indexing

```mermaid
sequenceDiagram
    autonumber
    actor Student as Student
    participant UI as StudentPortal Component
    participant API as FastAPI (/api/textbooks/upload)
    participant Parser as Multi-Format Parser
    participant Chunker as Semantic Chunker
    participant Chroma as ChromaDB Vector Store
    participant DB as PostgreSQL Database

    Student->>UI: Select PDF / Word (.docx) / Excel (.xlsx) / TXT file
    UI->>API: POST /api/textbooks/upload (multipart/form-data)
    API->>Parser: Parse raw document and extract text + page/sheet boundaries
    Parser-->>API: Structured text stream with section headers
    API->>Chunker: Semantic paragraph chunking with overlap (CHUNK_OVERLAP=120)
    Chunker-->>API: Chunks with document ID, page, and chapter metadata
    API->>Chroma: Embed and index chunks into vector collection
    API->>DB: Save Textbook record with file metadata and summary data
    API-->>UI: Return { textbook_id, title, total_pages, status: "processed" }
    UI-->>Student: Instant display in Library with Document Preview & Summarizer
```

