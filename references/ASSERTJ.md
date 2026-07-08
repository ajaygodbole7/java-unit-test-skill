# AssertJ

AssertJ Core 3.27.x requires Java 8+. Single entry point:

```java
import static org.assertj.core.api.Assertions.*;   // assertThat, tuple, entry, offset, ...
```

A BDD variant is available via `org.assertj.core.api.BDDAssertions.then(...)`,
interchangeable with `assertThat`. Use one per codebase.

## Correctness traps

Configuration and description calls must come before the terminal assertion. A
failing assertion short-circuits the chain, so anything after it does not run.

```java
// wrong: description/config after the assertion is ignored
assertThat(order.getStatus()).isEqualTo(PAID).as("order should be paid");
assertThat(actual).isEqualTo(expected).withFailMessage("mismatch");
assertThat(actual).isEqualTo(expected).usingComparator(cmp);

// correct: set them first
assertThat(order.getStatus()).as("order should be paid").isEqualTo(PAID);
assertThat(actual).withFailMessage("mismatch").isEqualTo(expected);
assertThat(actual).usingComparator(cmp).isEqualTo(expected);
```

An `assertThat(...)` without a terminal assertion compiles but verifies nothing:

```java
assertThat(actual.equals(expected));   // asserts nothing, always passes
assertThat(a == b);                    // same

assertThat(actual).isEqualTo(expected);   // correct
```

Enable SpotBugs (`RV_RETURN_VALUE_IGNORED_INFERRED`) or SonarQube (S2970) to
catch this class of mistake automatically.

## Descriptions & failure messages

- `as("...")` / `describedAs("...")`: prefix the failure with context; use for
  boolean assertions especially, where the default message is just `true`/
  `false`. Supports `String.format` args: `as("age of %s", name)`.
- `withFailMessage(...)` / `overridingErrorMessage(...)`: replace the message
  entirely. Prefer the `Supplier<String>` overload when the message is
  expensive to build (only evaluated on failure).

## Object & value assertions

```java
assertThat(result).isNotNull().isEqualTo(expected);
assertThat(order.getStatus()).isEqualTo(OrderStatus.PENDING);
assertThat(value).isInstanceOf(Money.class);
assertThat(user).matches(u -> u.getAge() >= 18, "is an adult");
```

## Numbers, temporal values, and strings

Compare numbers by value, not representation. `BigDecimal` equality is
scale-sensitive, so `new BigDecimal("75.0").equals(new BigDecimal("75.00"))` is
`false`; `isEqualByComparingTo` uses `compareTo` and is the correct assertion for
money and other decimals.

```java
assertThat(order.getTotal()).isEqualByComparingTo("75.00");   // scale-insensitive
assertThat(count).isPositive();
assertThat(discount).isGreaterThanOrEqualTo(BigDecimal.ZERO);
```

For a `Duration` or `Instant`, assert a tolerance band or an ordering when the
value is derived from elapsed time; a band tolerates jitter that an exact match
would not.

```java
assertThat(ceiling).isBetween(Duration.ofHours(8), Duration.ofHours(8).plusSeconds(5));
assertThat(ceiling).isLessThanOrEqualTo(maxLifespan);
assertThat(order.getCreatedAt()).isBefore(order.getPaidAt());
```

A tolerance band is a fallback for a value that cannot be pinned with an injected
`Clock`; prefer a fixed `Clock` so the expected value is exact (see
`UNIT-TESTING-PRACTICES.md`).

String shape and size are contracts too. `matches(regex)`, `hasSize(n)`,
`hasSameSizeAs(other)`, `isNotBlank()`, and `startsWith`/`endsWith` state intent
better than manual length arithmetic.

```java
assertThat(id).hasSize(13).matches("[0-9A-HJKMNP-TV-Z]{13}");   // e.g. Crockford base32
assertThat(errorCode).isNotBlank();
```

## Optional

Assert presence and the contained value in one chain rather than calling `get()`,
and assert absence with `isEmpty()`.

```java
assertThat(repository.findById(id)).isPresent().contains(expectedOrder);
assertThat(repository.findById(id)).get().extracting(Order::getStatus).isEqualTo(PAID);
assertThat(repository.findById("missing")).isEmpty();
```

## Field-by-field recursive comparison

Use when the type has no value-based `equals`, or when you want to ignore
generated fields. Comparison is limited to the **actual** object's fields (not
symmetrical), and since 3.17 it does **not** call overridden `equals`; it
recurses (re-enable per type/field with `usingOverriddenEquals()` if needed).

```java
// Ignore fields that are nondeterministic or irrelevant (ids, timestamps).
assertThat(created)
    .usingRecursiveComparison()
    .ignoringFields("id", "createdAt")
    .isEqualTo(expectedOrder);

// Or ignore by type / regex.
assertThat(created)
    .usingRecursiveComparison()
    .ignoringFieldsOfTypes(Instant.class)
    .ignoringFieldsMatchingRegexes(".*Id")
    .isEqualTo(expectedOrder);

// Build the expected object with nulls for unpredictable fields and skip them.
assertThat(created)
    .usingRecursiveComparison()
    .ignoringExpectedNullFields()
    .isEqualTo(expectedWithNullId);

// Enforce identical types as well as values.
assertThat(actual).usingRecursiveComparison().withStrictTypeChecking().isEqualTo(expected);
```

Selective and tolerant comparison:

```java
// Compare ONLY the named fields (inverse of ignoringFields); subfields included.
assertThat(created)
    .usingRecursiveComparison()
    .comparingOnlyFields("customerId", "status")
    .isEqualTo(expectedOrder);

// Compare a field/type with tolerance instead of exact equality.
// Prefer the BiPredicate form (withEqualsForType/withEqualsForFields) over a Comparator.
assertThat(created)
    .usingRecursiveComparison()
    .withEqualsForFields((Instant a, Instant b) -> Duration.between(a, b).abs().toMillis() < 50, "createdAt")
    .isEqualTo(expectedOrder);
```

For objects with a generated id or timestamp (see `UNIT-TESTING-PRACTICES.md`),
ignore the generated field, compare only the meaningful fields, or compare the
volatile field with tolerance.

## Extracting

Prefer method references: they are type-safe and refactor-safe. String field
names reach private fields and nested paths (`"race.name"`) but are not checked
at compile time; use them only for that case.

```java
// single value per element
assertThat(users)
    .extracting(User::getName)
    .containsExactly("Bob", "Alice", "Charlie");

// multiple values per element -> tuples
assertThat(orders)
    .extracting(Order::getId, Order::getStatus)
    .containsExactly(
        tuple("order-1", OrderStatus.PAID),
        tuple("order-2", OrderStatus.PENDING));

// extract several fields of a single object (no tuple needed)
assertThat(created)
    .extracting(Order::getCustomerId, Order::getStatus)
    .containsExactly("customer-123", OrderStatus.PENDING);
```

## Collections

- `containsExactly(...)`: exact values, **in order**.
- `containsExactlyInAnyOrder(...)`: exact values, order irrelevant (use this
  when order is not part of the contract, e.g. results from a `HashSet`/map).
- `containsOnly(...)`: same values, ignoring order and duplicates.
- `contains(...)` / `containsAnyOf(...)`: presence checks.
- `containsSequence(...)` / `containsSubsequence(...)`: ordered runs.

Assert on elements without extracting:

```java
assertThat(users).allSatisfy(u -> assertThat(u.getAge()).isGreaterThanOrEqualTo(18));
assertThat(users).anySatisfy(u -> assertThat(u.getName()).isEqualTo("Sam"));
assertThat(users).noneSatisfy(u -> assertThat(u.getRole()).isEqualTo(Role.ADMIN));
// or predicate form
assertThat(users).allMatch(u -> u.getAge() >= 18, "adults");
```

Target a subset with `filteredOn(...)`, then assert:

```java
assertThat(orders)
    .filteredOn(o -> o.getStatus() == OrderStatus.PAID)
    .extracting(Order::getId)
    .containsExactly("order-1", "order-3");

assertThat(orders).filteredOn("status", OrderStatus.PAID).hasSize(2);   // property form
```

Assert each element against its own index-matched requirement (as many
consumers as elements) with `satisfiesExactly` (order matters) or
`satisfiesExactlyInAnyOrder`:

```java
assertThat(events).satisfiesExactly(
    e -> assertThat(e.getType()).isEqualTo(CREATED),
    e -> assertThat(e.getType()).isEqualTo(PAID));
```

Compare a collection of objects field-by-field when the element type has no
value `equals` (recursive element comparator, with the same ignore/compare-only
config as `usingRecursiveComparison`):

```java
assertThat(actualOrders)
    .usingRecursiveFieldByFieldElementComparatorIgnoringFields("id", "createdAt")
    .containsExactly(expectedOrder1, expectedOrder2);
```

## Navigation

```java
assertThat(results).first().isEqualTo(expectedFirst);
assertThat(results).last().isEqualTo(expectedLast);
assertThat(results).element(1).isEqualTo(expectedSecond);

// single-element collections: prefer singleElement() over hasSize(1) + get(0)
assertThat(results).singleElement().isEqualTo(theOnlyOne);

// strongly typed navigation with an InstanceOfAssertFactory
import static org.assertj.core.api.InstanceOfAssertFactories.STRING;
assertThat(names).singleElement(as(STRING)).startsWith("Ma");
```

## Maps

```java
assertThat(grouped)
    .containsOnlyKeys(Category.FOOD, Category.ELECTRONICS)
    .hasEntrySatisfying(Category.FOOD, items -> assertThat(items).hasSize(2));
assertThat(map).containsEntry("k", "v").doesNotContainKey("x");
```

## Exceptions

```java
// standard
assertThatThrownBy(() -> service.process(invalid))
    .isInstanceOf(InvalidInputException.class)
    .hasMessage("Input cannot be null")
    .hasNoCause();

// typed alternative (has shortcuts: assertThatIllegalArgumentException(), etc.)
assertThatExceptionOfType(ValidationException.class)
    .isThrownBy(() -> service.validate(input))
    .withMessageContaining("invalid")
    .withNoCause();

// navigate to the cause to use full throwable assertions on it
assertThatThrownBy(() -> service.run())
    .cause()
    .isInstanceOf(IOException.class)
    .hasMessageContaining("timeout");

// no exception expected
assertThatCode(() -> service.process(valid)).doesNotThrowAnyException();
assertThatNoException().isThrownBy(() -> service.process(valid));
```

Pin a message substring only when the message is yours. Assert a stable slice
(a code, a config key you emit) with `hasMessageContaining(...)` rather than the
exact text, so the test survives wording changes while still pinning the
behaviour. For an exception thrown by a type you do not own (a library or
framework), assert the type alone (`isInstanceOf(...)`); its message is not your
contract and will drift. Where an exception carries structured data, drill into
it with `extracting(...)`; where it wraps another, assert the root cause.

```java
// stable substring, not the whole message
assertThatThrownBy(() -> factory.create(config))
    .isInstanceOf(IllegalStateException.class)
    .hasMessageContaining("orders.base-url");

// assert a field of a structured exception
assertThatThrownBy(() -> new CartSnapshot(cartId, List.of()))
    .isInstanceOf(BusinessRuleException.class)
    .extracting(ex -> ((BusinessRuleException) ex).getSlug())
    .isEqualTo("cart-empty");

// assert on the root cause of a wrapped exception
assertThatThrownBy(() -> service.start())
    .hasRootCauseInstanceOf(IllegalArgumentException.class)
    .rootCause()
    .hasMessageContaining("refresh-lock");
```

BDD capture style: `Throwable thrown = catchThrowable(() -> ...);` then
`assertThat(thrown)...`, or `catchThrowableOfType(MyEx.class, () -> ...)` to get
the typed instance and assert on its custom fields.

## Soft assertions

```java
SoftAssertions softly = new SoftAssertions();
softly.assertThat(response.getStatus()).isEqualTo(Status.SUCCESS);
softly.assertThat(response.getMessage()).isNotEmpty();
softly.assertThat(response.getTimestamp()).isBeforeOrEqualTo(now);
softly.assertAll();   // throws once, listing every failed check
```

(`SoftAssertions` is not supported on Android.) Prefer soft assertions over a
chain of hard assertions when validating several independent properties of one
object, so a single run surfaces all problems.

## Custom domain assertions

For recurring domain checks, write an assertion class extending `AbstractAssert`
with a static `assertThat` entry point and a controlled failure message.

```java
public class OrderAssert extends AbstractAssert<OrderAssert, Order> {

    public OrderAssert(Order actual) {
        super(actual, OrderAssert.class);
    }

    public static OrderAssert assertThat(Order actual) {
        return new OrderAssert(actual);
    }

    public OrderAssert hasStatus(OrderStatus expected) {
        isNotNull();                                  // inherited base assertion
        if (actual.getStatus() != expected) {
            failWithMessage("Expected status <%s> but was <%s>", expected, actual.getStatus());
        }
        return this;                                  // return this to allow chaining
    }
}
```

Gather these entry points in one `MyProjectAssertions` class (static
`assertThat` overloads) for a single import. Use custom assertions for recurring
domain concepts; keep one-off checks inline.

## Conditions

A `Condition` is a named predicate usable with `is`/`has` (single value) and
`are`/`have`/`areAtLeast`/`haveExactly` (collections); combine with `not`,
`allOf`, `anyOf`.

```java
Condition<Order> paid = new Condition<>(o -> o.getStatus() == OrderStatus.PAID, "paid");

assertThat(order).is(paid);
assertThat(orders).areAtLeast(2, paid);
assertThat(order).is(anyOf(paid, cancelled));
```

## Assumptions

`assumeThat(...)` mirrors the assertion API but aborts the test (marks it
skipped) when the condition is not met. Use for environment or precondition
gating, not for the behaviour under test.

```java
assumeThat(System.getProperty("os.name")).containsIgnoringCase("linux");
// test body runs only on Linux; otherwise it is reported as skipped
```

## References

- JUnit: https://junit.org/
- AssertJ: https://assertj.github.io/doc/
