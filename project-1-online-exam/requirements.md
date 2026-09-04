# Project 1: Online Exam Platform — Requirements & System Specification

> **Bootcamp:** Node.js Backend Engineering Bootcamp  
> **Project #:** 1 (First Project)  
> **Primary Goals:** Learn project design, API extraction & implementation from scratch + Comprehensive Unit Testing  
> **Tech Stack:** Node.js · Express.js · TypeScript · MongoDB (Mongoose) · Jest  
> **Figma Design Link:** [Online Exam UI Design (Figma)](https://www.figma.com/design/jJl1SNjeasOAF0WSlK8epD/Exam-Online-Elevate?node-id=0-1&p=f&t=ugdI6dG6TNAfpT7k-0)

---

## 1. Project Overview & Philosophy

The **Online Exam Platform** is a streamlined application that allows students to register, take timed multiple-choice quizzes, view performance insights on a personalized dashboard, and review correct answers after quiz submission.

### Design Principles for Trainees:
1. **Design & API Extraction from Scratch:** Practice analyzing the [Figma Design](https://www.figma.com/design/jJl1SNjeasOAF0WSlK8epD/Exam-Online-Elevate?node-id=0-1&p=f&t=ugdI6dG6TNAfpT7k-0), identifying the required endpoints, and drafting your own API contracts before writing code.
2. **Modular Architecture (Vertical Slices):** Group files by feature modules (`auth/`, `quiz/`, `dashboard/`) rather than technical layers so code remains cohesive and easy to navigate.
3. **Simplicity First (KISS):** Keep data structures and API routes uncluttered so you can focus on clean code and robust unit tests.
4. **Never Trust the Client:** Enforce all timing, scoring, and answer validation logic strictly on the server side.
5. **Test-Driven Mentality:** Write unit tests for all core business logic, utility calculations, and controllers using Jest.

---

## 2. Tech Stack & Architecture Overview

```
                      ┌─────────────────────────────────────────┐
                      │             Express App (app.ts)        │
                      │       Mounts Modular Feature Routes     │
                      └────────────────────┬────────────────────┘
                                           │
         ┌─────────────────────────────────┼─────────────────────────────────┐
         │                                 │                                 │
┌────────▼────────┐               ┌────────▼────────┐               ┌────────▼────────┐
│   Auth Module   │               │   Quiz Module   │               │Dashboard Module │
│ (routes/ctrl/   │               │ (routes/ctrl/   │               │ (routes/ctrl/   │
│ service/models) │               │ service/models) │               │ service/models) │
└─────────────────┘               └─────────────────┘               └─────────────────┘
```

- **Runtime & Language:** Node.js + TypeScript (Strict Mode enabled).
- **Web Framework:** Express.js.
- **Database & ODM:** MongoDB with Mongoose.
- **Auth:** JSON Web Tokens (JWT) & bcrypt for password hashing.
- **Unit Testing:** Jest (with zero real DB dependencies in unit tests).

---

## 3. Recommended Modular Project Structure

Trainees should structure their project using **Feature Modules (Vertical Slices)**. Each module encapsulates its own routes, controllers, services, models, DTOs, and unit tests:

```
src/
├── config/                  # DB connection, env variables & application configuration
├── common/                  # Cross-cutting concerns shared across all modules
│   ├── middlewares/         # Global error handler, JWT auth middleware, validation pipe
│   ├── utils/               # Shared helpers (response formatting, date helpers)
│   └── errors/              # Custom AppError exception classes
├── modules/                 # Modular Feature Slices
│   ├── auth/                # Auth Feature Module (Register, Login, Password Reset)
│   │   ├── auth.controller.ts
│   │   ├── auth.service.ts
│   │   ├── auth.model.ts
│   │   ├── auth.routes.ts
│   │   ├── auth.dto.ts
│   │   └── auth.service.spec.ts
│   ├── quiz/                # Quiz Engine Feature Module (Catalog, Timed Attempt, Scoring)
│   │   ├── quiz.controller.ts
│   │   ├── quiz.service.ts
│   │   ├── quiz.model.ts
│   │   ├── quiz-attempt.model.ts
│   │   ├── quiz.routes.ts
│   │   ├── utils/
│   │   │   └── score-calculator.ts
│   │   └── tests/
│   │       ├── quiz.service.spec.ts
│   │       └── score-calculator.spec.ts
│   └── dashboard/           # Dashboard & Analytics Feature Module
│       ├── dashboard.controller.ts
│       ├── dashboard.service.ts
│       ├── dashboard.routes.ts
│       └── dashboard.service.spec.ts
├── app.ts                   # Express application setup & mounting feature routes
└── server.ts                # Application entry point & HTTP listener bootstrap
```

### Why Modular Architecture?
- **High Cohesion & Low Coupling:** Everything related to a single feature (e.g., `quiz`) lives in one folder. Adding or changing a feature doesn't require jumping between distant directories.
- **Scalability:** Teams of trainees can work on separate modules (`auth` vs `quiz`) concurrently with minimal git merge conflicts.
- **Locality of Tests:** Unit tests live alongside the code they test (e.g. `auth.service.spec.ts` inside `modules/auth/`).

---

## 4. Functional Requirements

Review the [Figma UI Design](https://www.figma.com/design/jJl1SNjeasOAF0WSlK8epD/Exam-Online-Elevate?node-id=0-1&p=f&t=ugdI6dG6TNAfpT7k-0) and implement the 4 core feature areas below:

### Module 1: Authentication Cycle (`modules/auth`)
- **User Registration:** Allow new users to create an account with name, email, and password.
- **User Login:** Authenticate credentials and issue a JWT token.
- **Forgot & Reset Password:** Allow users to request password recovery and reset their password using a reset token.

### Module 2: User Dashboard & Quiz Catalog (`modules/dashboard` & `modules/quiz`)
- **Dashboard Performance Insights:** Display personalized user statistics:
  - **Quizzes Passed:** Total count of passed quizzes.
  - **Fastest Completion Time:** Shortest time taken (in seconds) to complete any passed quiz.
  - **Total Correct Answers:** Total number of correct answers across all quiz attempts.
- **Quiz Catalog:** List available quizzes showing title, description, duration (in minutes), question count, and passing threshold percentage.

### Module 3: Timed Quiz Engine (`modules/quiz`)
- **Quiz Instructions & Details:** Show quiz instructions, duration, and question count before starting.
- **Start Quiz Session:** Initialize an active quiz attempt with a server-generated start timestamp. Return questions to the user **without revealing the correct options**.
- **Submit Quiz Attempt:** Accept user answers, enforce server-side timer validation (`now <= startTime + durationMinutes + grace_period`), evaluate scores for single-choice and multi-choice questions, and persist attempt metrics.

### Module 4: Results & Answer Review (`modules/quiz`)
- **Attempt Result Summary:** Display overall percentage score, pass/fail status, and time spent.
- **Detailed Question Review:** Show a question-by-question breakdown containing question text, user's selected option(s), correct option(s), and correctness indicator.

---

## 5. Data Requirements & Schema Design Challenge

> 💡 **Design Exercise for Trainees:** You are responsible for designing the database models and Mongoose schemas from scratch based on the domain entities below.

### Entities to Model:
1. **User Entity (`modules/auth/auth.model.ts`):**
   - Must store user identification details, unique email, hashed password, and timestamp tracking.
2. **Quiz & Question Entities (`modules/quiz/quiz.model.ts`):**
   - Needs to store quiz metadata (title, description, instructions, duration limit in minutes, pass score percentage).
   - Questions can be **single-choice** or **multi-choice**.
   - Each question contains options (ID & text) and correct option IDs.
   - *Design Decision for You:* Decide whether questions should be embedded array documents inside `Quiz` or stored in a separate collection referenced by `quizId`. Explain your decision to your mentor!
3. **Quiz Attempt Entity (`modules/quiz/quiz-attempt.model.ts`):**
   - Tracks a user's attempt on a specific quiz.
   - Must record starting server timestamp, completion/submitted timestamp, calculated score percentage, pass/fail status, total correct count, and user's selected option IDs per question.

---

## 6. API Design & Extraction Challenge for Trainees

> 🎯 **Extraction Assignment:** Instead of being given pre-defined API endpoints, you must extract and design the REST API contract yourselves from the [Figma Design](https://www.figma.com/design/jJl1SNjeasOAF0WSlK8epD/Exam-Online-Elevate?node-id=0-1&p=f&t=ugdI6dG6TNAfpT7k-0) and functional requirements.

### Deliverables Required Before Coding Controllers:
1. **Identify Resources & Endpoints:** List all HTTP routes required for Authentication, Dashboard Insights, Quiz Catalog, Quiz Attempt Engine, and Results Review.
2. **Select HTTP Verbs:** Assign appropriate REST methods (`GET`, `POST`, `PUT`, `DELETE`).
3. **Define Request/Response Payloads:** Specify request bodies, path parameters, query params, and JSON response bodies.
4. **Determine Auth Protection:** Mark which endpoints are public vs protected by JWT Bearer authentication.

> 📝 **Mentor Review:** Review your proposed API endpoint table with your mentor before starting your Express route implementations!

---

## 7. Business Logic Rules (Scoring & Timers)

### 1. MCQ Scoring Rules
- **Single-Choice (`type: "single"`):** User selects 1 option ID. If it matches the correct option ID, award `question.points`. Otherwise `0`.
- **Multi-Choice (`type: "multi"`):** User selects multiple option IDs. Award full points **only** if the selected options exactly match the correct options (all correct options selected, zero incorrect options selected).

### 2. Server-Side Timer Validation
- When user starts a quiz, record `startTime = new Date()`.
- When user submits answers, compare `submittedAt` vs `startTime`:
  ```typescript
  const maxAllowedTimeMs = startTime.getTime() + (durationMinutes * 60 + 10) * 1000; // 10s grace period for network latency
  if (now.getTime() > maxAllowedTimeMs) {
    // Attempt expired! Mark score as 0 and passed as false
  }
  ```

---

## 8. Best Practices for Coding & REST APIs

### A. Software Engineering & Coding Best Practices

1. **Modular Architecture & Separation of Concerns:**
   - Group files by domain modules (`auth/`, `quiz/`, `dashboard/`).
   - Inside each module, keep responsibility clear:
     - **Controllers:** Handle HTTP requests/responses ONLY (parse `req.body`/`req.params`, call service, send status code).
     - **Services:** Contain ALL business logic, scoring formulas, and timer checks.
     - **Models:** Handle Mongoose database schemas ONLY.
2. **Strict Environment Variable Validation:**
   - Validate required environment variables (`PORT`, `MONGO_URI`, `JWT_SECRET`) at application startup in `config/` so the app fails fast if a configuration is missing.
3. **Centralized Error Handling (`common/errors` & `common/middlewares`):**
   - Never let standard Express route handlers crash quietly or leak raw stack traces to users.
   - Create a custom `AppError` class that extends `Error` with HTTP status codes.
   - Use a global error middleware (`(err, req, res, next) => { ... }`) to catch all unhandled errors.
4. **Input Sanitization & Validation:**
   - Validate every incoming payload using a validation library (e.g. Zod, Joi, or class-validator) inside a middleware before reaching the controller handler.
5. **Asynchronous Error Catching:**
   - Wrap async route handlers using an `asyncHandler` wrapper function or use `express-async-errors` so rejected promises reach the error middleware via `next(err)`.

---

### B. REST API Design Best Practices

1. **Resource-Oriented Naming:**
   - Use plural nouns for resource paths (e.g., `/api/quizzes`, `/api/attempts`).
   - Use sub-resources for nested entities (e.g., `/api/quizzes/:id/start`).
2. **Proper HTTP Method Usage:**
   - `GET`: Read resources (idempotent, no side effects).
   - `POST`: Create a new resource or initiate an action (e.g., submit attempt).
   - `PUT` / `PATCH`: Update resources.
   - `DELETE`: Remove resources.
3. **Standard HTTP Status Codes:**
   - `200 OK`: Successful retrieval or update.
   - `201 Created`: Successful creation (e.g. user registration, starting an attempt).
   - `400 Bad Request`: Validation failure or missing fields.
   - `401 Unauthorized`: Missing or invalid JWT token.
   - `403 Forbidden`: Authenticated user lacks permission.
   - `404 Not Found`: Resource does not exist.
   - `500 Internal Server Error`: Unhandled server exception.
4. **Consistent JSON Response Envelopes:**
   Always return a predictable JSON payload format across all endpoints:

   **Success Response:**
   ```json
   {
     "success": true,
     "data": { ... }
   }
   ```

   **Error Response:**
   ```json
   {
     "success": false,
     "error": {
       "code": "EXPIRED_TIMER",
       "message": "Quiz submission expired (timer exceeded)"
     }
   }
   ```

---

## 9. Required Unit Testing Coverage (Jest)

Trainees must submit unit tests co-located inside each module (`modules/<feature>/`):

1. **`ScoreCalculator` Utility (`modules/quiz/utils/score-calculator.spec.ts`):**
   - Single choice correct vs incorrect.
   - Multi-choice full match vs partial selection vs extra incorrect selection.
2. **`AuthService` Unit Tests (`modules/auth/auth.service.spec.ts`):**
   - Duplicate email registration attempt.
   - Password hashing verification.
   - Invalid credentials login attempt.
   - Successful login token generation.
3. **`QuizService` Unit Tests (`modules/quiz/tests/quiz.service.spec.ts`):**
   - Server-side timer expiry test.
   - Score calculation and pass/fail determination.
   - Attempt submission idempotency (prevent double submission).
4. **`AuthController` & `QuizController` Unit Tests:**
   - Status 200/201 response formatting.
   - Express `next(err)` error handling delegation.

---

## 10. Step-by-Step Trainee Execution Roadmap

1. **Phase 1: Setup, Figma Analysis & Design**
   - Initialize project repository with Express, TypeScript, Mongoose, and Jest config.
   - Analyze the [Figma UI Design](https://www.figma.com/design/jJl1SNjeasOAF0WSlK8epD/Exam-Online-Elevate?node-id=0-1&p=f&t=ugdI6dG6TNAfpT7k-0).
   - **Extract and Document API Endpoints Table** (methods, paths, request/response formats) and get mentor approval.
   - **Design Database Schemas** and get mentor approval before coding models.
2. **Phase 2: Auth Module (`modules/auth`)**
   - Build User model, AuthService, AuthController, AuthRoutes.
   - Write unit tests for `AuthService` and `AuthController`.
3. **Phase 3: Quiz Management & Attempt Engine (`modules/quiz`)**
   - Build Quiz and QuizAttempt models.
   - Implement `ScoreCalculator` utility with 100% unit test coverage.
   - Implement `QuizService` with timer validation and score calculation.
   - Write unit tests for `QuizService`.
4. **Phase 4: Dashboard Insights & Results Review (`modules/dashboard`)**
   - Implement MongoDB aggregate query for dashboard insights.
   - Implement quiz result review endpoint returning correct answers vs user answers.
   - Write unit tests for dashboard service.
