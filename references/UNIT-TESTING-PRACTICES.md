# Unit Testing Practices

Extends the core rules in `SKILL.md`.

## State over interaction testing

Assert on the return value or resulting observable state, not on which methods
were called. `verify(...)` couples the test to the implementation, so a
refactor that preserves behaviour breaks it.

Use `verify(...)` only when the interaction is the contract:
- a side effect with no observable state (a payment must not be charged twice);
- a must-not-happen guarantee (`verify(x, never())` when a cache hit bypasses
  the DB);
- a fire-and-forget call to a boundary with no observable result.

```java
// state
assertThat(service.getCustomer("c-123")).isEqualTo(expected);

// interaction, only because "must not call" is the contract
service.getValue("cached-key");
verify(externalService, never()).fetchValue("cached-key");
```

Before verifying a `save(...)` argument, check whether asserting on the stored
object's state tests the same thing.

## Test-double preference order

1. Real implementation, when it is fast and deterministic with no problematic
   transitive dependencies. All value and domain objects qualify.
2. Fake: an in-memory working implementation, or a lambda for a functional
   interface, when the real one is slow or side-effecting.
3. Mock or stub, only at an architectural seam, stubbing state (`when`) rather
   than asserting interactions (`verify`).

Do not mock types you do not own. Wrap a third-party client or framework class
behind your own interface and fake or mock that. A mock of another library's
API encodes assumptions about its behaviour that a unit test cannot validate.

## Fake fidelity

A fake can diverge from the real implementation. Define the behaviour as a
contract test and run it against both:

```java
abstract class UserRepositoryContract {
    abstract UserRepository newRepository();

    @Test
    void should_return_saved_user_by_id() {
        var repo = newRepository();
        var user = aUser().withId("u-1").build();
        repo.save(user);
        assertThat(repo.findById("u-1")).contains(user);
    }
}
class InMemoryUserRepositoryTest extends UserRepositoryContract {
    UserRepository newRepository() { return new InMemoryUserRepository(); }
}
// The real implementation runs the same contract in its own (integration) test.
```

When a component is coupled to infrastructure through a port (a store, a cache, a
message bus), test it against an in-memory twin of that port so the component's
own logic runs as a fast unit test. Quarantine parity with the real dependency to
a separate `*IT` that runs the same contract against both: the unit test covers
the behaviour, the integration test covers the fidelity of the twin.

## Equals and hashCode contract

A type with identity-based equality (an entity keyed by an assigned id) deserves
a focused test of the contract rather than incidental coverage: two instances
with the same id are equal and share a hash code, instances with different ids
are not equal, and two instances that have no id yet are never equal to each
other. Testing this directly catches a broken `equals`/`hashCode` before it
corrupts a `HashSet` or a map key.

## Determinism

Remove every source of nondeterminism:
- Time: inject a `Clock`; do not call `LocalDateTime.now()` / `Instant.now()` in
  code under test. Builders default to a fixed instant, not `now()`.
- Concurrency and async: do not use `Thread.sleep`. Use Awaitility
  (`await().untilAsserted(...)`), or Reactor `StepVerifier` for reactive code.
  For an in-process concurrency invariant (mutual exclusion, cleanup on release),
  coordinate threads with a `CountDownLatch` or `CyclicBarrier` and wait with a
  bounded `await(timeout, unit)`, then assert an observed maximum or a
  post-condition. Do not pace threads with sleeps.
- Randomness: inject a seeded generator; assert on properties that hold for any
  value, not a specific draw.
- Identifiers and sequences: inject the id or sequence generator and supply a
  deterministic source (a counter or a fixed queue), so minted ids are stable and
  ordered, the same way a fixed `Clock` fixes time.

```java
AtomicInteger counter = new AtomicInteger();
IdGenerator ids = () -> "id-" + counter.incrementAndGet();
```
- Ordering: do not depend on `HashMap`/`HashSet` iteration order or filesystem
  listing order; use `containsExactlyInAnyOrder` when order is not part of the
  contract.

```java
private final Clock clock = Clock.fixed(Instant.parse("2025-01-01T00:00:00Z"), ZoneOffset.UTC);
private final OrderService service = new OrderService(orderRepository, clock);
```

## Lifecycle hooks

Inlining collaborators as `private final` fields keeps setup visible and avoids
shared mutable state, which is why it is preferred over `@BeforeEach`. Two hooks
remain appropriate. Use `@BeforeAll` for a fixture that is expensive to build and
immutable for the life of the class (a generated key pair, a parsed grammar, a
configured validator), so the cost is paid once. Use `@AfterEach` to reset
ambient state a test unavoidably mutates, such as a thread-local or a static
registry, so one test cannot leak into the next. Neither reintroduces the
per-test mutable state the `@BeforeEach` guidance is meant to prevent.

## No logic in tests; DAMP over DRY

- No control flow. A test with `if`/`for`/`switch` or a computed expected value
  needs its own test. Enumerate cases with `@ParameterizedTest`; write literal
  expected values rather than recomputing them with the production logic.
- DAMP over DRY. Some duplication is acceptable if it keeps each test readable in
  isolation. Builders and object mothers are appropriate sharing; a
  `buildExpectedResult()` helper that recomputes the assertion is not.

## Test size and hermeticity

Unit tests are small: single process, no network, no disk, no real database, no
sleep, milliseconds to run, no shared global state. Anything that needs I/O or a
container is an integration test and belongs in a separate suite (`*IT.java`).

Test a handler, a factory method, or a controller method as a plain method call
on a constructed instance, asserting on the returned value and headers. The
behaviour is a unit concern; routing it through the framework (a dispatcher or an
HTTP layer) turns the test into an integration test that belongs in the `*IT`
suite. Keep the behaviour in a fast POJO test and let the slice test cover only
the wiring.

## Test discovery

A test the runner does not discover is skipped silently and the build still
passes, which is worse than a failing test.

- Maven Surefire discovers unit tests by class name: `*Test`, `Test*`, `*Tests`,
  `*TestCase`. A class named otherwise (e.g. `FooSpec`, `FooShould`) compiles and
  never runs. Name unit test classes `*Test`. Gradle discovers by annotation and
  is not affected, but `*Test` remains the safe default.
- JUnit 5 does not run `private` or `static` `@Test` methods, and ignores test
  methods in a non-`@Nested` inner class. Keep test methods package-private and
  non-static; annotate grouping inner classes with `@Nested`.
- Keep integration tests (real DB, network, container) in `*IT.java`, run by
  Maven Failsafe under `mvn verify`. This keeps them out of the fast unit lane
  (`mvn test`, Surefire) and lets each suite be discovered by its own runner.
  In Gradle there is no Failsafe: put integration tests in a separate source set
  or a dedicated `integrationTest` task so `gradle test` stays fast. Adjust the
  task and plugin names to the project's build.

## Coverage and mutation testing

Line and branch coverage measure execution, not assertion, and are easily gamed.
Use coverage to find untested code, not as a target. Mutation testing (PIT)
introduces faults and checks whether assertions catch them, which identifies
tests that execute code without verifying its effect.

Write each negative test so that a green result requires the guard under test to
be present. Ask: would this test still pass if the production check were deleted?
If so, the assertion is not exercising the check. A test for signature rejection
that would also pass against an unsigned token proves nothing; construct the
input so only the specific guard makes it pass. This is the mutation-testing
mindset applied while writing the test.

## Failure diagnostics

- Annotate non-obvious assertions with `as("...")` / `describedAs("...")`.
- Use `SoftAssertions` to report all failures on an object at once.
- Use `usingRecursiveComparison()` for whole-object equality so the diff names
  the differing field.

## Mockito strictness

Static `mock()` does not flag unused stubs. Enable strict stubs to surface dead
`when(...)` lines:

```java
private final Foo foo = mock(Foo.class, withSettings().strictness(Strictness.STRICT_STUBS));
```

Or run the suite under `@ExtendWith(MockitoExtension.class)` (strict by default)
as a lint pass.

## Choosing edge cases

For every boundary, test a matched pair that straddles it: the largest accepted
value and the smallest rejected one (a 60-second skew allows 59 seconds and
rejects 61). A single-sided test still passes when the comparison is off by one;
the pair fails unless the boundary is exactly right, which is also what makes
these tests strong under mutation testing. Pin each case to the constant the
production code uses, not a copied literal, so the test tracks the limit if it
changes.

```java
assertThatThrownBy(() -> new Sku("S".repeat(Sku.MAX_LENGTH + 1)))
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessageContaining("at most");
```

Cover fail-closed behaviour on hostile or ambiguous input: assert the safe
outcome (rejected, empty result, no privileges) for malformed ids,
injection-shaped strings, and missing context.

Some contracts are about what must be absent. Assert the negative directly: a
secret never appears in `toString` or an exception message, a rendered payload
contains no carriage return or line feed (log-forging), an optional field is
omitted rather than present-but-empty.

```java
assertThat(session.toString())
    .as("token material must never appear in toString")
    .doesNotContain(rawToken)
    .contains("<redacted>");
```

## What not to unit test

Skip trivial getters and setters, generated code, and DTOs with no behaviour.
Do not test framework or library code you do not own.

Spend the effort where there is logic to get wrong: branching, boundary and
null handling, error paths, and state transitions.
