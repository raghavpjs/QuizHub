# Quiz Website Backend

A REST API backend for a Quiz/Exam platform with two roles — **student** and
**lecturer** — built on top of a shared, reusable question bank. No frontend
is included.

## Overview

- **Lecturers** create MCQ questions and quizzes. Questions can be reused from
  a shared question bank, or created on the fly while building a quiz (new
  ones are automatically added to the bank).
- **Students** browse quizzes, take a quiz (without seeing correct answers),
  submit their answers for instant grading, and review or delete their past
  results.

## Tech stack

- Node.js + Express (REST API, ES modules)
- MongoDB + Mongoose
- JWT (`jsonwebtoken`) for authentication, `bcryptjs` for password hashing
- `joi` for request validation
- `helmet` + `cors` for baseline security, `morgan` for request logging

## Architecture

```text
Controller -> Service -> Repository -> Model -> MongoDB
```

- **Controllers** parse the request and shape the response.
- **Services** hold business logic (e.g. "add new questions to the bank while
  creating a quiz", "grade a submission").
- **Repositories** are the only layer that talks to Mongoose models.
- **Models** define the Mongoose schemas.

```text
src/
├── config/          # env loading, DB connection
├── controllers/      # auth, question, quiz (also handles results)
├── services/          # business logic
├── repositories/       # data access (Mongoose)
├── models/              # Mongoose schemas: User, Question, Quiz, Result
├── routes/                # Express routers
├── middleware/              # auth, role, validation, error handling
├── validators/                # Joi schemas
├── utils/                       # AppError, catchAsync, JWT helpers
├── app.js                        # Express app (no listen/connect — testable)
└── server.js                      # connects to DB, then starts the server
```

## Requirements

- Node.js v18+
- npm
- MongoDB (local) or a MongoDB Atlas cluster

## Installation

```bash
npm install
```

## Environment variables

Create a `.env` file in the project root:

```env
PORT=5000
MONGODB_URI=mongodb://127.0.0.1:27017/quiz_platform
JWT_SECRET=change_this_secret
JWT_EXPIRES_IN=1d
```

`PORT` and `JWT_EXPIRES_IN` are optional (default to `5000` and `1d`).
`MONGODB_URI` and `JWT_SECRET` are required — the app exits on startup if
either is missing. `.env` is git-ignored and must never be committed.

## Running the server

```bash
npm run dev     # nodemon, auto-restarts on changes
npm start       # plain node
```

The server listens on `http://localhost:5000` (or your configured `PORT`).
Health check: `GET /health` → `{ "data": { "status": "ok" } }`.

## Authentication

1. Sign up or log in to receive a JWT (`POST /api/auth/signup` or `/login`).
2. Send it on every protected request:

   ```http
   Authorization: Bearer <token>
   ```

The token payload contains `userId` and `role`. There is no email
verification, password reset, or refresh token — signup/login only.

## Response conventions

- Most success responses use the envelope `{ "data": ... }`.
- A few endpoints (question update/delete, result delete) instead return
  `{ "status": "success", ... }` — noted per endpoint below.
- Errors: `{ "message": "..." }`.

## API Endpoints

Base URL: `/api`

### Auth — `/api/auth`

#### `POST /api/auth/signup`
- Auth: none
- Body: `{ "email": string, "password": string (min 6 chars), "role": "student" | "lecturer" }`
- Success `201`: `{ "data": { "user": { "id", "email", "role" }, "token": "..." } }`
- Errors: `400` invalid input, `409` email already registered

#### `POST /api/auth/login`
- Auth: none
- Body: `{ "email": string, "password": string }`
- Success `200`: `{ "data": { "user": { "id", "email", "role" }, "token": "..." } }`
- Errors: `400` invalid input, `401` invalid email or password

### Questions — `/api/questions`

All routes require authentication. Create/update/delete/list are
**lecturer-only**.

A question can be sent using either naming convention — `statement`/`choices`/
`correctAnswer` or `question`/`options`/`answer` — the API accepts both.

#### `GET /api/questions`
- Auth: required, role: `lecturer`
- Query (optional): `?tag=java`, `?difficulty=Easy`
- Success `200`: `{ "data": [ { "_id", "statement", "choices", "correctAnswer", "tags", "difficulty", "createdBy", "createdAt", "updatedAt" }, ... ] }`
- Errors: `401` unauthenticated, `403` wrong role

#### `POST /api/questions`
- Auth: required, role: `lecturer`
- Body:
  ```json
  {
    "statement": "What does JVM stand for?",
    "choices": ["Java Virtual Machine", "Java Variable Manager", "Java Visual Machine", "None"],
    "correctAnswer": "Java Virtual Machine",
    "tags": ["java", "jvm"],
    "difficulty": "Easy"
  }
  ```
- `createdBy` is set automatically from the authenticated user, not the body.
- `difficulty` defaults to `"Easy"` if omitted.
- Success `201`: the created question document (not wrapped in `data`)
- Errors: `400` invalid input (e.g. fewer than 2 choices, `correctAnswer` not among choices), `401` unauthenticated, `403` wrong role

#### `PUT /api/questions/:id`
- Auth: required, role: `lecturer`
- Body: any subset of `statement`/`question`, `choices`/`options`, `correctAnswer`/`answer`, `tags`, `difficulty` — all optional; only provided fields are updated
- If both a choices list and a correct answer are present in the request, `correctAnswer` must be one of the choices
- Success `200`: `{ "status": "success", "data": <updated question> }`
- Errors: `400` invalid input, `401` unauthenticated, `403` wrong role, `404` question not found

#### `DELETE /api/questions/:id`
- Auth: required, role: `lecturer`
- Success `200`: `{ "status": "success", "message": "Question deleted successfully" }`
- Errors: `401` unauthenticated, `403` wrong role, `404` question not found

### Quizzes — `/api/quizzes`

#### `POST /api/quizzes`
- Auth: required, role: `lecturer`
- Body:
  ```json
  {
    "title": "Java Basics",
    "description": "Basic Java quiz",
    "questions": [
      { "questionId": "665f1c2b8a1e2a0012345678" },
      {
        "statement": "What is JVM?",
        "choices": ["Java Virtual Machine", "Java Variable Manager", "Java Visual Machine", "None"],
        "correctAnswer": "Java Virtual Machine",
        "tags": ["java", "jvm"],
        "difficulty": "Easy"
      }
    ]
  }
  ```
- Behavior: each entry in `questions` is either `{ "questionId": "<existing id>" }` to reuse a question from the bank, or a full question payload to create a new one. New questions are created under the shared bank first (attributed to the lecturer, `difficulty` defaults to `"Medium"` if omitted), then referenced by the quiz. At least one question is required.
- Success `201`: `{ "data": { "_id", "title", "description", "questions": [...ids], "createdBy", "createdAt", "updatedAt" } }`
- Errors: `400` invalid input, `401` unauthenticated, `403` wrong role, `404` a referenced `questionId` does not exist

#### `GET /api/quizzes`
- Auth: required (student or lecturer)
- Success `200`:
  ```json
  { "data": [ { "id": "...", "title": "Java Basics", "description": "...", "questionCount": 5, "createdBy": "lecturer@example.com" } ] }
  ```
- Errors: `401` unauthenticated

#### `GET /api/quizzes/:quizId`
- Auth: required, role: `student`
- Success `200`: `{ "data": { "id", "title", "description", "questions": [ { "id", "statement", "choices" } ] } }` — **`correctAnswer` is stripped**
- Errors: `400` invalid `quizId` format, `401` unauthenticated, `403` wrong role, `404` quiz not found

#### `POST /api/quizzes/:quizId/submit`
- Auth: required, role: `student`
- Body:
  ```json
  { "answers": [ { "questionId": "665f1c2b8a1e2a0012345678", "answer": "Java Virtual Machine" } ] }
  ```
- Behavior: grades each answer (case-insensitive, whitespace-trimmed comparison) and persists a `Result` document.
- Success `200`:
  ```json
  {
    "data": {
      "studentId": "...", "quizId": "...", "quizTitle": "Java Basics",
      "score": 70, "correct": 7, "wrong": 3, "totalQuestions": 10,
      "date": "Sep 11, 2026",
      "details": [ { "question": "...", "userAnswer": "...", "correctAnswer": "...", "isCorrect": true } ]
    }
  }
  ```
  Note: `score` in the saved/returned result is the **percentage**; the raw
  correct-answer count is in `correct`.
- Errors: `400` invalid input / malformed `questionId`, `401` unauthenticated, `403` wrong role, `404` quiz not found

### Results — `/api/results`

All routes require authentication (no role restriction is enforced beyond
that; a result can only be deleted by the student who owns it).

#### `GET /api/results`
- Auth: required
- Success `200`: `{ "data": [ <Result documents for the authenticated user, as returned by the submit endpoint>, ... ] }`
- Errors: `401` unauthenticated

#### `DELETE /api/results/:id`
- Auth: required
- Success `200`: `{ "status": "success", "message": "Result deleted successfully" }`
- Errors: `401` unauthenticated, `404` result not found or not owned by the caller

### Common error codes

```text
400 -> invalid request (validation failure, bad ObjectId)
401 -> unauthenticated (missing/invalid/expired token)
403 -> authenticated but wrong role
404 -> resource not found
409 -> duplicate email on signup
500 -> unexpected server error
```

## Example workflow

```text
1. Lecturer signs up               -> POST /api/auth/signup
2. Lecturer logs in                -> POST /api/auth/login
3. Lecturer browses question bank  -> GET  /api/questions?tag=java
4. Lecturer creates a quiz         -> POST /api/quizzes
   (any brand-new questions in the payload are added to the question bank)
5. Student signs up                -> POST /api/auth/signup
6. Student logs in                 -> POST /api/auth/login
7. Student lists quizzes           -> GET  /api/quizzes
8. Student fetches a quiz          -> GET  /api/quizzes/:quizId  (no answers included)
9. Student submits answers         -> POST /api/quizzes/:quizId/submit
10. Backend grades it and returns score / correct / wrong / percentage
11. Student reviews past results   -> GET    /api/results
12. Student deletes a result       -> DELETE /api/results/:id
```

## Notes on the codebase

- `app.js` builds the Express app without starting a server or connecting to
  MongoDB, so it can be imported directly in tests; `server.js` does the
  actual `connectDB()` + `listen()`.
- The question and quiz creation paths accept both `statement/choices/
  correctAnswer` and `question/options/answer` naming — this flexibility
  exists to tolerate slightly different frontend payload shapes.
- Passwords are hashed with bcrypt and the `password` field is excluded from
  query results by default (`select: false` on the User model).
