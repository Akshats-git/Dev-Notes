# 24 — Testing for Backend Engineers: Mocks, TDD and Coverage

**Video:** [#26 Testing for Backend Engineers: mocks, TDD and coverage](https://www.youtube.com/watch?v=HgVrfHX92y4) (1h 25m)

The series deliberately went 25 videos without writing a test: *a bad or hurried test is worse than none, because it gives false confidence.* This lecture covers testing properly: what a test is, the kinds of tests (with a better mental model than "unit vs integration"), test doubles, dependency injection, testing against real databases, time and flaky tests, TDD, and how to measure test and code quality (coverage, mutation testing, complexity).

The demo app is the Go task board from the real-time video (11 Go files). Code is only illustrative.

---

## Contents

1. [What a test is](#what-a-test-is)
2. [Test runners and assertions](#test-runners-and-assertions)
3. [Regressions: why tests matter most](#regressions-why-tests-matter-most)
4. [Unit, integration and end-to-end tests](#unit-integration-and-end-to-end-tests)
5. [A better model: small, medium, large](#a-better-model-small-medium-large)
6. [Other test "kinds"](#other-test-kinds)
7. [Test doubles](#test-doubles)
8. [State vs interaction verification](#state-vs-interaction-verification)
9. [Don't mock what you don't own](#dont-mock-what-you-dont-own)
10. [Dependency injection and the functional core](#dependency-injection-and-the-functional-core)
11. [Databases in tests](#databases-in-tests)
12. [Network and time](#network-and-time)
13. [Hermetic tests](#hermetic-tests)
14. [Test-driven development](#test-driven-development)
15. [Flaky tests](#flaky-tests)
16. [Measuring tests: coverage and mutation testing](#measuring-tests-coverage-and-mutation-testing)
17. [Measuring code: complexity](#measuring-code-complexity)
18. [Static tools: linter, formatter, type checker](#static-tools-linter-formatter-type-checker)
19. [Summary](#summary)
20. [Interview questions](#interview-questions)

---

## What a test is

[0:00](https://www.youtube.com/watch?v=HgVrfHX92y4&t=0s)

> A test is **a second program that runs your program and makes claims about what happened.**

Scope here: what a backend engineer writes and runs (about 80% of testing in a normal team), not manual QA, test-management tools, mobile or browser automation.

Even without frameworks, this is a test:

```go
got := Add(2, 3)
if got != 5 { panic("expected 5") }
```

Frameworks exist because hand-rolling conditions, runners and mocking doesn't scale: solve it once in a library.

---

## Test runners and assertions

[7:49](https://www.youtube.com/watch?v=HgVrfHX92y4&t=469s)

A **test runner** finds every test across the code base, runs them and reports passes/failures. In Go, `go test` literally **generates a `main` function** (`_testmain.go`), compiles everything into a test binary (`board.test`) and runs it. (Shown with `go test -work` and `-x`.) Every runner does the same: write the main entry point, collect results.

An **assertion** is the check inside a test that fails the test when a condition is false (`assertEquals`, `assertNotNil`... or in Go simply `if got != want { t.Errorf(...) }`).

**Write meaningful failure messages**: file, line, *wanted X, got Y*, so whoever sees the failure knows what to check.

---

## Regressions: why tests matter most

[12:23](https://www.youtube.com/watch?v=HgVrfHX92y4&t=743s)

A **regression** = a bug you already fixed that comes back (e.g. a teammate changes a function, interface or behaviour).

The key payoff of a suite: run it in **CI** (e.g. GitHub Actions) on every PR before merging. If anything fails, the PR is causing a regression. You find out immediately, not from user complaints. That lets you **ship features quickly without fearing you broke old ones**.

---

## Unit, integration and end-to-end tests

[14:26](https://www.youtube.com/watch?v=HgVrfHX92y4&t=866s)

| Type | Definition |
|---|---|
| **Unit** | One unit in isolation. But *what is a unit* (function? struct? package? service?) has **no universal definition**. You learn to recognise it |
| **Integration** | Two or more parts working together; usually your code plus something you didn't write (e.g. a database) |
| **End-to-end (E2E)** | The whole system the way a **user** goes through it: authenticate first, then use the session to navigate, create a task... Integration tests can bypass things like login |

The famous **test pyramid** (wide base of unit tests, fewer integration, a few E2E on top) is about **cost**: higher = slower and more expensive (CPU, memory, time, not just money).

### Demo: the same check four ways

| # | Setup | Time |
|---|---|---|
| 1 | Board with a **mock bus** | ~0 s |
| 2 | Real bus and hub | ~0 s |
| 3 | Real HTTP server on a port, real POST | ~0 s |
| 4 | Real **Postgres** (container) | **10.5 s** (mostly container boot) |

> The expensive part isn't how *much logic* a test covers. It's **whether the test leaves the process** and talks to the outside.

---

## A better model: small, medium, large

[21:06](https://www.youtube.com/watch?v=HgVrfHX92y4&t=1266s)

Instead of "how much of the system does a test cover?", ask **"how much is a test allowed to touch?"**

| Size | Allowed |
|---|---|
| **Small** | One process. No network, no database, no file system, no artificial sleeping |
| **Medium** | Talks to things on the **same machine** (local DB container, a local Redis). Cannot reach the internet |
| **Large** | Can do anything |

Why it's better: nobody agrees on what a "unit" is, but **the machine can check whether a test opened a socket**. It becomes an **enforceable rule**, not an opinion. It also nearly eliminates **flakiness**: flakiness lives in the network, external services and clocks, which small/medium tests avoid.

---

## Other test "kinds"

[24:10](https://www.youtube.com/watch?v=HgVrfHX92y4&t=1450s)

**Regression**, **functional** (spec-first; a variation of integration), **performance** (speed, memory, CPU), **security** (privacy/permissions) tests are **variations/purposes**, not separate scopes. A regression test can be a unit test, and a security test can be E2E. The scope axis (small/medium/large) is the real one.

---

## Test doubles

[27:13](https://www.youtube.com/watch?v=HgVrfHX92y4&t=1633s)

A handler may hit Postgres, a payment provider and an email service. In a test you give it something that **looks and acts like** the external service: a **test double** (like a movie stunt double). People call all of these "mocks", but there are five kinds:

| Double | What it does | Example |
|---|---|---|
| **Dummy** | A value passed only because the signature requires it. Never used | A no-op logger |
| **Stub** | Returns a **pre-written answer** every time, regardless of input | `getUser(7)` → a fixed user object |
| **Spy** | A stub that also **records how it was used** | Records "name changed X → Y" |
| **Mock** | Pre-programmed with **expectations**; fails if they aren't met | "`save` must be called exactly once with these args" |
| **Fake** | A **working but non-production** implementation | An in-memory map standing in for the user repository |

---

## State vs interaction verification

[33:23](https://www.youtube.com/watch?v=HgVrfHX92y4&t=2003s)

Two ways to check a move-task handler:

| Style | Question |
|---|---|
| **State verification** | "Is the task now in the *done* column? Did the sequence number go up by one?" |
| **Interaction / behaviour verification** | "Was `publish` called once with an event of type `task.moved`?" |

If a refactor changes *how* things happen but not the outcome (publish one combined event instead of two), **state tests keep passing** while **interaction tests break**. These are **change-detector tests**. Google has a good write-up.

> **Rule:** assert **what the system is now**, not how it got there. Use interaction verification only when truly necessary (e.g. "the email was sent exactly once").

---

## Don't mock what you don't own

[35:58](https://www.youtube.com/watch?v=HgVrfHX92y4&t=2158s)

You mock a payment client so `charge(...)` returns `{success: true, newBalance: ...}`. Tests pass, production fails, because **every field name, error shape and duplicate-charge behaviour came from your assumptions**.

Fix:

1. **Wrap the third party in your own small interface** with your own success/failure types, and use that wrapper in production *and* tests. You own it, so you may mock it.
2. Write a few **contract tests** against the provider's **sandbox API** to verify your wrapper matches reality (request/response/failure shapes). They're small, fast, and don't touch real payment data.

---

## Dependency injection and the functional core

[38:31](https://www.youtube.com/watch?v=HgVrfHX92y4&t=2311s)

A handler that creates its own DB and Redis clients can't have them replaced from outside. **Dependency injection (DI)**: pass dependencies in through parameters/constructors. (Frameworks exist to automate it as apps grow.)

**Functional core, imperative shell:**

```
┌──────────────── imperative shell (thin) ────────────────┐
│  DB, HTTP calls, email, clock … talks to the outside    │
│    ┌────────── functional core (pure) ───────────┐      │
│    │ decisions, calculations, business rules      │      │
│    │ take values → return values; nothing to mock │      │
│    └──────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────┘
```

The core is trivially testable (nothing to mock). The shell is hard to test, but it's so thin there's little to test. You're more likely to write tests when you structure code this way.

---

## Databases in tests

[41:06](https://www.youtube.com/watch?v=HgVrfHX92y4&t=2466s)

Instinct: replace the DB with an in-memory fake (H2 in Java, maps in Go, in-memory stores in Node). You gain speed but **lose fidelity**. The bugs that live in the data layer (wrong column name, missing index, unique constraint you forgot, a transaction that doesn't behave, Postgres-specific JSON operators) live in exactly the layer you replaced.

**Use a real database and make it cheap:**

| Technique | How |
|---|---|
| **Transaction per test** | Begin a transaction, run the test, **roll back**. Clean slate. Fast. Requires your code to accept a transaction handle (DI again) |
| **Template database** | Run migrations once on a template DB. Each test creates its own DB as a copy of the template (milliseconds). Tests can run **in parallel** |

**Testcontainers** (SDKs in most languages): starts a real container (Postgres 18) from your test code and gives you the address. Demo: container boot ~7 s once per package; the nine tests then each copy the template and run in milliseconds. Total 10.5 s (7 s setup + 3 s tests).

---

## Network and time

[46:52](https://www.youtube.com/watch?v=HgVrfHX92y4&t=2812s)

**Network:** wrap external APIs in your own interface (as above). If you need to test the real HTTP behaviour of your client (headers, etc.), start a **real HTTP server on a real port in the test** (Go: `httptest`) and point the client at it. Node has **MSW**, which intercepts at the network layer.

**Time:** the classic bug: the code does something async, the test does `sleep(100ms)` then checks. It passes locally; on a loaded CI box it takes 110 ms and fails. Raising the sleep is not a fix. Two real fixes:

1. **Wait for a condition, not a duration**: wait on a channel/callback/event.
2. **Inject a clock**: pass a fake clock so the test controls time and timeouts.

---

## Hermetic tests

[50:28](https://www.youtube.com/watch?v=HgVrfHX92y4&t=3028s)

A **hermetic** test brings its own environment (starts what it needs, stops it after). It doesn't depend on a hand-set-up DB or a service that must already be running. So it can run anywhere: laptop, SSH box, CI. Testcontainers = hermetic about the database; an injected clock = hermetic about time.

---

## Test-driven development

[51:31](https://www.youtube.com/watch?v=HgVrfHX92y4&t=3091s)

TDD isn't a kind of test, it's a **workflow**: write the test first.

```
RED      write a test for something that doesn't exist yet; it must FAIL
GREEN    write the smallest code that passes (ugly is allowed)
REFACTOR clean up; the test must stay green      ← the step people skip
```

| Phase | Benefit |
|---|---|
| Red (design) | Forces you to imagine *using* the function before it exists. If the test is awkward to write, the API is wrong, and you learn in seconds |
| Red → Green (proof) | If you write tests after code, they pass first try, so you learn nothing (maybe they check nothing). Seeing red first proves the test can fail |
| Refactor | Pay back the shortcuts taken in green |

**Demo:** Requirement: a task can't go straight from *to-do* to *done*. Red: moving to-do → done should return an error but returns `nil`. Green: one `if`. Refactor: extract `canMove(from, to)`. The suite stays green.

### The TDD debate

- **Kent Beck** (author of TDD), 2023 article on what the loop actually means.
- **Ian Cooper**, *"TDD: Where did it all go wrong?"* People treated "unit" as "class", wrote one test class per production class, mocked everything, and every refactor broke hundreds of tests. **The unit is the behaviour**, tested via the module's public interface, so you can reorganise internals and tests still pass (ties to *state over interaction*).
- **DHH**, *"TDD is dead. Long live testing."* Against test-first culture and designing for mockability.

**The speaker's view:** TDD is great **when you know what the answer should look like but not how to build it**: a JSON parser, a pricing rule, a state machine. It's the wrong tool when you're still **exploring** approaches and trade-offs, which is often the case with backend feature design.

---

## Flaky tests

[1:02:21](https://www.youtube.com/watch?v=HgVrfHX92y4&t=3741s)

A **flaky test** passes and fails on the same code. It is worse than a missing test: a missing test is a known hole, while a flaky one trains the team to say *"just rerun it"*, so when a real failure appears someone reruns it instead of fixing it.

A 2014 study (Luo et al., 201 real flaky-test fixes) found the top causes:

| Cause | Fix |
|---|---|
| **Async wait**: waiting the wrong way | Wait for a condition/signal, not a sleep |
| **Concurrency**: races inside the test | Proper synchronisation primitives |
| **Test-order dependency**: test B only passes after test A left some state | See below |

For order dependency (passes alone? no; passes after another test? yes), you have three options: **delete** the test (often fine and underrated), **quarantine** it (move it out of the blocking suite with an **owner's name, date and reason**), or add a retry (not recommended). The speaker prefers quarantine with name/date/reason.

---

## Measuring tests: coverage and mutation testing

[1:07:35](https://www.youtube.com/watch?v=HgVrfHX92y4&t=4055s)

### Code coverage

The percentage of code **executed** while tests ran. The tool instruments the code (splits into basic blocks, adds a counter to each), runs the tests, reads the counters.

- It only records **whether code ran**, not whether anything was **checked**. A test that calls every function and asserts nothing gives 100%.
- **Line coverage** vs **branch coverage** give different numbers: an `if` with no `else`, tested with the condition true, is 100% line coverage but 50% branch coverage.
- A 2014 ICSE study (Inozemtseva & Holmes) held suite size constant and found only a **low-to-moderate** correlation between coverage and bugs found. Google's guideline numbers: 60% acceptable, 75% commendable, 90% exemplary, but there is no universal target.
- Takeaways: **what's not covered (the red lines) matters more than the percentage.** Mandating 100% leads to low-value tests written just to hit the number.

### Mutation testing

A way of **testing your tests**. A tool (e.g. **Stryker**) makes small deliberate **mutations** (`>` → `>=`, `+` → `-`, return value → `0`), each producing a **mutant**, and runs your suite against each.

- Test fails → mutant **killed** (good, tests noticed).
- Tests still pass → mutant **survived** (your tests don't really check that behaviour).
- **Mutation score** = % of mutants killed.

**Demo:** a pagination function with weak assertions ("returns an array", "length > 0") has **100% coverage** but Stryker made 19 mutants and only 10 died: score **52.63%**. A surviving mutant changed `i <= offset + limit` to `i < offset + limit` (changing how many items are returned) and every test still passed.

---

## Measuring code: complexity

[1:15:53](https://www.youtube.com/watch?v=HgVrfHX92y4&t=4553s)

| Metric | What it measures | Notes |
|---|---|---|
| **Cyclomatic complexity** (McCabe, 1976) | Number of independent paths. Formula: edges − nodes + 2, or simply **1 + number of decision points** (each `if`, loop, `case`, `&&`, `||`) | A function with no branches = 1; one `if` = 2. The usual guideline is **≤ 10**. McCabe himself noted an exception: a flat `switch` with 20 cases scores 21 yet is easy to read |
| **Cognitive complexity** (SonarSource) | How hard the code is to **understand**. Doesn't penalise a flat switch; **penalises nesting** | An `if` inside a loop inside an `if` is hard to hold in your head; three sequential `if`s are not |

In the demo, a hub's `Run` method scored **8** cyclomatic (3rd on the list) but **19** cognitive (top), and reading it confirms it's deeply nested.

> Don't trust either number blindly. Use them as **flags** that point you at code to look at, then decide as a team.

---

## Static tools: linter, formatter, type checker

[1:20:40](https://www.youtube.com/watch?v=HgVrfHX92y4&t=4840s)

| Tool | Does |
|---|---|
| **Linter** | Matches **patterns** in source: "this condition is always true", "you ignored this returned error". Doesn't check syntax (compiler's job) or logic correctness. Examples: ESLint, Go's lint tools |
| **Formatter** | Rewrites code to the team's standard layout. Doesn't change behaviour |
| **Type checker** | Catches type misuse (e.g. arithmetic on a variable declared as string). Go, TypeScript, Rust |
| **Static analysis** | A deeper linter: follows values through the program without running it |

There is no objective, deterministic tool for "code quality". These help to a degree.

---

## Summary

1. A **test is a program** that runs yours and asserts. A **runner** supplies the main and reports. Make failure messages meaningful.
2. The main payoff is catching **regressions** in CI, so you can ship fast.
3. Think **small / medium / large** (what a test may touch: process, machine, anything) rather than arguing over "unit". It's checkable and kills flakiness.
4. Know the five **test doubles** (dummy, stub, spy, mock, fake). Prefer **state verification** over interaction verification. **Don't mock what you don't own**: wrap third parties and contract-test against their sandbox.
5. Use **dependency injection** and a **functional core / imperative shell**.
6. Use a **real database** in tests (transaction-per-test or template database, via Testcontainers). Don't use in-memory replacements.
7. Never `sleep` in tests: wait for conditions or inject a clock. Aim for **hermetic** tests.
8. **TDD** (red → green → refactor) shines when you know the expected output but not how to build it.
9. **Flaky tests** erode trust: quarantine with owner/date/reason, or delete.
10. **Coverage** shows what ran, not what was verified. **Mutation testing** shows if tests actually catch changes. **Cyclomatic vs cognitive complexity** are flags, not rules.

## Interview questions

**Q: What is a test, fundamentally?**
A second program that exercises your code and asserts on its behaviour.

**Q: Why write tests at all?**
Chiefly to catch regressions as the code base evolves, and secondarily as documentation and design feedback.

**Q: Unit vs integration vs E2E?**
Unit: one piece in isolation. Integration: your code with other parts (e.g. a real DB). E2E: the whole system as a user flows through it. Costs increase up the pyramid.

**Q: Why does the speaker prefer "small/medium/large"?**
"Unit" has no agreed definition, but whether a test opens a socket or touches the DB is mechanically checkable, so it can be enforced. It also limits flakiness.

**Q: Name the five test doubles.**
Dummy, stub, spy, mock, fake.

**Q: State vs interaction verification, and which to prefer?**
State checks the resulting state. Interaction checks which calls were made. Prefer state; interaction tests break under harmless refactors (change-detector tests).

**Q: What does "don't mock what you don't own" mean?**
Wrap external libraries/services in your own interface and mock that. Verify the wrapper against the real thing (sandbox) with a few contract tests.

**Q: Why not use an in-memory database for tests?**
It loses fidelity. Constraints, indexes, transactions and DB-specific features live in the real database, which is exactly what you replaced.

**Q: How do you keep real-DB tests fast?**
Transaction-per-test with rollback, or per-test copies of a template database, with the container started once (Testcontainers).

**Q: How do you test time-dependent or async code without `sleep`?**
Wait on a condition/signal, or inject a fake clock.

**Q: What is the TDD cycle and when is it useful?**
Red (failing test) → green (minimal pass) → refactor. Best when the expected output is clear but the implementation isn't (parsers, pricing rules, state machines).

**Q: What are the common causes of flaky tests?**
Async waiting, concurrency races, and test-order dependency.

**Q: Why isn't 100% coverage the goal?**
Coverage measures execution, not assertions. It's gameable, correlates only weakly with bug-finding once suite size is controlled, and encourages low-value tests.

**Q: What is mutation testing?**
Deliberately mutating code and checking whether tests fail. The mutation score is the fraction of mutants killed. It reveals tests that run code but check nothing.

**Q: Cyclomatic vs cognitive complexity?**
Cyclomatic counts independent paths (1 + decision points). Cognitive measures how hard code is to understand, penalising nesting more than flat branching.
