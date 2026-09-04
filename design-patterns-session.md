# Design Patterns — Session Handbook

> **Audience:** mid-level backend engineers (Node.js · Express · TypeScript · Mongoose · Jest)
> **Duration:** ~2 hours
> **Running example:** the Online Exam Platform you are building (`online-exam/team-*`)
> **Companion:** [`standards/clean-code.md`](./standards/clean-code.md) — patterns are how you *satisfy* those rules, not a replacement for them.

---

## Session agenda

| # | Block | Minutes | Outcome |
| --- | --- | --- | --- |
| 1 | What a design pattern is (and is not) | 10 | Shared definition + vocabulary |
| 2 | Why patterns exist — the forces behind them | 10 | Change-driven thinking, not catalog memorizing |
| 3 | The principles patterns are built on | 10 | SOLID + "encapsulate what varies" |
| 4 | The three categories + full GoF map | 10 | Mental index of 23 patterns |
| 5 | Deep dive: 14 most-used patterns | 55 | Problem → bad code → pattern → trade-offs |
| 6 | Look-alike patterns compared | 10 | Strategy vs State vs Template Method, etc. |
| 7 | Anti-patterns, overuse, how to choose | 10 | Know when *not* to use one |
| 8 | Apply it to the exam platform + exercises | 15 | Concrete refactors in their own repo |

---

# Part 1 — What is a design pattern?

## 1.1 Definition

> A **design pattern** is a named, reusable solution to a problem that recurs in a particular
> context, described together with its consequences and trade-offs.

Three words carry the weight:

- **Recurring** — it shows up again and again across projects. A one-off trick is not a pattern.
- **Context** — it solves the problem *under certain forces*. Change the forces and the pattern stops fitting.
- **Consequences** — every pattern buys something and pays for it somewhere else. A pattern without a
  stated cost is marketing, not engineering.

Origin: architect **Christopher Alexander** (*A Pattern Language*, 1977) described patterns for towns
and buildings. In 1994 the **"Gang of Four"** (Gamma, Helm, Johnson, Vlissides) applied the idea to
object-oriented software in *Design Patterns: Elements of Reusable Object-Oriented Software* — the 23
patterns everyone still refers to.

## 1.2 Anatomy of a pattern

Every catalog entry has the same shape. Use this shape when you discuss design in a review:

| Section | Question it answers |
| --- | --- |
| **Name** | What do we call it so the team knows instantly? ("Wrap it in a Decorator") |
| **Intent** | One sentence: what does it accomplish? |
| **Problem / Motivation** | What pain shows up without it? |
| **Solution / Structure** | Which participants, which relationships? |
| **Consequences** | What gets easier, what gets harder, what it costs |
| **Known uses** | Where you already meet it in real libraries |

## 1.3 What a design pattern is **not**

| Not this | Why |
| --- | --- |
| **A library or framework** | You cannot `npm install strategy`. A pattern is a shape you write yourself; a library is code someone shipped. Libraries often *implement* patterns. |
| **An algorithm** | Quicksort solves a computation. Strategy solves *how to swap* the algorithm. Algorithms answer "what steps"; patterns answer "how to structure". |
| **A finished design** | A pattern is a template. You adapt names, types, and participants to your domain. |
| **An architecture style** | Layered, Hexagonal, MVC, Microservices, Event-Driven are *architectural* patterns — system-level. GoF patterns are class/object-level. Different scale, both useful. |
| **A goal** | Nobody gets paid for "using 8 patterns". You get paid for code that survives change. |

## 1.4 Levels of patterns

```
Architecture patterns      Layered · Hexagonal (Ports & Adapters) · MVC · CQRS · Event-Driven
        ↑ system shape
Design patterns (GoF)      Strategy · Factory · Decorator · Observer · ...
        ↑ class & object shape
Idioms                     language-specific tricks: TS discriminated unions, Node module caching
```

This session is about the middle layer, with two enterprise patterns added because you use them daily:
**Repository** and **Dependency Injection**.

---

# Part 2 — Why use design patterns?

## 2.1 The five real benefits

1. **Shared vocabulary.** "Extract a Strategy for the scoring rules" replaces three minutes of
   whiteboard. Code review comments get shorter and less personal.
2. **Proven trade-offs.** Somebody already hit this wall and documented the cost. You skip a design
   iteration.
3. **Change absorption.** Almost every pattern isolates *one axis of change* so the rest of the code
   stops caring about it. That is the whole game.
4. **Testability.** Patterns push dependencies to interfaces and constructor parameters, which is
   exactly what makes a unit test possible without a real database.
5. **Onboarding speed.** A new engineer who sees `PaymentStrategy` knows the shape before reading it.

## 2.2 The forces you are actually fighting

Patterns are answers to questions like:

- What is *likely to change* here — the algorithm, the data source, the number of steps, the transport?
- Who is *allowed to know* about whom? (dependency direction)
- How do I add behaviour **without editing** a class that already works and is already tested?

If you cannot name the axis of change, you do not need a pattern yet.

## 2.3 The honest costs

| Cost | Shape it takes |
| --- | --- |
| **Indirection** | More files, more hops to read one behaviour. |
| **Cognitive load** | A junior reading 4 classes for a 3-line `if` will be slower, not faster. |
| **Premature generality** | An abstraction fitted to one imagined future usually fits the real future badly. |
| **Wrong pattern** | Costs more than no pattern — it actively misleads readers about intent. |

## 2.4 The rule of thumb

> Write the simple thing first. When the **second** reason to change arrives, refactor toward the
> pattern that isolates that change. Patterns are usually a **refactoring destination**, not a
> starting blueprint.

Exception: patterns you already know your system needs on day one (Repository at the data boundary,
DI at the composition root, Middleware chain in Express) — those are cheap and standard.

---

# Part 3 — The principles patterns are built on

Patterns are consequences of a handful of principles. Learn the principles and you can *derive* the
pattern instead of memorizing it.

## 3.1 Encapsulate what varies

Find what changes, put it behind a stable interface, and let the rest of the code depend on the
interface. Every creational pattern varies *what gets created*; every behavioural pattern varies
*what gets executed*; every structural pattern varies *what gets connected*.

## 3.2 Program to an interface, not an implementation

```ts
// Bad — service is welded to Mongoose and to nodemailer
class AuthService {
  async register(dto: RegisterDto) {
    const user = await UserModel.create(dto);      // Mongoose, hard-wired
    await nodemailer.createTransport(...).sendMail(...); // SMTP, hard-wired
  }
}

// Good — service depends on capabilities it declares
class AuthService {
  constructor(
    private readonly users: UserRepository,   // interface
    private readonly mailer: Mailer,          // interface
  ) {}
  async register(dto: RegisterDto) {
    const user = await this.users.create(dto);
    await this.mailer.send(welcomeEmail(user));
  }
}
```

The second version is unit-testable with plain fakes, and swapping SMTP for SendGrid touches one file.

## 3.3 Favor composition over inheritance

> **GoF** = the *Gang of Four*: Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides — the four
> authors of *Design Patterns: Elements of Reusable Object-Oriented Software* (1994). "GoF patterns"
> means the 23 patterns catalogued in that book.

**Inheritance** (`class B extends A`) says *B is an A*. It is decided at compile time, allows one
parent, and every member of the parent — including the ones you did not want — becomes part of B.

**Composition** (`class B { constructor(private readonly a: A) {} }`) says *B has an A*. It is decided
at runtime, allows any number of collaborators, and B exposes only what it chooses to expose.

### 3.3.1 What goes wrong with inheritance

**a) Subclass explosion.** Behaviours that combine multiply as classes.

```ts
// Notifications: 3 channels × 2 concerns (retry, audit) = 12 classes, and the next concern doubles it
class EmailNotifier {}
class RetryingEmailNotifier extends EmailNotifier {}
class AuditedEmailNotifier extends EmailNotifier {}
class AuditedRetryingEmailNotifier extends RetryingEmailNotifier {}   // …and again for SMS, push
```

With composition the concerns are orthogonal, so they add instead of multiply — 3 channels + 2
wrappers = 5 classes, any combination available at runtime (this is the **Decorator** pattern):

```ts
const notifier = new AuditedNotifier(new RetryingNotifier(new EmailNotifier(config), 3), auditLog);
```

**b) The fragile base class problem.** A subclass depends on the base class's *internals*, not just its
signatures. Change the base and subclasses break silently.

```ts
class BaseRepository {
  async findById(id: string) { return this.model.findById(id).lean(); }
  async findMany(filter: object) { return this.model.find(filter).lean(); }
}

class QuizRepository extends BaseRepository {
  // Relies on findMany() internally calling .lean(). The day the base adds .populate(),
  // this subclass silently starts returning a different shape. Nothing in the types changed.
  async findPublished() { return this.findMany({ status: 'published' }); }
}
```

**c) You inherit everything, including what you must not have.** A subclass that must reject a
parent's method violates the **L**iskov Substitution Principle — the code no longer works when you
substitute the child for the parent.

```ts
class Attempt { submit() { /* … */ } }
class SubmittedAttempt extends Attempt {
  submit(): never { throw new AlreadySubmittedError(); }   // breaks every caller holding an Attempt
}
```

**d) One parent only.** Real requirements are multi-axis (channel × formatting × transport). A single
inheritance chain can encode one axis; composition encodes all of them (this is **Bridge**).

**e) It welds construction to behaviour.** A subclass is chosen by `new`, at compile time. A collaborator
is chosen by whoever wires the object — so it can differ per environment, per tenant, per test.

### 3.3.2 Side by side

| | Inheritance (`extends`) | Composition (constructor collaborator) |
| --- | --- | --- |
| Relationship | *is-a* | *has-a* / *uses-a* |
| Bound at | Compile time | Runtime |
| How many | One parent | Any number |
| Coupling | To the parent's internals | To the collaborator's interface |
| Swap in a test | Only by subclassing again | Pass a fake in the constructor |
| Adding a behaviour | New subclass per combination (multiplies) | New wrapper/collaborator (adds) |
| Reuse unit | Whole class | One capability |

### 3.3.3 Where this shows up in the catalog

Most GoF patterns are, structurally, "replace this subclass with an object you hold":

| Instead of a subclass… | Use | Because |
| --- | --- | --- |
| `NegativeScoringService extends ScoringService` | **Strategy** | The algorithm must change at runtime |
| `CachingQuizRepository extends MongooseQuizRepository` | **Decorator** | Concerns must combine freely |
| `SendGridNotifier extends Notifier` where the SDK shape differs | **Adapter** | The two interfaces do not match |
| `SmsEmailPushNotifier` hierarchies crossing two axes | **Bridge** | Two axes vary independently |
| Subclass per lifecycle status | **State** | Behaviour follows a value, not a type |

### 3.3.4 When inheritance IS the right call

Composition is the default, not a ban. Inheritance fits when:

- **The skeleton is real and stable, and only steps vary** — that is exactly **Template Method**
  (see 5.14): `ReportGenerator` with abstract `fetchData()` / `render()`.
- **A framework requires it** — `class AppError extends Error`, Express error classes, custom Mongoose
  types. Fighting that adds noise for nothing.
- **The subtype genuinely satisfies LSP** — anywhere the parent works, the child works, with no method
  that throws "not supported".
- Keep it **shallow**: two levels is the practical limit. Three or more and no one can predict
  behaviour without reading the whole chain.

### 3.3.5 TypeScript specifics

- `implements` gives you the contract with **zero** inherited implementation — the safe half of
  inheritance. Prefer `class X implements Mailer` over `class X extends BaseMailer`.
- TypeScript is **structurally** typed: any object with the right shape satisfies an interface, so a
  test fake needs no base class and no `extends`.
- A stateless strategy can just be a **function** — composition without a class at all:
  `type ScoringStrategy = (answers: Answer[]) => number`.
- Shared behaviour with no "is-a" relationship: use a plain helper module or a mixin, not a base class.

> **Rule:** reach for `extends` only when you would defend the sentence "a B *is a* A, everywhere, with
> no exceptions". Otherwise hold the object instead of inheriting it.

## 3.4 SOLID in one table

| Principle | One line | Patterns that deliver it |
| --- | --- | --- |
| **S**ingle Responsibility | One reason to change per module | Facade, Repository, Command |
| **O**pen/Closed | Open to extension, closed to modification | Strategy, Decorator, Chain of Responsibility |
| **L**iskov Substitution | Any implementation must be usable through the interface without surprises | All polymorphic patterns depend on it |
| **I**nterface Segregation | Many small interfaces beat one fat one | Adapter, Repository split by use case |
| **D**ependency Inversion | High-level policy must not depend on low-level detail | DI, Repository, Adapter, Factory |

## 3.5 The dependency rule (clean architecture)

```
controller  →  service (domain policy)  →  repository interface
                                                 ↑ implemented by
                                          Mongoose repository  (infrastructure)
```

Dependencies point **inward**. The domain never imports Express, Mongoose, or nodemailer. Patterns are
the tools that let you honour that arrow.

---

# Part 4 — The three categories

| Category | Question it answers | GoF members |
| --- | --- | --- |
| **Creational** | *How is an object created?* Decouple code from concrete constructors. | Factory Method, Abstract Factory, Builder, Prototype, Singleton |
| **Structural** | *How are objects composed?* Assemble parts into bigger structures without rigid coupling. | Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy |
| **Behavioral** | *How do objects interact and share responsibility?* | Chain of Responsibility, Command, Interpreter, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor |

## 4.1 Full GoF map — one line each

### Creational

| Pattern | Use when |
| --- | --- |
| **Factory Method** | A subclass/config decides which concrete class to instantiate. |
| **Abstract Factory** | You need *families* of related objects that must stay consistent (all-Postgres or all-Mongo). |
| **Builder** | An object needs many optional parts, or step-by-step assembly (query builders). |
| **Prototype** | Copying an existing configured object is cheaper/safer than rebuilding it. |
| **Singleton** | Exactly one instance must exist and be globally reachable — the most abused pattern in the list. |

### Structural

| Pattern | Use when |
| --- | --- |
| **Adapter** | Two incompatible interfaces must work together (third-party SDK vs your port). |
| **Bridge** | Abstraction and implementation must vary independently (notification type × channel). |
| **Composite** | Clients should treat single items and trees of items uniformly (nested question groups). |
| **Decorator** | Add behaviour to one object at runtime without touching its class (caching, logging, retry). |
| **Facade** | A simple entry point over a complicated subsystem. |
| **Flyweight** | Thousands of objects share immutable state; store it once (rare in typical web APIs). |
| **Proxy** | Same interface, but control access: lazy load, cache, permissions, remote call. |

### Behavioral

| Pattern | Use when |
| --- | --- |
| **Chain of Responsibility** | A request passes through handlers until one deals with it (Express middleware). |
| **Command** | Turn a request into an object: queue it, log it, retry it, undo it (job queues). |
| **Interpreter** | You have a small language/grammar to evaluate (filter expressions). |
| **Iterator** | Traverse a collection without exposing its internals (JS `Symbol.iterator`, generators). |
| **Mediator** | Many components talk to each other; centralize the protocol instead of an N×N mesh. |
| **Memento** | Capture and restore state without breaking encapsulation (undo, draft snapshots). |
| **Observer** | One-to-many notification when state changes (`EventEmitter`, webhooks). |
| **State** | Behaviour changes with an internal state, and `switch` blocks are multiplying (attempt lifecycle). |
| **Strategy** | Interchangeable algorithms picked at runtime (scoring, pricing, auth). |
| **Template Method** | Fixed algorithm skeleton, variable steps supplied by subclasses. |
| **Visitor** | New operations must be added over a stable object structure without editing it. |

## 4.2 Two non-GoF patterns you use every day

| Pattern | Use when |
| --- | --- |
| **Repository** | Isolate persistence behind a collection-like interface so the domain never sees Mongoose. |
| **Dependency Injection** | Supply collaborators from outside instead of constructing them inside. |

---

# Part 5 — Deep dive: the 15 patterns you will actually use

Each entry follows the same shape: **Intent → Problem → Solution → When to use / When not → Known uses → Common mistakes.**

---

## 5.1 Strategy *(Behavioral)*

**Intent:** Define a family of interchangeable algorithms, encapsulate each one, and make them
swappable at runtime.

### Problem

Quiz scoring differs per quiz type: plain correct-count, negative marking for wrong answers,
weighted-by-difficulty. The naive version grows a `switch` and a new reason to edit a tested class
every time the business invents a rule.

```ts
// Bad — one class, many reasons to change (violates SRP and Open/Closed)
class ScoringService {
  score(quiz: Quiz, answers: Answer[]): number {
    switch (quiz.scoringType) {
      case 'simple':
        return answers.filter((a) => a.isCorrect).length;
      case 'negative': {
        const correct = answers.filter((a) => a.isCorrect).length;
        const wrong = answers.length - correct;
        return correct - wrong * 0.25;
      }
      case 'weighted':
        return answers.reduce((sum, a) => (a.isCorrect ? sum + a.question.weight : sum), 0);
      default:
        throw new Error(`Unknown scoring type: ${quiz.scoringType}`);
    }
  }
}
```

### Solution

```ts
// domain/scoring/scoring-strategy.ts
export interface ScoringStrategy {
  score(answers: Answer[]): number;
}

export class SimpleScoring implements ScoringStrategy {
  score(answers: Answer[]): number {
    return answers.filter((answer) => answer.isCorrect).length;
  }
}

export class NegativeMarkingScoring implements ScoringStrategy {
  constructor(private readonly penaltyPerWrongAnswer = 0.25) {}
  score(answers: Answer[]): number {
    const correct = answers.filter((answer) => answer.isCorrect).length;
    const wrong = answers.length - correct;
    return correct - wrong * this.penaltyPerWrongAnswer;
  }
}

export class WeightedScoring implements ScoringStrategy {
  score(answers: Answer[]): number {
    return answers.reduce((total, answer) => (answer.isCorrect ? total + answer.question.weight : total), 0);
  }
}

// The context depends on the interface only.
export class ScoringService {
  constructor(private readonly strategy: ScoringStrategy) {}
  score(answers: Answer[]): number {
    return this.strategy.score(answers);
  }
}
```

Selecting a strategy is a *lookup*, not a `switch` spread through the codebase:

```ts
const SCORING_STRATEGIES: Record<ScoringType, ScoringStrategy> = {
  simple: new SimpleScoring(),
  negative: new NegativeMarkingScoring(),
  weighted: new WeightedScoring(),
};

export const scoringStrategyFor = (type: ScoringType): ScoringStrategy => SCORING_STRATEGIES[type];
```

In TypeScript a strategy can be a plain function when it has no state:

```ts
export type ScoringStrategy = (answers: Answer[]) => number;
export const simpleScoring: ScoringStrategy = (answers) => answers.filter((a) => a.isCorrect).length;
```

### Structure

```
Client ──▶ Context ──▶ «interface» Strategy
                          ▲      ▲       ▲
                    StrategyA StrategyB StrategyC
```

### When to use
- Several variants of one algorithm, chosen at runtime by config, tenant, or data.
- You want each variant unit-tested in isolation (a strategy has no HTTP, no DB — pure input/output).
- A `switch`/`if-else` chain over a "type" field keeps gaining branches.

### When not to use
- Two branches that will never grow. An `if` is cheaper and honest.
- Variants differ in *data only* — use a config object, not three classes.

### Known uses
`passport` authentication strategies · `bcrypt` vs `argon2` hashers behind one `PasswordHasher` ·
Express `body-parser` per content type · sorting comparators.

### Common mistakes
- Leaking the strategy choice everywhere — resolve it once, at the edge.
- Strategies with different method signatures. If they cannot substitute, it is not a Strategy.
- Passing the whole `Request` into a strategy — pass the domain input it needs.

---

## 5.2 Factory Method *(Creational)*

**Intent:** Define an interface for creating an object, but let the choice of concrete class live in
one place instead of scattered `new` calls.

### Problem

```ts
// Bad — every caller must know every concrete class and how to configure it
const notifier =
  channel === 'email'
    ? new EmailNotifier(process.env.SMTP_URL!, templates)
    : channel === 'sms'
      ? new SmsNotifier(process.env.TWILIO_SID!, process.env.TWILIO_TOKEN!)
      : new PushNotifier(firebaseApp);
```

Every new channel means editing every call site — and construction details bleed into business code.

### Solution

```ts
export interface Notifier {
  notify(userId: string, message: NotificationMessage): Promise<void>;
}

export class NotifierFactory {
  constructor(private readonly config: NotificationConfig) {}

  create(channel: NotificationChannel): Notifier {
    switch (channel) {
      case 'email':
        return new EmailNotifier(this.config.smtpUrl);
      case 'sms':
        return new SmsNotifier(this.config.twilio);
      case 'push':
        return new PushNotifier(this.config.firebase);
      default:
        throw new UnsupportedChannelError(channel);
    }
  }
}
```

The `switch` did not disappear — it got **confined to one file** that exists for exactly this purpose.
That is the win: one reason to change, one place to change it.

> **Naming note.** *Factory Method* (GoF) is a method a subclass overrides to decide the concrete
> class. A class whose only job is `create(...)` is usually called a *Simple Factory* / *Static
> Factory* — not in the GoF book, universally used, and the version you will write most.

### When to use
- Construction needs configuration, secrets, or several collaborators.
- The concrete type depends on runtime input (channel, tenant, provider, file format).
- Tests need to build valid domain objects without repeating setup.

### When not to use
- The class has a trivial constructor and one implementation. `new Thing()` is fine.

### Known uses
`mongoose.model()` · `express()` · `createServer()` · `jwt.sign` provider selection · test data factories.

### Common mistakes
- A factory that returns `any` — you throw away the type safety you built the interface for.
- Factories that also *do work* (send the email as well as create the sender) — that is a service.

---

## 5.3 Builder *(Creational)*

**Intent:** Construct a complex object step by step, so the same construction process can produce
different representations — and invalid intermediate states never escape.

### Problem

```ts
// Bad — telescoping parameters; nobody can read the call site
const quiz = new Quiz('Math 101', 30, true, false, 10, undefined, ['algebra'], 0.25);
```

### Solution

```ts
export class QuizBuilder {
  private readonly draft: Partial<QuizProps> = { tags: [], questions: [] };

  withTitle(title: string): this {
    this.draft.title = title;
    return this;
  }

  withDuration(minutes: number): this {
    this.draft.durationMinutes = minutes;
    return this;
  }

  withQuestion(question: Question): this {
    this.draft.questions = [...(this.draft.questions ?? []), question];
    return this;
  }

  shuffled(): this {
    this.draft.shuffleQuestions = true;
    return this;
  }

  build(): Quiz {
    if (!this.draft.title) throw new InvalidQuizError('title is required');
    if (!this.draft.questions?.length) throw new InvalidQuizError('a quiz needs at least one question');
    return new Quiz(this.draft as QuizProps);   // validated once, at the boundary
  }
}

const quiz = new QuizBuilder()
  .withTitle('Math 101')
  .withDuration(30)
  .withQuestion(question1)
  .shuffled()
  .build();
```

**Where Builder earns its keep in tests** — the Object Mother / test-data-builder idiom kills setup
duplication in Jest specs:

```ts
const aQuiz = (overrides: Partial<QuizProps> = {}): Quiz =>
  new Quiz({ title: 'Sample', durationMinutes: 10, questions: [aQuestion()], ...overrides });

it('rejects submission after the deadline', () => {
  const quiz = aQuiz({ durationMinutes: 1 });
  // ...
});
```

### When to use
- Many optional parameters, or parameters that only make sense in combination.
- Object must be validated as a whole before it exists.
- Fluent, readable construction matters (queries, test fixtures, HTTP clients).

### When not to use
- 2–3 required fields. An options object literal (`new Quiz({ title, durationMinutes })`) is simpler
  and TypeScript already names the fields for you.

### Known uses
`mongoose.Query` (`find().where().sort().limit()`) · Knex/TypeORM query builders · `supertest` request chains.

### Common mistakes
- A builder with no validation in `build()` — then it is just a verbose object literal.
- Reusing one builder instance across tests: mutable shared state leaks between cases.

---

## 5.4 Singleton *(Creational — handle with care)*

**Intent:** Guarantee exactly one instance of a class, with a global access point.

### The legitimate uses
Database connection pool, application config, logger, cache client. Things that are genuinely one per
process and expensive to create.

```ts
// Node modules are cached, so a module-level instance is already a singleton — idiomatic and enough.
// src/config/database.ts
let connection: Connection | null = null;

export const connectDatabase = async (uri: string): Promise<Connection> => {
  if (connection) return connection;
  connection = await mongoose.createConnection(uri).asPromise();
  return connection;
};
```

Classic form, for reference:

```ts
export class Logger {
  private static instance: Logger | null = null;
  private constructor(private readonly level: LogLevel) {}

  static getInstance(): Logger {
    Logger.instance ??= new Logger(currentLogLevel());
    return Logger.instance;
  }
}
```

### Why it is the most criticized pattern
- **Hidden dependency.** `Logger.getInstance()` inside a service means the signature lies about what
  the service needs.
- **Untestable.** Global state survives between Jest tests; one test's mutation breaks the next.
- **Concurrency & scaling.** "One per process" is not "one per system" once you run 4 pods.

### The better default

```ts
// Create it once at the composition root, then INJECT it. Still one instance — no global reach-in.
const logger = createLogger(config);
const quizService = new QuizService(quizRepository, logger);
```

### When to use
- Genuinely process-wide, stateless-or-immutable resources, injected rather than reached for.

### When not to use
- To share mutable domain state. That is a global variable wearing a suit.
- Anywhere a unit test would need a different instance.

### Common mistakes
- Singleton + mutable state + tests = flaky suite. If you must, expose a `reset()` for tests.
- Using it to dodge wiring effort. The wiring is the cheap part; the coupling is the expensive part.

---

## 5.5 Repository *(Enterprise pattern — your data boundary)*

**Intent:** Mediate between the domain and data mapping, exposing a collection-like interface so the
domain never knows a database exists.

### Problem

```ts
// Bad — service is a Mongoose client; unit tests need a real (or heavily mocked) DB
class QuizService {
  async publish(quizId: string) {
    const quiz = await QuizModel.findById(quizId).populate('questions').lean();
    if (!quiz) throw new NotFoundError('quiz');
    await QuizModel.updateOne({ _id: quizId }, { $set: { status: 'published' } });
  }
}
```

### Solution

```ts
// domain — owns the interface (the "port")
export interface QuizRepository {
  findById(id: QuizId): Promise<Quiz | null>;
  save(quiz: Quiz): Promise<void>;
  findPublishedByAuthor(authorId: UserId): Promise<Quiz[]>;
}

// infrastructure — owns the Mongoose detail (the "adapter")
export class MongooseQuizRepository implements QuizRepository {
  constructor(private readonly model: Model<QuizDocument>) {}

  async findById(id: QuizId): Promise<Quiz | null> {
    const document = await this.model.findById(id).lean();
    return document ? toDomain(document) : null;
  }

  async save(quiz: Quiz): Promise<void> {
    await this.model.updateOne({ _id: quiz.id }, { $set: toPersistence(quiz) }, { upsert: true });
  }

  async findPublishedByAuthor(authorId: UserId): Promise<Quiz[]> {
    const documents = await this.model.find({ authorId, status: 'published' }).lean();
    return documents.map(toDomain);
  }
}

// service — pure policy, trivially testable
export class QuizService {
  constructor(private readonly quizzes: QuizRepository) {}

  async publish(quizId: QuizId): Promise<void> {
    const quiz = await this.quizzes.findById(quizId);
    if (!quiz) throw new NotFoundError('quiz');
    quiz.publish();                 // domain rule lives in the entity
    await this.quizzes.save(quiz);
  }
}
```

Test with a fake, no mocks of Mongoose internals:

```ts
class InMemoryQuizRepository implements QuizRepository {
  private readonly store = new Map<QuizId, Quiz>();
  async findById(id: QuizId) { return this.store.get(id) ?? null; }
  async save(quiz: Quiz) { this.store.set(quiz.id, quiz); }
  async findPublishedByAuthor(authorId: UserId) {
    return [...this.store.values()].filter((q) => q.authorId === authorId && q.isPublished);
  }
}
```

### When to use
- Any service with business rules that also touches persistence — which is most of your modules.
- You want unit tests that run in milliseconds without `mongodb-memory-server`.

### When not to use
- Thin CRUD proxies with zero domain logic — a repository that only forwards `findById` adds a layer
  for nothing. Be honest about which modules have rules.

### Common mistakes
- **Leaking the ODM through the interface**: `findById(): Promise<QuizDocument>` or a method returning
  a Mongoose `Query`. Now the domain depends on Mongoose again and the boundary is fake.
- Repository methods that take Mongo filter objects (`find(filter: FilterQuery<Quiz>)`) — same leak.
- Putting business rules inside the repository. Repositories store and retrieve; services decide.

---

## 5.6 Dependency Injection *(Enterprise pattern — how the pieces meet)*

**Intent:** Give an object its collaborators from outside instead of letting it construct or look them
up itself. It is the mechanism that makes almost every other pattern usable.

```ts
// Bad — hidden, unswappable dependencies
class AuthService {
  private readonly users = new MongooseUserRepository(UserModel);
  private readonly mailer = new SmtpMailer(process.env.SMTP_URL!);
}

// Good — declared dependencies, swappable in tests and in production
class AuthService {
  constructor(
    private readonly users: UserRepository,
    private readonly mailer: Mailer,
    private readonly tokens: TokenIssuer,
  ) {}
}
```

**Composition root** — the single place where the object graph is wired (`src/app.ts` or
`src/container.ts`). Everything else stays ignorant of concrete classes.

```ts
// src/container.ts
export const buildContainer = (config: AppConfig) => {
  const users = new MongooseUserRepository(UserModel);
  const mailer = new SmtpMailer(config.smtpUrl);
  const tokens = new JwtTokenIssuer(config.jwtSecret);
  const authService = new AuthService(users, mailer, tokens);
  return { authService };
};
```

### When to use
Everywhere a class needs an I/O collaborator. No framework required — constructor parameters are DI.

### Common mistakes
- Injecting the container itself (Service Locator) — dependencies become hidden again.
- Injecting *everything*, including pure functions and value objects. Inject boundaries, not helpers.

---

## 5.7 Adapter *(Structural)*

**Intent:** Convert the interface of an existing class into the interface a client expects — making
incompatible things work together.

### Problem

Your domain wants `Mailer.send(email)`. SendGrid's SDK wants
`sgMail.send({ to, from, subject, html, dynamicTemplateData })`. Sprinkling SDK calls through services
welds your business logic to a vendor.

### Solution

```ts
// domain port
export interface Mailer {
  send(email: OutgoingEmail): Promise<void>;
}

// infrastructure adapter
export class SendGridMailer implements Mailer {
  constructor(private readonly client: MailService) {}

  async send(email: OutgoingEmail): Promise<void> {
    await this.client.send({
      to: email.to,
      from: email.from,
      subject: email.subject,
      html: email.body,
    });
  }
}

// swapping vendors = one new adapter, zero domain edits
export class ConsoleMailer implements Mailer {
  async send(email: OutgoingEmail): Promise<void> {
    console.info(`[mail] ${email.subject} -> ${email.to}`);
  }
}
```

### When to use
- Wrapping any third-party SDK, legacy module, or external API.
- Two subsystems with the same concept but different vocabularies (`user_id` vs `userId`).

### When not to use
- You control both sides and can simply change one interface.

### Known uses
Every "Ports & Adapters" (hexagonal) architecture · payment gateway wrappers · storage drivers
(S3 / local disk behind `FileStorage`).

### Common mistakes
- An adapter that also adds behaviour (retry, caching) — that is a Decorator; keep them separate.
- Passing vendor types through the port (`send(msg: SgMailData)`) — the adapter then adapts nothing.

---

## 5.8 Facade *(Structural)*

**Intent:** Provide one simple interface over a complicated subsystem.

### Problem

The controller orchestrates six collaborators to submit an exam attempt: load attempt, validate
deadline, score answers, persist, update dashboard stats, send a notification. Controllers should
translate HTTP, not run business workflows.

### Solution

```ts
// One entry point; the subsystem stays decomposed behind it.
export class ExamSubmissionFacade {
  constructor(
    private readonly attempts: AttemptRepository,
    private readonly scoring: ScoringService,
    private readonly dashboard: DashboardStatsUpdater,
    private readonly notifier: Notifier,
    private readonly clock: Clock,
  ) {}

  async submit(command: SubmitAttemptCommand): Promise<AttemptResult> {
    const attempt = await this.attempts.findById(command.attemptId);
    if (!attempt) throw new NotFoundError('attempt');
    attempt.ensureNotExpired(this.clock.now());

    const result = this.scoring.score(command.answers);
    attempt.complete(result);
    await this.attempts.save(attempt);

    await this.dashboard.recordAttempt(attempt);
    await this.notifier.notify(attempt.studentId, resultMessage(result));
    return result;
  }
}

// controller shrinks to HTTP translation
export const submitAttempt = async (req: Request, res: Response, next: NextFunction) => {
  try {
    const result = await submissionFacade.submit(toCommand(req));
    res.status(200).json({ success: true, data: result });
  } catch (error) {
    next(error);
  }
};
```

### When to use
- A common use case requires several subsystem calls in a fixed order.
- You want to keep the subsystem's parts independently testable while giving callers one door.

### When not to use
- The "subsystem" is one class. A facade over one thing is a pointless hop.

### Known uses
`mongoose.connect()` over the driver · your `app.ts` bootstrap · SDK "client" objects.

### Common mistakes
- The facade grows into a God object that owns all logic. It **orchestrates**, it does not implement.
- Adding a facade *and* keeping callers who reach into the subsystem directly — pick one door.

---

## 5.9 Decorator *(Structural)*

**Intent:** Attach responsibilities to an object dynamically, keeping the same interface. The
alternative to a subclass explosion.

### Problem

You want caching, logging, and retry on the quiz repository. Subclassing gives you
`CachedLoggedRetryingQuizRepository`, and every combination is a new class.

### Solution

```ts
// Each decorator implements the SAME interface and wraps another instance of it.
export class CachingQuizRepository implements QuizRepository {
  constructor(
    private readonly inner: QuizRepository,
    private readonly cache: Cache,
  ) {}

  async findById(id: QuizId): Promise<Quiz | null> {
    const cached = await this.cache.get<Quiz>(`quiz:${id}`);
    if (cached) return cached;

    const quiz = await this.inner.findById(id);
    if (quiz) await this.cache.set(`quiz:${id}`, quiz, { ttlSeconds: 60 });
    return quiz;
  }

  save(quiz: Quiz): Promise<void> { return this.inner.save(quiz); }
  findPublishedByAuthor(authorId: UserId): Promise<Quiz[]> { return this.inner.findPublishedByAuthor(authorId); }
}

export class LoggingQuizRepository implements QuizRepository {
  constructor(private readonly inner: QuizRepository, private readonly logger: Logger) {}

  async findById(id: QuizId): Promise<Quiz | null> {
    this.logger.debug('quiz.findById', { id });
    return this.inner.findById(id);
  }
  // ...delegate the rest
}

// Compose at the composition root — order is explicit and swappable.
const repository = new LoggingQuizRepository(
  new CachingQuizRepository(new MongooseQuizRepository(QuizModel), redisCache),
  logger,
);
```

### When to use
- Cross-cutting concerns: caching, logging, metrics, retry, rate limiting, authorization.
- Behaviour must be composable and optional per environment (no cache in tests).

### When not to use
- Only one combination will ever exist — put it in the class.
- The added behaviour changes the contract (different return type) — that is not a Decorator.

### Known uses
Express middleware wrapping handlers · `React.memo` · TypeScript `@decorators` (a language feature
inspired by, but not identical to, the pattern) · Node stream `pipe` chains.

### Common mistakes
- Forgetting to delegate a method — silent behaviour loss. Let the compiler help: implement the interface.
- Deep stacks (5+ layers) — debugging becomes archaeology.

---

## 5.10 Proxy *(Structural)*

**Intent:** Same interface as the real object, but the proxy controls access to it.

Four common flavours:

| Flavour | Purpose |
| --- | --- |
| **Virtual** | Delay expensive creation/loading until first use (lazy DB connection). |
| **Protection** | Enforce permissions before delegating (only quiz author may edit). |
| **Remote** | Local stand-in for a remote service (HTTP client with the domain interface). |
| **Caching** | Serve stored results instead of recomputing. |

```ts
export class AuthorizedQuizRepository implements QuizRepository {
  constructor(
    private readonly inner: QuizRepository,
    private readonly currentUser: UserContext,
  ) {}

  async findById(id: QuizId): Promise<Quiz | null> {
    const quiz = await this.inner.findById(id);
    if (quiz && !quiz.isVisibleTo(this.currentUser)) throw new ForbiddenError();
    return quiz;
  }
  // ...
}
```

**Decorator vs Proxy** — structurally identical, different *intent*: a Decorator **adds** behaviour a
client asked for; a Proxy **controls** access to something the client thinks it is using directly.

JavaScript also ships a literal `Proxy` object for interception:

```ts
const auditedConfig = new Proxy(config, {
  get(target, key: string) {
    logger.debug('config.read', { key });
    return target[key as keyof AppConfig];
  },
});
```

### Common mistakes
- Using a Proxy to hide N+1 queries — lazy loading that fires a query per property access.

---

## 5.11 Chain of Responsibility *(Behavioral)*

**Intent:** Pass a request along a chain of handlers; each either handles it or forwards it.

**You already use it**: Express middleware *is* this pattern.

```ts
// Each handler: do its bit, then call next() — or end the chain.
export const authenticate = (req: Request, res: Response, next: NextFunction) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return next(new UnauthorizedError('missing token'));
  req.user = tokenIssuer.verify(token);
  next();
};

export const authorize = (...roles: Role[]) => (req: Request, res: Response, next: NextFunction) => {
  if (!roles.includes(req.user.role)) return next(new ForbiddenError());
  next();
};

router.post('/quizzes', authenticate, authorize('instructor'), validate(createQuizSchema), createQuiz);
```

Domain-level version — submission validation rules that must run in order and can be reconfigured:

```ts
export interface SubmissionRule {
  check(attempt: Attempt, next: () => void): void;
}

export class NotExpiredRule implements SubmissionRule {
  constructor(private readonly clock: Clock) {}
  check(attempt: Attempt, next: () => void): void {
    if (attempt.deadline < this.clock.now()) throw new AttemptExpiredError();
    next();
  }
}
```

### When to use
- A pipeline of independent checks/transforms whose order and membership can change.
- Different routes need different subsets of the same steps.

### When not to use
- A fixed pair of checks — two `if`s read better than a chain.

### Common mistakes
- Forgetting `next()` — the request hangs until timeout. Classic Express bug.
- Handlers that know about each other. Each link must be independent.

---

## 5.12 Observer *(Behavioral)*

**Intent:** One-to-many dependency — when a subject changes state, all its observers are notified
automatically.

### Problem

`AttemptService.submit()` also updates dashboard stats, emails the student, and writes an audit log.
Three unrelated reasons to edit the submit method, and its unit test needs three mocks.

### Solution

```ts
export type DomainEvent = { name: 'attempt.submitted'; payload: { attemptId: string; score: number } };

export interface EventBus {
  publish(event: DomainEvent): Promise<void>;
  subscribe(name: DomainEvent['name'], handler: (event: DomainEvent) => Promise<void>): void;
}

// Node's EventEmitter is an Observer implementation you already have.
export class InProcessEventBus implements EventBus {
  private readonly emitter = new EventEmitter();

  async publish(event: DomainEvent): Promise<void> { this.emitter.emit(event.name, event); }
  subscribe(name: DomainEvent['name'], handler: (event: DomainEvent) => Promise<void>): void {
    this.emitter.on(name, (event) => void handler(event).catch((error) => logger.error(error)));
  }
}

// Producer knows nothing about consumers.
await this.events.publish({ name: 'attempt.submitted', payload: { attemptId, score } });

// Consumers register themselves at the composition root.
events.subscribe('attempt.submitted', updateDashboardStats);
events.subscribe('attempt.submitted', emailResultToStudent);
```

### When to use
- Side effects that are genuinely optional to the core operation.
- Several independent reactions to one fact, likely to grow.

### When not to use
- The reaction is **required** for correctness (must run in the same transaction). Events make control
  flow implicit; failures become invisible. Call it directly instead.

### Known uses
Node `EventEmitter` · Mongoose middleware hooks (`pre('save')`) · WebSocket broadcast · webhooks.

### Common mistakes
- Unbounded listeners → memory leak (`MaxListenersExceededWarning`). Unsubscribe.
- Swallowing errors in handlers, so a failed email silently loses data.
- Event chains 4 deep: nobody can answer "what happens when a student submits?"

---

## 5.13 Command *(Behavioral)*

**Intent:** Encapsulate a request as an object — so it can be passed, queued, logged, retried, undone.

```ts
export interface Command<TResult = void> {
  execute(): Promise<TResult>;
}

export class GradeAttemptCommand implements Command<AttemptResult> {
  constructor(
    private readonly attemptId: AttemptId,
    private readonly attempts: AttemptRepository,
    private readonly scoring: ScoringService,
  ) {}

  async execute(): Promise<AttemptResult> {
    const attempt = await this.attempts.findById(this.attemptId);
    if (!attempt) throw new NotFoundError('attempt');
    const result = this.scoring.score(attempt.answers);
    attempt.complete(result);
    await this.attempts.save(attempt);
    return result;
  }
}
```

Because the request is an object, infrastructure can treat all requests uniformly:

```ts
class RetryingInvoker {
  constructor(private readonly attempts = 3) {}
  async run<T>(command: Command<T>): Promise<T> {
    let lastError: unknown;
    for (let i = 0; i < this.attempts; i++) {
      try { return await command.execute(); } catch (error) { lastError = error; }
    }
    throw lastError;
  }
}
```

### When to use
- Background jobs and queues (BullMQ payload = a serialized command).
- Undo/redo, audit trails, request replay.
- CQRS-style handlers: `CreateQuizCommand` → `CreateQuizHandler`.

### When not to use
- A plain method call with no queuing, logging, or undo requirement.

### Common mistakes
- Commands that hold live DB handles but get serialized to a queue — serialize *data*, resolve
  dependencies on the worker side.

---

## 5.14 Template Method *(Behavioral)*

**Intent:** Define the skeleton of an algorithm in a base class, letting subclasses override specific
steps without changing the structure.

```ts
export abstract class ReportGenerator {
  // The template method: fixed order, final.
  async generate(userId: UserId): Promise<Buffer> {
    const data = await this.fetchData(userId);
    const rows = this.transform(data);
    return this.render(rows);
  }

  protected abstract fetchData(userId: UserId): Promise<RawStats>;
  protected abstract render(rows: ReportRow[]): Promise<Buffer>;

  // Shared default step — a "hook" a subclass may override.
  protected transform(data: RawStats): ReportRow[] {
    return data.attempts.map(toReportRow);
  }
}

export class PdfReportGenerator extends ReportGenerator {
  protected fetchData(userId: UserId) { return this.stats.forStudent(userId); }
  protected render(rows: ReportRow[]) { return this.pdf.render(rows); }
}
```

**Template Method vs Strategy:** same goal (vary part of an algorithm), different mechanism —
Template Method uses **inheritance and is fixed at compile time**; Strategy uses **composition and is
swappable at runtime**. Prefer Strategy unless the shared skeleton is substantial and stable.

### When to use
- Several flows share a genuinely identical skeleton and differ in 1–3 steps.

### When not to use
- Subclasses start overriding the template method itself, or need steps in another order.

### Known uses
Jest lifecycle (`beforeEach` / test / `afterEach`) · Mongoose schema hooks · framework base controllers.

### Common mistakes
- Deep hierarchies (3+ levels). Two levels is the practical limit.
- Protected methods that call each other in undocumented order — the skeleton must be readable in one place.

---

## 5.15 State *(Behavioral)*

**Intent:** Let an object change its behaviour when its internal state changes — the object appears to
change class.

### Problem

```ts
// Bad — every method re-implements the same state machine
class Attempt {
  submit() {
    if (this.status === 'expired') throw new AttemptExpiredError();
    if (this.status === 'submitted') throw new AlreadySubmittedError();
    if (this.status !== 'in_progress') throw new InvalidStateError();
    // ...
  }
  answer() { /* the same three checks again */ }
  resume() { /* and again */ }
}
```

### Solution

```ts
export interface AttemptState {
  readonly name: AttemptStatus;
  answer(attempt: Attempt, answer: Answer): void;
  submit(attempt: Attempt): void;
}

export class InProgressState implements AttemptState {
  readonly name = 'in_progress';
  answer(attempt: Attempt, answer: Answer): void { attempt.record(answer); }
  submit(attempt: Attempt): void { attempt.transitionTo(new SubmittedState()); }
}

export class SubmittedState implements AttemptState {
  readonly name = 'submitted';
  answer(): never { throw new AlreadySubmittedError(); }
  submit(): never { throw new AlreadySubmittedError(); }
}

export class Attempt {
  private state: AttemptState = new InProgressState();
  answer(answer: Answer): void { this.state.answer(this, answer); }
  submit(): void { this.state.submit(this); }
  transitionTo(state: AttemptState): void { this.state = state; }
}
```

Invalid transitions become impossible instead of being re-checked in every method.

### When to use
- A real lifecycle with 3+ states and rules about legal transitions (attempt, order, subscription).

### When not to use
- Two states and one flag. `isPublished` is fine.

### Common mistakes
- States that know every other state (transition spaghetti) — centralize the transition table if it grows.
- Persisting the state *object* instead of the state *name*. Store the name; rebuild the object on load.

---

## 5.16 Honorable mentions — recognize, do not memorize

| Pattern | 20-second pitch | Where you meet it |
| --- | --- | --- |
| **Abstract Factory** | Family of related products that must match (all-Mongo repos vs all-in-memory repos). | Test container vs prod container |
| **Composite** | Uniform treatment of leaf and tree (nested question sections, permission trees). | UI trees, file systems |
| **Bridge** | Two independent axes of variation: notification *type* × delivery *channel*. | Cross-platform drivers |
| **Iterator** | Traverse without exposing internals. | `for...of`, generators, cursor pagination |
| **Mediator** | Components talk through a hub instead of N×N. | Socket.IO rooms, workflow orchestrators |
| **Memento** | Snapshot and restore state. | Draft autosave, undo |
| **Visitor** | Add operations over a stable structure. | AST tooling, linters, TypeScript compiler API |
| **Flyweight** | Share immutable data across many instances. | Interned strings, glyph caches |
| **Prototype** | Clone a configured object. | `structuredClone` of a template quiz |
| **Interpreter** | Evaluate a small grammar. | Query filter DSLs, permission expressions |

---

# Part 6 — Look-alike patterns compared

Most confusion in interviews and reviews comes from patterns with the same *structure* and different
*intent*. Intent is what you name in a review comment.

## 6.1 Strategy vs State vs Template Method

| | Strategy | State | Template Method |
| --- | --- | --- | --- |
| **Varies** | The algorithm | The behaviour per lifecycle state | Some steps of a fixed algorithm |
| **Who chooses** | The client / config | The object itself, by transitioning | The subclass, at compile time |
| **Mechanism** | Composition | Composition + self-transition | Inheritance |
| **Do the parts know each other?** | No | Usually yes (they set the next state) | Base knows the step names |
| **Exam example** | Scoring rules | Attempt lifecycle | Report generation skeleton |

## 6.2 Adapter vs Facade vs Proxy vs Decorator

| | Adapter | Facade | Proxy | Decorator |
| --- | --- | --- | --- | --- |
| **Interface vs wrapped object** | **Different** | **New, simpler** | **Same** | **Same** |
| **Intent** | Make incompatible things fit | Simplify a subsystem | Control access | Add behaviour |
| **How many objects wrapped** | 1 | Many | 1 | 1 (stackable) |
| **Exam example** | `SendGridMailer` | `ExamSubmissionFacade` | `AuthorizedQuizRepository` | `CachingQuizRepository` |

## 6.3 Factory Method vs Abstract Factory vs Builder

| | Factory Method | Abstract Factory | Builder |
| --- | --- | --- | --- |
| **Produces** | One product | A family of related products | One complex product |
| **Key question** | "Which class?" | "Which consistent set?" | "Which parts, in which order?" |
| **Call shape** | `factory.create(type)` | `factory.createRepo()`, `factory.createBus()` | `.withX().withY().build()` |

## 6.4 Observer vs Command vs Chain of Responsibility

| | Observer | Command | Chain of Responsibility |
| --- | --- | --- | --- |
| **Coupling** | Publisher ignores subscribers | Caller holds a request object | Sender ignores which link handles it |
| **Receivers** | 0..N, all run | 1, later | 1..N, until one stops the chain |
| **Exam example** | `attempt.submitted` fan-out | Queued grading job | Express middleware stack |

---

# Part 7 — Anti-patterns, overuse, and how to choose

## 7.1 Pattern abuse smells

| Smell | What it looks like | Fix |
| --- | --- | --- |
| **Patternitis** | `AbstractQuizFactoryProviderStrategy` for one implementation | Delete layers until deleting one hurts |
| **Speculative generality** | Interfaces with exactly one implementation, "for later" | YAGNI. Add the interface when the second implementation exists (test fakes count) |
| **Golden hammer** | Every problem gets a Strategy | Name the axis of change first |
| **Anemic pattern** | Repository that returns Mongoose documents; adapter that leaks vendor types | Fix the boundary or drop the pretence |
| **God facade** | The facade owns all the logic and the subsystem is empty | Push logic back down |
| **Singleton soup** | Globals reached from everywhere; flaky tests | Inject from the composition root |
| **Event spaghetti** | Handler publishes an event whose handler publishes an event... | Direct calls for required work; events for optional reactions |

## 7.2 Choosing a pattern — decision flow

```
1. What exactly is changing / likely to change?
   ├── How an object is CREATED ........... Factory · Builder · Abstract Factory · Prototype
   ├── What ALGORITHM runs ................ Strategy · Template Method · State
   ├── WHO is notified .................... Observer · Mediator
   ├── HOW things CONNECT ................. Adapter · Facade · Bridge · Composite
   ├── EXTRA behaviour on an existing type  Decorator · Proxy
   └── WHEN/IF a request runs ............. Command · Chain of Responsibility

2. Is it changing NOW, or might it change someday?
   ├── Now / already twice ................ apply the pattern
   └── Someday ............................ write the simple version, note the seam

3. Can I name the cost out loud?
   └── If not, you do not understand the pattern yet — do not merge it.
```

## 7.3 The three questions for a code review

1. **What varies here, and is it isolated?**
2. **Which direction do the dependencies point?**
3. **Could I unit test this without touching Mongo, the network, or the clock?**

If all three answers are good, the design is fine — whether or not it has a pattern name.

---

# Part 8 — Applying it to the Online Exam Platform

Concrete, high-value refactors for `online-exam/team-*`. Each maps to a rule in
[`standards/clean-code.md`](./standards/clean-code.md).

| Area in your project | Pattern | Payoff |
| --- | --- | --- |
| `src/modules/*/**.repository.ts` returning Mongoose docs | **Repository** + mapper | Services unit-testable with in-memory fakes; `CC-2xx` layering satisfied |
| `services` instantiating models/mailers inline | **Dependency Injection** at `app.ts` | Every service testable without `jest.mock` of modules |
| Scoring / grading rules | **Strategy** | New quiz types without editing tested code |
| Attempt lifecycle (`in_progress → submitted → expired`) | **State** | Illegal transitions impossible, not re-checked in five methods |
| Auth, validation, error handling middleware | **Chain of Responsibility** (already) | Name it, keep links independent, always call `next()` |
| Email / SMS providers | **Adapter** behind a `Notifier` port | Vendor swap = one file |
| Caching, logging, retry around repositories | **Decorator** | No cache code in the domain; disabled in tests by not wrapping |
| Submit-attempt orchestration in the controller | **Facade** / application service | Controllers become HTTP translation only |
| Post-submission side effects (stats, email, audit) | **Observer** (events) | Adding a reaction does not touch `submit()` |
| Building quizzes/questions in tests | **Builder** (test data builders) | Specs stop drowning in setup |
| Token issuing, password hashing | **Strategy** behind `TokenIssuer` / `PasswordHasher` | Swap `jsonwebtoken`/`bcrypt` without touching auth rules |

## 8.1 Live exercise (15 min, pairs)

Pick **one** module in your team folder and do exactly one of these:

1. **Extract a Repository interface** for one entity and write an `InMemory*Repository`. Then rewrite
   one service spec to use the fake instead of mocking Mongoose. Measure the runtime before/after.
2. **Extract a Strategy** from the largest `switch` / `if-else` chain you can find. Show the lookup map
   and one unit test per strategy.
3. **Wrap one repository in a Decorator** that logs slow queries (>100 ms) — without editing the
   repository class.

Deliverable: a diff plus two sentences — *what varies*, and *what the pattern cost you*.

## 8.2 Discussion prompts

- Where in your codebase would a pattern make things **worse**?
- Which of your interfaces have exactly one implementation, and is that justified?
- If you had to swap MongoDB for PostgreSQL, how many files would change today?

---

# Part 9 — Cheat sheet

## 9.1 Signals → pattern

| You see / say this | Reach for |
| --- | --- |
| "Another `case` in this switch" | Strategy · Factory · State |
| "This test needs a real database" | Repository · Dependency Injection |
| "`new` with 6 arguments, half optional" | Builder |
| "The SDK's shape does not fit our code" | Adapter |
| "I need caching/logging here but cannot touch the class" | Decorator |
| "This controller is 120 lines of orchestration" | Facade / application service |
| "Only the author may read this" | Proxy (protection) |
| "Every route repeats these three checks" | Chain of Responsibility |
| "Now also send an email when X happens" | Observer |
| "This work should run later / be retried" | Command |
| "These four flows differ in two steps" | Template Method |
| "This entity has a lifecycle with rules" | State |

## 9.2 One-line intents (all 23 GoF + 2)

**Creational** — Factory Method: *let one place decide the class*. Abstract Factory: *consistent
families*. Builder: *step-by-step assembly*. Prototype: *clone a configured object*. Singleton: *one
instance, injected not reached for*.

**Structural** — Adapter: *make it fit*. Bridge: *two axes, independent*. Composite: *tree looks like a
leaf*. Decorator: *add behaviour, same interface*. Facade: *one simple door*. Flyweight: *share
immutable state*. Proxy: *same interface, controlled access*.

**Behavioral** — Chain of Responsibility: *pass it along*. Command: *request as object*. Interpreter:
*evaluate a grammar*. Iterator: *traverse without exposing*. Mediator: *talk through a hub*. Memento:
*snapshot and restore*. Observer: *notify many*. State: *behaviour follows lifecycle*. Strategy: *swap
the algorithm*. Template Method: *fixed skeleton, variable steps*. Visitor: *new operations, stable
structure*.

**Enterprise** — Repository: *collection-like data boundary*. Dependency Injection: *collaborators come
from outside*.

## 9.3 Closing rules

1. Patterns are a **vocabulary**, not a checklist.
2. Isolate what varies; leave everything else boring.
3. Prefer composition. Reach for inheritance only when the skeleton is real and stable.
4. Every pattern has a cost — say it out loud before you merge it.
5. The best design is the simplest one that survives the **next** change you can actually name.

---

# Part 10 — Further reading

- Gamma, Helm, Johnson, Vlissides — *Design Patterns: Elements of Reusable Object-Oriented Software* (1994) — the catalog.
- Freeman & Robson — *Head First Design Patterns* — the friendliest first pass.
- Martin Fowler — *Patterns of Enterprise Application Architecture* — Repository, Unit of Work, Data Mapper.
- Robert C. Martin — *Clean Architecture* — the dependency rule, in depth.
- Refactoring.Guru — visual catalog with TypeScript examples.
- Fowler, *Refactoring* — the moves that get you *to* a pattern safely.
