# Team D — Code Review Feedback

**Reviewed:** `online-exam/team-d` · 68 TypeScript files, 14 spec files (159 tests)
**Standards applied:** [`clean-code.md`](../../standards/clean-code.md) · [`typescript.md`](../../standards/typescript.md) · [`testing-jest.md`](../../standards/testing-jest.md) · [Project Requirements](../requirements.md)

---

## 1. Summary

By far the most complete test suite of the five teams — 14 spec files including six
controller specs, which is the only project that satisfies requirement §9.4. The dashboard
aggregation is the best implementation of Module 2 in the cohort. ESLint, Prettier, a
`type-check` script, and path aliases are all configured.

The gap is the timed quiz engine. **There is no server-side timer validation anywhere** —
the attempt model stores `startedAt` but nothing ever compares it to the quiz duration, so
requirement §7.2, one of the two named business rules in the brief, is unimplemented. The
other significant issues are the password hash in auth responses and `process.env.JWT_SECRET!`
scattered across six files.

| Area | Verdict |
| --- | --- |
| Architecture & layering | Strong — clear modules, strategies pattern, asyncWrap, allowedTo |
| Type safety | Good structure, weakened by `JWT_SECRET!` and `useUnknownInCatchVariables: false` |
| Requirements coverage | Timer rule entirely missing; everything else present |
| Testing | Strongest in the cohort — 159 tests, controllers included |
| Security | Password hash returned on sign-in/sign-up |

---

## 2. Requirements coverage

| Requirement | Status | Evidence |
| --- | --- | --- |
| M1 Register / Login / Forgot / Reset | ✅ | `modules/auth/auth.service.ts` |
| M2 Dashboard: passed, fastest, correct | ✅ | `modules/dashboard/dashboard.service.ts:9-57` — single aggregation, fastest gated on `isPassed`, seconds |
| M2 Quiz catalog | ✅ | `dashboard.service.ts:60-67` |
| M3 Start attempt, answers hidden | ⚠️ | `modules/quiz/quiz.service.ts:31-56` — depends on `getRandomQuestionsByCourseId` projection (§3, MAJ-3) |
| M3 Server-side timer + 10s grace | ❌ | not implemented (§3, CRIT-1) |
| M3 Single/multi scoring | ⚠️ | `quizAttempt.helper.ts:30-32` exact match ✅, but compares option **text** and accepts duplicates (§3, MAJ-2) |
| M4 Result summary + review | ✅ | `quizAttempt.service.ts:48-78` |
| §8.A.2 Env validated at startup | ❌ | only `DB_URI` is checked (`config/db.ts:4-9`); `JWT_SECRET` is asserted with `!` in 6 places (§3, CRIT-3) |
| §8.A.3 AppError + global handler | ✅ | `common/errors/appError.ts` + `app.ts:39-46`, stack gated on `NODE_ENV` |
| §8.A.4 Validation middleware | ⚠️ | `common/middlewares/validate.ts` validates `req.body` only (§3, MAJ-4) |
| §8.A.5 Async errors reach `next` | ✅ | `common/middlewares/asyncWrap.ts` applied consistently |
| §8.B.4 Response envelope | ⚠️ | `{status, message, data}` — consistent, but not the specified `{success, data}` / `{success, error:{code, message}}` |
| §9.1 `ScoreCalculator` util + spec | ⚠️ | `quizAttempt.helper.ts` + `quizAttempt.helper.spec.ts` (6 tests) — right idea, different name and location |
| §9.2 AuthService tests | ✅ | 16 tests |
| §9.3 QuizService tests | ⚠️ | 14 + 10 tests, but no timer-expiry test (nothing to test) |
| §9.4 Controller tests | ✅ | 6 controller specs — only team to meet this |

---

## 3. Findings

### Critical

**CRIT-1 — `modules/quizAttempt/quizAttempt.service.ts:12-56` — requirement §7.2 (critical):
no server-side timer validation exists anywhere in the project.**

`submitQuiz` checks attempt ownership, `IN_PROGRESS` status, and question membership, then
grades and saves. There is no comparison of `Date.now()` against `attempt.startedAt +
quiz.duration`. A search for `expire`, `duration`, `timeLimit`, or `grace` across
`modules/quizAttempt/` returns only the model field and a `populate` string.

The brief calls this out as one of two explicit business rules (§7.2) and repeats it as a
design principle (§1.4 "Never Trust the Client" — *"Enforce all timing ... logic strictly on
the server side"*).

```ts
const quiz = await quizSercive.validateQuizExists(attempt.quizId.toString());

const maxAllowedTimeMs = attempt.startedAt.getTime() + (quiz.duration * 60 + 10) * 1000;
if (Date.now() > maxAllowedTimeMs) {
  attempt.status = "COMPLETED";
  attempt.score = 0;
  attempt.isPassed = false;
  attempt.completedAt = new Date();
  await attempt.save();
  throw new AppError("Quiz submission expired (timer exceeded)", statusCode.BadRequest);
}
```

Note you already load `quiz` at `:37` — move that call above the grading block and the check
is five lines. Also consider snapshotting `expiresAt` onto the attempt at start time, so
editing a quiz's duration mid-attempt cannot change a running deadline.

Then add the test requirement §9.3 asks for: *"Server-side timer expiry test."*

**CRIT-2 — `modules/auth/auth.service.ts:44-47, 62-65` — CC-801 (critical): sign-up and sign-in
return the full Mongoose user document, including the password hash.**

```ts
const user = await User.findOne({ email }).select('+password -__v');
...
return { token, user }        // password hash goes to the client
```

`signIn` explicitly re-selects the hidden `password` field for comparison and then returns
the same document. `signUp` (`:30-47`) returns `newUser` straight from `User.create`.

Fix — map to a response shape, and never let the persistence document reach the controller:

```ts
const toUserResponse = (user: IUser) => ({
  id: user._id, firstName: user.firstName, lastName: user.lastName,
  email: user.email, role: user.role,
});
return { token, user: toUserResponse(user) };
```

**CRIT-3 — `process.env.JWT_SECRET!` in 6 locations — TS-504, CC-701 (critical).**

`common/utils/token.util.ts:29,36` · `common/middlewares/authenticate.ts:18` ·
`modules/auth/auth.service.ts:37,61,166`

```ts
return jwt.sign({ email, jti }, process.env.JWT_SECRET!, { ... });
```

The `!` asserts a value the runtime cannot guarantee. With `JWT_SECRET` unset, `jwt.sign`
receives `undefined` and throws at the first login — in production, not at boot. Requirement
§8.A.2 asks for startup validation of `PORT`, `MONGO_URI`, and `JWT_SECRET`.

`config/db.ts:4-9` already does this correctly for `DB_URI`. Generalise it:

```ts
// src/config/env.ts
const envSchema = Joi.object({
  NODE_ENV: Joi.string().valid("development", "test", "production").required(),
  PORT: Joi.number().default(3000),
  DB_URI: Joi.string().uri().required(),
  JWT_SECRET: Joi.string().min(32).required(),
}).unknown(true);

const { value, error } = envSchema.validate(process.env);
if (error) throw new Error(`Invalid environment: ${error.message}`);
export const env = value;
```

Then `process.env` appears in exactly one file, and `!` disappears from all six call sites.

### Major

**MAJ-1 — `modules/quiz/quiz.service.ts:31-70` — CC-504 (major): the attempt is created before
the "no questions" check, leaving an orphan row.**

```ts
const attempt = await QuizAttempt.create({ ...questions.map(...), status: "IN_PROGRESS" });

if (!questions || questions.length === 0) {
  throw new AppError("No questions found for this quiz", statusCode.NotFound)
}
```

When a course has no questions the guard fires *after* the write, so an `IN_PROGRESS`
attempt with an empty answer array is left in the database. Move the guard above
`QuizAttempt.create` — it costs nothing and removes the partial-write path.

**MAJ-2 — `modules/quizAttempt/quizAttempt.helper.ts:18-34` — requirement §7.1 (major): grading
compares option **text**, and duplicate selections score as correct.**

```ts
const studentOptions = (Array.isArray(...) ? ... : [...]).map((opt) => String(opt).trim().toLowerCase());
const correctAnswers  = (Array.isArray(...) ? ... : [...]).map((ans) => String(ans).trim().toLowerCase());

const isCorrect =
  studentOptions.length === correctAnswers.length &&
  studentOptions.every((opt) => correctAnswers.includes(opt));
```

Two defects:

1. **Duplicates pass.** For correct options `["a", "b"]`, a student submitting `["a", "a"]`
   satisfies both conditions — length 2 === 2, and every element is in the correct list. Use
   sets: `new Set(studentOptions).size === correctAnswers.length && ...`.
2. **Text-based matching.** Requirement §5 models options as *"(ID & text)"* with *"correct
   option IDs"*. Matching on normalised text means fixing a typo in an option retroactively
   changes historical grades, and two options with the same text are indistinguishable.

The exact-set-match rule itself is right — the comparison key is wrong.

**MAJ-3 — `modules/quiz/quiz.service.ts:39` — requirement M3 (major): verify the start payload
cannot leak correct answers.**

`startQuiz` returns `questions` exactly as `QuestionService.getRandomQuestionsByCourseId`
produced them. Whether `correctAnswer` reaches the client depends entirely on that method's
projection. Requirement M3 is explicit: *"Return questions to the user without revealing the
correct options."* Do not rely on a projection two modules away — map explicitly at the
boundary:

```ts
questions: questions.map((q) => ({ id: q._id, text: q.text, options: q.options, type: q.type })),
```

and add a test asserting the response contains no `correctAnswer` key. Note the same object
is stored on the attempt (`:33-37`), which is correct — the server copy should keep them.

**MAJ-4 — `common/middlewares/validate.ts:7` — CC-206 (major): only `req.body` is validated.**

```ts
const { error } = schema.validate(req.body, { abortEarly: false });
```

`req.params` and `req.query` reach controllers unchecked — including `attemptId` and
`quizId`, which are passed straight into `Types.ObjectId(...)` and Mongoose filters. A
malformed id produces a `CastError` surfacing as a 500 rather than a 400.

```ts
export const validate = (schemas: { body?: ObjectSchema; params?: ObjectSchema; query?: ObjectSchema }) => ...
```

Also, this middleware writes its own response body (`:11-14`) instead of delegating to the
error handler, so validation failures use a different envelope than every other error
(`CC-402`). `next(new AppError(messages.join(", "), statusCode.BadRequest))` keeps it in one
place.

**MAJ-5 — `jest.config.ts:11` — JT-004 (major): `forceExit: true`.**

```ts
forceExit: true,
```

`forceExit` kills the worker while handles are still open — it hides a resource leak rather
than fixing it. One of your specs opens a real Mongoose connection; close it in `afterAll`
(`await mongoose.disconnect()`) and remove the flag. If the run then hangs, `--detectOpenHandles`
will name the culprit.

**MAJ-6 — 205 `as jest.Mock` casts across the spec files — JT-206 (major).**

```ts
(quizAttemptService.submitQuiz as jest.Mock).mockResolvedValue(mockResult);
```

`jest.mocked()` keeps the mock bound to the real signature, so a change to
`submitQuiz(userId, attemptId, dto)` fails at compile time instead of producing a test that
passes against a signature that no longer exists:

```ts
const attemptService = jest.mocked(quizAttemptService);
attemptService.submitQuiz.mockResolvedValue(mockResult);
```

This is a mechanical find-and-replace and it upgrades 159 tests at once.

**MAJ-7 — `tsconfig.json:11` — TS-102 (major): `"useUnknownInCatchVariables": false`.**

This turns every `catch (error)` binding back into `any`, project-wide. Removing the line
(the default is `true` under `strict`) surfaces the places where a caught value is used
without narrowing — which is exactly the check you want on error paths.

### Minor

- **`modules/quizAttempt/quizAttempt.service.ts:111` — TS-401 (minor):** `stats[0].totals[0]`
  double-indexes an aggregation result without guards. `noUncheckedIndexedAccess` is off in
  `tsconfig.json`, so TypeScript will not warn. Enable it (team A does) and add the guard.
- **`app.ts:39` — TS-101 (minor):** `(err: any, ...)` on the error middleware. Type it
  `unknown` and narrow with `err instanceof AppError`.
- **`app.ts:41-45` — CC-405 (minor):** unknown errors return `err.message` verbatim; a driver
  or Mongoose message can leak schema details. Return a fixed message for 5xx and log the
  original. The `NODE_ENV`-gated stack (`:44`) is handled correctly — keep that.
- **`modules/Question/` — CC-006 (minor):** capitalised directory beside `quiz/`, `auth/`,
  `course/`. On a case-sensitive CI filesystem an import of `../question/...` breaks.
- **`modules/quiz/quiz.service.ts:10` — CC-002 (minor):** `quizSercive` (typo), exported and
  imported under that name by `quizAttempt.service.ts:6`.
- **`modules/course/` — requirement §3 (minor):** `course` and `diploma` are not in the brief.
  Fine as an extension — mention the modelling decision to your mentor, since `startQuiz`
  now draws questions from a *course* rather than the quiz (`quiz.service.ts:39`), which
  changes the M3 contract.
- **`quizAttempt.helper.spec.ts:70` — JT-103 (minor):** *"should handle single non-array
  selectedOption gracefully"* — "gracefully" is not a behaviour. Name the expected outcome.

---

## 3b. REST API findings (`RS-###`)

Reviewed against [`restful-api.md`](../../standards/restful-api.md).

**RS-002 + RS-005 (major) — `app.ts:26-32`: singular collection mounts, one in camelCase.**

```ts
app.use("/api/v1/diploma", diplomaRouter);
app.use('/api/v1/course', courseRouter);
app.use('/api/v1/question', questionRouter);
app.use('/api/v1/quiz', quizRouter);
app.use('/api/v1/quizAttempt', quizAttemptRouter);      // camelCase in a URL

// Good
app.use('/api/v1/diplomas', diplomaRouter);
app.use('/api/v1/courses', courseRouter);
app.use('/api/v1/questions', questionRouter);
app.use('/api/v1/quizzes', quizRouter);
app.use('/api/v1/quiz-attempts', quizAttemptRouter);
```

`/quizAttempt` is the one that will bite: URL paths are case-sensitive, and a normalising proxy
or a client that lowercases the path gets a 404 that reproduces nowhere else.

**RS-001 + RS-005 (major) — `modules/auth/auth.routes.ts`: every auth path is a camelCase verb.**

| Now | Should be |
| --- | --- |
| `POST /api/v1/auth/signUp` | `POST /api/v1/auth/registrations` or `/auth/register` |
| `POST /api/v1/auth/signIn` | `POST /api/v1/auth/sessions` or `/auth/login` |
| `POST /api/v1/auth/forgetPassword` | `POST /api/v1/auth/password-reset-requests` |
| `POST /api/v1/auth/verifyCode` | `POST /api/v1/auth/password-reset-requests/verification` |
| `POST /api/v1/auth/resetPassword` | `POST /api/v1/auth/password-resets` |
| `POST /api/v1/auth/createAdmin` | `POST /api/v1/admins` |

Auth is the accepted exception to "no verbs" — `/login` and `/register` are idiomatic. The
non-negotiable part is the casing: use kebab-case, not camelCase.

**RS-004 (major) — `modules/quizAttempt/quizAttempt.routes.ts`: `POST /:attemptId` is
ambiguous.**

```ts
quizAttemptRouter.post("/:attemptId", ...)      // this is "submit"
quizAttemptRouter.get("/:attemptId", ...)       // this is "read"
```

`POST` to a member URL reads as "create something inside this attempt" without saying what.
Name the sub-resource:

```ts
quizAttemptRouter.post("/:attemptId/submission", ...)
```

Note also that `GET /latest` and `GET /statistics` are registered above `GET /:attemptId`,
which is the correct order — literal paths before parameterised ones. Keep that.

**RS-505 (major) — `app.ts:17-23`: `helmet` installed but never imported; unrestricted CORS;
no rate limiting.**

```ts
app.use(cors());               // reflects any origin
app.use(express.json());       // no size limit (RS-306)
```

`helmet` is in `dependencies`. Two lines fix both:

```ts
app.use(helmet());
app.use(cors({ origin: env.CLIENT_URL, credentials: true }));
app.use(express.json({ limit: '100kb' }));
app.use('/api/v1/auth', rateLimit({ windowMs: 15 * 60 * 1000, max: 20 }), authRouter);
```

**RS-402 (major) — `app.ts:41-45`: error body carries no `code` and no `success` flag.**

```ts
res.status(status).json({ status: err.status || "error", message: ..., ...(dev && { stack }) });
```

`status: "error"` is a string duplicating the HTTP status; the brief asks for
`{ success: false, error: { code, message } }`. Add a `code` to `AppError` and emit it.

**RS-302 (major) — `common/middlewares/validate.ts:7`: params and query never validated.**
Same as MAJ-4. `:attemptId` and `:quizId` flow straight into `Types.ObjectId(...)`.

**Clean:** **`/api/v1` versioning — the only team that did it** (RS-601). Literal routes ordered
before parameterised ones. `201` used consistently on creation (12 call sites, RS-201). Named
status constants via `statusCode.*` rather than magic numbers (RS-206). `allowedTo(...)` role
guard composed at the router level on admin operations (RS-502). Ownership enforced in the
query filter (`quizAttempt.service.ts:13-17`, `:67`, RS-503). Stack trace gated on
`NODE_ENV === "development"` (RS-403). A JSON catch-all 404 via
`app.all("/{*splat}", ...)` (RS-104).

---

## 4. Done well — keep doing this

- **The test suite.** 159 tests over 14 files, including six controller specs — the only team
  meeting requirement §9.4. The controller specs
  (`quizAttempt.controller.spec.ts:10-31`) build partial `req`/`res` doubles with
  `mockReturnThis()` and assert on both the service call and the response body. That is the
  right shape for a controller test.
- **`jest.config.ts:12-14`** enables `clearMocks`, `resetMocks`, and `restoreMocks` — no test
  can inherit another's mock state. Exactly what `JT-005` asks for.
- **`modules/dashboard/dashboard.service.ts:9-57`** — one aggregation returning all three
  required statistics, with `fastestCompletionTime` correctly gated on `$isPassed`, expressed
  in seconds, and `Infinity` normalised to `null` for the no-attempts case. Best Module 2
  implementation in the cohort.
- **Question strategies** (`modules/Question/strategies/`) — a factory plus an interface per
  question type is real polymorphism where four other teams wrote an `if (type === "multi")`.
- **`allowedTo` + `authenticate` + `asyncWrap`** applied consistently at the router level, so
  authorization is declarative and auditable (`CC-207` satisfied).
- **Tooling**: ESLint, Prettier with import sorting, `type-check` script wired into
  `prebuild`, path aliases, `forceConsistentCasingInFileNames`. The best-configured project
  of the five.
- **`quizAttempt.service.ts:23-29`** — submitted answers are checked against the attempt's own
  question set before grading. Correct "never trust the client" control.

---

## 5. Priority order

1. **CRIT-1** — implement the timer check and its test. This is a named requirement and the
   core of the "timed quiz engine" module.
2. **CRIT-2** — stop returning the password hash from `signUp`/`signIn`.
3. **CRIT-3** — one env module, validated at boot; delete all six `JWT_SECRET!`.
4. **MAJ-2** — grade on option ids and de-duplicate selections.
5. **MAJ-3** — map the start-quiz payload explicitly and assert no `correctAnswer` leaks.
6. **MAJ-1** ordering fix, **MAJ-4** validate params/query.
7. **MAJ-6** `jest.mocked()` sweep, **MAJ-5** drop `forceExit`, **MAJ-7** restore
   `useUnknownInCatchVariables`.
