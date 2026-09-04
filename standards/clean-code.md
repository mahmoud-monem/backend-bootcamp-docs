# Clean Code Rule Catalog (`CC-###`)

Applies to production TypeScript under `src/**`, excluding `*.spec.ts` / `*.test.ts`
(those are governed by [`testing-jest.md`](./testing-jest.md)).

Read [`README.md`](./README.md) for severity meanings, the verify-before-report protocol,
and the false-positive rules.

**Bands**

| Band | Topic |
| --- | --- |
| `CC-0xx` | Naming |
| `CC-1xx` | Functions and function size |
| `CC-2xx` | Architecture, layering, dependency rule |
| `CC-3xx` | Duplication and abstraction |
| `CC-4xx` | Error handling |
| `CC-5xx` | Async correctness |
| `CC-6xx` | Comments, dead code, noise |
| `CC-7xx` | Configuration, constants, magic values |
| `CC-8xx` | Data and API boundary hygiene |

---

## CC-0xx — Naming

### CC-001 — Name does not reveal intent
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** Identifiers named `data`, `res` (outside an Express handler), `result`, `temp`,
`obj`, `arr`, `item`, `val`, `x`, `d`, `flag`, `info` where a domain word exists.
Grep hint: `\b(const|let)\s+(data|result|temp|obj|arr|val|info|flag)\b`

```ts
// Bad
const data = await this.quizRepository.findById(id);
const arr = data.questions.filter((x) => x.active);

// Good
const quiz = await this.quizRepository.findById(id);
const activeQuestions = quiz.questions.filter((question) => question.isActive);
```

**Why:** The reader should not have to trace an assignment chain to learn what a variable holds.
**Not a violation when:** The scope is 2–3 lines and the type is obvious at a glance
(`const [a, b] = pair`), or the name follows a framework contract (`req`, `res`, `next`, `err`).

---

### CC-002 — Abbreviated or misspelled identifiers
**Severity:** minor · **Applies to:** `src/**/*.ts`, directory and file names
**Detect:** Invented abbreviations (`usr`, `qz`, `cfg`, `resp`, `pwd`, `attmpt`) or typos
in a path or export name. Directory typos count — they propagate into every import.

```ts
// Bad
import { hashPwd } from "../../utils/secuirty/hash.js";
const usrRepo = new UsrRepository();

// Good
import { hashPassword } from "../../utils/security/hash.js";
const userRepository = new UserRepository();
```

**Why:** Abbreviations save no meaningful typing and break search; a misspelled folder is
permanent friction for everyone importing from it.
**Not a violation when:** The abbreviation is industry standard (`id`, `url`, `http`, `jwt`,
`dto`, `otp`, `api`, `db`).

---

### CC-003 — Type information encoded in the name
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** Hungarian-style suffixes/prefixes: `strName`, `userArray`, `quizObj`,
`IUser` for interfaces, `_privateField` on a `private` member.

```ts
// Bad
interface IQuiz { ... }
const quizArray: Quiz[] = [];

// Good
interface Quiz { ... }
const quizzes: Quiz[] = [];
```

**Why:** The type system already states the type; the name should state the meaning.
**Not a violation when:** The suffix is domain vocabulary, not type vocabulary
(`quizIds`, `startDate`, `attemptCount`), or it names a *layer role* rather than a type
kind — see the collision guidance below.

#### Resolving a `User` class vs `User` interface collision

A name collision means two concepts are competing for one word. Disambiguate by **role**,
never by type kind (`IUser`, `UserInterface`, `UserType`, `UserClass`).

| Concept | Name | Layer |
| --- | --- | --- |
| Domain entity with behaviour | `User` | domain |
| Mongoose persistence shape | `UserDocument` | infrastructure |
| Shape returned to a client | `UserResponse` | boundary |
| Shape accepted from a client | `CreateUserInput`, `UpdateUserDto` | boundary |

```ts
// Bad — kind-based disambiguation
interface IUser { id: string; email: string }
class User implements IUser { ... }

// Good — role-based disambiguation
export interface UserDocument { email: string; password: string; role: Role }   // infra
export class User { constructor(readonly id: string, readonly email: string) {} } // domain
export interface UserResponse { id: string; email: string; role: Role }          // boundary
```

Before renaming, check whether the second declaration should exist at all:

1. **Identical shapes** — do not declare both. Derive one (`type User = z.infer<typeof userSchema>`)
   or have the class implement the contract. Two hand-maintained declarations is `TS-202`.
2. **A class already produces a type.** `class User {}` emits a value *and* a `User` type.
   A separate `interface User` describing the same members is usually redundant — delete it.
3. **Deliberate declaration merging** — `interface User` merging into `class User` in the
   same scope to add members (Mongoose instance methods, mixins) is a language feature, not
   a collision. Not a violation.

Do not resolve a collision by import aliasing (`import { User as UserModel }`), which moves
the ambiguity to every call site and lets each file pick a different alias.

---

### CC-004 — Boolean not phrased as a predicate
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** Boolean variables, fields, or returns without an `is`/`has`/`can`/`should`/
`was` prefix.

```ts
// Bad
const active = quiz.publishedAt !== null;
if (user.verified && !user.deleted) { ... }

// Good
const isActive = quiz.publishedAt !== null;
if (user.isVerified && !user.isDeleted) { ... }
```

**Why:** A predicate name makes a condition readable as a sentence and prevents
`if (user.verification)` truthiness bugs.
**Not a violation when:** The field name is fixed by an external contract (a third-party
API payload, an existing DB column).

---

### CC-005 — Inconsistent vocabulary for one concept
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** Two or more words for the same domain concept inside one module or across
one layer: `attempt` / `submission` / `try`; `student` / `user` / `learner`;
`get` / `fetch` / `retrieve` / `load` for identical operations.

```ts
// Bad
quizRepository.getById(id);
questionRepository.fetchById(id);
attemptRepository.retrieveById(id);

// Good
quizRepository.findById(id);
questionRepository.findById(id);
attemptRepository.findById(id);
```

**Why:** Synonyms force the reader to prove two things are the same. One concept, one word.
**Not a violation when:** The words genuinely denote different concepts — document the
distinction at the type level (`Attempt` vs `Submission` as separate types).

---

### CC-006 — File or folder name disagrees with its contents
**Severity:** minor · **Applies to:** `src/**`
**Detect:** `*.controller.ts` exporting service logic; `utils/` holding domain rules;
casing that breaks the module's convention (`Question/` beside `quiz/`, `DataBase/` beside
`config/`); `*.controlers.ts`-style typos.

**Why:** Path conventions are how a reader navigates without grep. Inconsistent casing also
breaks builds on case-sensitive CI filesystems even when it works on macOS.
**Not a violation when:** The whole project consistently uses that convention.

---

## CC-1xx — Functions

### CC-101 — Function is too long to hold in one view
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** A function body over ~40 lines, or one that needs a scroll to read. Count
statements, not blank lines. Pair this with CC-102 — length is a symptom, mixed
responsibilities are the disease.

**Why:** Length is the cheapest available proxy for "this does more than one thing".
**Not a violation when:** The body is a flat, non-branching data mapping (a long object
literal, a Mongoose schema definition, a route table).

---

### CC-102 — Function does more than one thing
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** A single function that spans two or more distinct concerns: validation +
persistence + notification, or fetch + transform + respond. Reliable tell — you cannot
name it without "and", or it has section comments (`// hash password`, `// send email`).

```ts
// Bad
async register(req: Request, res: Response) {
  if (!req.body.email || !req.body.password) return res.status(400).json({ message: "bad" });
  const exists = await UserModel.findOne({ email: req.body.email });
  if (exists) return res.status(409).json({ message: "exists" });
  const hashed = await bcrypt.hash(req.body.password, 10);
  const user = await UserModel.create({ ...req.body, password: hashed });
  const otp = Math.floor(100000 + Math.random() * 900000).toString();
  user.confirmOTP = otp;
  await user.save();
  await transporter.sendMail({ to: user.email, subject: "Confirm", text: otp });
  const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET!);
  return res.status(201).json({ token, user });
}

// Good
// auth.service.ts
async register(dto: RegisterDto): Promise<AuthResult> {
  await this.assertEmailAvailable(dto.email);
  const user = await this.userRepository.create({
    ...dto,
    password: await this.passwordHasher.hash(dto.password),
  });
  await this.verificationService.sendConfirmationOtp(user);
  return { token: this.tokenService.issue(user), user: toUserResponse(user) };
}
```

**Why:** One reason to change per function. Anything else cannot be tested, reused, or
reasoned about in isolation.
**Not a violation when:** The function is an explicit orchestrator whose every step is a
named call at the same level of abstraction (as in the `Good` example).

---

### CC-103 — Mixed levels of abstraction in one function
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** High-level policy calls sitting next to low-level mechanics: a service method
that calls `this.gradeAttempt(...)` on one line and does `Math.round(raw * 100) / 100` with
a regex on the next.

```ts
// Bad
async submitAttempt(dto: SubmitDto) {
  const attempt = await this.attemptRepository.findById(dto.attemptId);
  let correct = 0;
  for (const answer of dto.answers) {
    const question = attempt.gradingSheet.find((q) => String(q.id) === String(answer.questionId));
    if (question && question.correctIds.sort().join() === answer.selected.sort().join()) correct++;
  }
  const score = Math.round((correct / attempt.gradingSheet.length) * 10000) / 100;
  await this.attemptRepository.finish(attempt.id, score);
}

// Good
async submitAttempt(dto: SubmitDto): Promise<AttemptResult> {
  const attempt = await this.attemptRepository.findById(dto.attemptId);
  const score = this.scoreCalculator.grade(attempt.gradingSheet, dto.answers);
  return this.attemptRepository.finish(attempt.id, score);
}
```

**Why:** A function should read as a single-altitude story. Descending into mechanics
mid-paragraph is what makes code unreadable.
**Not a violation when:** The function *is* the low-level unit (`scoreCalculator.grade`).

---

### CC-104 — Too many positional parameters
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** More than three positional parameters, or two-plus adjacent parameters of the
same type (call sites become order-dependent).

```ts
// Bad
createQuiz(title: string, description: string, diplomaId: string, duration: number, isPublished: boolean)
createQuiz("Intro", "Basics", "d1", 30, true);

// Good
createQuiz(input: CreateQuizInput)
createQuiz({ title: "Intro", description: "Basics", diplomaId, durationMinutes: 30, isPublished: true });
```

**Why:** Named fields make call sites self-documenting and immune to argument swaps that
the compiler cannot catch.
**Not a violation when:** The parameters are of distinct types and the order is
conventional (`(req, res, next)`, `(key, value)`).

---

### CC-105 — Boolean flag parameter that switches behaviour
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** A parameter whose only job is to pick between two code paths;
call sites read `doThing(id, true)`.

```ts
// Bad
async listQuizzes(userId: string, isAdmin: boolean) {
  return isAdmin ? this.repository.findAll() : this.repository.findPublished();
}

// Good
async listAllQuizzes(): Promise<Quiz[]> { return this.repository.findAll(); }
async listPublishedQuizzes(): Promise<Quiz[]> { return this.repository.findPublished(); }
```

**Why:** A flag parameter means the function does two things; the caller already knows
which one it wants.
**Not a violation when:** The flag is data passed through to storage
(`create({ isPublished })`), not a branch selector.

---

### CC-106 — Output parameter or hidden mutation of an argument
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** A function that mutates an object or array it received and relies on the caller
seeing the change, especially when it also returns something else.

```ts
// Bad
function applyDefaults(quiz: Quiz): void { quiz.durationMinutes ??= 30; }

// Good
function withDefaults(quiz: Quiz): Quiz { return { ...quiz, durationMinutes: quiz.durationMinutes ?? 30 }; }
```

**Why:** Invisible side effects on inputs break local reasoning and make ordering bugs
untraceable.
**Not a violation when:** Mutation is the documented purpose and the name says so
(`normalizeInPlace`), or it is a Mongoose document being deliberately mutated before `save()`.

---

### CC-107 — Deep nesting instead of guard clauses
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** Three or more nested `if`/`for`/`try` levels, or an `if` wrapping the whole body.

```ts
// Bad
async function confirm(email: string, otp: string) {
  const user = await userRepository.findByEmail(email);
  if (user) {
    if (!user.isVerified) {
      if (user.otp === otp) { ... }
      else { throw new BadRequestError("invalid otp"); }
    } else { throw new ConflictError("already verified"); }
  } else { throw new NotFoundError("user"); }
}

// Good
async function confirm(email: string, otp: string) {
  const user = await userRepository.findByEmail(email);
  if (!user) throw new NotFoundError("user");
  if (user.isVerified) throw new ConflictError("already verified");
  if (user.otp !== otp) throw new BadRequestError("invalid otp");
  ...
}
```

**Why:** Guard clauses put failure next to its cause and keep the happy path at one indent.
**Not a violation when:** The nesting is inherent (a nested loop over a genuine matrix).

---

### CC-108 — Function returns different shapes depending on the path
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** One function returning `null` in one branch, a domain object in another, and a
`{ message }` error envelope in a third.

```ts
// Bad
async function findQuiz(id: string) {
  const quiz = await QuizModel.findById(id);
  if (!quiz) return { error: "not found" };
  if (!quiz.isPublished) return null;
  return quiz;
}

// Good
async function findPublishedQuiz(id: string): Promise<Quiz> {
  const quiz = await this.repository.findById(id);
  if (!quiz || !quiz.isPublished) throw new NotFoundError(`quiz ${id}`);
  return quiz;
}
```

**Why:** Every caller has to re-implement the discrimination, and one of them will forget.
**Not a violation when:** The union is explicit in the return type and discriminated
(`Promise<Result<Quiz, QuizError>>`) — see `TS-204`.

---

### CC-109 — Dead parameter or unused injected dependency
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** Parameters never referenced in the body; constructor-injected services never
used. Grep hint: enable `noUnusedParameters`, then read the compiler output.
**Why:** A parameter nobody reads is a lie about what the function needs.
**Not a violation when:** The signature is fixed by a framework contract — Express error
middleware must declare four parameters. Prefix with `_` in that case.

---

### CC-110 — Class or module with too many responsibilities
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** A file over ~300 lines; a service that touches four or more unrelated
collections; a `utils.ts` that exports hashing, date maths, mail templating, and pagination.

```ts
// Bad — src/common/utils/index.ts exports hashPassword, signToken, sendMail, paginate, slugify

// Good
common/security/password-hasher.ts
common/security/token-service.ts
common/mail/mailer.ts
common/pagination/paginate.ts
```

**Why:** A grab-bag module makes everything depend on everything; one change forces the
whole graph to rebuild and retest.
**Not a violation when:** The file is a pure re-export barrel with no logic.

---

## CC-2xx — Architecture and layering

The dependency rule for this codebase:

```
routes  ->  controller  ->  service  ->  repository  ->  model (Mongoose)
                    \-> validation (schema)      \-> external adapters (mail, cache, storage)
```

Dependencies point one direction only. Nothing below the service layer may import from a
controller. Domain logic must not import Express or Mongoose types.

---

### CC-201 — Business logic in the controller
**Severity:** major · **Applies to:** `src/**/*.controller.ts`, `src/**/*routes*.ts`
**Detect:** A controller that computes, branches on domain state, hashes, signs tokens,
grades, or performs multi-step workflows. Grep hint inside controllers:
`bcrypt|argon2|jwt\.sign|\.aggregate\(|Model\.|transporter|nodemailer`

```ts
// Bad
async submit(req: Request, res: Response) {
  const attempt = await AttemptModel.findById(req.params.id);
  let score = 0;
  for (const answer of req.body.answers) if (isCorrect(attempt, answer)) score++;
  attempt.score = (score / attempt.questions.length) * 100;
  attempt.status = attempt.score >= 60 ? "passed" : "failed";
  await attempt.save();
  res.json({ success: true, data: attempt });
}

// Good
async submit(req: Request, res: Response, next: NextFunction) {
  try {
    const result = await this.attemptService.submit(req.params.id, req.body);
    res.status(200).json({ success: true, data: result });
  } catch (error) { next(error); }
}
```

**Why:** Logic in a controller is reachable only through HTTP, so it can only be tested
through HTTP, and it cannot be reused by a job, a CLI, or another endpoint.
**Not a violation when:** The controller only maps request → DTO, calls one service, and
maps the result → response.

---

### CC-202 — Data access outside the repository layer
**Severity:** major · **Applies to:** `src/**/*.controller.ts`, `src/**/*.service.ts`, middleware
**Detect:** Direct Mongoose model calls outside `*.repository.ts` / `repos/`.
Grep hint: `\b[A-Z]\w*Model\.(find|findOne|findById|create|updateOne|deleteOne|aggregate)`

```ts
// Bad — quiz.service.ts
const quiz = await QuizModel.findById(id).populate("questions").lean();

// Good — quiz.service.ts
const quiz = await this.quizRepository.findByIdWithQuestions(id);
```

**Why:** Query details leaking upward couples business rules to the schema and to Mongoose
itself; every query change then ripples through services and controllers.
**Not a violation when:** The project has deliberately no repository layer *and* is
consistent about it — then the service layer is the data boundary and controllers still
may not query.

---

### CC-203 — Layer imported in the wrong direction
**Severity:** critical · **Applies to:** `src/**/*.ts`
**Detect:** A service, repository, model, or util importing a controller, a route module,
or `express` types. Grep hint in `*.service.ts` / `*.repository.ts` / `models/`:
`from "express"|\.controller|\.routes`

```ts
// Bad — quiz.service.ts
import { Request, Response } from "express";
async listQuizzes(req: Request, res: Response) { ... }

// Good — quiz.service.ts
async listQuizzes(filter: QuizFilter): Promise<Quiz[]> { ... }
```

**Why:** Inverted dependencies destroy the layer boundary — the domain becomes unusable
outside HTTP and untestable without a fake `req`/`res`.
**Not a violation when:** The file is a controller, middleware, or the app bootstrap.

---

### CC-204 — Cross-module reach-around
**Severity:** major · **Applies to:** `src/modules/**`
**Detect:** Module A importing module B's repository, model, or internal helper instead of
B's public service. Grep hint: `from "\.\./[a-z-]+/(?!.*\.service)` inside `src/modules/`.

```ts
// Bad — attempt.service.ts
import { QuestionModel } from "../quiz/questions/question.model.js";

// Good — attempt.service.ts
import { quizService } from "../quiz/quiz.service.js";
const questions = await quizService.getGradingSheet(quizId);
```

**Why:** Reaching into another module's internals freezes those internals as a public API
by accident.
**Not a violation when:** The import is from a shared `common/` module explicitly meant to
be consumed by everyone.

---

### CC-205 — Framework or infrastructure types in the domain
**Severity:** major · **Applies to:** `src/**/*.service.ts`, domain types, pure logic modules
**Detect:** Domain functions accepting or returning `Request`, `Response`, `Document`,
`ObjectId`, or a raw Mongoose model instance.

```ts
// Bad
function calculateScore(attempt: HydratedDocument<Attempt>, req: Request): number

// Good
function calculateScore(gradingSheet: GradingSheet, answers: SubmittedAnswer[]): Score
```

**Why:** Pure domain functions are the cheapest thing in a codebase to test and the most
expensive thing to lose. Infrastructure types make them neither.
**Not a violation when:** The function is explicitly an adapter (repository, mapper,
middleware).

---

### CC-206 — Missing input validation at the HTTP boundary
**Severity:** critical · **Applies to:** `src/**/*.routes.ts`, `src/**/*.controller.ts`
**Detect:** `req.body`, `req.params`, or `req.query` flowing into a service or a query with
no Zod/Joi schema or validation middleware on that route. Grep hint: routes with no
`validate(`/`schema`/`ZodSchema`/`Joi.object` in the chain.

```ts
// Bad
router.post("/quizzes", isAuthenticated, quizController.create);
// controller: await this.quizService.create(req.body)

// Good
router.post("/quizzes", isAuthenticated, validate(createQuizSchema), quizController.create);
// controller: await this.quizService.create(req.validated.body)
```

**Why:** Unvalidated input reaching a Mongoose query is how mass-assignment and
NoSQL-operator injection (`{ email: { $ne: null } }`) happen.
**Not a violation when:** The handler takes no input, or validation demonstrably runs in a
router-level middleware applied to the whole subtree.

---

### CC-207 — Authorization decided ad hoc inside a handler
**Severity:** critical · **Applies to:** `src/**/*.controller.ts`, `src/**/*.service.ts`
**Detect:** Inline `if (req.user.role === "admin")` checks scattered across handlers instead
of a named guard, or an admin-only route with no role check at all.

```ts
// Bad
async deleteQuiz(req: Request, res: Response) {
  if (req.user.role !== "admin") return res.status(403).json({ message: "no" });
  ...
}

// Good
router.delete("/quizzes/:id", isAuthenticated, authorize(Role.Admin), quizController.delete);
```

**Why:** Scattered checks are impossible to audit; the one handler that forgets is the breach.
**Not a violation when:** The rule is genuinely record-level (owner-only access) — then it
belongs in the service, expressed as a named policy function.

---

### CC-208 — Singleton and construction wiring mixed into modules
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** A module that both defines a class and eagerly constructs it with concrete
dependencies at import time, when other modules need to inject fakes.
**Why:** Import-time construction makes the dependency graph implicit and unit tests need
`jest.mock` gymnastics to substitute anything.
**Not a violation when:** The project consistently uses module-level singletons and the
constructor still accepts its dependencies (so tests can build their own instance).

---

## CC-3xx — Duplication and abstraction

### CC-301 — Copy-pasted logic
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** The same 5+ line block appearing two or more times, most often: pagination
maths, OTP generation, "find or 404", response envelopes, `try/catch` + `next(error)`.

```ts
// Bad — repeated in six controllers
const page = Number(req.query.page) || 1;
const limit = Math.min(Number(req.query.limit) || 10, 100);
const skip = (page - 1) * limit;

// Good
const { skip, limit, page } = parsePagination(req.query);
```

**Why:** Duplicated rules drift. The fix lands in five of six copies and the sixth becomes
a bug report.
**Not a violation when:** The blocks are coincidentally similar but change for different
reasons — forcing a shared abstraction there is worse (see CC-303).

---

### CC-302 — Duplicated literal that encodes one rule
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** The same magic literal in 2+ places: `60` (pass mark), `10` (bcrypt rounds),
`"admin"`, `15 * 60 * 1000`, `"1h"`.

```ts
// Bad
if (score >= 60) attempt.status = "passed";      // attempt.service.ts
const passed = attempts.filter((a) => a.score >= 60);  // dashboard.service.ts

// Good
export const PASS_MARK_PERCENT = 60;
if (score >= PASS_MARK_PERCENT) ...
```

**Why:** One business rule, one definition. Otherwise changing the pass mark is a grep-and-pray.
**Not a violation when:** The literal is a local, self-evident constant (`0`, `1`, `-1`,
array indices).

---

### CC-303 — Premature or speculative abstraction
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** A generic base class, options bag, or "engine" with exactly one caller;
`if (options.mode === ...)` branches nobody triggers; config flags with a single value.

**Why:** YAGNI. A wrong abstraction costs more than the duplication it prevented, because
it must be unwound before anything can change.
**Not a violation when:** A second real caller exists today.

---

### CC-304 — Manual work the language or a dependency already does
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** Hand-rolled deep clone, `Object.keys().forEach` where `Object.entries().map`
reads better, custom email regex where a validation schema is already in use, hand-written
date arithmetic where `dayjs` is already a dependency.
**Why:** Hand-rolled versions of solved problems carry edge-case bugs that nobody tests for.
**Not a violation when:** The dependency is not already installed and adding it for this
one use is heavier than the ten lines it replaces.

---

## CC-4xx — Error handling

### CC-401 — Swallowed error
**Severity:** critical · **Applies to:** `src/**/*.ts`
**Detect:** `catch` blocks that are empty, only `console.log`, or return a default while
losing the cause. Grep hint: `catch\s*\([^)]*\)\s*\{\s*\}` and `catch` blocks with no
`throw`/`next(`.

```ts
// Bad
try { await this.mailer.send(user.email, otp); } catch (error) { console.log(error); }

// Good
try {
  await this.mailer.send(user.email, otp);
} catch (error) {
  this.logger.error({ error, userId: user.id }, "confirmation email failed");
  throw new ServiceUnavailableError("could not send confirmation email", { cause: error });
}
```

**Why:** A swallowed error turns a loud failure into a silent wrong result — the most
expensive class of production bug.
**Not a violation when:** The failure is genuinely optional *and* the catch says so
explicitly (logged with context, with a comment stating why it is safe to continue).

---

### CC-402 — Errors handled by ad-hoc HTTP responses instead of a central handler
**Severity:** major · **Applies to:** `src/**/*.controller.ts`, `src/**/*.service.ts`
**Detect:** `res.status(...).json({ message })` scattered through catch blocks; services
that build HTTP responses; the same "something went wrong" envelope repeated everywhere.

```ts
// Bad
catch (error) { res.status(500).json({ success: false, message: "Something went wrong" }); }

// Good
catch (error) { next(error); }   // one Express error middleware maps AppError -> status + body
```

**Why:** Central handling is the only way to keep response shape, logging, and status
mapping consistent — and the only place to guarantee stack traces never leak.
**Not a violation when:** The file *is* the central error middleware.

---

### CC-403 — Throwing or rejecting with a non-Error value
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** `throw "message"`, `throw { status: 404 }`, `Promise.reject("nope")`.

```ts
// Bad
throw { status: 404, message: "quiz not found" };

// Good
throw new NotFoundError(`quiz ${id} not found`);
```

**Why:** Non-Error throws have no stack trace, break `instanceof` dispatch in the error
middleware, and confuse every logger.
**Not a violation when:** Never. Define a small `AppError` hierarchy.

---

### CC-404 — Error message with no actionable context
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** `"error"`, `"failed"`, `"something went wrong"`, `"invalid"` with no subject.

```ts
// Bad
throw new BadRequestError("invalid");

// Good
throw new BadRequestError(`attempt ${attemptId} already submitted at ${attempt.submittedAt}`);
```

**Why:** The message is the only thing the on-call reader gets. Include the identifier and
the state that made it fail — never the secret or the password.
**Not a violation when:** The message is deliberately generic to avoid user enumeration
(login failures) — then log the detail server-side.

---

### CC-405 — Internal detail leaked to the client
**Severity:** critical · **Applies to:** error middleware, `src/**/*.controller.ts`
**Detect:** `res.json({ error: err.stack })`, forwarding a raw Mongoose validation error,
returning a hashed password or `__v` in a response body.

```ts
// Bad
catch (error) { res.status(500).json({ error }); }

// Good
catch (error) {
  logger.error({ error, requestId: req.id }, "unhandled");
  res.status(500).json({ success: false, message: "internal error", requestId: req.id });
}
```

**Why:** Stack traces and driver errors disclose paths, versions, and schema shape to an attacker.
**Not a violation when:** `NODE_ENV !== "production"` and the branch is explicit.

---

### CC-406 — Validation, expected, and unexpected failures conflated
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** Everything thrown as a generic `Error`, so the middleware cannot distinguish
400 from 404 from 500 and defaults everything to 500.
**Why:** Status mapping should follow from the error type, not from string matching on messages.
**Not a violation when:** A typed error hierarchy exists and is used.

---

### CC-407 — `try/catch` with no reason to catch
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** A `catch` block that re-throws the same information, rewraps every error as one
status, or exists only because "services should handle their errors". Grep hint:
`catch` blocks whose body is a single `throw new`, or that appear in every method of a file.

A global error middleware is the default owner of failure. A local `catch` must earn its
place by doing one of exactly four things:

| Reason | What the catch adds |
| --- | --- |
| **Translate** | A foreign error becomes a domain error (`code: 11000` → `ConflictError`) |
| **Add context** | Wraps with the identifier and state that made it fail, preserving `cause` |
| **Compensate** | Undoes a partial write before rethrowing (see `CC-504`) |
| **Decide** | A genuinely optional step continues, logged, with a comment saying why that is safe |

If none apply, do not catch. Let it propagate to the one middleware that maps errors to HTTP.

```ts
// Bad — rewraps every failure as 500, destroying the status the service chose
} catch (error: any) {
  next(new CustomError(error.message, 500));
}

// Bad — "handling" that loses the error entirely and continues with undefined
const quiz = await this.quizRepo.findById(id).catch((err: any) => { console.error(err); });

// Bad — the same six lines in twelve controllers
} catch (error) {
  res.status(500).json({ success: false, message: 'Something went wrong' });
}

// Good — translate, then let everything else flow
} catch (error) {
  if ((error as { code?: number }).code === 11000) {
    throw new ConflictError('User with this email already exists');
  }
  throw error;
}

// Good — compensate, then rethrow unchanged
} catch (error) {
  await this.attemptRepo.deleteOne({ filter: { _id: attempt._id } });
  throw error;
}

// Good — controller has no try/catch at all
submit = asyncWrap(async (req: Request, res: Response) => {
  const result = await this.attemptService.submit(req.params.id, req.validated.body);
  res.status(200).json({ success: true, data: result });
});
```

Responsibility per layer:

| Layer | `try/catch`? | Responsibility |
| --- | --- | --- |
| Controller | No — wrap in `asyncHandler` / `asyncWrap` | map request → DTO → service → response |
| Service | Only for the four reasons above | throw typed domain errors |
| Repository | Only to translate driver errors | keep Mongoose behind the boundary |
| Error middleware | Yes — the one that matters | map domain error → status + envelope, log unknowns |

**Why:** Catching everywhere produces the failures in `CC-401` (swallowed), `CC-402`
(duplicated response building), and `CC-406` (status collapsed to 500). Catching in one place
means status mapping, logging, and response shape have exactly one definition.
**Not a violation when:** The catch does one of the four jobs above, or the file is the
central error middleware.

> **Express 4 caveat:** a rejected promise from an `async` handler or middleware does **not**
> reach the error middleware. Every async handler must be wrapped (`asyncHandler`), or the
> app must use `express-async-errors` or Express 5. An unwrapped `async` middleware that
> throws leaves the request hanging with no response — see `CC-501`.

---

## CC-5xx — Async correctness

### CC-501 — Missing `await` on a promise
**Severity:** critical · **Applies to:** `src/**/*.ts`
**Detect:** A call to an async function used as a statement with no `await`, no `return`,
and no `.catch`. Grep hint: statement lines calling `Model.`, `repository.`, `service.`,
`.save()`, `.send()` without `await`. Enable `@typescript-eslint/no-floating-promises`.

```ts
// Bad
user.save();                       // fire and forget — caller "succeeds" before the write lands
this.mailer.sendConfirmation(user);

// Good
await user.save();
await this.mailer.sendConfirmation(user);
```

**Why:** A floating promise means the response can be sent before the work happens, and a
rejection becomes an unhandled rejection that can take down the process.
**Not a violation when:** Fire-and-forget is intentional, and the call has an attached
`.catch(...)` plus a comment saying why the result is not awaited.

---

### CC-502 — Sequential `await` in a loop over independent work
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** `for (const x of xs) await doSomething(x)` where iterations do not depend on
each other.

```ts
// Bad
for (const question of questions) await QuestionModel.create(question);

// Good
await this.questionRepository.createMany(questions);
// or, when there is no bulk API:
await Promise.all(questions.map((question) => this.questionRepository.create(question)));
```

**Why:** N sequential round trips where one batch or one parallel wave would do.
**Not a violation when:** Order matters, the work mutates shared state, or unbounded
parallelism would exhaust the connection pool — then bound the concurrency, do not
serialise by accident.

---

### CC-503 — Mixing `async/await` with `.then/.catch` in one flow
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** `await something().catch((err: any) => { ... })` inside an `async` function,
especially where the catch returns `undefined` and the code continues.

```ts
// Bad
const quiz = await QuizModel.findById(id).catch((err: any) => { console.log(err); });
quiz.title;   // quiz may be undefined here, and TS cannot see it

// Good
const quiz = await this.quizRepository.findById(id);   // let it throw; the middleware maps it
if (!quiz) throw new NotFoundError(`quiz ${id}`);
```

**Why:** The two styles have different failure semantics; mixing them produces
"impossible" undefineds that survive type-checking.
**Not a violation when:** `.catch` is used deliberately for a narrow fallback and the
result type reflects it.

---

### CC-504 — Multi-step write with no transaction or compensation
**Severity:** major · **Applies to:** `src/**/*.service.ts`, `src/**/*.repository.ts`
**Detect:** Two or more dependent writes across collections with no session/transaction and
no rollback path (create attempt → decrement quota → write audit).
**Why:** A crash between writes leaves the database in a state no code path expects.
**Not a violation when:** The writes are independent, or the second step is idempotent and
retried by design.

---

### CC-505 — Blocking or CPU-heavy work on the request path
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** `*Sync` filesystem calls, `bcrypt.hashSync`, `JSON.parse` on large uploads, or
tight loops over full collections inside a handler. Grep hint: `Sync\(`

**Why:** One event loop. Blocking it stalls every concurrent request, not just this one.
**Not a violation when:** The code runs at startup or in a script/CLI entrypoint.

---

## CC-6xx — Comments, dead code, noise

### CC-601 — Commented-out code
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** Blocks of `//` lines containing statements. Grep hint:
`^\s*//\s*(const|let|await|return|if|await|res\.|this\.)`
**Why:** Version control already remembers. Commented code rots and misleads.
**Not a violation when:** It is an illustrative example inside documentation prose.

---

### CC-602 — Comment restates the code
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** `// increment counter` above `counter++`; `// get user` above `getUser()`.

```ts
// Bad
// hash the password
const hashed = await bcrypt.hash(password, 10);

// Good
// Cost 12 is the current OWASP baseline; raise as hardware improves.
const hashed = await bcrypt.hash(password, BCRYPT_COST);
```

**Why:** Comments should carry what the code cannot: *why*, trade-offs, external constraints.
**Not a violation when:** The comment is a section divider in a long route table, or a
public API docstring.

---

### CC-603 — Stale `TODO`/`FIXME` with no owner or issue
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** `TODO`, `FIXME`, `HACK`, `XXX` without a ticket reference.
**Why:** An unowned TODO is a wish, not a plan.
**Not a violation when:** It links an issue (`TODO(#412): ...`).

---

### CC-604 — Dead code
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** Exports nobody imports, unreachable branches after `return`/`throw`, feature
flags permanently false, `*.old.ts` / `*.backup.ts` files.
**Why:** Dead code is read, maintained, and refactored at full cost for zero value.
**Not a violation when:** It is a public library export or a documented extension point.

---

### CC-605 — Debug logging left in production code
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** `console.log` / `console.dir` in `src/**` when the project has a logger
(`pino`, `morgan`, `jet-logger`). Escalate to **critical** when the logged value contains a
password, token, OTP, or full request body.

```ts
// Bad
console.log("user", user, "token", token);

// Good
logger.debug({ userId: user.id }, "issued access token");
```

**Why:** `console.log` has no level, no structure, and no redaction — it is how secrets end
up in log aggregators.
**Not a violation when:** The file is a script, seed, or CLI entrypoint.

---

## CC-7xx — Configuration, constants, magic values

### CC-701 — `process.env` read deep inside a module
**Severity:** major · **Applies to:** `src/**/*.ts` outside `src/config/**`
**Detect:** `process.env.X` outside the config module. Grep hint: `process\.env\.`

```ts
// Bad — auth.service.ts
const token = jwt.sign(payload, process.env.JWT_SECRET!, { expiresIn: process.env.JWT_TTL });

// Good — config/env.ts validates once at boot
export const env = envSchema.parse(process.env);
// auth.service.ts
const token = jwt.sign(payload, env.JWT_SECRET, { expiresIn: env.JWT_TTL });
```

**Why:** Scattered env reads mean the app boots successfully and fails later, in one
endpoint, in production. Validate once, fail at startup, type the result.
**Not a violation when:** The file is the config/bootstrap module itself.

---

### CC-702 — Secret or credential in source
**Severity:** critical · **Applies to:** all tracked files
**Detect:** Literal JWT secrets, API keys, database URIs with passwords, SMTP credentials,
default fallbacks like `process.env.JWT_SECRET || "secret"`, committed `.env`.
**Why:** Committed secrets are permanently compromised — rotation is the only fix.
**Not a violation when:** It is an obvious placeholder in `.env.example`.

---

### CC-703 — Unexplained magic number or string
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** Unnamed numeric literals in business expressions (`* 900000`, `>= 60`,
`15 * 60 * 1000`) and bare status strings (`"pending"`, `"admin"`).

```ts
// Bad
if (Date.now() - user.otpSentAt.getTime() > 600000) throw new BadRequestError("expired");

// Good
const OTP_TTL_MS = 10 * 60 * 1000;
if (Date.now() - user.otpSentAt.getTime() > OTP_TTL_MS) throw new OtpExpiredError();
```

**Why:** A named constant states the rule and gives it one place to change.
**Not a violation when:** The literal is self-evident in context (`0`, `1`, `100` in a
percentage conversion right beside the word `percent`).

---

### CC-704 — Domain state as free-form strings
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** Status/role values compared as raw strings across files (`"passed"`, `"admin"`,
`"pending"`) with no enum or union type.
**Why:** A typo in one comparison is a silent branch that never runs. See `TS-203`.
**Not a violation when:** The value comes from an external system and is deliberately
opaque.

---

## CC-8xx — Data and API boundary hygiene

### CC-801 — Persistence document returned straight to the client
**Severity:** major · **Applies to:** `src/**/*.controller.ts`, `src/**/*.service.ts`
**Detect:** A Mongoose document or `.lean()` result handed to `res.json` with no mapper.
Fields like `password`, `confirmOTP`, `forgetPasswordOTP`, `__v`, `refreshTokens` reaching
the response.

```ts
// Bad
const user = await this.userRepository.findById(id);
res.json({ success: true, data: user });      // includes password hash and OTP fields

// Good
res.json({ success: true, data: toUserResponse(user) });
// toUserResponse picks id, name, email, role, createdAt — nothing else
```

**Why:** Schema is not API. Without an explicit mapper, every new field is published by
default, and the first sensitive one is a breach.
**Not a violation when:** A schema-level projection guarantees exclusion (`select: false`
on secrets) *and* the response type is explicit.

---

### CC-802 — Unfiltered request body passed to a write
**Severity:** critical · **Applies to:** `src/**/*.ts`
**Detect:** `Model.create(req.body)`, `findByIdAndUpdate(id, req.body)`, `{ ...req.body }`
spread into a document.

```ts
// Bad
await UserModel.findByIdAndUpdate(req.params.id, req.body);   // a client can set role: "admin"

// Good
const { name, avatarUrl } = updateProfileSchema.parse(req.body);
await this.userRepository.updateProfile(req.params.id, { name, avatarUrl });
```

**Why:** Mass assignment — the client picks which columns to write, including `role` and
`isVerified`.
**Not a violation when:** The body has already been parsed by a schema that strips unknown
keys and the write uses the parsed object.

---

### CC-803 — Unbounded query or missing pagination
**Severity:** major · **Applies to:** `src/**/*.repository.ts`, `src/**/*.service.ts`
**Detect:** `find({})` with no `limit`, or a client-supplied `limit` with no ceiling.
**Why:** One collection scan or a `?limit=1000000` request is a denial of service that
needs no attacker.
**Not a violation when:** The collection is bounded by design (roles, config docs).

---

### CC-804 — Sensitive value logged or compared unsafely
**Severity:** critical · **Applies to:** `src/**/*.ts`
**Detect:** Passwords, OTPs, or tokens in log calls or error messages; plain `===`
comparison of a secret/OTP; a password field without `select: false` in the schema.
**Why:** Logs are read by more people than the database; naive comparison of secrets leaks
timing and lands the value in stack traces.
**Not a violation when:** The value is redacted at the logger level and the redaction is
demonstrably configured.

---

### CC-805 — Domain invariant enforced only in the UI or only in one path
**Severity:** major · **Applies to:** `src/**/*.service.ts`
**Detect:** A rule ("cannot submit an attempt twice", "cannot start an unpublished quiz")
enforced in one endpoint but missing on a second path that reaches the same state.
**Why:** An invariant enforced in some paths is not an invariant. Put it in the service or
the schema, where every path passes.
**Not a violation when:** The database enforces it (unique index, required field) and the
code relies on that error.
