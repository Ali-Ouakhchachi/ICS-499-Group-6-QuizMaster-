# Quiz Master — Software Requirements Specification (SRS)
### FP2: Project Elaboration Milestone
**Course:** Software Engineering | **Professor:** Siva Jasthi
**Status:** Living Document — FP2 Deliverable | **Version:** 1.0.0

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Functional Requirements](#2-functional-requirements)
   - 2.1 [User Roles & Authentication](#21-user-roles--authentication)
   - 2.2 [Runtime AI Engine Mechanics](#22-runtime-ai-engine-mechanics)
   - 2.3 [Quiz Engine & State Management](#23-quiz-engine--state-management)
   - 2.4 [Feedback System](#24-feedback-system)
3. [Technical Architecture & Data Schema](#3-technical-architecture--data-schema)
   - 3.1 [Recommended Tech Stack](#31-recommended-tech-stack)
   - 3.2 [System Architecture Diagram (Textual)](#32-system-architecture-diagram-textual)
   - 3.3 [Database Entity Relationship Map](#33-database-entity-relationship-map)
4. [AI-Friendly Repository Blueprint](#4-ai-friendly-repository-blueprint)
   - 4.1 [Folder Directory Tree](#41-folder-directory-tree)
   - 4.2 [Prompt File Conventions](#42-prompt-file-conventions)
   - 4.3 [.gitignore Layout](#43-gitignore-layout)
5. [Edge Cases & Resilience Strategy](#5-edge-cases--resilience-strategy)
6. [Non-Functional Requirements](#6-non-functional-requirements)
7. [Glossary](#7-glossary)

---

## 1. Project Overview

**Quiz Master** is an AI-heavy, full-stack, database-backed web application that enables dynamic quiz generation, intelligent answer evaluation, and personalized learning feedback — all powered by live LLM API calls at runtime.

Unlike traditional quiz platforms backed by static question banks, Quiz Master treats the LLM as a core first-class service: every quiz is generated fresh, every open-ended response is evaluated by AI, and every post-quiz debrief is personalized. The system supports two concurrent user roles — **Students** and **Instructors** — each with distinct workflows and data visibility scopes.

### 1.1 Project Goals

| # | Goal | Rationale |
|---|------|-----------|
| G1 | Dynamic, AI-generated quizzes | Eliminates rote memorization of question banks |
| G2 | Multi-role access control | Separates student UX from instructor management |
| G3 | Persistent, stateful quiz sessions | Prevents cheating from browser refreshes or disconnections |
| G4 | AI-graded short-answer evaluation | Enables open-ended questions beyond MCQ formats |
| G5 | Personalized post-quiz explanations | Drives genuine learning, not just scoring |
| G6 | Deployable to shared hosting (Bluehost/FTP) | Real-world deployment constraint for the project |

### 1.2 Scope Boundaries

**In Scope:**
- Web application (desktop-first, mobile-responsive)
- Two user roles: Student, Instructor
- AI-generated Multiple Choice, True/False, and Short Answer questions
- AI-graded short answers and AI-generated post-quiz feedback
- Quiz session persistence (resume on disconnect)
- Instructor dashboard with aggregate class analytics
- JWT-based authentication

**Out of Scope (FP2 → Consider for FP4+):**
- Mobile native apps
- Third-party SSO (Google, GitHub login)
- Real-time collaborative quizzes (multiplayer)
- Payment/subscription tiers
- Plagiarism detection

---

## 2. Functional Requirements

### 2.1 User Roles & Authentication

#### 2.1.1 Role Definitions

| Role | Description | Key Capabilities |
|------|-------------|------------------|
| **Student** | Learner who takes quizzes | Take quizzes, view own history & scores, receive AI feedback |
| **Instructor** | Educator who configures quizzes | Create quiz configs, view all student results, access analytics |
| **Admin** *(future)* | System administrator | Manage users, override data, system config |

> **Note:** For FP2/FP3 scope, Admin capabilities are handled directly via database access. A full Admin UI is a stretch goal.

#### 2.1.2 Authentication Flow

**FR-AUTH-01 — Registration**
- All new users register with: `email`, `password`, `display_name`, `role` (Student default).
- Instructors are granted the Instructor role either by an Admin or via a pre-shared invite code stored in the environment config.
- Passwords must be hashed using **bcrypt** (minimum 12 salt rounds) before being stored. Plaintext passwords must **never** be written to the database or any log file.

**FR-AUTH-02 — Login**
- Users submit `email` + `password`.
- Backend validates credentials, hashes the provided password, compares against stored hash.
- On success: issue a signed **JWT** (`access_token`, 1-hour TTL) and a **refresh token** (stored in `httpOnly` cookie, 7-day TTL).
- On failure: return generic `401 Unauthorized` — do **not** distinguish between "email not found" and "wrong password" in the response body (prevents user enumeration).

**FR-AUTH-03 — Token Refresh**
- Client automatically calls `POST /api/auth/refresh` when it receives a `401` on any protected route.
- Backend validates the refresh token from the `httpOnly` cookie and issues a new access token.
- If the refresh token is expired or invalid, the user is logged out and redirected to `/login`.

**FR-AUTH-04 — Logout**
- `POST /api/auth/logout` clears the `httpOnly` refresh token cookie server-side.
- Client discards the access token from memory (never store JWTs in `localStorage`).

**FR-AUTH-05 — Route Authorization**
- All API routes beyond `/auth/*` and `/public/*` require a valid JWT Bearer token in the `Authorization` header.
- Role-based middleware (`requireRole('instructor')`) guards all Instructor endpoints.
- A Student attempting to access an Instructor route receives `403 Forbidden`.

#### 2.1.3 Student Capabilities

| FR ID | Capability | Description |
|-------|-----------|-------------|
| FR-STU-01 | Browse available quizzes | View list of published quizzes with topic, difficulty, question count |
| FR-STU-02 | Start a quiz | Initiate a new quiz session; backend generates questions via LLM |
| FR-STU-03 | Resume a quiz | Re-enter an in-progress quiz session if browser was closed |
| FR-STU-04 | Submit answers | Submit one answer per question; cannot modify after moving to next question |
| FR-STU-05 | View score | See final score immediately after submission |
| FR-STU-06 | View AI feedback | Access personalized explanation for each incorrect answer |
| FR-STU-07 | View quiz history | See a paginated list of all completed quizzes with dates and scores |

#### 2.1.4 Instructor Capabilities

| FR ID | Capability | Description |
|-------|-----------|-------------|
| FR-INS-01 | Create quiz configuration | Define topic, difficulty, question count, question types, time limit |
| FR-INS-02 | Edit quiz configuration | Update settings before quiz is made available to students |
| FR-INS-03 | Publish / unpublish quiz | Toggle quiz visibility for students |
| FR-INS-04 | View individual student results | See every student's responses and scores for a given quiz |
| FR-INS-05 | View aggregate analytics | Class-wide average score, per-question difficulty heatmap, completion rates |
| FR-INS-06 | Delete a quiz | Soft-delete (mark as archived) a quiz config; retains historical score data |

---

### 2.2 Runtime AI Engine Mechanics

This section defines exactly how the backend interfaces with the LLM at runtime. This is the most critical architectural subsystem of the application.

#### 2.2.1 LLM Service Abstraction Layer

The backend must implement a dedicated `AIService` module that serves as the **only** point of contact with the external LLM API. Business logic controllers must never call the LLM API directly.

```
Controller → AIService → LLM Provider (OpenAI / Anthropic)
```

This abstraction means switching providers (e.g., OpenAI GPT-4o → Anthropic Claude) requires changing only the `AIService` module, not the entire codebase.

#### 2.2.2 Quiz Generation Workflow

**Trigger:** Student clicks "Start Quiz" on a configured quiz.

**Step-by-step backend process:**

```
1. POST /api/quiz-sessions/start
   ├── Auth middleware validates JWT
   ├── Fetch quiz config by quiz_config_id from DB
   ├── Check if student already has an active session → if yes, return existing session
   └── Call AIService.generateQuestions(config)

2. AIService.generateQuestions(config):
   ├── Load system prompt from: /backend/prompts/quiz-generation/system.txt
   ├── Load user prompt template from: /backend/prompts/quiz-generation/user.template.txt
   ├── Interpolate template variables:
   │   ├── {topic}         → config.topic
   │   ├── {difficulty}    → config.difficulty (easy | medium | hard)
   │   ├── {question_count}→ config.question_count
   │   └── {question_types}→ config.allowed_types (mcq, true_false, short_answer)
   ├── Attach JSON schema constraint to prompt (see §2.2.3)
   ├── Call LLM API with timeout: 15 seconds
   ├── Parse and validate response (see §2.2.4)
   └── Return validated question array

3. On success:
   ├── Insert questions into DB (Questions table)
   ├── Create new row in Quiz_Sessions table
   ├── Set session.status = 'active', session.expires_at = NOW() + time_limit
   └── Return session_id + first question to client

4. On failure: see §5 (Edge Cases)
```

#### 2.2.3 Structural Prompt Design

**System Prompt** (stored in `/backend/prompts/quiz-generation/system.txt`):

```
You are a strict JSON-only quiz question generator. You must respond ONLY with a valid 
JSON array — no markdown, no preamble, no explanations. Every response must conform 
exactly to the schema provided in the user message. If you cannot generate a question 
that fits the schema, omit it from the array entirely. Never generate partial objects.
```

**User Prompt Template** (stored in `/backend/prompts/quiz-generation/user.template.txt`):

```
Generate {question_count} quiz questions on the topic: "{topic}".
Difficulty level: {difficulty}.
Allowed question types: {question_types}.

Your response MUST be a JSON array matching this exact schema. Return nothing else.

SCHEMA:
[
  {
    "type": "mcq",
    "question_text": "string (the question)",
    "options": ["string", "string", "string", "string"],
    "correct_answer": "string (must match one option exactly)",
    "explanation": "string (why this is the correct answer)"
  },
  {
    "type": "true_false",
    "question_text": "string",
    "correct_answer": "True" | "False",
    "explanation": "string"
  },
  {
    "type": "short_answer",
    "question_text": "string",
    "sample_answer": "string (a model correct answer for grading reference)",
    "grading_rubric": "string (key concepts that must be present in a correct answer)"
  }
]
```

> **Critical design rule:** The `explanation` and `grading_rubric` fields are generated at question-creation time and stored in the database. They are **never** re-fetched from the LLM at grading time, which eliminates a second latency-sensitive API call in the critical path.

#### 2.2.4 LLM Response Parsing & Validation

The AI response must be treated as **untrusted input** and validated against the expected schema before any data is written to the database.

**Parsing pipeline in `AIService`:**

```javascript
// Pseudocode
function parseAndValidateLLMResponse(rawText, expectedCount) {
  // Step 1: Strip common LLM artifacts
  let cleaned = rawText.trim();
  cleaned = cleaned.replace(/^```json\s*/i, '').replace(/```\s*$/, '');
  cleaned = cleaned.replace(/^```\s*/i, '').replace(/```\s*$/, '');

  // Step 2: Attempt JSON parse
  let parsed;
  try {
    parsed = JSON.parse(cleaned);
  } catch (e) {
    throw new AIParseError('LLM returned non-JSON content', { raw: rawText });
  }

  // Step 3: Validate top-level structure
  if (!Array.isArray(parsed)) {
    throw new AIParseError('LLM response is not an array');
  }

  // Step 4: Validate each item against schema
  const valid = [];
  for (const item of parsed) {
    if (!item.type || !item.question_text) continue; // skip malformed
    if (item.type === 'mcq') {
      if (!Array.isArray(item.options) || item.options.length < 2) continue;
      if (!item.options.includes(item.correct_answer)) continue;
    }
    if (item.type === 'true_false') {
      if (!['True', 'False'].includes(item.correct_answer)) continue;
    }
    valid.push(item);
  }

  // Step 5: Check if we have enough questions
  if (valid.length < Math.ceil(expectedCount * 0.7)) {
    throw new AIParseError(`Insufficient valid questions: got ${valid.length}, needed ${expectedCount}`);
  }

  return valid;
}
```

**Error escalation:**
- `AIParseError` is caught by the controller.
- The controller retries the LLM call **once** with a simplified prompt.
- If the retry also fails, a `503 Service Temporarily Unavailable` response is returned to the client with a user-friendly message: *"Quiz generation is temporarily unavailable. Please try again in a moment."*

#### 2.2.5 Short Answer AI Grading

**Trigger:** Student submits a short-answer response.

**Workflow:**

```
1. Student submits: { session_id, question_id, answer_text }
2. Backend fetches from DB: question.sample_answer, question.grading_rubric
3. Call AIService.gradeShortAnswer(student_answer, sample_answer, rubric)
4. Grading prompt (from /backend/prompts/grading/user.template.txt):
   "Student answer: {student_answer}
    Model answer: {sample_answer}
    Key concepts required: {grading_rubric}
    
    Evaluate on a scale of 0–2:
    2 = Correct (all key concepts present)
    1 = Partially correct (some concepts present)
    0 = Incorrect
    
    Respond ONLY with this JSON: { "score": 0|1|2, "rationale": "one sentence" }"
5. Parse response, store User_Responses.ai_score and User_Responses.ai_rationale
6. Return { score, rationale } to client
```

---

### 2.3 Quiz Engine & State Management

#### 2.3.1 Session Architecture

Every active quiz is represented as a **Quiz Session** row in the database. The client never stores authoritative quiz state — it only stores the `session_id` in memory (or `sessionStorage`). On every page load, the client calls `GET /api/quiz-sessions/:session_id` to hydrate state from the server.

This design ensures:
- Browser refreshes do not reset progress
- The server is the single source of truth for score and timer
- Cheating via client-side manipulation is structurally impossible

#### 2.3.2 Timer Implementation

Timers are **server-enforced**, not client-enforced.

| Field | Location | Description |
|-------|----------|-------------|
| `expires_at` | `Quiz_Sessions` table | Absolute UTC timestamp when the session expires |
| Timer display | Client only | Client counts down from `expires_at - NOW()` |
| Submission gate | Server | `POST /api/quiz-sessions/:id/submit-answer` rejects if `NOW() > expires_at` |

When a student submits after `expires_at`, the server returns `410 Gone` with body `{ "error": "session_expired" }`. The client catches this and auto-submits whatever answers have been collected so far.

**Client timer implementation:**
```javascript
// Client polls server every 60s to keep server time in sync
// Prevents client clock drift from allowing extra time
const serverExpiry = new Date(session.expires_at);
const remaining = serverExpiry - Date.now();
startCountdown(remaining);
```

#### 2.3.3 Question Navigation Rules

- Questions are served **one at a time** — the client receives only the current question, never the full question set.
- Once an answer is submitted for a question, the response is locked. `PATCH /api/quiz-sessions/:id/answers/:question_id` returns `409 Conflict` if the answer already exists.
- The server tracks `current_question_index` in the `Quiz_Sessions` table. The client cannot skip ahead by incrementing an index locally.

#### 2.3.4 Scoring Logic

| Question Type | Scoring Rule |
|--------------|--------------|
| Multiple Choice | 1 point for exact match to `correct_answer` |
| True/False | 1 point for exact match ("True" or "False") |
| Short Answer | 0–2 points from AI grading (scaled to 0–1 for normalized scoring) |

**Final score calculation:**
```
raw_score = SUM(User_Responses.score) for session
max_score = COUNT(questions in session) * 1.0  [MCQ/TF weight 1, SA weight 1]
percentage = (raw_score / max_score) * 100
```

Score is calculated and written to `Scores` table only when `session.status` transitions to `'completed'`.

#### 2.3.5 Session Lifecycle States

```
created → active → completed
              ↓
           expired  (auto-transition on expires_at)
              ↓
           abandoned (student never returned within 24h of expiry)
```

A cron job (or scheduled function) runs every hour to transition `expired` sessions older than 24 hours to `abandoned` and auto-calculates their final score from whatever answers were submitted.

---

### 2.4 Feedback System

#### 2.4.1 Post-Quiz Feedback Generation

**Trigger:** Student clicks "View My Feedback" after completing a quiz.

**This is a non-blocking, on-demand operation** — feedback is not generated at submission time (to keep submission fast). It is generated lazily on first request and cached.

**Workflow:**

```
1. GET /api/quiz-sessions/:id/feedback
2. Check Feedback table for existing entry → if found, return cached result immediately
3. If not cached:
   a. Fetch all User_Responses for session where score < max
   b. For each incorrect/partial response, fetch: question_text, student_answer, correct_answer, explanation
   c. Batch all incorrect responses into a single LLM call (max 10 per call to respect token limits)
   d. Prompt (from /backend/prompts/feedback/user.template.txt):
      "The student just completed a quiz on {topic}. Below are questions they got wrong.
       For each, explain in 2–3 sentences WHY the student's answer was incorrect and what 
       the correct concept is. Be encouraging and educational, not condescending.
       
       Return ONLY a JSON array: [{ "question_id": int, "feedback_text": "string" }]
       
       Questions:
       {question_feedback_array}"
   e. Parse and validate response
   f. Store result in Feedback table with session_id reference
4. Return feedback array to client
```

#### 2.4.2 Feedback Display Rules

- Feedback is shown only for incorrect or partially correct answers.
- Correct answers show a brief "✓ Great job!" confirmation — no LLM call needed for these.
- The `explanation` field stored at question-generation time is shown alongside the AI feedback for context.
- Feedback UI shows: question text → student's answer → correct answer → AI explanation.

---

## 3. Technical Architecture & Data Schema

### 3.1 Recommended Tech Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Frontend** | React 18 + Vite | Fast dev server, component model ideal for quiz UI |
| **Styling** | Tailwind CSS v3 | Utility-first; avoids CSS bloat for a focused app |
| **State Management** | Zustand | Lightweight; avoids Redux boilerplate for this scope |
| **HTTP Client** | Axios | Interceptors simplify JWT refresh handling |
| **Backend** | Node.js + Express | JavaScript fullstack reduces context switching; large ecosystem |
| **ORM** | Prisma | Type-safe schema, auto-migrations, excellent DX |
| **Database** | MySQL 8.0 | Widely supported on shared hosting (Bluehost); relational model fits quiz data |
| **Authentication** | jsonwebtoken + bcrypt | Industry-standard; no external auth service dependency |
| **LLM Provider** | OpenAI GPT-4o (primary) / Anthropic Claude (fallback) | GPT-4o excels at structured JSON output; Claude as failover |
| **Deployment** | Bluehost / shared hosting via FTP | Per course project constraint |
| **Process Manager** | PM2 | Keeps Node.js backend alive on shared host |

> **Why MySQL over PostgreSQL?** Bluehost's shared hosting plans offer native MySQL via cPanel. While PostgreSQL is architecturally superior, MySQL is the pragmatic choice given the deployment constraint.

> **Why not Next.js?** Next.js SSR/SSG features add complexity that isn't needed for this SPA+API architecture. Vite + Express keeps the separation of concerns cleaner and is easier to deploy to shared hosting via FTP.

### 3.2 System Architecture Diagram (Textual)

```
┌─────────────────────────────────────────────────────────────┐
│                      CLIENT BROWSER                         │
│   React + Vite SPA                                          │
│   ┌─────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│   │  Auth   │  │  Quiz Engine │  │  Feedback Viewer     │  │
│   │ Context │  │  Component   │  │  Component           │  │
│   └────┬────┘  └──────┬───────┘  └──────────┬───────────┘  │
└────────┼──────────────┼─────────────────────┼──────────────┘
         │   HTTPS / REST API + JWT            │
         ▼              ▼                      ▼
┌─────────────────────────────────────────────────────────────┐
│                  EXPRESS.JS BACKEND (Node.js)               │
│                                                             │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────┐    │
│  │ Auth       │  │ Quiz       │  │ Feedback           │    │
│  │ Controller │  │ Controller │  │ Controller         │    │
│  └────────────┘  └─────┬──────┘  └────────┬───────────┘    │
│                        │                  │                 │
│              ┌─────────▼──────────────────▼───────┐        │
│              │         AI Service Layer            │        │
│              │  /backend/services/AIService.js     │        │
│              │  Loads prompts from /prompts/       │        │
│              └─────────────────┬───────────────────┘        │
│                                │                            │
│              ┌─────────────────▼───────────────────┐        │
│              │         Prisma ORM                  │        │
│              └─────────────────┬───────────────────┘        │
└────────────────────────────────┼────────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │     MySQL Database       │
                    │   (Bluehost cPanel)      │
                    └─────────────────────────┘
                                 ▲
                    ┌────────────┴────────────┐
                    │  OpenAI / Anthropic API  │
                    │  (External HTTPS calls)  │
                    └─────────────────────────┘
```

### 3.3 Database Entity Relationship Map

#### Table: `Users`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | INT | PK, AUTO_INCREMENT | Unique user identifier |
| `email` | VARCHAR(255) | UNIQUE, NOT NULL | Login credential |
| `password_hash` | VARCHAR(255) | NOT NULL | bcrypt hash |
| `display_name` | VARCHAR(100) | NOT NULL | Shown in UI |
| `role` | ENUM('student','instructor') | NOT NULL, DEFAULT 'student' | Access control |
| `created_at` | DATETIME | DEFAULT NOW() | Registration timestamp |
| `updated_at` | DATETIME | ON UPDATE NOW() | Last profile update |

#### Table: `Quiz_Configs`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | INT | PK, AUTO_INCREMENT | Quiz config identifier |
| `instructor_id` | INT | FK → Users.id, NOT NULL | Who created it |
| `title` | VARCHAR(255) | NOT NULL | Display name of quiz |
| `topic` | VARCHAR(255) | NOT NULL | Injected into LLM prompt |
| `difficulty` | ENUM('easy','medium','hard') | NOT NULL | Injected into LLM prompt |
| `question_count` | INT | NOT NULL, DEFAULT 10 | How many questions to generate |
| `allowed_types` | JSON | NOT NULL | Array: ["mcq","true_false","short_answer"] |
| `time_limit_minutes` | INT | NULLABLE | NULL = no time limit |
| `is_published` | TINYINT(1) | DEFAULT 0 | 0 = draft, 1 = visible to students |
| `is_archived` | TINYINT(1) | DEFAULT 0 | Soft delete flag |
| `created_at` | DATETIME | DEFAULT NOW() | |
| `updated_at` | DATETIME | ON UPDATE NOW() | |

#### Table: `Quiz_Sessions`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | INT | PK, AUTO_INCREMENT | Session identifier |
| `student_id` | INT | FK → Users.id, NOT NULL | Who is taking the quiz |
| `quiz_config_id` | INT | FK → Quiz_Configs.id, NOT NULL | Which quiz |
| `status` | ENUM('active','completed','expired','abandoned') | DEFAULT 'active' | Session lifecycle |
| `current_question_index` | INT | DEFAULT 0 | Server-side progress tracker |
| `started_at` | DATETIME | DEFAULT NOW() | |
| `expires_at` | DATETIME | NULLABLE | NULL if no time limit |
| `completed_at` | DATETIME | NULLABLE | Set on completion |

**Index:** `UNIQUE(student_id, quiz_config_id)` — one active session per student per quiz.

#### Table: `Questions`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | INT | PK, AUTO_INCREMENT | Question identifier |
| `session_id` | INT | FK → Quiz_Sessions.id, NOT NULL | Belongs to this session |
| `question_index` | INT | NOT NULL | Order within session (0-based) |
| `question_type` | ENUM('mcq','true_false','short_answer') | NOT NULL | |
| `question_text` | TEXT | NOT NULL | The question |
| `options` | JSON | NULLABLE | For MCQ: ["A","B","C","D"] |
| `correct_answer` | VARCHAR(500) | NOT NULL | For MCQ/TF: exact string; for SA: sample answer |
| `explanation` | TEXT | NULLABLE | LLM-generated explanation (stored at generation time) |
| `grading_rubric` | TEXT | NULLABLE | For short_answer only |
| `created_at` | DATETIME | DEFAULT NOW() | |

#### Table: `User_Responses`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | INT | PK, AUTO_INCREMENT | |
| `session_id` | INT | FK → Quiz_Sessions.id, NOT NULL | |
| `question_id` | INT | FK → Questions.id, NOT NULL | |
| `student_answer` | TEXT | NOT NULL | What the student submitted |
| `is_correct` | TINYINT(1) | NULLABLE | 1/0 for MCQ/TF; NULL until graded for SA |
| `ai_score` | DECIMAL(3,2) | NULLABLE | 0.00–1.00 normalized score for SA |
| `ai_rationale` | TEXT | NULLABLE | AI grading rationale for SA |
| `submitted_at` | DATETIME | DEFAULT NOW() | |

**Constraint:** `UNIQUE(session_id, question_id)` — one answer per question per session.

#### Table: `Scores`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | INT | PK, AUTO_INCREMENT | |
| `session_id` | INT | FK → Quiz_Sessions.id, UNIQUE, NOT NULL | One score per session |
| `student_id` | INT | FK → Users.id, NOT NULL | Denormalized for query performance |
| `quiz_config_id` | INT | FK → Quiz_Configs.id, NOT NULL | Denormalized |
| `raw_score` | DECIMAL(6,2) | NOT NULL | Sum of points earned |
| `max_score` | DECIMAL(6,2) | NOT NULL | Maximum possible points |
| `percentage` | DECIMAL(5,2) | NOT NULL | (raw/max)*100 |
| `calculated_at` | DATETIME | DEFAULT NOW() | |

#### Table: `Feedback`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | INT | PK, AUTO_INCREMENT | |
| `session_id` | INT | FK → Quiz_Sessions.id, UNIQUE, NOT NULL | One feedback record per session |
| `feedback_json` | JSON | NOT NULL | Array of `{ question_id, feedback_text }` |
| `generated_at` | DATETIME | DEFAULT NOW() | |

#### Entity Relationships Summary

```
Users ──< Quiz_Configs          (one instructor → many configs)
Users ──< Quiz_Sessions         (one student → many sessions)
Quiz_Configs ──< Quiz_Sessions  (one config → many sessions)
Quiz_Sessions ──< Questions     (one session → many questions, generated per-session)
Quiz_Sessions ──< User_Responses
Questions ──< User_Responses    (one question → one response per session)
Quiz_Sessions ──1 Scores        (one session → exactly one score record)
Quiz_Sessions ──1 Feedback      (one session → exactly one feedback record)
```

---

## 4. AI-Friendly Repository Blueprint

This section documents the canonical folder structure. AI coding agents (Copilot, Cursor, Claude Code) must be able to locate any file within 2 hops from the root. All path references in code must use the constants defined in `/backend/config/paths.js` — never hardcoded strings.

### 4.1 Folder Directory Tree

```
quiz-master/
│
├── README.md                        ← Project overview, setup instructions
├── .gitignore                        ← See §4.3
├── .env.example                      ← Template with all required env vars (no real values)
│
├── /docs/                            ← All project documentation
│   ├── QUIZ_MASTER_SRS_FP2.md        ← This document (FP2 milestone)
│   ├── API_REFERENCE.md              ← All endpoint definitions
│   └── DEPLOYMENT.md                 ← Step-by-step Bluehost/FTP deploy guide
│
├── /frontend/                        ← React + Vite SPA
│   ├── index.html
│   ├── vite.config.js
│   ├── tailwind.config.js
│   ├── package.json
│   ├── .env.example
│   │
│   └── /src/
│       ├── main.jsx                  ← App entry point
│       ├── App.jsx                   ← Router root
│       │
│       ├── /components/              ← Reusable UI atoms/molecules
│       │   ├── Button.jsx
│       │   ├── Timer.jsx
│       │   ├── ProgressBar.jsx
│       │   └── QuestionCard.jsx
│       │
│       ├── /pages/                   ← One file per route
│       │   ├── LoginPage.jsx
│       │   ├── RegisterPage.jsx
│       │   ├── StudentDashboard.jsx
│       │   ├── QuizPage.jsx          ← Active quiz UI
│       │   ├── ResultsPage.jsx
│       │   ├── FeedbackPage.jsx
│       │   └── InstructorDashboard.jsx
│       │
│       ├── /hooks/                   ← Custom React hooks
│       │   ├── useAuth.js
│       │   ├── useQuizSession.js
│       │   └── useTimer.js
│       │
│       ├── /store/                   ← Zustand state stores
│       │   ├── authStore.js
│       │   └── quizStore.js
│       │
│       └── /api/                     ← Axios instance + API call functions
│           ├── axiosInstance.js      ← Configured with interceptors for JWT refresh
│           ├── authApi.js
│           ├── quizApi.js
│           └── feedbackApi.js
│
└── /backend/                         ← Express.js Node.js API
    ├── server.js                     ← Entry point; app + server setup
    ├── package.json
    ├── .env.example
    │
    ├── /config/
    │   ├── database.js               ← Prisma client singleton
    │   ├── paths.js                  ← Canonical path constants (PROMPTS_DIR, etc.)
    │   └── constants.js              ← App-wide constants (TOKEN_TTL, MAX_RETRIES, etc.)
    │
    ├── /prompts/                     ← *** AI PROMPT DIRECTORY — fully decoupled ***
    │   │                              ← Prompts are .txt files, NOT JS strings
    │   │
    │   ├── /quiz-generation/
    │   │   ├── system.txt            ← System prompt for question generation
    │   │   ├── user.template.txt     ← User prompt template with {variables}
    │   │   └── schema.json           ← The expected JSON schema for validation
    │   │
    │   ├── /grading/
    │   │   ├── system.txt            ← System prompt for short-answer grading
    │   │   ├── user.template.txt     ← Grading prompt template
    │   │   └── schema.json           ← Expected grading response schema
    │   │
    │   └── /feedback/
    │       ├── system.txt            ← System prompt for feedback generation
    │       ├── user.template.txt     ← Feedback prompt template
    │       └── schema.json           ← Expected feedback response schema
    │
    ├── /routes/                      ← Express router definitions
    │   ├── auth.routes.js
    │   ├── quiz.routes.js
    │   ├── session.routes.js
    │   └── feedback.routes.js
    │
    ├── /controllers/                 ← Request handlers (thin; delegate to services)
    │   ├── auth.controller.js
    │   ├── quiz.controller.js
    │   ├── session.controller.js
    │   └── feedback.controller.js
    │
    ├── /services/                    ← Business logic (thick; no req/res objects here)
    │   ├── AIService.js              ← ONLY file that calls LLM API
    │   ├── QuizService.js
    │   ├── SessionService.js
    │   ├── ScoringService.js
    │   └── FeedbackService.js
    │
    ├── /middleware/
    │   ├── authenticateToken.js      ← JWT validation middleware
    │   ├── requireRole.js            ← Role-based guard middleware
    │   └── errorHandler.js           ← Global Express error handler
    │
    ├── /utils/
    │   ├── promptLoader.js           ← Reads .txt prompts from /prompts/ directory
    │   ├── templateInterpolator.js   ← Replaces {variables} in prompt templates
    │   └── logger.js                 ← Centralized logging (never log secrets)
    │
    └── /prisma/
        ├── schema.prisma             ← Database schema definition
        └── /migrations/              ← Auto-generated migration history
```

### 4.2 Prompt File Conventions

All prompts live in `/backend/prompts/` and follow these rules so AI agents can locate and update them without touching business logic:

**Rule 1 — File separation by purpose:**
Each LLM interaction has its own subdirectory with exactly three files: `system.txt`, `user.template.txt`, and `schema.json`.

**Rule 2 — Template variable syntax:**
All dynamic values in prompt templates use `{snake_case_variable}` syntax. The `templateInterpolator.js` utility handles substitution. Never use ES6 template literal syntax (`${var}`) in `.txt` files.

**Rule 3 — Schema files are the source of truth:**
The `schema.json` files define the expected structure of the LLM response. The parsing/validation code in `AIService.js` references these schema files at runtime — they are not duplicated in code.

**Rule 4 — Prompts are version-controlled:**
Prompt files are committed to git just like code. Changes to prompts must go through pull requests. This creates an audit trail of prompt evolution.

**Rule 5 — Loading convention:**
```javascript
// /backend/utils/promptLoader.js
const { PROMPTS_DIR } = require('../config/paths');
const path = require('path');
const fs = require('fs');

function loadPrompt(category, file) {
  const filePath = path.join(PROMPTS_DIR, category, file);
  return fs.readFileSync(filePath, 'utf-8');
}

module.exports = { loadPrompt };
```

### 4.3 .gitignore Layout

```gitignore
# ─────────────────────────────────────────
# ENVIRONMENT & SECRETS — NEVER COMMIT
# ─────────────────────────────────────────
.env
.env.local
.env.production
.env.staging
*.pem
*.key

# ─────────────────────────────────────────
# NODE DEPENDENCIES
# ─────────────────────────────────────────
node_modules/
frontend/node_modules/
backend/node_modules/

# ─────────────────────────────────────────
# BUILD ARTIFACTS
# ─────────────────────────────────────────
frontend/dist/
frontend/build/
backend/dist/

# ─────────────────────────────────────────
# DATABASE
# ─────────────────────────────────────────
*.db
*.sqlite
*.sqlite3
backend/prisma/*.db

# ─────────────────────────────────────────
# LOGS
# ─────────────────────────────────────────
*.log
logs/
npm-debug.log*

# ─────────────────────────────────────────
# OS / EDITOR ARTIFACTS
# ─────────────────────────────────────────
.DS_Store
Thumbs.db
.vscode/settings.json
.idea/

# ─────────────────────────────────────────
# PM2 RUNTIME
# ─────────────────────────────────────────
.pm2/

# ─────────────────────────────────────────
# WHAT TO COMMIT (explicit reminder)
# ─────────────────────────────────────────
# ✅ .env.example  (template only, no real values)
# ✅ backend/prompts/**  (prompt files are code)
# ✅ backend/prisma/schema.prisma
# ✅ backend/prisma/migrations/
# ✅ All source code in /frontend/src and /backend
```

**Required `.env` variables** (document in `.env.example`):

```env
# ── Backend ──────────────────────────────
NODE_ENV=development
PORT=3001
DATABASE_URL=mysql://user:password@localhost:3306/quiz_master

JWT_SECRET=your_jwt_secret_minimum_32_chars
JWT_REFRESH_SECRET=your_refresh_secret_minimum_32_chars
JWT_ACCESS_TTL=3600
JWT_REFRESH_TTL=604800

OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o
ANTHROPIC_API_KEY=sk-ant-...       # Fallback provider
AI_TIMEOUT_MS=15000
AI_MAX_RETRIES=1

# ── Frontend ─────────────────────────────
VITE_API_BASE_URL=http://localhost:3001/api
```

---

## 5. Edge Cases & Resilience Strategy

*This section is written from the perspective of a skeptical QA engineer stress-testing the system.*

### 5.1 LLM API Timeout Mid-Quiz Generation

**Scenario:** A student clicks "Start Quiz." The backend calls the LLM. The LLM hangs for 20+ seconds.

| Layer | Mitigation |
|-------|-----------|
| `AIService` | Hard timeout of `AI_TIMEOUT_MS` (15s) using `AbortController` / `axios.timeout` |
| Controller | On `TimeoutError`: retry once with a simplified prompt (fewer questions, lower specificity) |
| Controller | On second failure: return `503` with `{ error: "quiz_unavailable", retry_after: 30 }` |
| Frontend | Display: *"Our quiz generator is warming up. Please try again in 30 seconds."* |
| Frontend | Disable "Start Quiz" button for `retry_after` seconds to prevent spam |
| **No session is created** | Session row is only inserted to DB after questions are successfully generated |

### 5.2 Malformed AI JSON Output

**Scenario:** The LLM returns valid text but it is not valid JSON (e.g., includes markdown prose, truncated mid-array).

| Layer | Mitigation |
|-------|-----------|
| `AIService.parseAndValidateLLMResponse()` | Strip markdown fences before parsing (see §2.2.4) |
| Parser | `try/catch` around `JSON.parse()` — never crash on parse failure |
| Parser | Partial validation: skip malformed question objects, keep valid ones |
| `AIService` | If valid question count < 70% of requested count, classify as failure and retry |
| Retry prompt | Simplified prompt adds: *"CRITICAL: Return ONLY raw JSON. No markdown. No explanation."* |
| Fallback | If retry also malformed: return `503`; log the raw response to server logs for debugging |

### 5.3 Student Disconnects During Timed Exam

**Scenario:** A student loses internet connection with 5 minutes remaining, closes their laptop, and returns 2 hours later.

| Condition | Behavior |
|-----------|----------|
| Returns before `expires_at` | Session is still active; client calls `GET /api/quiz-sessions/:id` to resume from `current_question_index` |
| Returns after `expires_at` | Session has `status = 'expired'`; all submitted answers are scored; remaining questions marked unanswered (score 0) |
| Cron job processes session | Calculates final score from submitted answers only; writes to `Scores` table |
| Student views results page | Shows score based on answers submitted before expiry; includes note: *"Session expired — score reflects answers submitted before time ran out"* |

### 5.4 Concurrent Session Attempt (Double-Start Attack)

**Scenario:** A student opens two browser tabs and clicks "Start Quiz" in both simultaneously.

| Layer | Mitigation |
|-------|-----------|
| Database | `UNIQUE(student_id, quiz_config_id)` constraint on `Quiz_Sessions` rejects duplicate insert |
| Backend | Race condition: second request gets `409 Conflict`; controller catches unique constraint error and returns the existing session instead |
| Frontend | Both tabs end up with the same `session_id` — harmless, both show the same quiz state |

### 5.5 Instructor Deletes a Quiz While Students Are Mid-Session

**Scenario:** An instructor archives a quiz while 5 students are actively taking it.

| Layer | Mitigation |
|-------|-----------|
| `DELETE /api/quizzes/:id` | Sets `is_archived = 1` only (soft delete); does **not** cascade to active sessions |
| Active sessions | Continue to completion unaffected; questions already generated and stored in `Questions` table |
| Historical data | `Scores` and `User_Responses` are retained even for archived quizzes |
| Student completing after archive | Session completes normally; score is calculated and stored |
| Instructor dashboard | Archived quizzes can be shown with a filter toggle; their historical data remains accessible |

### 5.6 Short-Answer Grading AI Returns Invalid Score

**Scenario:** The grading LLM returns `{ "score": "maybe", "rationale": "..." }` — a non-numeric score.

| Layer | Mitigation |
|-------|-----------|
| `AIService` grading parser | Validates `score` field is a number in range [0, 2]; if not, defaults to `1` (partial credit) |
| Rationale | Stores: *"[Auto-grading encountered an issue. Score defaulted to partial credit.]"* + AI's rationale |
| Instructor override | Future feature: instructors can manually override AI scores for short answers |
| Logging | Log the raw invalid response for prompt tuning |

### 5.7 Student Submits Empty Answer

**Scenario:** Student clicks "Submit" without typing anything for a short-answer question.

| Layer | Mitigation |
|-------|-----------|
| Frontend | Disable "Submit" button if `answer_text.trim().length === 0`; show inline validation: *"Please enter an answer before submitting."* |
| Backend | Additional server-side check: if `answer_text.trim() === ""`, return `400 Bad Request` — never pass empty string to LLM grader |

### 5.8 LLM API Key Expired or Rate Limited

**Scenario:** The OpenAI API key hits its rate limit or is invalidated.

| Layer | Mitigation |
|-------|-----------|
| `AIService` | Check for `429 Too Many Requests` or `401 Unauthorized` from OpenAI |
| Failover | Switch to Anthropic Claude endpoint (configured via `ANTHROPIC_API_KEY`) |
| If both fail | Return `503` with `{ "error": "ai_unavailable" }` to client |
| Alert | Log a `CRITICAL` level alert; in production, trigger email/Slack alert to team |
| Rate limit tracking | Implement token-bucket counter to proactively throttle before hitting the API limit |

### 5.9 Database Connection Loss

**Scenario:** The MySQL database goes down mid-request (Bluehost maintenance, etc.).

| Layer | Mitigation |
|-------|-----------|
| Prisma | Connection pool with auto-reconnect; first request after reconnect will re-establish |
| Global error handler | `errorHandler.js` middleware catches `PrismaClientKnownRequestError` and returns `503` |
| Health check endpoint | `GET /api/health` returns `{ "db": "ok/error", "ai": "ok/error" }` for monitoring |
| Student impact | If quiz answer cannot be saved, return error to client; client prompts: *"Connection issue — please try submitting again"* |

---

## 6. Non-Functional Requirements

| ID | Category | Requirement |
|----|----------|-------------|
| NFR-01 | Performance | API responses (excluding LLM calls) must complete in < 500ms at p95 |
| NFR-02 | Performance | LLM calls have a hard timeout of 15 seconds; failures surface within 16 seconds |
| NFR-03 | Security | All passwords hashed with bcrypt (12+ rounds); no plaintext credentials in logs |
| NFR-04 | Security | JWTs stored only in memory or httpOnly cookies; never in localStorage |
| NFR-05 | Security | All API endpoints except auth use HTTPS and require valid JWT |
| NFR-06 | Security | LLM API keys stored only in server environment variables; never exposed to client |
| NFR-07 | Reliability | Application must handle LLM provider failure gracefully without crashing |
| NFR-08 | Maintainability | Prompts are decoupled from code (separate `.txt` files); engineers can update prompts without touching JS |
| NFR-09 | Deployability | Application must deploy to Bluehost shared hosting via FTP; PM2 manages the Node process |
| NFR-10 | Scalability | Database queries on `Quiz_Sessions`, `Scores` must use indexed `student_id` and `quiz_config_id` |

---

## 7. Glossary

| Term | Definition |
|------|-----------|
| **LLM** | Large Language Model (e.g., OpenAI GPT-4o, Anthropic Claude) |
| **JWT** | JSON Web Token — stateless authentication credential |
| **AIService** | The backend module that is the sole interface to the LLM API |
| **Quiz Config** | Instructor-defined settings for a quiz (topic, difficulty, etc.) — not a quiz instance |
| **Quiz Session** | A specific student's active attempt at a quiz config |
| **Prompt Template** | A `.txt` file with `{variable}` placeholders, interpolated at runtime |
| **Soft Delete** | Marking a record as archived rather than deleting it, preserving historical data |
| **httpOnly Cookie** | A browser cookie inaccessible to JavaScript — used to store refresh tokens securely |
| **Prisma** | Node.js ORM used for type-safe database access and schema migrations |
| **AI Parse Error** | Custom error class thrown when LLM response cannot be validated against expected schema |

---

*End of FP2 Software Requirements Specification — Quiz Master*
*Push this file to the repository root as: `docs/QUIZ_MASTER_SRS_FP2.md`*
