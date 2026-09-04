# Jest & Unit Testing Rule Catalog (`JT-###`)

Applies to `**/*.spec.ts`, `**/*.test.ts`, `tests/**`, `__tests__/**`, `jest.config.*`,
and test setup files.

Read [`README.md`](./README.md) for severity meanings, the verify-before-report protocol,
and the false-positive rules.

**Bands**

| Band | Topic |
| --- | --- |
| `JT-0xx` | Jest configuration and project setup |
| `JT-1xx` | File structure, naming, organisation |
| `JT-2xx` | Mocking policy |
| `JT-3xx` | Test design |
| `JT-4xx` | Assertions |
| `JT-5xx` | Async and timing |
| `JT-6xx` | Isolation and determinism |
| `JT-7xx` | Coverage and missing tests |

**Definitions used throughout**

- **Unit test** — exercises one module (a service, a pure function, a mapper) with all
  I/O collaborators replaced by test doubles. No database, no network, no filesystem.
- **Integration test** — exercises several real modules together, typically through
  `supertest` against the Express app, against an in-memory or disposable database.
  Different rules apply; they are called out where they differ.

---

## JT-0xx — Jest configuration

### JT-001 — No test script, or tests not discovered
**Severity:** critical · **Applies to:** `package.json`, `jest.config.*`
**Detect:** No `test` script; a `testMatch` that misses existing spec files; a project with
`*.spec.ts` files and no Jest setup at all.

```jsonc
// Bad
"scripts": { "dev": "ts-node-dev src/index.ts" }        // spec files exist, nothing runs them

// Good
"scripts": { "test": "jest", "test:watch": "jest --watch", "test:cov": "jest --coverage" }
```

**Why:** Tests that never run are worse than no tests — they signal safety that is not there.
**Not a violation when:** The package genuinely has no tests yet — then the finding is
`JT-701`, not this one.

---

### JT-002 — `testMatch` inconsistent with the files on disk
**Severity:** major · **Applies to:** `jest.config.*`
**Detect:** `testMatch: ['**/*.test.ts']` while the repo contains `*.spec.ts`, or vice
versa. Confirm by listing both patterns on disk.
**Why:** Half the suite silently does not run, and `passWithNoTests` makes CI green anyway.
**Not a violation when:** Both patterns are covered.

---

### JT-003 — Type errors suppressed in the test transform
**Severity:** major · **Applies to:** `jest.config.*`
**Detect:** `diagnostics: false` or `diagnostics: { ignoreCodes: [...] }` in the `ts-jest`
transform — especially `2307` (module not found) and `2345` (argument type mismatch).

```ts
// Bad
transform: { "^.+\\.tsx?$": ["ts-jest", { diagnostics: { ignoreCodes: [151002, 2307] } }] }

// Good
transform: { "^.+\\.tsx?$": ["ts-jest", { tsconfig: "tsconfig.json" }] }
// fix the underlying path mapping instead of muting the error
```

**Why:** Ignoring `2307` means a test can import a module that does not exist and still
"pass" — the mock resolves to `undefined` and assertions quietly test nothing.
**Not a violation when:** A single documented code is ignored for a known upstream
`@types` defect, with a comment.

---

### JT-004 — `forceExit` or `--detectOpenHandles` masking leaked resources
**Severity:** major · **Applies to:** `jest.config.*`, test scripts
**Detect:** `forceExit: true`; a test run that only terminates because of it.

```ts
// Bad
const config: Config = { preset: "ts-jest", forceExit: true };

// Good
// jest.config.ts — no forceExit
// tests close what they open:
afterAll(async () => {
  await mongoose.disconnect();
  await redisClient.quit();
});
```

**Why:** `forceExit` kills the process mid-flight — pending writes, open connections, and
unflushed assertions vanish. It hides a real resource leak that also exists in production.
**Not a violation when:** A third-party dependency is proven to leak and the workaround is
documented with a link.

---

### JT-005 — Mock state not reset between tests by configuration
**Severity:** major · **Applies to:** `jest.config.*`
**Detect:** None of `clearMocks`, `resetMocks`, `restoreMocks` enabled **and** no
`beforeEach(() => jest.clearAllMocks())` in the suites.

```ts
// Good — jest.config.ts
{ clearMocks: true, restoreMocks: true }
```

**Why:** Without a reset, call counts and queued `mockResolvedValueOnce` values leak into
the next test, making results depend on file order.
**Not a violation when:** Every suite resets explicitly in `beforeEach`.

---

### JT-006 — Test environment or setup not isolated from production config
**Severity:** major · **Applies to:** `jest.config.*`, `setup*.ts`, test scripts
**Detect:** Tests running with `NODE_ENV=development`; a setup file connecting to the same
`MONGO_URI` as the dev database; `.env` loaded without a `.env.test` override.
**Why:** A test suite that can write to a real database will eventually delete a real
collection.
**Not a violation when:** The test config demonstrably points at a disposable or in-memory
instance.

---

## JT-1xx — Structure and naming

### JT-101 — Test file not colocated or not discoverable
**Severity:** minor · **Applies to:** test files
**Detect:** A spec far from its subject with no convention; a `tests/` folder in one module
and colocated specs in another within the same project.
**Why:** Colocation is how a reader knows a module has tests without searching.
**Not a violation when:** The project consistently uses a top-level `tests/` tree.

---

### JT-102 — One spec file covering several modules
**Severity:** minor · **Applies to:** test files
**Detect:** A single `auth.test.ts` importing and asserting on the controller, the service,
the repository, and the middleware.
**Why:** One spec, one subject. Mixed specs cannot say what broke when they go red.
**Not a violation when:** It is explicitly an integration or end-to-end spec named as such.

---

### JT-103 — `describe` / `it` names that do not state a behaviour
**Severity:** minor · **Applies to:** test files
**Detect:** `it("works")`, `it("test 1")`, `it("should return")`, names describing the
implementation rather than the observable outcome.

```ts
// Bad
it("calls findOne and bcrypt", async () => { ... });

// Good
it("rejects sign-up when the email is already registered", async () => { ... });
```

**Why:** The test name is the failure message in CI. It must say what the system promised.
**Not a violation when:** The `describe` chain already supplies the subject and the `it`
completes the sentence.

---

### JT-104 — Setup buried far from the assertion
**Severity:** minor · **Applies to:** test files
**Detect:** Deeply nested `describe` blocks with `beforeEach` layers, so a reader must
climb three levels to learn what the input was.

```ts
// Bad — four nested describes, each mutating a shared `payload`

// Good
const validSignUp = (overrides: Partial<SignUpDto> = {}): SignUpDto => ({
  email: "user@example.com",
  password: "UserPassword123",
  firstName: "Ada",
  lastName: "Lovelace",
  ...overrides,
});

it("rejects a password shorter than 8 characters", async () => {
  await expect(authService.signUp(validSignUp({ password: "short" }))).rejects.toThrow(ValidationError);
});
```

**Why:** A test should be readable top to bottom in one screen. Builders beat nested setup.
**Not a violation when:** One shallow `beforeEach` constructs the subject under test.

---

### JT-105 — Shared mutable state between tests
**Severity:** major · **Applies to:** test files
**Detect:** A `let` fixture declared at module scope and mutated inside tests; a shared
array pushed to across cases.

```ts
// Bad
let user = { id: "1", attempts: [] as Attempt[] };
it("adds an attempt", () => { user.attempts.push(attempt); expect(user.attempts).toHaveLength(1); });
it("starts empty", () => { expect(user.attempts).toHaveLength(0); });   // depends on file order

// Good
const makeUser = (): User => ({ id: "1", attempts: [] });
it("adds an attempt", () => { const user = makeUser(); ... });
```

**Why:** Shared mutable state makes tests order-dependent — they pass locally, fail in CI
sharding, and nobody can reproduce it.
**Not a violation when:** The shared value is genuinely immutable (a frozen constant).

---

## JT-2xx — Mocking policy

**Policy in one line:** mock at the boundary the unit owns — repositories, mailers, token
signers, clocks, HTTP clients. Never mock the unit under test, and never mock pure logic.

---

### JT-201 — Over-mocking: internals mocked instead of boundaries
**Severity:** major · **Applies to:** test files
**Detect:** `jest.mock` on pure helpers, mappers, validators, or calculators that have no
I/O; a test with more mock setup lines than assertion lines.

```ts
// Bad
jest.mock("./score-calculator");        // pure function, no I/O — now nothing verifies grading

// Good
jest.mock("./attempt.repository");      // I/O boundary
// scoreCalculator runs for real, so the test actually proves the grading rule
```

**Why:** Every mocked collaborator is behaviour the test no longer verifies. Mock what
crosses a process boundary; run the rest.
**Not a violation when:** The helper is genuinely non-deterministic (random, time, uuid) —
inject it instead (see `JT-602`).

---

### JT-202 — Real I/O inside a unit test
**Severity:** critical · **Applies to:** `*.spec.ts`, `*.test.ts` that are unit tests
**Detect:** `mongoose.connect`, a live `Model` call with no `jest.mock`, `nodemailer`
sending, `fetch`/`axios` to a URL, filesystem writes.

```ts
// Bad
beforeAll(async () => { await mongoose.connect(process.env.MONGO_URI!); });

// Good — unit test
jest.mock("./quiz.repository");
const quizRepository = jest.mocked(quizRepositoryModule).quizRepository;
quizRepository.findById.mockResolvedValue(makeQuiz());
```

**Why:** A unit test that touches the network is slow, flaky, order-dependent, and can
destroy data. It also stops being a unit test without anyone renaming it.
**Not a violation when:** The file is an integration test — then it must be named and
located as one, and must use a disposable/in-memory database.

---

### JT-203 — The unit under test is itself mocked
**Severity:** critical · **Applies to:** test files
**Detect:** `jest.mock("./x.service")` inside `x.service.spec.ts`; a "controller test" that
mocks the controller; assertions that only check a mock was called with what the test
itself passed in.

```ts
// Bad — quiz.service.spec.ts
jest.mock("./quiz.service");
it("creates a quiz", async () => {
  await quizService.createQuiz(dto);
  expect(quizService.createQuiz).toHaveBeenCalledWith(dto);   // asserts on the mock, tests nothing
});

// Good — quiz.service.spec.ts
jest.mock("./quiz.repository");
it("persists a quiz with the default duration when none is supplied", async () => {
  quizRepository.create.mockResolvedValue(makeQuiz({ durationMinutes: 30 }));
  const quiz = await quizService.createQuiz(makeCreateQuizDto({ durationMinutes: undefined }));
  expect(quiz.durationMinutes).toBe(30);
});
```

**Why:** This is the highest-value finding in any AI-written test suite: it is a test that
can never fail, sitting in a green coverage report.
**Not a violation when:** Never.

---

### JT-204 — Mocks not reset between tests
**Severity:** major · **Applies to:** test files
**Detect:** `jest.mock` usage with no `clearMocks`/`resetMocks` in config and no
`beforeEach(() => jest.clearAllMocks())`; `mockResolvedValueOnce` queues left over from a
previous test.
**Why:** Leftover call history makes `toHaveBeenCalledTimes` assertions pass or fail based
on which test ran first.
**Not a violation when:** Config-level reset is enabled (see `JT-005`).

---

### JT-205 — Mock returns a shape the real collaborator never returns
**Severity:** major · **Applies to:** test files
**Detect:** `mockResolvedValue({} as any)`, a mock returning a bare object where the real
repository returns a document with methods, or a mock whose fields do not exist on the
real type.

```ts
// Bad
(User.findOne as jest.Mock).mockResolvedValue({ id: "1", pass: "x" });   // real field is `password`

// Good
const userRepository = jest.mocked(userRepositoryModule).userRepository;
userRepository.findByEmail.mockResolvedValue(makeUser({ password: "hashed" }));
```

**Why:** A mock that lies about the contract makes the test pass while production breaks —
the test is now actively harmful.
**Not a violation when:** The extra/missing fields are irrelevant to the assertion *and*
the mock is typed so the compiler enforces the rest.

---

### JT-206 — Untyped mocks
**Severity:** minor · **Applies to:** test files
**Detect:** `as jest.Mock` / `as any` casts repeated throughout a file instead of
`jest.mocked(...)`.

```ts
// Bad
(TokenUtils.sign as jest.Mock).mockReturnValue("token");

// Good
const tokenUtils = jest.mocked(TokenUtils);
tokenUtils.sign.mockReturnValue("token");
```

**Why:** `jest.mocked` keeps the mock tied to the real signature, so a refactor of the
collaborator breaks the test at compile time instead of at runtime.
**Not a violation when:** Building a deliberately partial double —
`as unknown as Partial<X>` is acceptable in tests when the partiality is the point.

---

### JT-207 — Auto-mock leaves collaborators returning `undefined`
**Severity:** major · **Applies to:** test files
**Detect:** `jest.mock("./x")` with no factory and no `mockResolvedValue` for a method the
code path calls — the call returns `undefined` and the test asserts on downstream noise.
**Why:** The test then verifies the behaviour of `undefined`, not of the system.
**Not a violation when:** The undefined return is exactly what the real collaborator gives
and the test says so.

---

### JT-208 — Mocking what you do not own, without a contract check
**Severity:** minor · **Applies to:** test files
**Detect:** Deep mocks of third-party clients (`mongoose.Model` chains, `nodemailer`
transports) with hand-written chainable stubs (`{ populate: () => ({ lean: () => data }) }`).
**Why:** Hand-built chain stubs encode your guess of the library's API; when the guess is
wrong, the test passes and production does not.
**Not a violation when:** The third-party call sits behind your own repository interface
and *that* is what you mock — which is the fix.

---

## JT-3xx — Test design

### JT-301 — Test asserts more than one behaviour
**Severity:** minor · **Applies to:** test files
**Detect:** An `it` with several unrelated `expect` groups, or a name containing "and":
`it("hashes the password and creates the user and sends the email and returns a token")`.

```ts
// Bad
it("signs up correctly", async () => {
  const result = await authService.signUp(dto);
  expect(cryptoUtil.hash).toHaveBeenCalled();
  expect(userRepository.create).toHaveBeenCalled();
  expect(emailService.send).toHaveBeenCalled();
  expect(result.token).toBeDefined();
});

// Good
it("stores the password hashed, never in plain text", async () => { ... });
it("sends a confirmation email to the new address", async () => { ... });
it("returns an access token for the created user", async () => { ... });
```

**Why:** A multi-behaviour test reports one failure for many possible causes, and the first
failing assertion hides the rest.
**Not a violation when:** Several `expect`s describe one behaviour (asserting three fields
of one returned object).

---

### JT-302 — No Arrange / Act / Assert shape
**Severity:** minor · **Applies to:** test files
**Detect:** Setup, invocation, and assertions interleaved; several `act` calls in one test.
**Why:** One act per test is what makes the failure attributable.
**Not a violation when:** The extra call is setup for the behaviour under test
(create-then-read).

---

### JT-303 — Logic inside a test
**Severity:** major · **Applies to:** test files
**Detect:** `if`, `for`, `while`, `try/catch`, or arithmetic computing the expected value
inside an `it`.

```ts
// Bad
it("grades the attempt", () => {
  let expected = 0;
  for (const answer of answers) if (answer.isCorrect) expected += 100 / answers.length;
  expect(scoreCalculator.grade(sheet, answers)).toBe(expected);   // reimplements the SUT
});

// Good
it.each([
  { correct: 0, total: 4, expected: 0 },
  { correct: 2, total: 4, expected: 50 },
  { correct: 4, total: 4, expected: 100 },
])("scores $correct/$total as $expected percent", ({ correct, total, expected }) => {
  expect(scoreCalculator.grade(sheetOf(total), answersWithCorrect(correct))).toBe(expected);
});
```

**Why:** Logic in a test can carry the same bug as the code, so both agree and both are
wrong. Expected values should be literal.
**Not a violation when:** The loop is in a builder or `it.each` table, not in the assertion.

---

### JT-304 — Conditional assertions
**Severity:** major · **Applies to:** test files
**Detect:** `if (result) expect(...)`, `expect(x).toBe(y)` inside a `catch`, or assertions
skipped by an early `return`.

```ts
// Bad
try { await authService.signUp(dto); } catch (error) { expect(error).toBeInstanceOf(AppError); }
// if signUp resolves, the catch never runs and the test passes

// Good
await expect(authService.signUp(dto)).rejects.toThrow(AppError);
```

**Why:** A conditional assertion is a test that passes when the code does nothing.
**Not a violation when:** `expect.assertions(n)` guarantees the assertion ran.

---

### JT-305 — Only the happy path is tested
**Severity:** major · **Applies to:** test files
**Detect:** A service with documented failure modes (not found, already exists, expired
OTP, unauthorized, invalid state transition) whose spec asserts only success.
**Why:** The error branches are where the security and correctness bugs live; the happy
path is the one manual testing already covers.
**Not a violation when:** The unit genuinely has one path.

---

### JT-306 — Tests assert implementation instead of behaviour
**Severity:** minor · **Applies to:** test files
**Detect:** Assertions only on call counts and call order for internal steps, with nothing
checking the returned value or the resulting state.

```ts
// Bad
expect(cryptoUtil.hash).toHaveBeenCalledTimes(1);
expect(User.create).toHaveBeenCalledTimes(1);
// nothing checks what was actually stored or returned

// Good
expect(userRepository.create).toHaveBeenCalledWith(expect.objectContaining({ password: "hashed-value" }));
expect(result.user.password).toBeUndefined();
```

**Why:** Implementation assertions break on every refactor and pass through real behaviour
changes — the worst of both.
**Not a violation when:** The interaction *is* the behaviour (an email must be sent, a
token must be revoked).

---

### JT-307 — The same call repeated to make several assertions
**Severity:** minor · **Applies to:** test files
**Detect:** The unit under test invoked twice in one test to assert two things about the
same outcome.

```ts
// Bad
await expect(authService.signUp(dto)).rejects.toThrow(AppError);
await expect(authService.signUp(dto)).rejects.toThrow("Email is already exists");

// Good
await expect(authService.signUp(dto)).rejects.toMatchObject({
  constructor: AppError,
  message: "Email is already exists",
});
// or capture once:
const error = await authService.signUp(dto).catch((e: unknown) => e);
expect(error).toBeInstanceOf(AppError);
expect((error as AppError).message).toBe("Email is already exists");
```

**Why:** Calling twice doubles the side effects and can pass for the wrong reason when the
unit is not idempotent.
**Not a violation when:** The second call is the behaviour being tested (idempotency,
rate limiting).

---

### JT-308 — Snapshot used where an explicit assertion belongs
**Severity:** minor · **Applies to:** test files
**Detect:** `toMatchSnapshot()` on API responses, service results, or objects containing
ids and timestamps.
**Why:** Snapshots of dynamic data are re-recorded on every failure, so the "test" becomes
a record of whatever the code currently does.
**Not a violation when:** Asserting on large, stable, human-reviewed output (a rendered
template) with inline snapshots.

---

### JT-309 — Test named for a bug ticket rather than a rule
**Severity:** minor · **Applies to:** test files
**Detect:** `it("fixes #412")`, `it("regression test")`.
**Why:** A regression test states the rule that must hold forever; the ticket number goes
in a comment.
**Not a violation when:** The name states the behaviour and merely cites the ticket.

---

## JT-4xx — Assertions

### JT-401 — Assertion too weak to fail
**Severity:** major · **Applies to:** test files
**Detect:** `toBeDefined()`, `toBeTruthy()`, `not.toBeNull()`, `expect(result).toBeDefined()`
as the only assertion on a returned value.

```ts
// Bad
expect(result).toBeDefined();

// Good
expect(result).toEqual({ id: "quiz_1", title: "Intro", durationMinutes: 30 });
```

**Why:** `toBeDefined` passes for `{}`, `0`, `"error"`, and every wrong answer.
**Not a violation when:** Existence genuinely is the behaviour (an optional field is
populated) and a stronger check would over-specify.

---

### JT-402 — No assertion at all
**Severity:** critical · **Applies to:** test files
**Detect:** An `it` body with no `expect`; a test that only calls the unit and relies on
"it did not throw" without saying so.

```ts
// Bad
it("creates a quiz", async () => { await quizService.createQuiz(dto); });

// Good
it("creates a quiz without throwing for a minimal valid payload", async () => {
  await expect(quizService.createQuiz(minimalDto)).resolves.toMatchObject({ title: minimalDto.title });
});
```

**Why:** A test with no assertion reports coverage and verifies nothing.
**Not a violation when:** Never — if "does not throw" is the behaviour, assert
`resolves.not.toThrow()` explicitly.

---

### JT-403 — Error assertion does not pin the error
**Severity:** minor · **Applies to:** test files
**Detect:** `rejects.toThrow()` with no argument, or `toThrow(Error)`.

```ts
// Bad
await expect(attemptService.submit(dto)).rejects.toThrow();

// Good
await expect(attemptService.submit(dto)).rejects.toThrow(AttemptAlreadySubmittedError);
```

**Why:** A bare `toThrow()` passes for a `TypeError` from a typo — the test goes green on a
crash.
**Not a violation when:** The error type is genuinely unspecified by the contract.

---

### JT-404 — Wrong matcher for the comparison
**Severity:** minor · **Applies to:** test files
**Detect:** `toBe` on objects/arrays (reference equality), `toEqual` where
`toStrictEqual` is meant, `toContain` on an object array where `toContainEqual` is needed.
**Why:** `toBe` on an object passes only by accident of reference sharing; `toEqual`
ignores `undefined` fields, hiding a whole class of mapper bugs.
**Not a violation when:** Reference identity is the point.

---

### JT-405 — Over-specified assertion
**Severity:** minor · **Applies to:** test files
**Detect:** `toEqual` on a whole entity including `createdAt`, `_id`, `__v` when the test
is about one field.

```ts
// Bad
expect(user).toEqual({ _id: "...", email: "...", createdAt: someDate, __v: 0, ... });

// Good
expect(user).toMatchObject({ email: "user@example.com", role: Role.Student });
```

**Why:** Over-specified assertions break on unrelated changes and train the team to update
tests without reading them.
**Not a violation when:** The full shape *is* the contract (an API response body).

---

## JT-5xx — Async and timing

### JT-501 — Async assertion not awaited
**Severity:** critical · **Applies to:** test files
**Detect:** `expect(promise).rejects.toThrow(...)` or `.resolves...` without `await` or
`return`; an async test whose last statement is an un-awaited call.

```ts
// Bad
it("rejects duplicates", () => { expect(authService.signUp(dto)).rejects.toThrow(AppError); });

// Good
it("rejects duplicates", async () => { await expect(authService.signUp(dto)).rejects.toThrow(AppError); });
```

**Why:** The test finishes before the assertion resolves — it passes unconditionally, and
the rejection may surface later as an unhandled rejection in an unrelated test.
**Not a violation when:** The promise is returned from the test body.

---

### JT-502 — `done` callback mixed with promises
**Severity:** minor · **Applies to:** test files
**Detect:** A test taking `done` while also returning a promise or using `async`.
**Why:** Jest fails or hangs on this combination, and errors before `done()` are swallowed.
**Not a violation when:** Testing a genuinely callback-based API, with `done` only.

---

### JT-503 — Real timers, sleeps, or wall-clock waits
**Severity:** major · **Applies to:** test files
**Detect:** `setTimeout` in a test, `await new Promise((r) => setTimeout(r, 500))`, an
inflated `jest.setTimeout(30000)`.

```ts
// Bad
await new Promise((resolve) => setTimeout(resolve, 1000));   // wait for the OTP to expire

// Good
jest.useFakeTimers().setSystemTime(new Date("2026-01-01T00:00:00Z"));
jest.advanceTimersByTime(OTP_TTL_MS + 1);
```

**Why:** Sleeps are the primary source of flaky suites and slow CI, and they encode a
timing guess that is wrong on a loaded machine.
**Not a violation when:** An integration test waits on a real external system with a bounded
polling helper.

---

### JT-504 — Unhandled promise rejection inside a test
**Severity:** major · **Applies to:** test files
**Detect:** Fire-and-forget calls in tests; mocks configured to reject that nothing awaits.
**Why:** The rejection lands in whichever test is running when it settles, producing
failures that move around between runs.
**Not a violation when:** The rejection is explicitly caught and asserted on.

---

## JT-6xx — Isolation and determinism

### JT-601 — Test depends on execution order
**Severity:** major · **Applies to:** test files
**Detect:** A test that reads data another test wrote; assertions on cumulative call
counts; a suite that fails under `--randomize` or when a single test is run with `-t`.
**Why:** Order-dependent suites break the moment someone adds, removes, or shards a test.
**Not a violation when:** Never — sequence, if it matters, belongs inside one test.

---

### JT-602 — Non-deterministic input not controlled
**Severity:** major · **Applies to:** test files
**Detect:** `new Date()`, `Date.now()`, `Math.random()`, `crypto.randomUUID()`, or
`faker` without a seed, used in the unit under test or in an expected value.

```ts
// Bad
expect(otp.expiresAt).toEqual(new Date(Date.now() + 600000));   // off by the test's own runtime

// Good
jest.useFakeTimers().setSystemTime(new Date("2026-01-01T00:00:00Z"));
expect(otp.expiresAt).toEqual(new Date("2026-01-01T00:10:00Z"));
```

**Why:** Non-determinism produces tests that fail once a week for nobody's reason, and the
team learns to re-run instead of read.
**Not a violation when:** The random value is irrelevant to any assertion.

---

### JT-603 — Test depends on the environment
**Severity:** major · **Applies to:** test files
**Detect:** Reads of `process.env` inside a test with no `beforeEach` override; assumptions
about locale, timezone, or the filesystem layout.

```ts
// Bad
const token = jwt.sign(payload, process.env.JWT_SECRET!);   // undefined in CI

// Good
beforeEach(() => { process.env.JWT_SECRET = "test-secret-at-least-32-characters-long"; });
afterEach(() => { delete process.env.JWT_SECRET; });
```

**Why:** "Works on my machine" is a test defect, not an environment defect.
**Not a violation when:** A committed `.env.test` or `setupFiles` sets the values
deterministically.

---

### JT-604 — Shared database state between integration tests
**Severity:** major · **Applies to:** integration tests
**Detect:** No per-test cleanup; tests relying on seed data another test created; a
`beforeAll` seed with mutations in individual tests.
**Why:** One failing test then cascades into ten, and the real failure is unfindable.
**Not a violation when:** Each test creates its own uniquely-keyed data and cleans up.

---

### JT-605 — Skipped, focused, or commented-out tests
**Severity:** major · **Applies to:** test files
**Detect:** `it.skip`, `describe.skip`, `xit`, `it.todo` left in place, and — most
serious — `it.only` / `describe.only`, which silently disables the rest of the file in CI.
**Why:** `.only` turns a full suite into one test while CI still reports green.
**Not a violation when:** `it.todo` marks planned work in a suite being actively written.

---

## JT-7xx — Coverage and missing tests

### JT-701 — Module with meaningful logic and no tests
**Severity:** major · **Applies to:** `src/**`
**Detect:** A service, calculator, mapper, guard, or validator with no corresponding spec.
Escalate to **critical** for authentication, authorization, grading, and scoring code.
Report at file level: `src/modules/auth/auth.service.ts:1 — JT-701 (critical): no tests`.
**Why:** Untested auth and grading code is where a silent regression costs the most.
**Not a violation when:** The module is a thin pass-through (a route table, a barrel, a
config object).

---

### JT-702 — Coverage chased instead of behaviour
**Severity:** minor · **Applies to:** test files
**Detect:** Tests that call every method once with no meaningful assertion (see JT-401,
JT-402); tests written per method rather than per rule; large `collectCoverageFrom`
exclusions hiding untested modules.
**Why:** Line coverage measures execution, not verification. A suite can reach 90% and
assert nothing.
**Not a violation when:** The tests state real rules and coverage follows as a by-product.

---

### JT-703 — No test for a documented failure mode
**Severity:** major · **Applies to:** test files
**Detect:** A service that throws domain errors (`NotFoundError`, `ConflictError`,
`ExpiredOtpError`) with no spec asserting each one.
**Why:** Error paths are the contract the client depends on for status codes.
**Not a violation when:** The error is a re-throw of a collaborator's error with no logic.

---

### JT-704 — Boundary and edge cases untested
**Severity:** minor · **Applies to:** test files
**Detect:** Numeric or temporal rules (pass mark, OTP TTL, attempt limits, pagination
caps) tested only in the middle of the range.

```ts
// Good
it.each([
  { score: 59.9, expected: AttemptStatus.Failed },
  { score: 60, expected: AttemptStatus.Passed },
  { score: 100, expected: AttemptStatus.Passed },
])("marks a score of $score as $expected", ({ score, expected }) => {
  expect(gradeAttempt(score)).toBe(expected);
});
```

**Why:** Off-by-one at the threshold is the defect that boundary tests exist to catch.
**Not a violation when:** The rule has no boundary.

---

### JT-705 — Controller or route tested only through mocks of everything
**Severity:** minor · **Applies to:** `*.controller.spec.ts`
**Detect:** A controller spec that mocks the service, fakes `req`/`res`, and asserts
`res.json` was called — verifying only the four lines of glue.
**Why:** Controllers are glue; the valuable test at that layer is an integration test
through `supertest` that exercises routing, validation middleware, auth, and error mapping.
**Not a violation when:** The controller contains real mapping logic worth isolating — in
which case `CC-201` may also apply.
