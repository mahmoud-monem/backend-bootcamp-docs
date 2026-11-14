# Project 1: Online Exam Platform — Requirements & System Specification

> **Bootcamp:** Node.js Backend Engineering Bootcamp (Cohort C4)
> **Project #:** 1 (First Team Project)
> **Team Size:** 5 Backend Engineers per Team
> **Timeline & Methodology:** 1 Week (7 Days) · Waterfall Delivery Model
> **Primary Goals:** API & Domain Extraction from Figma · Clean Modular Architecture · Concurrency, Streams & Worker Threads in Production · Comprehensive Jest Unit Testing
> **Tech Stack:** Node.js (LTS) · Express.js · TypeScript (Strict Mode) · MongoDB & Mongoose · Zod · Jest
> **Figma Design:** [Online Exam Elevate (Figma)](https://www.figma.com/design/Feva7gSQLYtQEcMPawPp1z/Exam-Online-Elevate--Copy-?t=ZGeY0sAxcDicWQPm-0)

---

## 0. How to Use This Document

You have **three sources of truth**, and your team is expected to cross-read all three before writing controller code:

| Source | What It Gives You |
|---|---|
| **This specification** | Complete domain scope, `Student` & `Admin` roles, numbered Business Rules (`BR-01`–`BR-22`), edge cases, runtime/concurrency requirements, canonical `/api/v1` endpoint contracts, and the 5-person 7-day Waterfall execution plan. |
| **[Figma Design](https://www.figma.com/design/Feva7gSQLYtQEcMPawPp1z/Exam-Online-Elevate--Copy-?t=ZGeY0sAxcDicWQPm-0)** | Exact UI flows, field names, empty/error states, two-level `Diploma → Quiz → Question` hierarchy, `Quiz History`, `Answer Quiz` review screen, and `Admin` modals (`Add Diploma`, `Add Quiz`, `Add Question`). |
| **Session 1 & Session 2 Labs** | Production Node.js runtime engineering: Event-Loop Lag Monitoring (`perf_hooks`), async `libuv` cryptography, Graceful Shutdown, Backpressured Streams (`pipeline` + `AbortController`), and self-healing `worker_threads`. |

**What this document intentionally expects your team to design cleanly:** exact Mongoose schema indexes, DTO mappings, shared TypeScript types, and clean unit test suites adhering to the standards in [`docs/standards/`](../standards/README.md).

---

## 1. Actors, Role-Based Access Control & Scope Boundaries

Inspection of the Figma canvas reveals **two distinct personas** (`Student` and `Admin`), each with dedicated screens and permissions. Every protected route must enforce both **authentication (valid JWT)** and **authorization (`role` + resource ownership)**.

| Actor | Role Value | Core Capabilities (From Figma) | Access Boundaries |
|---|---|---|---|
| **Student** | `"student"` | Sign up, sign in, recover password via 3-step OTP flow, view Student Dashboard KPIs, browse & search Diplomas/Quizzes, read Quiz Instructions, start & submit Timed Quiz Attempts, view Instant Score modal, inspect `Quiz History`, and open `Answer Quiz` detailed review. | **Never** sees `correctOptionIds` before attempt submission. **Cannot** view or submit another student's attempt (`404 Not Found` on IDOR). **Cannot** mutate Diplomas, Quizzes, or Questions. |
| **Admin** | `"admin"` | Sign in to Admin Dashboard, manage catalog via `Add Diploma`, `Add Quiz`, and `Add Question` (`radio button` / `check box`), update/publish quizzes, bulk-import Questions via NDJSON stream, export Attempt analytics/reports via stream, and run batch analytics. | Strictly gated behind `requireRole("admin")`. Every `/api/v1/admin/*` route MUST reject unauthenticated (`401`) and non-admin (`403`) requests. |

### Explicit 1-Week Project Scope Boundaries

To ensure a **team of 5 engineers finishes cleanly within 7 days**, the following items visible on the Figma mockups are **explicitly out of scope (UI-only or optional stretch)**:

1. **Social OAuth Buttons** (`Continue with Google`, `Twitter`, `Facebook`, `Apple` on Sign In / Sign Up): Leave as optional stretch; implement full Email + Password auth and OTP recovery first.
2. **UI Language Switcher** (`English` dropdown in auth header): Backend error messages and data only need English.
3. **File Upload Binary Storage (S3/Cloudinary):** `Add Photo` on `Add Diploma` and `Add Quiz` should store and validate a clean image URL string (`imageUrl`, validated with Zod `.url()`), rather than building multipart S3 upload infrastructure in Week 1.
4. **Live SMTP Provider:** For the 3-step OTP recovery flow, abstract an `EmailSender` interface (e.g., logging the OTP cleanly in dev/test mode or supporting optional Nodemailer SMTP env config) so tests and local evaluations work without third-party API keys.

---


## 2. Figma Screen Inventory & Functional Requirements by Module


### Module 1: Authentication & 3-Step Password Recovery (`src/modules/auth/`)


Figma frames covered: **Sign up**, **Sign in**, **Forgot your password?**, **Verify code**, **Set a Password**, and **Log Out**.


1. **Student Registration (`Sign up`):**
   - Exact Figma fields: `firstName`, `lastName`, `email`, `password`, `confirmPassword`, plus optional `phone` and `avatarUrl`.
   - Enforce `password === confirmPassword` via Zod schema refinement and strong password rules (minimum 8 characters, complexity rule).
   - Store email normalized (`trim().toLowerCase()`), hash password with **async** `bcrypt.hash`, assign default `role: "student"` server-side (ignore any client-supplied `role` field), and return sanitized user DTO + JWT (`201 Created`).
2. **User Login (`Sign in`):**
   - Exact Figma fields: `email`, `password`.
   - Verify password with **async** `bcrypt.compare`. Return `401 Unauthorized` on invalid credentials without specifying whether email or password failed.
   - Issue signed JWT (`accessToken`) containing `{ sub: userId, role, tokenVersion }`.
3. **Log Out (`Log Out` sidebar CTA):**
   - Increment `tokenVersion` on the `User` document (or revoke token in active store) and reject stale tokens in `authMiddleware` so logout truly invalidates existing JWTs.
4. **3-Step Password Recovery Flow (OTP Code Verification):**
   - **Step 1 — `Forgot your password?`:** Accepts `email`. Generates a cryptographically secure 6-digit OTP via **async/non-blocking** `crypto.randomInt(100000, 1000000)`, stores its SHA-256 hash (`resetOtpHash`) and 10-minute expiration (`resetOtpExpiresAt`) on the user, and dispatches via `EmailSender`. Always returns `200 OK` with a generic message so account existence cannot be enumerated. Enforce resend cooldown / rate limiting for the `"Didn't receive a code? Resend"` action.
   - **Step 2 — `Verify code`:** Accepts `email` and `code`. Compares hashed OTP (`crypto.timingSafeEqual`) against `resetOtpHash`, checks `resetOtpExpiresAt > now` and max attempt counter (e.g., 5 failed attempts locks/clears OTP), and transitions the reset flow to verified state (or issues a short-lived single-use reset token).
   - **Step 3 — `Set a Password`:** Accepts `email` (or reset token), `newPassword`, and `confirmPassword` (`newPassword === confirmPassword`). Updates password hash via async `bcrypt.hash`, clears all `resetOtp*` fields, and increments `tokenVersion` to revoke prior sessions.

---


### Module 2: Diplomas, Quizzes & Question Bank (`src/modules/diploma/`, `src/modules/quiz/`, `src/modules/question/`)

Figma frames covered: **Student Dashboard Catalog**, **Select Diploma (Exams List)**, **Admin Dashboard**, **Add Diploma Modal**, **Add Quiz Modal**, and **Add Question Modal**.


1. **Two-Level Catalog Hierarchy (`Diploma → Quiz → Question`):**
   - **Diplomas (Subjects / Career Tracks):** e.g., _Front-end Web Development_, _Android Development_, _Flutter Development_, _UI & UX Design_, _AI & ML Development_, _Back-End Web Development_.
     - Fields: `title` (`Diploma Name`), `description`, `imageUrl` (`Add Photo`), `isActive`, `quizzesCount` (computed/maintained).
     - Student browsing supports pagination (`page`, `limit`), `"View All"`, and case-insensitive keyword search (`?search=`) matching the Figma `Search Quiz` bar.
   - **Quizzes (Exams within a Diploma):** Clicking a Diploma displays its quizzes (e.g., `HTML`, `CSS`, `Bootstrap`, `JavaScript`, `Angular`, `React`), grouped optionally by `category` (e.g., _Front-End Quiz_, _Framework Quiz_).
     - Each quiz displays: `imageUrl`, `title` (`Quiz Name`), `description`, `category`, `durationMinutes` (`Time`, e.g., `15`), `questionsCount` (`20 Question`), `passingScorePercentage` (default `50`), `totalPoints`, `instructions` (array of strings for the pre-exam modal), and `isPublished`.
   - **Questions (Bank per Quiz):** Belong to a `quizId` (separate `Question` collection referencing `quizId` so quizzes stay lightweight, paginated/indexed cleanly, and compatible with streaming bulk import).
     - Exact Figma fields from `Add Question`:
       - `type`: `"single"` (Figma dropdown: **`radio button`**) or `"multi"` (Figma dropdown: **`check box`**).
       - `text`: Question prompt (`Add question`).
       - `options`: Array of option objects `{ optionId: string, text: string }` (`Add Answer 1` .. `Add Answer 4`, minimum 2 options, default 4).
       - `correctOptionIds`: Array of valid `optionId`s (`length === 1` for `"single"`, `>= 1` for `"multi"`).
       - `points`: Positive integer weight (default `1`).
2. **Admin Catalog Management (`Admin` Role Only):**
   - **Add / Update / Archive Diploma** (`Add Diploma` modal: `imageUrl`, `title`, `description`).
   - **Add / Update / Publish / Delete Quiz** (`Add Quiz` modal: `diplomaId`, `imageUrl`, `durationMinutes`, `title`, `description`, `category`, `passingScorePercentage`, `instructions`). Quizzes default to `isPublished: false` until valid questions exist (`BR-07`).
   - **Add / Update / Delete Question** (`Add Question` modal supporting `Add Another question` / `Done` flows, plus `Select Question style` dropdown `"single"` / `"multi"`). Keep `quiz.questionsCount` and `quiz.totalPoints` synchronized on question mutations.


---


### Module 3: Timed Quiz Engine (`src/modules/attempt/`)

  Figma frames covered: **Instructions Modal**, **Active Question 1..20 (Radio & Checkbox)**, and **Your Score Summary Modal**.

1. **Pre-Exam Quiz Instructions:**
   - Endpoint returning quiz summary (`title`, `durationMinutes`, `questionsCount`, `passingScorePercentage`, `instructions[]`) before creating an attempt or starting the timer.
2. **Start Quiz Session:**
   - Verifies the quiz exists and is `isPublished: true` (`questionsCount >= 1`).
   - Creates a `QuizAttempt` document with `status: "IN_PROGRESS"`, server timestamp `startedAt = new Date()`, and `expiresAt = new Date(startedAt.getTime() + durationMinutes * 60_000)`.
   - Snapshots the ordered list of questions (`questionId`, `text`, `type`, `options`, `points`, `correctOptionIds`) onto the attempt so mid-attempt Admin edits never corrupt grading (`BR-09`).
   - **Sanitized Response Payload:** Returns `attemptId`, `quizTitle`, `durationMinutes`, `startedAt`, `expiresAt`, and `questions` containing **only** `questionId`, `text`, `type`, `points`, and `options: [{ optionId, text }]`. **`correctOptionIds` MUST NEVER be returned while in progress (`BR-10`).**
3. **Submit Quiz Attempt:**
   - Accepts `answers: Array<{ questionId: string; selectedOptionIds: string[] }>`.
   - Verifies attempt ownership (`attempt.studentId === req.user.id`, else `404`).
   - Atomically transitions status from `"IN_PROGRESS"` → `"SUBMITTED"` (or `"EXPIRED"`) to defeat double-submit race conditions (`BR-12`).
   - Enforces server-side deadline (`submittedAt <= expiresAt + 10_000` ms grace window — `BR-11`).
   - Evaluates score using pure deterministic `ScoreCalculator` (`BR-13`, `BR-14`, `BR-15`).
   - Returns payload matching Figma's **`Your score`** modal:
     - `scorePercentage` (e.g., `80`)
     - `correctCount` (e.g., `18`)
     - `incorrectCount` (e.g., `2`)
     - `earnedPoints`, `totalPoints`
     - `passed` (`true` / `false`)
     - `timeSpentSeconds`

---


### Module 4: Student Dashboard, Quiz History & Answer Review (`src/modules/dashboard/` & `src/modules/attempt/`)

> Figma frames covered: **Student Dashboard**, **Quiz History**, and **Answer Quiz Detailed Review Modal**.

1. **Student Dashboard (`GET /api/v1/dashboard/me`):**
   - Returns authenticated user profile header (`firstName`, `lastName`, `avatarUrl`) alongside **3 KPI metrics** matching the Figma profile banner (`27`, `13 min`, `200`):
     - `quizzesPassed`: Count of submitted attempts where `passed === true`.
     - `fastestTimeSeconds`: Minimum `timeSpentSeconds` strictly across **passed** attempts (`passed === true`). Returns `null` if the student has 0 passed quizzes (`BR-16`).
     - `totalCorrectAnswers`: Sum of `correctCount` across all submitted attempts.
   - Must execute via a **MongoDB Aggregation Pipeline** (`$match` + `$group` / `$facet`), never by loading all attempts into Node.js memory (`BR-17`).
   - May include `diplomasPreview` (initial diplomas page) or pair cleanly with `GET /api/v1/diplomas`.
2. **Quiz History (`GET /api/v1/dashboard/history`):**
   - Paginated list of the student's completed attempts (`status` in `["SUBMITTED", "EXPIRED"]`), newest first, containing: `attemptId`, `quizId`, `quizTitle`, `quizImageUrl`, `questionsCount`, `durationMinutes`, `correctCount`, `scorePercentage`, `passed`, `timeSpentSeconds`, and `submittedAt` — powering Figma rows such as `"18 corrected answers in 12 min."`.
3. **Detailed Answer Review (`GET /api/v1/attempts/:attemptId/review` — `Answer Quiz` modal):**
   - Restricted to the owning student (`BR-05`) and **strictly blocked (`403` / `400 ATTEMPT_NOT_SUBMITTED`) if `status === "IN_PROGRESS"`** (`BR-10`).
   - Returns every question in the attempt snapshot with: `questionId`, `text`, `type`, `options`, `selectedOptionIds`, `correctOptionIds`, `isCorrect`, `pointsEarned`, `maxPoints` — allowing the UI to highlight wrong selections in red and correct answers in green.

---

## 3. Mandatory Session 1 & Session 2 Runtime & Concurrency Requirements

> Every team must directly apply the Node.js runtime architecture learned in **Session 1 (Event Loop & libuv)** and **Session 2 (Streams & Worker Threads)** inside this production codebase:

### A. Session 1 Integrations (Event Loop, `perf_hooks`, libuv Thread Pool, Graceful Shutdown)

1. **Live Event-Loop Lag Monitoring (`GET /api/v1/health`):**
   - Initialize `monitorEventLoopDelay({ resolution: 10 })` from `node:perf_hooks` during server bootstrap and call `.enable()`.
   - Expose `GET /api/v1/health` returning `{ status, uptimeSeconds, dbStatus, eventLoop: { meanDelayMs, maxDelayMs, p99DelayMs } }` so event-loop saturation under load is immediately observable.
2. **Non-Blocking Request Paths & Async `libuv` Operations:**
   - **Zero `*Sync` calls** (`bcrypt.hashSync`, `bcrypt.compareSync`, `crypto.pbkdf2Sync`, `fs.readFileSync`, `fs.writeFileSync`) anywhere in request handlers, services, or repositories.
   - Always use async `bcrypt.hash` / `bcrypt.compare` and async/non-blocking `crypto.randomInt` / streams so CPU crypto runs cleanly off the main event loop via `libuv`.
3. **Graceful Shutdown Lifecycle (`SIGINT` & `SIGTERM`):**
   - Register shutdown handlers in `server.ts` that: (1) stop accepting new HTTP connections via `server.close(...)`, (2) terminate the Worker Thread pool cleanly (`worker.terminate()`), (3) close the Mongoose connection (`mongoose.connection.close()`), and (4) exit cleanly with code `0` (with a safety timeout fallback).

### B. Session 2 Integrations (Streams, `pipeline`, Backpressure, `AbortController`, Worker Threads)

1. **Streaming Bulk Question Import (`POST /api/v1/admin/quizzes/:quizId/questions/import`):**
   - Admins must be able to upload/stream large **NDJSON** question banks (Newline-Delimited JSON, one question JSON per line) to seed or populate quizzes without loading the entire payload into memory.
   - **Implementation Mandate:** Consume `req` (or an uploaded stream) as a Node.js `Readable` stream using `node:readline` (`createInterface({ input: stream, crlfDelay: Infinity })`) or a custom `Transform` line-parser.
   - Validate each parsed line individually with Zod, buffer into bounded chunks (e.g., batches of `100` valid documents) for `QuestionModel.insertMany(...)`, enforce backpressure cleanly, update quiz `questionsCount`/`totalPoints`, and return a structured import report: `{ totalLines, importedCount, skippedCount, errors: [{ line, message }] }`. Never call `fs.readFile` or accumulate unbounded arrays in memory.
2. **Backpressured Streaming Export (`GET /api/v1/admin/quizzes/:quizId/questions/export` & `GET /api/v1/admin/quizzes/:quizId/attempts/export`):**
   - Allow Admins to export Quiz question banks and quiz-wide attempt records as **NDJSON or CSV** streams, optionally compressed when `?compress=gzip` is passed.
   - **Implementation Mandate:** Use a Mongoose cursor (`Model.find(...).cursor()`) piped through a formatting `Transform` stream (and optional `createGzip()` from `node:zlib`) into HTTP `res` via `await pipeline(...)` from `node:stream/promises`.
   - Wire an **`AbortController`** to `req.on("close", ...)` (and pass `{ signal: abortController.signal }` to `pipeline`) so client disconnects abort the DB cursor and clean up streams immediately without leaking memory.
3. **Long-Lived Worker Thread for CPU-Bound Analytics & Batch Re-Scoring (`src/common/workers/`):**
   - Offload CPU-heavy attempt analytics (e.g., `GET /api/v1/admin/quizzes/:quizId/analytics` computing score distribution histograms, percentile rankings, p50/p90 latency, and per-question item discrimination/failure rates across thousands of attempts — or bulk attempt re-scoring when an Admin fixes a flawed question key) to a **`node:worker_threads` worker**.
   - **Implementation Mandate (Learned in Session 2 Task 2):**
     - Instantiate a **long-lived Worker Thread (or small bounded pool)** at startup/module init — **NEVER spawn `new Worker()` per HTTP request**.
     - Communicate via `postMessage({ requestId, taskType, payload })` backed by a `Map<number, { resolve, reject }>` with per-request timeouts.
     - Attach **both `.on("error")` and `.on("exit")` handlers**: if the worker throws or exits unexpectedly (`code !== 0`), immediately reject all pending promises in the map, clear stale references, and **automatically respawn a replacement worker** so subsequent requests never hang.
     - Keep `GET /api/v1/health` event-loop lag `< 100 ms` even while heavy analytics jobs execute in the worker.

---

## 4. Numbered Business Rules (`BR-01`–`BR-22`) & Edge-Case Matrix

### Business Rules

| Rule ID | Rule Name | Mandatory Requirement |
|---|---|---|
| **BR-01** | **Boot-Time Env Validation** | Parse & validate `NODE_ENV`, `PORT`, `MONGO_URI`, `JWT_SECRET` (min 32 chars in prod), `JWT_EXPIRES_IN`, and `OTP_EXPIRES_MINUTES` via Zod in `src/config/env.ts` on startup. Crash fast with a clear message if invalid. |
| **BR-02** | **Password Storage & Confirmation** | Sign-up and password reset MUST validate `password === confirmPassword`, hash with async `bcrypt.hash` (cost `>= 10`), and set `select: false` / DTO stripping so hashes never appear in responses or logs. |
| **BR-03** | **Non-Enumerating OTP Recovery** | `forgot-password` generates a 6-digit OTP via `crypto.randomInt`, stores only `sha256(otp)` + expiry (`10 min`), enforces resend cooldown, and returns generic `200 OK` regardless of email existence. |
| **BR-04** | **Two-Step Reset Verification** | Students cannot skip `verify-reset-code`; `reset-password` requires a verified OTP session, clears OTP metadata after use, and bumps `tokenVersion` to invalidate old JWTs. |
| **BR-05** | **Strict RBAC & Ownership Isolation** | Every non-public route requires valid JWT (`tokenVersion` matched). Admin routes require `role === "admin"`. Student attempt/review queries scope strictly by `{ _id, studentId: userId }` (`404` on mismatch). |
| **BR-06** | **Two-Level Catalog Integrity** | Every Quiz belongs to an active `Diploma`; every Question belongs to an existing `Quiz`. Deleting a non-empty Diploma or Quiz must either be blocked (`409 CONFLICT`) or perform explicit guarded soft-archive (`isActive: false`) — never leave dangling attempts or orphan queries returning `null`. |
| **BR-07** | **Quiz Publishability Guard** | A Quiz cannot be marked `isPublished: true` or started by a Student unless it contains `>= 1` valid Question and `durationMinutes >= 1`. |
| **BR-08** | **Question Style & Answer Key Integrity** | `type === "single"` requires `options.length >= 2` (Figma shows 4) and `correctOptionIds.length === 1`. `type === "multi"` requires `options.length >= 2` and `correctOptionIds.length >= 1`. Every ID in `correctOptionIds` MUST exist in `options[].optionId`. |
| **BR-09** | **Attempt Question Snapshot Isolation** | Starting an attempt snapshots question text, type, options, `points`, and `correctOptionIds` onto `QuizAttempt` so subsequent Admin edits or question deletions never break active or historical grading. |
| **BR-10** | **Zero Answer-Key Leakage** | While `attempt.status === "IN_PROGRESS"`, `correctOptionIds` and `isCorrect` MUST be stripped from all responses, and `GET /attempts/:id/review` MUST reject access (`403` / `400`). |
| **BR-11** | **Server-Authoritative Timer & Grace Window** | `startedAt` and `expiresAt = startedAt + durationMinutes * 60_000` are computed exclusively server-side. Submissions with `now <= expiresAt + 10_000` (10s latency grace) are graded normally; submissions past `expiresAt + 10_000` transition status to `"EXPIRED"` with `scorePercentage: 0`, `passed: false`, and return `400 ATTEMPT_EXPIRED`. |
| **BR-12** | **Atomic Double-Submit Idempotency** | Submission MUST claim the attempt atomically (e.g., `findOneAndUpdate({ _id, studentId, status: "IN_PROGRESS" }, { $set: { status: "SUBMITTING" } })` or conditional status transition on final write). Concurrent or duplicate submissions MUST receive `409 Conflict` (`ATTEMPT_ALREADY_SUBMITTED`), never score twice. |
| **BR-13** | **Exact-Set & Weighted MCQ Scoring** | Grade by `optionId` (never option text). Deduplicate `selectedOptionIds`. For both `single` and `multi`, award `question.points` **only** when the normalized set of selected IDs equals `correctOptionIds` (zero partial credit). |
| **BR-14** | **Full-Quiz Denominator Invariant** | `scorePercentage = Math.round((earnedPoints / totalQuizPoints) * 100)` where `totalQuizPoints` sums `points` across **every** question in the quiz snapshot. Unanswered or omitted questions score `0` earned points and remain in the denominator. |
| **BR-15** | **Pass/Fail Threshold** | `passed = scorePercentage >= quizPassingScorePercentage` (default `50`). Evaluated strictly server-side. |
| **BR-16** | **Fastest Time Over Passed Attempts Only** | Dashboard `fastestTimeSeconds` computes `MIN(timeSpentSeconds)` strictly filtering `{ status: "SUBMITTED", passed: true }`. Failed attempts (`0s` speed-runs or incomplete quizzes) MUST NOT count. Returns `null` when a student has 0 passed attempts. |
| **BR-17** | **Database-Side Dashboard Aggregation** | Dashboard KPIs (`quizzesPassed`, `fastestTimeSeconds`, `totalCorrectAnswers`) MUST be computed in MongoDB via Aggregation Pipeline (`$match`, `$group`, `$facet`) rather than fetching attempt documents into Node arrays. |
| **BR-18** | **Clamped Completion Time Storage** | Store `timeSpentSeconds = Math.max(0, Math.min(Math.round((submittedAt - startedAt) / 1000), durationMinutes * 60))` so the 10-second network grace window never records `15m 08s` for a `15m` exam. |
| **BR-19** | **Non-Blocking Runtime & Health Telemetry** | Request handlers use async crypto/I/O only. `GET /api/v1/health` reports DB readiness and live `monitorEventLoopDelay` metrics (`node:perf_hooks`). |
| **BR-20** | **Bounded-Memory Streaming I/O** | Bulk NDJSON question import streams line-by-line without `readFile`; exports stream MongoDB cursors through `pipeline(...)` with `createGzip()` support and `AbortController` disconnect cleanup. |
| **BR-21** | **Self-Healing Worker Thread Pool** | Heavy batch analytics / re-scoring runs on a long-lived `node:worker_threads` worker with `requestId` correlation plus `.on("error")` & `.on("exit")` recovery/respawn. |
| **BR-22** | **Canonical Prefix, Validation & Envelopes** | All routes mount under `/api/v1`, validate `body`/`params`/`query` with Zod before controllers execute, and return uniform `{ success: true, data }` or `{ success: false, error: { code, message } }` envelopes. |

### Concrete Edge-Case Matrix (Mandatory Test & Implementation Scenarios)

| Scenario | Expected Server Behavior |
|---|---|
| Student submits attempt twice simultaneously (two parallel `POST /submit` requests) | First request transitions state and grades; second hits atomic guard and fails with `409 Conflict` (`ATTEMPT_ALREADY_SUBMITTED`). |
| Student submits at `startedAt + 15m 07s` on a `15 min` quiz (within 10s grace window) | Accepted (`200 OK`), graded normally, `timeSpentSeconds` clamped to `900` (`15 * 60`). |
| Student submits at `startedAt + 15m 15s` (past 10s grace window) | Rejected with `400 Bad Request` (`ATTEMPT_EXPIRED`), attempt persisted as `status: "EXPIRED"`, `scorePercentage: 0`, `passed: false`. |
| Multi-choice question has `correctOptionIds: ["opt_a", "opt_c"]`; student sends `["opt_c", "opt_a", "opt_a"]` | Deduplicated set `{"opt_a", "opt_c"}` matches answer key → full `question.points` awarded. |
| Multi-choice question has `correctOptionIds: ["opt_a", "opt_c"]`; student sends `["opt_a"]` or `["opt_a", "opt_b", "opt_c"]` | Not an exact set match → `0` points awarded (no partial credit). |
| Quiz has 20 questions (`1 pt` each); student sends answers for only 10 questions and gets all 10 right | `earnedPoints = 10`, `totalQuizPoints = 20` → `scorePercentage = 50%`, `correctCount = 10`, `incorrectCount = 10`. |
| New student or student who failed 3 quizzes opens Student Dashboard | `quizzesPassed = 0`, `fastestTimeSeconds = null`, `totalCorrectAnswers = sum(correctCount across failed attempts)`. |
| Student calls `POST /admin/quizzes` or `GET /attempts/:otherId/review` | `403 FORBIDDEN` on admin route; `404 NOT_FOUND` on another student's attempt. |
| Student calls `GET /attempts/:id/review` while `status === "IN_PROGRESS"`                                                    | Rejected (`403 FORBIDDEN` / `ATTEMPT_NOT_SUBMITTED`) — never reveals answer key mid-exam.                                                                       |
| Admin uploads a 50,000-line NDJSON file where line 42 is malformed JSON                                                      | Invalid line logged in `errors[]`, remaining valid lines stream-inserted in batches; memory stays flat.                                                         |
| Client cancels HTTP request mid-stream during NDJSON/CSV gzip export                                                         | `req.on("close")` fires `abortController.abort()`, closing MongoDB cursor and zlib/transform streams cleanly without `ERR_STREAM_PREMATURE_CLOSE` crash.        |
| Worker thread crashes or exits mid-analytics job                                                                             | `.on("error")`/`.on("exit")` rejects all pending promises for that worker cleanly (`500` / error envelope), respawns a fresh worker, and next request succeeds. |

---


## 5. Canonical API Contract, Envelopes & Endpoint Reference


> **Assignment Note for Teams:** Before writing Express controllers, finalize your exact request/response DTO schemas in Swagger/Postman or an API Markdown table matching this contract and review it with your mentor.


### A. Base Path & JSON Response Envelopes

- **Base Prefix:** `/api/v1` (plus `/api/v1/health`)
- **Success Envelope:**
  ```json
  {
    "success": true,
    "data": {},
    "meta": { "page": 1, "limit": 10, "total": 42, "totalPages": 5 }
  }
  ```
  _(Omit `meta` on non-paginated responses.)_
- **Error Envelope:**
  ```json
  {
    "success": false,
    "error": {
      "code": "ATTEMPT_EXPIRED",
      "message": "Quiz submission grace window has expired",
      "details": []
    }
  }
  ```
- **HTTP Status Semantics:** `200 OK` (read/submit/update), `201 Created` (register/create/start attempt), `400 Bad Request` (validation error, expired attempt), `401 Unauthorized` (missing/invalid/revoked JWT), `403 Forbidden` (insufficient role or unsubmitted review), `404 Not Found` (missing resource or non-owned resource), `409 Conflict` (duplicate email, already submitted attempt, delete with dependents), `429 Too Many Requests` (OTP resend cooldown), `500 Internal Server Error` (unhandled exception — never leak stack trace in production).

### B. Required Endpoints Inventory

| Module                  | Method & Path                                         | Access  | Figma Screen / Engineering Purpose                                                                                             |
| ----------------------- | ----------------------------------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **Health**              | `GET /api/v1/health`                                  | Public  | DB status, uptime & live `perf_hooks` event-loop lag (**Session 1**)                                                           |
| **Auth**                | `POST /api/v1/auth/register`                          | Public  | `Sign up` (`firstName`, `lastName`, `email`, `password`, `confirmPassword`, `phone?`)                                          |
| **Auth**                | `POST /api/v1/auth/login`                             | Public  | `Sign in` (`email`, `password`) → JWT + sanitized user                                                                         |
| **Auth**                | `POST /api/v1/auth/logout`                            | Auth    | `Log Out` (increments `tokenVersion` / revokes session)                                                                        |
| **Auth**                | `POST /api/v1/auth/forgot-password`                   | Public  | `Forgot your password?` + `Resend` (sends 6-digit OTP)                                                                         |
| **Auth**                | `POST /api/v1/auth/verify-reset-code`                 | Public  | `Verify code` (validates OTP before allowing password set)                                                                     |
| **Auth**                | `POST /api/v1/auth/reset-password`                    | Public  | `Set a Password` (`newPassword`, `confirmPassword`)                                                                            |
| **Auth**                | `GET /api/v1/auth/me`                                 | Auth    | Fetch current authenticated profile                                                                                            |
| **Catalog (Student)**   | `GET /api/v1/diplomas`                                | Auth    | Student & Admin `Diplomas` catalog (`?search=&page=&limit=`)                                                                   |
| **Catalog (Student)**   | `GET /api/v1/diplomas/:diplomaId/quizzes`             | Auth    | `Select Diploma` screen showing published quizzes under a diploma                                                              |
| **Catalog (Student)**   | `GET /api/v1/quizzes/:quizId`                         | Auth    | `Instructions` modal (`title`, `durationMinutes`, `questionsCount`, `instructions[]`)                                          |
| **Attempts (Student)**  | `POST /api/v1/quizzes/:quizId/attempts`               | Student | `Start` quiz → creates attempt, sets server timer, returns stripped questions                                                  |
| **Attempts (Student)**  | `POST /api/v1/attempts/:attemptId/submit`             | Student | `Submit` quiz → validates timer, grades MCQ, returns `Your score` summary                                                      |
| **Attempts (Student)**  | `GET /api/v1/dashboard/history`                       | Student | `Quiz History` paginated list                                                                                                  |
| **Attempts (Student)**  | `GET /api/v1/attempts/:attemptId/review`              | Student | `Answer Quiz` detailed review (only after submission)                                                                          |
| **Dashboard (Student)** | `GET /api/v1/dashboard/me`                            | Student | MongoDB-aggregated KPIs (`quizzesPassed`, `fastestTimeSeconds`, `totalCorrectAnswers`)                                         |
| **Admin — Diplomas**    | `POST /api/v1/admin/diplomas`                         | Admin   | `Add Diploma` modal (`title`, `description`, `imageUrl`)                                                                       |
| **Admin — Diplomas**    | `PATCH / DELETE /api/v1/admin/diplomas/:diplomaId`    | Admin   | Edit or safely archive/delete Diploma                                                                                          |
| **Admin — Quizzes**     | `POST /api/v1/admin/diplomas/:diplomaId/quizzes`      | Admin   | `Add Quiz` modal (`title`, `description`, `category`, `imageUrl`, `durationMinutes`, `passingScorePercentage`, `instructions`) |
| **Admin — Quizzes**     | `PATCH / DELETE /api/v1/admin/quizzes/:quizId`        | Admin   | Edit, publish (`isPublished`), or safely delete Quiz                                                                           |
| **Admin — Questions**   | `POST /api/v1/admin/quizzes/:quizId/questions`        | Admin   | `Add Question` modal (`type`: `"single"` or `"multi"`, `text`, `options`, `correctOptionIds`, `points`)                        |
| **Admin — Questions**   | `PATCH / DELETE /api/v1/admin/questions/:questionId`  | Admin   | Update or remove Question (updates parent Quiz counts)                                                                         |
| **Admin — Streams**     | `POST /api/v1/admin/quizzes/:quizId/questions/import` | Admin   | NDJSON streaming bulk question import (**Session 2 Streams**)                                                                  |
| **Admin — Streams**     | `GET /api/v1/admin/quizzes/:quizId/questions/export`  | Admin   | Backpressured NDJSON/CSV streaming export (`?format=csv` or `ndjson`, `&compress=gzip`, **Session 2 `pipeline`**)              |
| **Admin — Streams**     | `GET /api/v1/admin/quizzes/:quizId/attempts/export`   | Admin   | Backpressured NDJSON/CSV streaming export of quiz attempts (`?format=csv` or `ndjson`, `&compress=gzip`, **Session 2 `pipeline`**) |
| **Admin — Worker**      | `GET /api/v1/admin/quizzes/:quizId/analytics`         | Admin   | Offloaded CPU-bound item difficulty & score percentiles (**Session 2 Worker Thread**)                                          |

---

## 6. Recommended Modular Architecture, 5-Engineer Ownership Matrix & 7-Day Waterfall Plan

### A. Recommended Modular Project Layout (Vertical Slices)

Keep files grouped by domain feature slice so all **5 teammates implement their Phase 2 modules in parallel with zero overlapping files or merge conflicts**:

```text
src/
├── config/
│   ├── env.ts                        # Zod fail-fast env schema (BR-01)
│   └── database.ts                   # Mongoose connection + disconnect helpers
├── common/
│   ├── errors/                       # AppError hierarchy (ValidationError, UnauthorizedError, ConflictError...)
│   ├── middlewares/                  # globalErrorHandler, asyncHandler, authJwt, requireRole, validateRequest
│   ├── telemetry/                    # perf_hooks monitorEventLoopDelay + /health controller/routes (Session 1)
│   ├── utils/                        # responseEnvelope helper, pagination parser, EmailSender abstraction
│   └── workers/                      # Long-lived worker manager + analytics worker script (Session 2)
├── modules/
│   ├── auth/                         # auth.{model,dto,repository,service,controller,routes}.ts + *.spec.ts
│   ├── diploma/                      # diploma.{model,dto,repository,service,controller,routes}.ts + *.spec.ts
│   ├── quiz/                         # quiz.{model,dto,repository,service,controller,routes}.ts + *.spec.ts
│   ├── question/                     # question.{model,dto,repository,service,stream-import,stream-export,controller,routes}.ts + *.spec.ts
│   ├── attempt/                      # attempt.{model,dto,repository,service,score-calculator,controller,routes}.ts + *.spec.ts
│   └── dashboard/                    # dashboard.{repository,service,controller,routes}.ts + *.spec.ts (KPIs, Quiz History & Admin Analytics)
├── app.ts                            # Express app wiring, security/rate-limit middleware, /api/v1 router
└── server.ts                         # Boot sequence, worker init, monitorEventLoopDelay, SIGINT/SIGTERM graceful shutdown
```

_(Note: Thin repositories/DAOs isolate Mongoose queries from services so service business logic stays 100% unit-testable with zero real DB connections, following [`docs/standards/clean-code.md`](../standards/clean-code.md) and [`docs/standards/testing-jest.md`](../standards/testing-jest.md).)_

### B. 5-Engineer Vertical-Slice Ownership Matrix

After completing the **Shared Phase 1 Foundation (All 5 Engineers — Days 1–2)** to freeze the API contracts, Mongoose schemas, and `src/config/env.ts` + `src/common/` skeleton, divide Phase 2 implementation across these 5 non-overlapping slices:

| Teammate | Primary Ownership | Key Deliverables & Engineering Focus |
|---|---|---|
| **Engineer 1 — Authentication & Identity** | `src/modules/auth/` | `User` model & repository, `authJwt` & `requireRole` middlewares, Sign Up (`confirmPassword`), Sign In, Log Out (`tokenVersion` invalidation), `/auth/me`, 3-step OTP Password Recovery (`forgot-password`, `verify-reset-code`, `reset-password` with async `crypto.randomInt` & `bcrypt.hash`), and `AuthService`/`AuthController` Jest unit tests. |
| **Engineer 2 — Diplomas & Quiz Catalog** | `src/modules/diploma/`, `src/modules/quiz/` | `Diploma` & `Quiz` models/repositories, Admin CRUD (`Add Diploma`, `Add Quiz`, publish guard `BR-07`, safe archive/delete `BR-06`), Student catalog browsing (`?search=`, pagination, diploma → quizzes listing, pre-exam `Instructions` endpoint), catalog seed script, and `DiplomaService`/`QuizService` Jest unit tests. |
| **Engineer 3 — Question Bank & Streaming Bulk I/O** | `src/modules/question/` | `Question` model & repository, Admin `Add/Update/Delete Question` (`"single"` [radio] & `"multi"` [checkbox] validation `BR-08`, synchronizing `quiz.questionsCount` & `quiz.totalPoints`), **Session 2 NDJSON Streaming Bulk Question Import** (`POST /api/v1/admin/quizzes/:quizId/questions/import` via `readline` + batched `insertMany`), **Session 2 Backpressured Streaming Question/Attempt Export** (`pipeline` + `createGzip` + `AbortController`), and `QuestionService` + stream unit tests. |
| **Engineer 4 — Timed Quiz Attempt Engine & Scoring** | `src/modules/attempt/` | Sole owner of `src/modules/attempt/`: `QuizAttempt` model & repository, `Start Quiz` session (question snapshot `BR-09`, stripping `correctOptionIds` `BR-10`, server `startedAt`/`expiresAt`), `Submit Quiz` (server timer + 10s grace `BR-11`, atomic double-submit guard `BR-12`), pure `ScoreCalculator` (`BR-13`–`BR-15`), `Answer Quiz` detailed review (`GET /api/v1/attempts/:attemptId/review`), and `ScoreCalculator` + `AttemptService` Jest unit tests. |
| **Engineer 5 — Dashboard, Quiz History, Worker Analytics & Runtime Telemetry** | `src/modules/dashboard/`, `src/common/workers/`, `src/common/telemetry/` | Student Dashboard MongoDB Aggregation Pipeline (`GET /api/v1/dashboard/me` — `quizzesPassed`, `fastestTimeSeconds` gated on `passed: true` `BR-16`, `totalCorrectAnswers`), Student `Quiz History` (`GET /api/v1/dashboard/history`), **Session 2 Self-Healing Long-Lived Worker Thread** (`GET /api/v1/admin/quizzes/:quizId/analytics` with `error`/`exit` recovery), **Session 1 `/api/v1/health` (`monitorEventLoopDelay`) & Graceful Shutdown**, and `DashboardService` + worker manager Jest unit tests. |

### C. 7-Day Waterfall Execution Plan (Sequential Phases)

All phases execute sequentially with explicit gate sign-offs before advancing to the next phase:

| Phase & Days | Milestone & Exit Gate |
|---|---|
| **Phase 1 — Requirements Analysis & System Design (Days 1–2)** | Analyze all Figma screens as a team; freeze the complete **REST API Contract Table**, **Mongoose ER Schemas** (`User`, `Diploma`, `Quiz`, `Question`, `QuizAttempt`), and **Zod DTO schemas**; get **Mentor Design Sign-Off**; merge the shared project skeleton (`src/config/`, `src/common/errors/`, `src/common/middlewares/`, `src/common/utils/`, `app.ts`, `server.ts`). **Exit Gate:** No feature module coding starts until Mentor approves the API & schema design. |
| **Phase 2 — Module Implementation / Coding (Days 3–4)** | Each of the 5 engineers implements their assigned vertical slice strictly against the frozen Phase 1 contracts and schemas with zero overlapping files (`Eng 1`: Auth + OTP + RBAC; `Eng 2`: Diplomas + Quizzes + Seed; `Eng 3`: Question Bank + Streaming Import/Export; `Eng 4`: Timed Attempt Engine + `ScoreCalculator` + Review; `Eng 5`: Dashboard Aggregation + Quiz History + Worker Analytics + `/health` & Shutdown). |
| **Phase 3 — Unit Testing & Component Verification (Day 5)** | Each engineer writes exhaustive co-located Jest unit tests (`*.spec.ts`) for their module's services, pure calculators, stream transformers, worker manager, and controllers using clean mocked repository boundaries (zero real DB/network calls). |
| **Phase 4 — System Integration & End-to-End Verification (Day 6)** | Merge all vertical slices onto `main`, seed realistic catalog data (e.g., *Front-End Web Development → HTML 20 Questions / 15 Min*), execute full Student & Admin end-to-end Postman/HTTP smoke scenarios, verify concurrent double-submit rejection (`409`), verify backpressured/aborted streaming I/O, verify Worker Thread error/exit crash recovery, and confirm `/api/v1/health` event-loop lag remains healthy under load. |
| **Phase 5 — Code Audit, Documentation & Final Delivery (Day 7)** | Audit full codebase against [`docs/standards/`](../standards/README.md) (check for `any`, `*Sync` blockers, leaked DB entities, or silent catches), finalize `README.md` + `.env.example` + API documentation ([`docs/readme-guide.md`](../readme-guide.md)), and complete final mentor sign-off / team presentation. |

---

## 7. Mandatory Unit Testing Coverage (Jest)

All unit tests live co-located inside their modules (`*.spec.ts` / `tests/*.spec.ts`), mock repository/external boundaries cleanly (never testing the mock itself — see [`docs/standards/testing-jest.md`](../standards/testing-jest.md)), and **must verify every critical branch and edge case**:

1. **`ScoreCalculator` (`modules/attempt/`):**
   - Single-choice exact match vs wrong choice vs omitted answer.
   - Multi-choice exact match (out-of-order and duplicate option IDs in payload) vs partial match (`0` pts) vs extra wrong option (`0` pts).
   - Weighted `question.points`, full-quiz denominator invariant (`totalQuizPoints`), rounding, and pass/fail boundary (`scorePercentage === passingScorePercentage`).
2. **`AuthService` (`modules/auth/auth.service.spec.ts`):**
   - Registration rejects duplicate email (`409`), enforces `confirmPassword`, ignores client `role`, and never calls `*Sync` crypto.
   - Login rejects wrong password (`401`) and issues JWT on valid credentials.
   - 3-step OTP reset validates wrong/expired OTP, blocks `reset-password` before `verify-reset-code`, and bumps `tokenVersion`.
3. **`AttemptService` (`modules/attempt/`):**
   - `startAttempt` blocks unpublished/empty quizzes (`BR-07`) and strips `correctOptionIds` (`BR-10`).
   - `submitAttempt` accepts within `+10s` grace window (`BR-11`), rejects past grace window (`400 ATTEMPT_EXPIRED`), enforces atomic single-submit idempotency (`409` on double submit — `BR-12`), and rejects non-owner attempts (`404`).
   - `getAttemptReview` blocks `IN_PROGRESS` attempts (`BR-10`) and returns expected selection/key diff after submission.
4. **`DashboardService` (`modules/dashboard/dashboard.service.spec.ts`):**
   - Verifies MongoDB aggregation pipeline stages / outputs: `fastestTimeSeconds` excludes failed attempts (`BR-16`), returns `null` when zero quizzes passed, and sums `totalCorrectAnswers`.
5. **Streaming & Worker Thread Resilience (`modules/question/`, `common/workers/`):**
   - NDJSON line parser/importer handles chunk boundaries split mid-line, isolates invalid JSON lines without crashing, and verifies zero full-file buffering.
   - Worker manager rejects pending requests cleanly on worker `"error"` / `"exit"` events and automatically respawns a replacement worker.
6. **Controllers & Middleware:**
   - Assert expected status codes (`200`, `201`), canonical envelope `{ success, data }`, and `next(err)` delegation when services throw.
