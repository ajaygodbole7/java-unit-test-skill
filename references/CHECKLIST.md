# Checklist

Self-review before finishing a test class.

## Pre-flight checklist

- [ ] Classified every dependency: real (domain/value objects), fake
      (functional/single-method interfaces), or mock (services, repositories,
      gateways, infrastructure).
- [ ] Checked for existing test builders/fixtures; created a Builder for any
      complex domain object that lacked one.
- [ ] Domain objects are real instances; none are mocked.
- [ ] All assertions use AssertJ (`assertThat`, `assertThatThrownBy`,
      `assertThatCode`, `assertThatExceptionOfType`).
- [ ] Every test follows the `// given` / `// when` / `// then` structure.
- [ ] Test method names use `lower_snake_case` and state scenario + expected
      outcome.
- [ ] Tests are independent; no shared mutable state.
- [ ] `when(...)` stubs are explicit about parameters that matter; `any()` used
      only where the argument is irrelevant to the scenario.
- [ ] Collaborators are `private final` fields via `mock(Type.class)`; no
      `@Mock` / `@InjectMocks` / unnecessary `@BeforeEach`.
- [ ] Coverage goes beyond the happy path: null handling, empty collections,
      boundary values, invalid states, error paths, and state transitions.
- [ ] Boundaries tested as a straddling pair, largest accepted and smallest
      rejected, pinned to the production constant, not a copied literal.
- [ ] Contracts about absence asserted directly (no secret in `toString`/
      messages, no CR/LF in rendered output, a field omitted rather than empty).
- [ ] Asserts on state/return values; `verify(...)` used only where the
      interaction is the contract (not to re-describe how the code is written).
- [ ] Test-double order respected: real → fake → mock; no mocking of types you
      don't own (third-party/framework wrapped behind your own interface).
- [ ] Any fake is backed by a shared contract test run against the real impl.
- [ ] Deterministic: injected `Clock` (no `now()` in builders/prod paths), no
      `Thread.sleep`, seeded randomness, deterministic id/sequence generators, no
      reliance on iteration order.
- [ ] `@BeforeEach` not used for mutable per-test setup; `@BeforeAll` only for
      expensive immutable fixtures, `@AfterEach` only to reset ambient state.
- [ ] No control flow or recomputed expected values in tests; readability
      favored over de-duplication (DAMP).
- [ ] Stays a small/hermetic test with no network, disk, or real DB (those go in
      an integration suite).

## Common pitfalls to avoid

- Mocking domain objects.
- Using JUnit assertions instead of AssertJ.
- Mocking a functional/single-method interface instead of faking it.
- Over-mocking; mock only what crosses an architectural boundary.
- Testing implementation details instead of observable behavior.
- Sharing mutable state between tests.
- Using CamelCase/camelCase for test method names.
- Missing or wrong `given`/`when`/`then` annotations.
- Building complex domain objects by hand when a builder exists.
- Using `any()` in `when(...)` where the parameter value should be explicit.
- Adding `verify(...)` calls solely to check parameters that should have been
  explicit in `when(...)`.
- Reaching for `ArgumentCaptor` when `argThat(...)` would express the check
  inline (only capture when multiple assertions on the value are needed).
- Defaulting to interaction testing (`verify`) where a state assertion would be
  more robust and refactor-safe.
- Mocking third-party or framework types directly instead of wrapping them.
- Writing a fake without a contract test, letting it drift from the real impl.
- `LocalDateTime.now()` / `Instant.now()` / `Thread.sleep` / unseeded randomness
  inside tests or builders.
- Control flow or recomputed expected values inside a test.
- Chasing a line-coverage number instead of asserting meaningful behavior.
- Leaving an `assertThat(...)` with no terminal assertion, or wrapping the
  comparison inside it (`assertThat(a.equals(b))`) so nothing is actually
  checked.
- Placing `as(...)` / `withFailMessage(...)` / `usingComparator(...)` *after*
  the terminal assertion, where AssertJ silently ignores them.
- Freezing every field to compare an object with a generated id/timestamp
  instead of `usingRecursiveComparison().ignoringFields(...)`.
- A test class the runner does not discover (silently skipped, build stays
  green): a class not matching `*Test`/`*Tests` under Surefire, a `private` or
  `static` `@Test` method, or a non-`@Nested` inner class.
- Comparing `BigDecimal` with `isEqualTo`/`equals` (scale-sensitive) instead of
  `isEqualByComparingTo`.
- Pinning the exact message of an exception thrown by a type you don't own; its
  wording is not your contract; assert the type instead.
- Testing only one side of a boundary, so an off-by-one comparison still passes.
- A negative test that would still pass with the production guard removed.
- Mocking a collaborator that has readable state instead of using an in-memory
  fake asserted on by state.
- Reaching for a mock of a type you don't own instead of subclassing the code
  under test and overriding a narrow seam.
- Calling `get()` on an `Optional` instead of `assertThat(opt).contains(...)` /
  `isPresent()` / `isEmpty()`.
