# API Documentation – AARVA Backend

The **AARVA** backend provides a RESTful API built on **FastAPI**. All JSON responses follow consistent schemas and secured endpoints are protected via **JWT Bearer Authentication** (except public auth, demo accounts, health checks, and OTP verification flows).

- **Swagger Interactive OpenAPI Docs**: `http://127.0.0.1:8000/docs`
- **ReDoc Interactive Documentation**: `http://127.0.0.1:8000/redoc`

---

## 1. Authentication & Session Endpoints (`/api/auth`)

| Method | Path | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/auth/send-otp` | Generate and dispatch 4-digit email verification OTP via SMTP | None |
| `POST` | `/api/auth/verify-otp` | Validate submitted OTP code for registration or password reset | None |
| `POST` | `/api/auth/reset-password` | Verify reset OTP and update account password | None |
| `POST` | `/api/auth/signup` | Register new student (all 4 personas) and issue JWT token | None |
| `POST` | `/api/auth/login` | Student / Admin login with email & password | None |
| `GET` | `/api/auth/me` | Fetch authenticated user's profile, interests, and progress | Bearer Token |
| `POST` | `/api/auth/logout` | Revoke active user session in database | Bearer Token |
| `GET` | `/api/auth/sessions` | List active device sessions for authenticated user | Bearer Token |
| `GET` | `/api/auth/chat-sessions` | List student chat sessions and conversation history | Bearer Token |

---

## 2. Textbooks & Document Ingestion (`/api/textbooks`)

Supports multi-format ingestion: **PDF (`.pdf`), Word (`.docx`), Excel spreadsheets (`.xlsx`), and plain text (`.txt`)**.

| Method | Path | Description | Auth Required |
|---|---|---|---|
| `GET` | `/api/textbooks/` | List all textbooks & documents uploaded by current user | Optional (scoped by `user_id` query param or Bearer) |
| `POST` | `/api/textbooks/upload` | Upload document, extract text, chunk and index in ChromaDB | Optional (associates with `user_id`) |
| `GET` | `/api/textbooks/{textbook_id}/file` | Stream uploaded document with appropriate `Content-Type` for in-browser viewing | None |
| `DELETE` | `/api/textbooks/{textbook_id}` | Delete document, metadata, and associated ChromaDB embeddings | Optional |

---

## 3. Document Summarization & RAG AI Chat (`/api/summary`, `/api/chat`)

| Method | Path | Description | Auth Required |
|---|---|---|---|
| `GET` | `/api/summary/{textbook_id}` | Retrieve pre-computed or generated 5-tier hierarchical summary (Complete, Chapter, Concepts, Glossary, Notes) | None |
| `GET` | `/api/summary/chat/history/{textbook_id}` | Retrieve persistent chat message history for a document | None |
| `POST` | `/api/summary/chat` | Query AI Tutor via Hybrid RAG (Chroma dense + BM25 sparse) with exact citations | None |
| `POST` | `/api/chat` | Direct top-level alias for Hybrid RAG AI Tutor Chat | None |

---

## 4. 3-Tier Adaptive Quiz Engine (`/api/quiz`)

| Method | Path | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/quiz/generate` | Generate dynamic practice questions for a document | None |
| `GET` | `/api/quiz/{textbook_id}` | Retrieve 3-tier adaptive practice challenge (Basic, Medium, Advanced) | None |
| `POST` | `/api/quiz/submit` | Grade answers, calculate percentage, award XP, update streak, and provide AI feedback on incorrect answers | None |
| `GET` | `/api/quiz/leaderboard/top` | Retrieve top students by XP and level rankings | None |

---

## 5. Student Dashboard & Study Tasks (`/api/students`)

| Method | Path | Description | Auth Required |
|---|---|---|---|
| `GET` | `/api/students/dashboard/{user_id}` | Fetch student dashboard metrics, streaks, active tasks, and recent tests | None |
| `POST` | `/api/students/award-xp` | Award XP, increment levels, and update learning progress milestones | None |
| `PUT` | `/api/students/preferences/{user_id}` | Update accessibility and learning preferences (TTS, STT, font size, language) | None |

---

## 6. Admin Portal Endpoints (`/api/admin`)

| Method | Path | Description | Auth Required |
|---|---|---|---|
| `GET` | `/api/admin/metrics` | System telemetry (total students, active sessions, documents, average quiz score) | Admin Role |
| `GET` | `/api/admin/students` | Paginated and persona-filtered roster of enrolled students | Admin Role |
| `GET` | `/api/admin/students/{student_id}` | Deep inspection of profile, uploaded documents, and login audits | Admin Role |
| `POST` | `/api/admin/students/{student_id}/toggle-status` | Toggle student account status (`is_active: true/false`) | Admin Role |

---

## 7. System & Health Endpoints

| Method | Path | Description | Auth Required |
|---|---|---|---|
| `GET` | `/` | System root and platform metadata | None |
| `GET` | `/api/health` | Health check verifying database and vector store readiness | None |

---

## 8. Sample Request & Response Payloads

### A. Reset Password (`POST /api/auth/reset-password`)
**Request:**
```json
{
  "email": "student@college.edu",
  "otp": "4821",
  "new_password": "newSecurePassword123"
}
```
**Response (`200 OK`):**
```json
{
  "success": true,
  "message": "Password reset successfully. You can now log in with your new password."
}
```

### B. Hybrid RAG AI Tutor Chat (`POST /api/chat`)
**Request:**
```json
{
  "textbook_id": 1,
  "query": "What is the difference between TCP slow start and congestion avoidance?",
  "language": "English",
  "conversation_history": [
    {"role": "user", "content": "Hello!"},
    {"role": "assistant", "content": "Hello Alex! How can I help you with Computer Networks today?"}
  ]
}
```
**Response (`200 OK`):**
```json
{
  "answer": "TCP Slow Start increases the congestion window exponentially (doubling every RTT) until the Slow Start Threshold (ssthresh) is reached. Once ssthresh is surpassed, TCP enters Congestion Avoidance, increasing the window additively (by 1 MSS per RTT) to probe network capacity gently.",
  "citations": [
    {
      "chapter": "Chapter 3: Transport Layer",
      "page": 42,
      "snippet": "During slow start, cwnd begins at 1 MSS and increments by 1 MSS for every acknowledged segment..."
    }
  ],
  "suggested_followups": [
    "How does Fast Retransmit work in TCP Reno?",
    "What triggers a transition back to Slow Start?"
  ]
}
```

### C. Quiz Submission & AI Remediation (`POST /api/quiz/submit`)
**Request:**
```json
{
  "user_id": 7,
  "textbook_id": 1,
  "topic": "Transport Layer Protocols",
  "score": 4,
  "total_questions": 5,
  "answers": [
    {"question_id": 1, "selected": "B", "correct": "B"},
    {"question_id": 2, "selected": "A", "correct": "C"}
  ]
}
```
**Response (`200 OK`):**
```json
{
  "success": true,
  "score": 4,
  "total": 5,
  "percentage": 80.0,
  "xp_awarded": 40,
  "new_xp": 280,
  "streak_days": 13,
  "remediation": [
    {
      "question_id": 2,
      "concept": "Three-Way Handshake",
      "explanation": "SYN-ACK is sent by the server in step 2, not ACK alone.",
      "ai_discussion_prompt": "Explain step 2 of the TCP 3-way handshake in detail."
    }
  ]
}
```

