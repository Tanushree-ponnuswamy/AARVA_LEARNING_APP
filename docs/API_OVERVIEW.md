# API Documentation – AARVA Backend

The **AARVA** backend provides a RESTful API built on **FastAPI**. All JSON responses follow consistent schemas.

- **Swagger Interactive Docs**: `http://127.0.0.1:8000/docs`
- **ReDoc**: `http://127.0.0.1:8000/redoc`

---

## 1. Authentication & Session (`/api/auth`)

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/auth/send-otp` | Generate and dispatch a 4-digit email verification OTP via SMTP |
| `POST` | `/api/auth/verify-otp` | Validate a submitted OTP code |
| `POST` | `/api/auth/reset-password` | Verify OTP and update account password; auto-login on success |
| `POST` | `/api/auth/signup` | Register a new student (personal info + preferences + OTP verified) |
| `POST` | `/api/auth/login` | Student login with email & password — returns JWT |
| `GET` | `/api/auth/me` | Fetch authenticated user's profile and progress (Bearer Token) |
| `POST` | `/api/auth/logout` | Revoke all active `UserSession` rows for the user (Bearer Token) |
| `GET` | `/api/auth/sessions` | List active login sessions for the current user (Bearer Token) |
| `GET` | `/api/auth/chat-sessions` | List all chat sessions for the current user (Bearer Token) |

---

## 2. Textbooks & Document Ingestion (`/api/textbooks`)

Supported formats: **PDF, DOCX, DOC, XLSX, TXT, MD, CSV, RST, and images (OCR via EasyOCR/pytesseract)**.

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/textbooks/` | List all documents uploaded by a user (scoped by `user_id` query param) |
| `POST` | `/api/textbooks/upload` | Upload a document, extract text, chunk, embed in ChromaDB, store metadata |
| `GET` | `/api/textbooks/{textbook_id}/file` | Stream the raw uploaded file with correct `Content-Type` for browser preview |
| `DELETE` | `/api/textbooks/{textbook_id}` | Delete record, ChromaDB vector chunks, and file from disk |

---

## 3. Document Summarization & RAG Chat (`/api/summary`, `/api/chat`)

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/summary/{textbook_id}` | Retrieve summary in one of 4 modes: `complete`, `chapter`, `concept`, `page` |
| `GET` | `/api/summary/chat/history/{textbook_id}` | Retrieve persisted chat message history for a document |
| `POST` | `/api/summary/chat` | Query AI Tutor via Hybrid RAG (Chroma dense + BM25 sparse + RRF) |
| `POST` | `/api/chat` | Direct alias for `POST /api/summary/chat` |

### Summary Modes (`GET /api/summary/{textbook_id}?mode=...`)

| Mode | Behaviour |
|---|---|
| `complete` | Returns overview, key points, topics covered, chapters, concepts, definitions, and important notes |
| `chapter` | Per-chapter breakdown (uses stored `summary_data.chapters` or LLM generation) |
| `concept` | AI-generated concept deep-dive |
| `page` | AI-generated page-level summary |

---

## 4. Quiz Engine (`/api/quiz`)

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/quiz/generate` | Generate quiz questions via Hybrid RAG + LLM (falls back to static bank if Ollama unavailable) |
| `GET` | `/api/quiz/{textbook_id}` | Retrieve the static fallback question set |
| `POST` | `/api/quiz/submit` | Grade answers, award XP, unlock badges, return performance feedback |
| `GET` | `/api/quiz/leaderboard/top` | Top 10 students by XP from `LearningProgress` |

### Quiz Generate Request
```json
{
  "textbook_id": 1,
  "topic": "Transport Layer",
  "num_questions": 5,
  "difficulty": "mixed"
}
```

### Quiz Submit Request
```json
{
  "user_id": 1,
  "textbook_id": 1,
  "topic": "Transport Layer Protocols",
  "answers": [
    {"question_id": 1, "selected_option": 1, "is_correct": true},
    {"question_id": 2, "selected_option": 0, "is_correct": false}
  ]
}
```

### Quiz Submit Response
```json
{
  "success": true,
  "score": 4,
  "total": 5,
  "percentage": 80.0,
  "passed": true,
  "xp_earned": 65,
  "feedback": "Great work! You have a solid grasp. Review the explanation for any missed questions.",
  "new_badges_unlocked": [],
  "new_badge_unlocked": null
}
```

---

## 5. Student Dashboard (`/api/student`)

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/student/dashboard/{user_id}` | Dashboard stats: textbook count, quiz history, XP, level, adaptive insights, badges |
| `POST` | `/api/student/award-xp` | Award XP and update level/title in `LearningProgress` |
| `PUT` | `/api/student/preferences/{user_id}` | Update learning preferences (TTS, STT, larger text, language) |

---

## 6. Admin Portal (`/api/admin`)

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/admin/metrics` | Total registered users + active sessions count (last 4 hours) |
| `GET` | `/api/admin/students` | All users — searchable by name/email, filterable by learner type |
| `GET` | `/api/admin/students/{student_id}` | Full profile + login activity log (last 10 entries) |
| `POST` | `/api/admin/students/{student_id}/toggle-status` | Toggle `is_active` for a student account |

---

## 7. System & Health

| Method | Path | Description |
|---|---|---|
| `GET` | `/` | Root — returns system status and portal metadata |
| `GET` | `/api/health` | Health check — confirms database and ChromaDB are ready |

---

## 8. Sample Payloads

### Hybrid RAG Chat (`POST /api/chat`)
**Request:**
```json
{
  "textbook_id": 1,
  "query": "Explain the difference between TCP slow start and congestion avoidance.",
  "language": "English",
  "conversation_history": [
    {"role": "user", "content": "Hello!"},
    {"role": "assistant", "content": "Hello! How can I help you?"}
  ]
}
```
**Response:**
```json
{
  "response": "TCP Slow Start doubles the congestion window each RTT until ssthresh is reached...",
  "sources": [
    {"chunk_id": "...", "page": 42, "chapter": "Chapter 3: Transport Layer", "text": "..."}
  ]
}
```

### Reset Password (`POST /api/auth/reset-password`)
**Request:**
```json
{
  "email": "student@college.edu",
  "code": "4821",
  "new_password": "newSecurePassword123"
}
```
**Response:**
```json
{
  "success": true,
  "message": "Password reset successfully.",
  "access_token": "<jwt>",
  "user": {
    "id": 1,
    "name": "Alex Morgan",
    "email": "student@college.edu",
    "role": "student",
    "learner_type": "college"
  }
}
```

