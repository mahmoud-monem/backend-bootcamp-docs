# Team E — Code Review Feedback

**Reviewed:** `online-exam/team-e` · 24 TypeScript files, **0 test files**
**Standards applied:** [`clean-code.md`](../../standards/clean-code.md) · [`typescript.md`](../../standards/typescript.md) · [`testing-jest.md`](../../standards/testing-jest.md) · [Project Requirements](../requirements.md)

---

## 1. Summary

The smallest codebase, and it contains the single best piece of domain code in the cohort:
`score-calculator.ts` implements requirement §7.1 exactly — single-choice, multi-choice
exact match, and per-question `points` weighting — and the timer check in `submitAttempt`
matches §7.2 line for line, grace period included. The quiz service also protects a real
invariant that nobody else thought about: questions cannot be edited once attempts exist.

Three things block it. **There are no tests at all** — Jest is not installed, there is no
config, and no `test` script — which makes the project's primary stated goal (§1
*"Comprehensive Unit Testing"*) unmet. **Four admin routes have no authentication**, and one
of them returns the correct answers for every quiz. And the error handler returns the full
error object, including the stack, to the client.

The good news: the hard part is done and done well. The remaining work is mostly wiring.

| Area | Verdict |
| --- | --- |
| Architecture & layering | Good — DI'd service, thin controllers, clear module split |
| Type safety | Strong config (`strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`) |
| Requirements coverage | Scoring and timer are the best in the cohort; dashboard module absent |
| Testing | **None** |
| Security | Unauthenticated admin routes; error handler leaks internals |

---

## 2. Requirements coverage

| Requirement | Status | Evidence |
| --- | --- | --- |
| M1 Register / Login / Forgot / Reset | ⚠️ | `modules/auth/` — sign-in/sign-up present; reset flow partial (3 files reference it) |
| M2 Dashboard: passed, fastest, correct | ❌ | no dashboard module; no code computes any of the three statistics |
| M2 Quiz catalog | ✅ | `modules/quiz/quiz.service.ts:63-77` |
| M3 Start attempt, answers hidden | ✅ | `quiz.service.ts:109-114` — maps options to `{id, text}` only |
| M3 Server-side timer + 10s grace | ✅ | `quiz.service.ts:132-150` — matches §7.2 exactly, and persists the expired attempt with score 0 |
| M3 Single/multi scoring | ✅ | `utils/score-calculator.ts` — exact set match, `question.points` awarded |
| M4 Result summary + review | ✅ | `quiz.service.ts:170-190` (`getResult`, `getReview`) |
| §8.A.2 Env validated at startup | ❌ | `config/env.variables.ts` defaults every secret to `""` (§3, CRIT-2) |
| §8.A.3 AppError + global handler | ⚠️ | `CustomError` exists; the handler leaks internals (§3, CRIT-3) |
| §8.A.4 Validation middleware | ⚠️ | `common/middlewares/validatation.middleware.ts` exists but is applied to no quiz route (§3, MAJ-1) |
| §8.A.5 Async errors reach `next` | ⚠️ | controllers `try/catch → next`, but rewrap every error as 500 (§3, CRIT-4) |
| §8.B.4 Response envelope | ⚠️ | success uses `{success, data}` ✅; errors return `{msg, cause, stack, err}` ❌ |
| §9.1 `ScoreCalculator` util + spec | ⚠️ | util is the best in the cohort; **no spec** |
| §9.2 AuthService tests | ❌ | none |
| §9.3 QuizService tests | ❌ | none |
| §9.4 Controller tests | ❌ | none |

---

## 3. Findings

### Critical

**CRIT-1 — `modules/quiz/quiz.routes.ts:11-19` — CC-207 (critical): four admin routes have no
authentication middleware, and one of them returns every quiz's correct answers.**

```ts
quizRouter.post("/admin", isAuth, QuizController.createQuiz.bind(QuizController));   // guarded
quizRouter.get("/admin", QuizController.listQuizzesForAdmin.bind(QuizController));        // open
quizRouter.get("/admin/:id", QuizController.getQuizForAdmin.bind(QuizController));        // open
quizRouter.put("/admin/:id", QuizController.updateQuiz.bind(QuizController));             // open
quizRouter.delete("/admin/:id", QuizController.deleteQuiz.bind(QuizController));          // open
```

`getQuizForAdmin` returns the raw quiz document — your own comment at
`quiz.service.ts:29` says *"includes `isCorrect` on every option, deliberately"*. Any
unauthenticated caller can `GET /api/quiz/admin` and read the answer key for every quiz,
then `PUT` or `DELETE` them. This defeats the entire timed-quiz mechanism.

There is also no role check anywhere — `isAuth` proves *who* you are, not that you are an
admin. Add both:

```ts
const adminOnly = [isAuth, authorize(Role.Admin)];
quizRouter.get("/admin", ...adminOnly, QuizController.listQuizzesForAdmin.bind(QuizController));
quizRouter.get("/admin/:id", ...adminOnly, ...);
quizRouter.put("/admin/:id", ...adminOnly, ...);
quizRouter.delete("/admin/:id", ...adminOnly, ...);
```

Applying the guard at the router level (`quizRouter.use("/admin", ...adminOnly)`) is safer
still — a new admin route then cannot be added unprotected by accident.

**CRIT-2 — `config/env.variables.ts:12-27` — TS-504, CC-702 (critical): every secret defaults
to the empty string.**

```ts
export const USER_ACCESS_SECRET_KEY = process.env.USER_ACCESS_SECRET_KEY || "";
export const JWT_SECRET = process.env.JWT_SECRET || "";
export const ENCRYPTION_SECRET = process.env.ENCRYPTION_SECRET || "";
```

With any of these unset the app starts normally and signs tokens with `""` — which means
tokens can be forged by anyone who guesses the (empty) secret. `|| ""` converts a loud
startup failure into a silent authentication bypass. Requirement §8.A.2 asks for the
opposite behaviour: fail fast at startup.

Two further problems in the same file:

```ts
export const NODE_ENV = process.env.NODE_ENV || "prod";       // :4  — reads env BEFORE dotenv loads it
env.config({ path: path.resolve(`./.env.dev`) });             // :5  — always loads .env.dev
```

`NODE_ENV` is read on line 4, one line *before* `dotenv` populates `process.env`, so it can
never come from the `.env` file. And the path is hard-coded to `.env.dev` regardless of
environment, so a production deploy loads development configuration.

Fix — one validated module:

```ts
import { config } from "dotenv";
import { z } from "zod";

config({ path: process.env.ENV_FILE ?? ".env" });

export const env = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().default(3000),
  MONGO_URI: z.string().url(),
  JWT_SECRET: z.string().min(32),
  USER_ACCESS_SECRET_KEY: z.string().min(32),
}).parse(process.env);
```

You already depend on `zod`.

**CRIT-3 — `common/middlewares/globalErr.middleware.ts:12-15` — CC-405 (critical): the error
handler serialises the entire error object to the client.**

```ts
res
  .status(err.statusCode || 400)
  .json({ msg: err.message, cause: err.cause, stack: err.stack, err });
```

The stack trace exposes absolute file paths and dependency versions; `err` may serialise a
Mongoose validation error carrying schema field names, or a driver error carrying part of
the connection string. Requirement §8.A.3 forbids this explicitly. The default status is
also wrong — an unclassified error is 500, not 400 (§8.B.3).

```ts
export default function globalErrHandler(err: unknown, req: Request, res: Response, _next: NextFunction) {
  const isKnown = err instanceof CustomError;
  const status = isKnown ? err.statusCode : 500;

  if (!isKnown) logger.error({ err, path: req.originalUrl }, "unhandled error");

  res.status(status).json({
    success: false,
    error: {
      code: isKnown ? err.code : "INTERNAL_ERROR",
      message: isKnown ? err.message : "Internal server error",
    },
  });
}
```

That shape also satisfies the §8.B.4 error envelope, which is currently unmet.

**CRIT-4 — `modules/auth/auth.controlers.ts:30-32` (and the other handlers in the file) —
CC-406, TS-102 (critical): every service error is rewrapped as a 500.**

```ts
} catch (error: any) {
  next(new CustomError(error.message, 500));
}
```

A `CustomError("Quiz not found", 404)` thrown by the service arrives at the client as a
**500** carrying the 404's message. The status code your service carefully chose is
discarded, and a genuine internal failure is indistinguishable from a validation error.

```ts
} catch (error: unknown) {
  next(error);                 // the global handler already knows how to map CustomError
}
```

**CRIT-5 — no tests exist — requirement §9 (critical).**

`package.json` has no `test` script, `devDependencies` contains no `jest`, `ts-jest`, or
`@types/jest`, and there is no `jest.config.*`. Requirement §9 lists four mandatory test
suites, and §1.5 names testing as one of the two primary goals of the project.

Start with the file that needs it least and pays back most —
`modules/quiz/utils/score-calculator.ts` is pure, dependency-free, and exercises the two
business rules in §7.1. Setup is three commands and a nine-line config:

```bash
npm i -D jest ts-jest @types/jest
npx ts-jest config:init
```

```ts
// jest.config.ts
export default {
  preset: "ts-jest/presets/default-esm",
  testEnvironment: "node",
  extensionsToTreatAsEsm: [".ts"],
  moduleNameMapper: { "^(\\.{1,2}/.*)\\.js$": "$1" },
  testMatch: ["**/*.spec.ts"],
};
```

Then the required §9.1 cases, which your implementation already handles correctly:

```ts
describe("scoreQuestion", () => {
  it("awards full points when the single correct option is selected", ...);
  it("awards zero when the wrong single option is selected", ...);
  it("awards full points when a multi-choice selection matches exactly", ...);
  it("awards zero for a partial multi-choice selection", ...);
  it("awards zero when a correct selection includes an extra incorrect option", ...);
});
```

After that: `AuthService` (§9.2), `QuizService` timer expiry and double-submit (§9.3), and
the two controller suites (§9.4).

### Major

**MAJ-1 — `modules/quiz/quiz.routes.ts` — CC-206 (major): no route validates its input.**

`validatation.middleware.ts` exists and `quiz.dto.ts` defines Zod schemas, but no quiz route
applies them. `submitAttempt` takes `req.body.answers` — an array of ids that flows into
`new Types.ObjectId(...)` at `quiz.service.ts:137-138`. A malformed id throws a `CastError`
that surfaces as an unhandled 500 rather than a 400. Bind the schemas you already wrote:

```ts
quizRouter.post("/attempts/:attemptId/submit", isAuth, validate(submitAttemptSchema), ...);
```

**MAJ-2 — no dashboard module — requirement §4 Module 2 (major).**

Nothing in the project computes quizzes passed, fastest completion time, or total correct
answers. The data needed is already on `quizAttempt` (`passed`, `correctCount`,
`startTime`, `submittedAt`), so this is one aggregation:

```ts
QuizAttemptModel.aggregate([
  { $match: { userId, status: "submitted" } },
  { $group: {
      _id: null,
      quizzesPassed: { $sum: { $cond: ["$passed", 1, 0] } },
      totalCorrectAnswers: { $sum: "$correctCount" },
      fastestCompletionTime: { $min: { $cond: ["$passed",
        { $divide: [{ $subtract: ["$submittedAt", "$startTime"] }, 1000] }, null] } },
  } },
]);
```

**MAJ-3 — `modules/quiz/quiz.service.ts:93-116` — requirement M3 (major): `startQuiz` allows
unlimited concurrent attempts on the same quiz.**

Every call creates a new `in_progress` attempt. A student can open ten attempts, see the
questions, and submit whichever one has the most time left. Guard it:

```ts
const active = await this._quizAttemptModel.findOne({ userId, quizId, status: "in_progress" });
if (active) throw new CustomError("You already have an attempt in progress", 409);
```

**MAJ-4 — `modules/quiz/quiz.service.ts:93-102` — CC-205 (major): the attempt stores no
duration snapshot.**

`submitAttempt` reads `quiz.durationMinutes` at submit time (`:133`). If an admin shortens
the quiz while a student is mid-attempt, the student's deadline moves backwards. Snapshot
`expiresAt` (or `durationMinutes`) onto the attempt at start, and validate against that.
The edit-lock at `:36-43` reduces the exposure but does not close it — `durationMinutes` is
not part of `input.questions`, so it can still be changed.

### Minor

- **`common/middlewares/validatation.middleware.ts` — CC-002 (minor):** filename typo.
- **`modules/auth/auth.controlers.ts` — CC-002 (minor):** "controlers"; also the only module
  using plural file names (`auth.services.ts`, `auth.models.ts`) while `quiz/` uses singular.
  Pick one (`CC-006`).
- **`modules/auth/auth.controlers.ts:30` — TS-102 (minor):** `catch (error: any)`. Use
  `unknown`; with CRIT-4's fix you do not touch the value at all.
- **`package.json` — TS-505 (minor):** `@types/jsonwebtoken` and `@types/nodemailer` are in
  `dependencies`. Type packages belong in `devDependencies` — they ship to production otherwise.
- **`package.json` — CC-006 (minor):** no `dev`, `build`, or `test` scripts; `start` runs
  `tsc --watch` alongside `node --watch`, which is a development command named `start`.
- **`modules/quiz/quiz.service.ts:4` — CC-601 (minor):** commented-out import left in place.
- **`modules/quiz/quiz.service.ts:16-19` — CC-801 (minor):** `createQuiz` returns the full
  document including `isCorrect`. Deliberate for an admin response per your comment — make it
  explicit in the type (`AdminQuizResponse`) so it cannot be reused on a student route by
  accident.
- **`app.ts:12` — CC-006 (minor):** `AuthRoute` is mounted at the app root while `quizRouter`
  is mounted at `/api/quiz`, so auth endpoints sit outside `/api`. Mount both under `/api`.

---

## 3b. REST API findings (`RS-###`)

Reviewed against [`restful-api.md`](../../standards/restful-api.md).

**RS-501 + RS-502 (critical) — `modules/quiz/quiz.routes.ts:11-19`: four admin routes with no
guard, and no role check anywhere in the project.** Full detail in CRIT-1. In REST terms this
is two separate rules: RS-501 (a protected route with no auth middleware) and RS-502 (`isAuth`
alone on operations that require an admin role). The structural fix is to guard the subtree so
a new admin route cannot be added unprotected:

```ts
const adminOnly = [isAuth, authorize(Role.Admin)];
quizRouter.use('/admin', ...adminOnly);
```

**RS-505 (major) — `app.ts`: no `helmet`, no CORS configuration at all, no rate limiting.**
`cors` is not even a dependency, so a browser client cannot call this API cross-origin, and
`/auth` login has no attempt limit. Add all three.

**RS-306 (minor) — `app.ts:10`: `express.json()` with no size limit.**

**RS-006 (minor) — `app.ts:12-16`: auth routes are mounted at the app root.**

```ts
app.use("/api/quiz", quizRouter);
app.use(AuthRoute);                 // endpoints land on /signin, /signup — outside /api

// Good
app.use('/api/v1/quizzes', quizRouter);
app.use('/api/v1/auth', AuthRoute);
```

**RS-002 + RS-601 (minor) — `app.ts:14`: singular collection, no version.** `/api/quiz` →
`/api/v1/quizzes`.

**RS-004 (minor) — `modules/quiz/quiz.routes.ts:22-31`: mixed resource modelling on one router.**

```ts
quizRouter.post("/:id/start", ...);                       // action sub-path
quizRouter.post("/attempts/:attemptId/submit", ...);      // attempts is a sibling, not a child
quizRouter.get("/attempts/:attemptId/result", ...);
quizRouter.get("/attempts/:attemptId/review", ...);
```

`/api/quiz/attempts/...` reads as "the attempts of the quiz collection". Attempts are a
first-class resource — give them their own router:

```ts
app.use('/api/v1/quizzes', quizRouter);      // POST /:id/attempts
app.use('/api/v1/attempts', attemptRouter);  // POST /:id/submission, GET /:id, GET /:id/review
```

Note `GET /:id` is registered at `:24` **before** `/attempts/...` at `:26-31`. Express matches
in order, so `GET /api/quiz/attempts/123/result` — a three-segment path — does not collide
today, but `GET /api/quiz/attempts` would bind `id="attempts"`. Splitting the routers removes
the hazard entirely.

**RS-403 (critical) — `common/middlewares/globalErr.middleware.ts:12-15`: `stack`, `cause`, and
the whole error object in the response body.** Same as CRIT-3.

**RS-402 (major) — same file: no `success` flag and no `error.code` on failures.** Success
responses already use `{success: true, data}`; errors use `{msg, cause, stack, err}`. Bring the
error side into the same contract.

**RS-301 (critical) — `modules/quiz/quiz.routes.ts`: no route binds a validation schema.**
Same as MAJ-1 — the Zod schemas in `quiz.dto.ts` exist but are never applied.

**RS-203 (major) — `modules/auth/auth.controlers.ts:30-32`: every service error becomes a 500.**
Same as CRIT-4. A 404 from the service reaches the client as a 500 carrying the 404's message.

**Clean:** success envelope is `{success: true, data}` — matching the brief (RS-401) and the
only team to get the success side exactly right. Ownership checked on every attempt operation
(`quiz.service.ts:121-123`, `:173-175`, RS-503). Correct answers stripped from the student
start payload (`:109-114`, RS-405) — the leak is via the unguarded admin route, not the student
one. Duration fields carry their unit (`durationMinutes`, `timeSpentSeconds`, RS-604).

---

## 4. Done well — keep doing this

- **`modules/quiz/utils/score-calculator.ts`** — the reference implementation of §7.1 for this
  cohort. Single-choice checks length **and** identity; multi-choice requires an exact set
  match; `question.points` is awarded and the percentage is `earnedPoints / totalPoints`, so
  weighted questions work. It is pure, has no dependencies, and takes an explicit
  `SubmittedAnswer[]` rather than a Mongoose document. Write its spec first — it will pass on
  the first run.
- **`quiz.service.ts:131-150`** — the timer check matches §7.2 exactly, *and* it persists the
  expired attempt with `score 0`, `passed false`, and the submitted answers before throwing.
  Most implementations throw and lose the submission; yours keeps the audit trail.
- **`quiz.service.ts:36-43` and `:51-54`** — questions cannot be edited and a quiz cannot be
  deleted once attempts exist. That is a domain invariant nobody else protected, and it is
  what makes grading historically consistent.
- **Ownership checks on every attempt operation** (`:121-123`, `:173-175`) — `submitAttempt`,
  `getResult`, and `getReview` all verify `attempt.userId === userId` before returning data.
- **`QuizService` takes its models by constructor injection** (`:11-14`) — which means the
  entire service is unit-testable with two fake models and no database. Use that.
- **`tsconfig.json`** enables `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, and
  `verbatimModuleSyntax` — stricter than three of the other teams.

---

## 5. Priority order

1. **CRIT-1** — put auth + a role guard on the four open admin routes. This is a live answer
   leak.
2. **CRIT-3** — stop returning `err` and `err.stack`; default to 500.
3. **CRIT-2** — validated env module; delete every `|| ""` on a secret and fix the
   `NODE_ENV`/dotenv ordering.
4. **CRIT-4** — `next(error)` instead of rewrapping as 500.
5. **CRIT-5** — install Jest and write `score-calculator.spec.ts` (§9.1). Then `AuthService`,
   `QuizService`, and the controllers.
6. **MAJ-1** bind the validation schemas, **MAJ-3** single active attempt, **MAJ-4** snapshot
   the deadline.
7. **MAJ-2** dashboard module.
