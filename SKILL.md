---
name: java-unit-testing
description: >-
  Write, review, or refactor Java unit tests with JUnit 5, AssertJ, and Mockito.
  Use when creating test classes for a Java service or component, adding test
  cases, modernizing or fixing existing tests, building test data
  (builders/fixtures), or deciding what to mock vs. fake vs. use real. Triggers
  on JUnit, AssertJ, Mockito, "unit test", *Test.java, @Test, test coverage, or
  any request to test Java code.
license: Apache-2.0
metadata:
  version: 1.1.0
  language: java
  frameworks: [junit5, assertj, mockito]
---

# Java Unit Testing

## Overview

A house style for Java unit tests using JUnit 5, AssertJ, and Mockito, packaged
as a tool-agnostic Agent Skill. Apply the core rules below to every test; use
the `references/` files for detailed patterns.

## When to use this skill

Use it whenever the task involves Java tests: creating a test class for a
service/component, adding cases, fixing or modernizing an existing suite,
building test data (builders/fixtures), or deciding what to mock vs. fake vs.
use real. It does **not** cover integration tests that need a real database,
network, or container. Those are a separate suite (`*IT.java`).

## Version management

Assumed tool versions live in `versions.json`. They are defaults, not
requirements: align them with the versions your project actually pins, and if a
project uses an older AssertJ/Mockito, verify that any newer API used in the
references exists before relying on it. Update `versions.json` first when
bumping a version.

## Core rules (non-negotiable)

**Structure**
- Test method names use `lower_snake_case` describing scenario + expected
  outcome: `should_return_processed_result_when_input_is_valid`.
- Name unit test classes `*Test` so the test runner discovers them (Maven
  Surefire matches `*Test`/`*Tests`); keep integration tests as `*IT` in a
  separate suite. A misnamed class is silently skipped with a passing build.
- Annotate phases with `// given`, `// when`, `// then` (fuse to `// when / then`
  for exception assertions).
- One *logical* assertion per test: one concept, verified once. Chained AssertJ
  calls count as one, and several `assertThat(...)` statements that check
  different facets of the *same* resulting state or object are still one logical
  assertion. A test that verifies two unrelated behaviours should be split.
- Prefer test-scoped setup in the `given` phase; inline simple collaborators as
  `private final` fields rather than `@BeforeEach`. This targets mutable per-test
  setup, not lifecycle hooks in general: `@BeforeAll` is appropriate for an
  expensive, immutable fixture shared read-only across the class (a generated key
  pair, a parsed schema), and `@AfterEach` for resetting ambient state a test
  touches (a thread-local, a static registry) so tests stay independent.
- Tests are independent and self-contained, never sharing mutable state.

**Mocking strategy**
- **Prefer state over interaction testing.** Assert on return values and
  observable state; reach for `verify(...)` only when the interaction itself is
  the contract (must-not-happen guarantees, unobservable side effects). See
  `references/UNIT-TESTING-PRACTICES.md`.
- **Never mock domain objects.** Use real instances via builders.
- **Fake, don't mock, functional / single-method interfaces** (a lambda beats a
  stub).
- **Prefer a hand-written fake or in-memory twin over a mock.** When a boundary
  collaborator has readable state (a repository, a store), an in-memory fake
  asserted on by state is the default double; reserve `mock(Type.class)` for a
  boundary with no readable state (a gateway, a notifier, a client). Mock only
  across an architectural boundary, and never a type you don't own. Subclass the
  code under test and override a narrow seam instead (see
  `references/PATTERNS.md`).
- Use the static `mock(Type.class)` assigned to `private final` fields; no
  `@Mock` / `@InjectMocks`.
- Be **explicit** in `when(...)`; reserve `any()` for arguments that don't
  matter. Verify complex arguments with `argThat(...)`, not `ArgumentCaptor`.

**Assertions**
- **Always AssertJ, never JUnit assertions.**
- **Every `assertThat(...)` must end in a terminal assertion.**
  `assertThat(a.equals(b))` asserts nothing; write `assertThat(a).isEqualTo(b)`.
- Put `as(...)`, `withFailMessage(...)`, `usingComparator(...)`, and
  `usingRecursiveComparison()` config **before** the terminal assertion, or it
  is silently ignored.
- Prefer specific matchers (`hasSize`, `containsExactly`, `extracting`) over
  generic `satisfies`; use `usingRecursiveComparison().ignoringFields(...)` for
  objects with generated ids/timestamps. See `references/ASSERTJ.md`.

**Determinism & coverage**
- No `now()` / `Thread.sleep` / unseeded randomness in tests or builders; inject
  a fixed `Clock`, use Awaitility for async.
- Cover edge cases and error paths, not just the happy path: nulls, empty
  collections, boundary values, invalid states, state transitions.

## Workflow

1. **Read the code under test.** Classify each dependency as `real`
   (domain/value objects), `fake` (functional/single-method interfaces), or
   `mock` (services, repositories, gateways, infrastructure).
2. **Reuse test builders/fixtures** before constructing domain objects; create a
   Builder for any complex domain object that lacks one; see
   `references/PATTERNS.md`.
3. **Write the tests** to the core rules above.
4. **Self-review** against `references/CHECKLIST.md`, then run the validation
   commands below.

## Reference guides

`SKILL.md` stays high-level; detail lives in these files.

- **`references/PATTERNS.md`**: code for each pattern: basic
  class, faking interfaces, real domain objects, test builders, AssertJ
  chaining, exceptions, parameterized tests, state-first verification,
  collections, nested organization, and two complete example test classes.
- **`references/ASSERTJ.md`**: AssertJ Core 3.27 usage: correctness traps
  (config-before-assertion, no-op assertions), recursive comparison
  (ignore / compare-only / tolerant field comparison), type-safe
  `extracting`/`tuple`, containment matchers, `filteredOn`, `satisfiesExactly`,
  navigation, exception assertions, soft assertions, custom domain assertions
  (`AbstractAssert`), conditions, assumptions, and numbers/temporal/`Optional`
  assertions. Links to the JUnit and AssertJ docs.
- **`references/UNIT-TESTING-PRACTICES.md`**: state-over-interaction testing;
  the real/fake/mock preference order and not mocking types you don't own;
  fake-fidelity contracts; determinism; no-logic-in-tests and DAMP-over-DRY;
  test size and hermeticity; coverage and mutation testing; Mockito strictness.
- **`references/CHECKLIST.md`**: pre-flight checklist and common pitfalls to
  self-review against before finishing.

## Validation

After writing or changing tests, run the suite and check quality; fix failures
before proceeding.

| # | What | Maven | Gradle |
| - | ---- | ----- | ------ |
| 1 | Compile + run unit tests | `./mvnw test` | `./gradlew test` |
| 2 | Line/branch coverage (guide, not target) | `./mvnw jacoco:report` | `./gradlew jacocoTestReport` |
| 3 | Mutation testing (real quality signal) | `./mvnw org.pitest:pitest-maven:mutationCoverage` | `./gradlew pitest` |

Coverage identifies untested code. Mutation testing (PIT) identifies tests that
execute code without asserting its effect.

The commands assume the Maven or Gradle wrapper. The exact task, goal, and
plugin names depend on the project's build; adjust the JaCoCo and PIT
configuration accordingly, and use whichever build tool the project already uses.

## Agent integration

`SKILL.md` and `references/` are the entire skill and are read directly by any
agent that supports the Agent Skills format. Two thin adapters carry no rules of
their own; they only route agents that use a different discovery convention to
this file:

- `AGENTS.md`: for agents that read an `AGENTS.md` at the repository root.
- `.kiro/steering/java-unit-testing.md`: a Kiro steering rule scoped to
  `*Test.java`.

`SKILL.md` is the single source of truth. Do not restate its rules elsewhere.
