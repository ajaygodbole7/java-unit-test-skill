# Patterns

Code for the patterns referenced in `SKILL.md`. The core rules apply: real
domain objects, fakes over mocks, explicit stubbing, AssertJ.

## 1. Basic test structure

```java
class ServiceUnderTestTest {

    private final ExternalService externalService = mock(ExternalService.class);
    private final ServiceUnderTest serviceUnderTest = new ServiceUnderTest(externalService);

    @Test
    void should_return_processed_result_when_input_is_valid() {
        // given
        var input = new Input("valid-data");
        when(externalService.process(input)).thenReturn("processed");

        // when
        var result = serviceUnderTest.execute(input);

        // then
        assertThat(result)
            .isNotNull()
            .extracting(Result::getValue)
            .isEqualTo("processed");
    }
}
```

Collaborators are `private final` fields; no `@BeforeEach`, no `@Mock`.

## 2. Fake functional interfaces instead of mocking them

```java
// DO: provide a small real implementation
@Test
void should_accept_valid_input() {
    // given
    Validator<String> validator = input ->
        input.length() > 5 ? ValidationResult.valid() : ValidationResult.invalid("Too short");
    var service = new ServiceUnderTest(validator);

    // when
    var result = service.process("valid-input");

    // then
    assertThat(result).isNotNull();
}
```

A single-method interface (validator, mapper, rule, clock, id-generator) is
cheaper and clearer as a lambda than as a stubbed mock.

## 3. Domain objects are always real, never mocked

```java
@Test
void should_process_adult_user() {
    // given
    var user = User.builder()
        .name("John")
        .age(30)
        .build();

    // when
    var result = service.processUser(user);

    // then
    assertThat(result.isAdult()).isTrue();
}
```

## 4. Test builders for complex domain objects

```java
public class OrderTestBuilder {
    private String orderId = "default-id";
    private List<OrderItem> items = new ArrayList<>();
    private OrderStatus status = OrderStatus.PENDING;
    private LocalDateTime createdAt = LocalDateTime.now();

    public static OrderTestBuilder anOrder() {
        return new OrderTestBuilder();
    }

    public OrderTestBuilder withId(String orderId) { this.orderId = orderId; return this; }
    public OrderTestBuilder withItems(OrderItem... items) { this.items = Arrays.asList(items); return this; }
    public OrderTestBuilder withStatus(OrderStatus status) { this.status = status; return this; }
    public OrderTestBuilder completed() { this.status = OrderStatus.COMPLETED; return this; }

    public Order build() {
        return new Order(orderId, items, status, createdAt);
    }
}
```

```java
@Test
void should_calculate_total_for_completed_order() {
    // given
    var order = anOrder()
        .withItems(
            new OrderItem("item-1", Money.of(10.00)),
            new OrderItem("item-2", Money.of(20.00)))
        .completed()
        .build();

    // when
    var total = service.calculateTotal(order);

    // then
    assertThat(total).isEqualTo(Money.of(30.00));
}
```

Provide sensible defaults so tests specify only the fields they care about.
Add intent-revealing shortcuts (`completed()`) for common states. A builder or
object-mother factory reused across the class lets each test state only the one
axis it varies (`anOrder().withStatus(CANCELLED).build()`), so the meaningful
difference in each scenario is obvious. Factories may take varargs for the common
case: `anOrder(anItem("sku-1"), anItem("sku-2"))`. A factory may also accept a
`Consumer<Builder>` so each test customizes one axis inline without a bespoke
overload: `anOrder(o -> o.withStatus(CANCELLED))`.

## 5. AssertJ chaining

```java
@Test
void should_return_filtered_and_sorted_users() {
    // given
    var users = List.of(new User("Alice", 30), new User("Bob", 25), new User("Charlie", 35));

    // when
    var result = service.getAdultUsersSortedByAge(users);

    // then
    assertThat(result)
        .hasSize(3)
        .extracting(User::getName)
        .containsExactly("Bob", "Alice", "Charlie");
}
```

For several fields on one object, prefer `extracting(...)` /
`usingRecursiveComparison()` over a wall of `satisfies` lambdas. Use
`satisfies` only when checks don't map cleanly to matchers.

## 6. Exceptions

```java
@Test
void should_throw_exception_when_input_is_invalid() {
    // given
    var invalidInput = new Input(null);

    // when / then
    assertThatThrownBy(() -> service.process(invalidInput))
        .isInstanceOf(InvalidInputException.class)
        .hasMessage("Input cannot be null")
        .hasNoCause();
}

@Test
void should_throw_exception_with_proper_context() {
    // given
    var input = new Input("invalid");

    // when / then
    assertThatExceptionOfType(ValidationException.class)
        .isThrownBy(() -> service.validate(input))
        .withMessageContaining("invalid")
        .satisfies(ex -> {
            assertThat(ex.getErrorCode()).isEqualTo("VALIDATION_FAILED");
            assertThat(ex.getFields()).contains("input");
        });
}
```

To assert that **no** exception is thrown, use `assertThatCode(...).doesNotThrowAnyException()`.

## 7. Parameterized tests

```java
@ParameterizedTest
@MethodSource("provideInvalidInputs")
void should_reject_invalid_inputs(String input, String expectedError) {
    // when / then
    assertThatThrownBy(() -> service.process(input))
        .isInstanceOf(ValidationException.class)
        .hasMessageContaining(expectedError);
}

private static Stream<Arguments> provideInvalidInputs() {
    return Stream.of(
        Arguments.of(null, "cannot be null"),
        Arguments.of("", "cannot be empty"),
        Arguments.of("   ", "cannot be blank"));
}

@ParameterizedTest
@CsvSource({ "10, 20, 30", "5, 15, 20", "100, 200, 300" })
void should_sum_two_numbers(int a, int b, int expected) {
    // when
    var result = calculator.add(a, b);

    // then
    assertThat(result).isEqualTo(expected);
}
```

Cover the full "blank" set for a string guard in one test by composing
`@NullAndEmptySource` (null and `""`) with `@ValueSource` for whitespace:

```java
@ParameterizedTest
@NullAndEmptySource
@ValueSource(strings = {"  ", "\t"})
void rejects_null_empty_or_blank_customer_id(String customerId) {
    // when / then
    assertThatThrownBy(() -> new OrderRequest(customerId, someItems()))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("customerId");
}
```

## 8. State-first assertion; interaction verification

Assert on the return value or observable state. `verify(...)` couples the test
to the implementation. See `references/UNIT-TESTING-PRACTICES.md`.

### 8a. State assertion

Assert what the method returns, or query the resulting state through a fake:

```java
@Test
void should_fetch_customer_by_specific_id() {
    // given
    var customerId = "customer-123";
    var customer = aCustomer().withId(customerId).build();
    when(customerRepository.findById(customerId)).thenReturn(Optional.of(customer));

    // when
    var result = service.getCustomer(customerId);

    // then
    assertThat(result).isEqualTo(customer);   // no verify() needed
}
```

Prefer a fake and assert its state over verifying a mock: assert that the order
was persisted with the right fields, not that `save` was called.

```java
// fake repository, assert on resulting state
@Test
void should_persist_created_order_with_pending_status() {
    // given
    var repository = new InMemoryOrderRepository();     // fake, not a mock
    var service = new OrderService(repository, fixedClock);
    var request = new OrderRequest("customer-123", items);

    // when
    var created = service.createOrder(request);

    // then
    assertThat(repository.findById(created.getId()))
        .get()
        .extracting(Order::getCustomerId, Order::getStatus)
        .containsExactly("customer-123", OrderStatus.PENDING);
}
```

Stub with the specific parameter, not `any()`. If the code passes the wrong
argument the stub returns nothing and the test fails at the assertion.

### 8b. Interaction verification

Use `verify(...)` only when there is no observable state to assert: a
must-not-happen guarantee, a side effect at an unobservable boundary, or a call
whose occurrence is the behaviour under test.

```java
// must-not-happen contract; no state to observe
@Test
void should_not_call_service_when_cache_hit() {
    // given
    var key = "cached-key";
    when(cache.get(key)).thenReturn(Optional.of("cached-value"));

    // when
    service.getValue(key);

    // then
    verify(externalService, never()).fetchValue(key);
}

// fire-and-forget side effect with no return value or queryable state
@Test
void should_send_confirmation_after_placing_order() {
    // given
    var order = anOrder().build();

    // when
    service.place(order);

    // then
    verify(emailService).sendConfirmation(order.getCustomerEmail());
}
```

To verify an argument, use the exact value or `argThat(...)` for complex
objects. Use `ArgumentCaptor` only when multiple assertions on the captured
value are needed. A fake plus a state assertion (8a) usually tests the same
thing without pinning the implementation.

```java
verify(orderRepository).save(argThat(order ->
    order.getCustomerId().equals("customer-123") &&
    order.getStatus() == OrderStatus.PENDING));
```

### 8c. When `any()` is acceptable

When the argument does not affect the scenario, e.g. asserting a call count:

```java
@Test
void should_log_every_request() {
    // given
    when(logger.log(any())).thenReturn(true);

    // when
    service.handleRequest(request1);
    service.handleRequest(request2);

    // then
    verify(logger, times(2)).log(any());
}
```

## 9. Collections

```java
@Test
void should_return_users_with_expected_properties() {
    // given
    var filter = new UserFilter(18);

    // when
    var users = service.findUsers(filter);

    // then
    assertThat(users)
        .isNotEmpty()
        .allSatisfy(user -> assertThat(user.getAge()).isGreaterThanOrEqualTo(18))
        .extracting(User::getName)
        .containsExactlyInAnyOrder("Alice", "Bob", "Charlie");
}

@Test
void should_group_items_by_category() {
    // given
    var items = List.of(
        new Item("A", Category.FOOD),
        new Item("B", Category.ELECTRONICS),
        new Item("C", Category.FOOD));

    // when
    var grouped = service.groupByCategory(items);

    // then
    assertThat(grouped)
        .containsOnlyKeys(Category.FOOD, Category.ELECTRONICS)
        .hasEntrySatisfying(Category.FOOD, foodItems -> assertThat(foodItems).hasSize(2))
        .hasEntrySatisfying(Category.ELECTRONICS, electronics -> assertThat(electronics).hasSize(1));
}
```

## 10. Nested organization

```java
@Nested
@DisplayName("When processing valid orders")
class ValidOrderProcessing {

    @Test
    void should_accept_order_with_items() {
        // given
        var order = anOrder().withItems(someItems()).build();
        // when
        var result = service.process(order);
        // then
        assertThat(result.isSuccess()).isTrue();
    }

    @Test
    void should_send_confirmation_email() {
        // given
        var order = anOrder().build();
        // when
        service.process(order);
        // then
        verify(emailService).sendConfirmation(order.getCustomerEmail());
    }
}

@Nested
@DisplayName("When processing invalid orders")
class InvalidOrderProcessing {

    @Test
    void should_reject_empty_order() {
        // given
        var order = anOrder().withItems().build();
        // when / then
        assertThatThrownBy(() -> service.process(order))
            .isInstanceOf(EmptyOrderException.class);
    }
}
```

## 11. Edge cases, error paths, state transitions

```java
@Test
void should_handle_empty_list() {
    // given
    var emptyList = List.<Item>of();
    // when
    var result = service.process(emptyList);
    // then
    assertThat(result).isEmpty();
}

@Test
void should_handle_external_service_failure_gracefully() {
    // given
    when(externalService.call()).thenThrow(new ServiceException("Service down"));
    // when
    var result = service.processWithFallback();
    // then
    assertThat(result.isSuccess()).isFalse();
    assertThat(result.getErrorMessage()).contains("Service unavailable");
}

@Test
void should_transition_from_pending_to_completed() {
    // given
    var order = anOrder().withStatus(OrderStatus.PENDING).build();
    // when
    order.complete();
    // then
    assertThat(order.getStatus()).isEqualTo(OrderStatus.COMPLETED);
    assertThat(order.getCompletedAt()).isNotNull();
}

@Test
void should_not_allow_completing_cancelled_order() {
    // given
    var order = anOrder().withStatus(OrderStatus.CANCELLED).build();
    // when / then
    assertThatThrownBy(order::complete)
        .isInstanceOf(IllegalStateException.class)
        .hasMessageContaining("Cannot complete cancelled order");
}
```

## 12. Complete example: service with repository + domain objects

The repository is a fake (its state is what the test asserts on); only the
payment gateway, an external boundary with no observable state, is a mock. Time is fixed
for determinism. Notice there is **no `verify(...)` on the repository**: the
persisted state proves the behavior.

```java
class OrderServiceTest {

    private final InMemoryOrderRepository orderRepository = new InMemoryOrderRepository();
    private final PaymentGateway paymentGateway = mock(PaymentGateway.class);
    private final Clock clock = Clock.fixed(Instant.parse("2025-01-01T00:00:00Z"), ZoneOffset.UTC);
    private final OrderService orderService = new OrderService(orderRepository, paymentGateway, clock);

    @Test
    void should_create_and_persist_order_with_pending_status() {
        // given
        var request = new OrderRequest("customer-123", List.of(new OrderItem("product-1", 2)));

        // when
        var created = orderService.createOrder(request);

        // then: assert on the returned value AND the persisted state
        assertThat(created)
            .extracting(Order::getCustomerId, Order::getStatus)
            .containsExactly("customer-123", OrderStatus.PENDING);
        assertThat(orderRepository.findById(created.getId()))
            .get()
            .extracting(o -> o.getItems().size(), Order::getStatus)
            .containsExactly(1, OrderStatus.PENDING);
    }

    @Test
    void should_mark_order_paid_after_successful_payment() {
        // given
        orderRepository.save(anOrder().withId("order-789").withStatus(OrderStatus.PENDING).build());
        when(paymentGateway.processPayment(any())).thenReturn(new PaymentResult(true, "transaction-123"));

        // when
        orderService.processPayment("order-789");

        // then: the outcome is the stored order's new state, not a save() call
        assertThat(orderRepository.findById("order-789"))
            .get()
            .extracting(Order::getStatus, Order::getPaymentTransactionId)
            .containsExactly(OrderStatus.PAID, "transaction-123");
    }
}
```

If `OrderService` genuinely has no queryable repository (write-only sink), then
verifying `save(argThat(...))` is legitimate, but reach for the fake first.

## 13. Complete example: functional interface as a fake

```java
class ValidationServiceTest {

    @Test
    void should_validate_email_format() {
        // given
        EmailValidator emailValidator = email -> email.contains("@") && email.contains(".");
        var service = new ValidationService(emailValidator);

        // when
        var validResult = service.validateEmail("user@example.com");
        var invalidResult = service.validateEmail("invalid-email");

        // then
        assertThat(validResult.isValid()).isTrue();
        assertThat(invalidResult.isValid()).isFalse();
    }

    @Test
    void should_apply_custom_business_rule() {
        // given
        BusinessRule<Order> minimumOrderRule = order ->
            order.getTotal().compareTo(Money.of(10.00)) >= 0;
        var service = new OrderValidationService(minimumOrderRule);
        var validOrder = anOrder().withTotal(Money.of(15.00)).build();
        var invalidOrder = anOrder().withTotal(Money.of(5.00)).build();

        // when
        var validResult = service.validate(validOrder);
        var invalidResult = service.validate(invalidOrder);

        // then
        assertThat(validResult.isValid()).isTrue();
        assertThat(invalidResult.isValid()).isFalse();
        assertThat(invalidResult.getError()).contains("minimum order");
    }
}
```

## 14. Recording fakes that assert an interaction as state

When the contract is an interaction (a call happened, was skipped, or was
ordered relative to another), a recording fake keeps the assertion state-based.
The fake counts and logs calls; the test reads those counters instead of
`verify(...)`.

```java
class RecordingPaymentGateway implements PaymentGateway {
    private final List<String> charges = new ArrayList<>();

    @Override
    public PaymentResult charge(String customerId, long amountCents) {
        charges.add(customerId + ":" + amountCents);
        return PaymentResult.ok("txn-" + charges.size());
    }

    List<String> charges() { return charges; }
}
```

```java
@Test
void does_not_charge_when_cart_is_empty() {
    // given
    var gateway = new RecordingPaymentGateway();
    var service = new OrderService(orderRepository, gateway, clock);

    // when / then
    assertThatThrownBy(() -> service.checkout(anEmptyCart()))
        .isInstanceOf(EmptyCartException.class);
    assertThat(gateway.charges()).isEmpty();   // interaction asserted as state
}
```

A recording fake reads more clearly than a cluster of `verify(gateway, never())`
calls and never over-specifies argument matching. It also expresses ordering
(`assertThat(fake.events()).containsExactly("claim", "charge")`) without
`InOrder`.

## 15. Echo-argument stubbing for a persistence sink

When a collaborator is genuinely a mock (a write-only boundary with no readable
state), stub `save`/`persist` to return its argument so the code under test
operates on the instance the test supplied and post-conditions stay assertable.

```java
when(orderRepository.save(any(Order.class)))
    .thenAnswer(invocation -> invocation.getArgument(0));
```

Prefer an in-memory fake with a queryable finder (section 12) where one is
possible; reach for echo-stubbing only for a sink that cannot be read back.

## 16. Exhaustive state-machine transitions with `@EnumSource`

Test a transition rule as a full matrix so a newly added enum constant is
covered automatically rather than silently skipped by a hand-listed set.

```java
@ParameterizedTest
@EnumSource(OrderStatus.class)
void delivered_is_terminal_and_rejects_every_transition(OrderStatus target) {
    // when / then
    assertThat(OrderStatus.DELIVERED.canTransitionTo(target)).isFalse();
}

@ParameterizedTest
@EnumSource(value = OrderStatus.class, names = {"PAID", "SHIPPED"})
void pending_can_advance_to_paid_or_shipped(OrderStatus target) {
    assertThat(OrderStatus.PENDING.canTransitionTo(target)).isTrue();
}
```

## 17. Fakes that inject failure or a race

A fake can reproduce an infrastructure fault or race deterministically, so
recovery and retry branches are covered with no real database. Model the fault
as state inside the fake.

```java
// first save collides, then the row is visible: exercises duplicate-key recovery
class DuplicateKeyOnFirstSaveRepository implements OrderRepository {
    private boolean firstSaveAttempted;
    private Order stored;

    @Override
    public void save(Order order) {
        stored = order;
        if (!firstSaveAttempted) {
            firstSaveAttempted = true;
            throw new DuplicateKeyException("orders_customer_id unique violation");
        }
    }

    @Override
    public Optional<Order> findById(String id) { return Optional.ofNullable(stored); }
}
```

The exception the fake throws stands in for the real boundary's failure mode;
the test asserts the service recovers to the expected end state.

To reproduce a race deterministically, override one read method of the fake to
mutate state on the first call (guarded by an `AtomicBoolean`), simulating a
concurrent write arriving mid-operation without threads or a scheduler:

```java
class AddOnFirstReadRepository extends InMemoryOrderRepository {
    private final AtomicBoolean injected = new AtomicBoolean(false);

    @Override
    public Optional<Order> findById(String id) {
        if (injected.compareAndSet(false, true)) {
            save(anOrder().withId(id).build());   // "concurrent" insert on first read
        }
        return super.findById(id);
    }
}
```

## 18. Round-trip and symmetry properties

For an inverse pair (encode/decode, serialize/parse), assert that a value
survives the round trip and that the encoded form is opaque. One property covers
a class of cases a single-direction test would miss.

```java
@Test
void token_round_trips_and_is_opaque() {
    // given
    var orderId = "order-42";

    // when
    var token = OpaqueId.encode(orderId);

    // then
    assertThat(token).isNotNull().isNotEqualTo(orderId);      // opaque
    assertThat(OpaqueId.decode(token)).isEqualTo(orderId);    // round-trips
}
```

## 19. Self-validating fixtures

Object-mother factories return real domain objects with sensible defaults; a
dedicated test asserts the generated data satisfies the domain's own
constraints, so a shared fixture cannot drift out of spec as the domain evolves.

```java
@Test
void an_order_fixture_satisfies_domain_constraints() {
    // when
    var order = anOrder().build();

    // then
    assertThat(order.getCustomerId()).isNotBlank();
    assertThat(order.getItems()).isNotEmpty();
    assertThat(order.getTotal()).isPositive();
}
```

If a fixture draws random values, seed the generator so the same data is produced
every run (see the determinism rule in `UNIT-TESTING-PRACTICES.md`); an unseeded
generator reintroduces the flakiness fixtures are meant to remove.

## 20. Consecutive-call stubbing

When one call must behave differently on successive invocations (an
idempotency or state-change contract), chain the stub so a single test proves
both outcomes.

```java
doNothing()
    .doThrow(new OrderNotFoundException("order-1"))
    .when(orderRepository).delete("order-1");

// first delete succeeds, second reports the order is gone
assertThatCode(() -> service.cancel("order-1")).doesNotThrowAnyException();
assertThatThrownBy(() -> service.cancel("order-1")).isInstanceOf(OrderNotFoundException.class);
```

If a stub must throw an abstract exception type with no public constructor,
instantiate a throwaway concrete subtype inline:

```java
when(orderRepository.save(any())).thenThrow(new DataAccessException("write failed") {});
```

## 21. Deterministic failure of an asynchronous collaborator

For a collaborator that returns a `CompletableFuture`, drive the failure,
timeout, and never-completing paths with an already-resolved future rather than
real threads or sleeps.

```java
when(gateway.chargeAsync(any()))
    .thenReturn(CompletableFuture.failedFuture(new GatewayException("declined")));

// timeout branch
var pending = new CompletableFuture<PaymentResult>();
pending.completeExceptionally(new TimeoutException("gateway timeout"));
when(gateway.chargeAsync(any())).thenReturn(pending);
```

If the code under test handles `InterruptedException`, assert that it restores the
flag: interrupt the thread, drive the call, then assert
`Thread.currentThread().isInterrupted()` is still true, and clear it with
`Thread.interrupted()` so later tests are unaffected.

## 22. Invalid-variant object mothers and a validation matrix

Beyond a valid builder, expose factories that take a valid base and break
exactly one field. Each isolates one validation rule, and a `@MethodSource`
matrix pairs every invalid variant with the field it should report.

```java
static Order orderWithBlankCustomerId(Order base) { return base.toBuilder().customerId("  ").build(); }
static Order orderWithNoItems(Order base)         { return base.toBuilder().items(List.of()).build(); }

@ParameterizedTest(name = "rejects order when {1} is invalid")
@MethodSource("invalidOrders")
void rejects_invalid_orders(Order order, String field) {
    assertThatThrownBy(() -> validator.validate(order))
        .isInstanceOf(ValidationException.class)
        .hasMessageContaining(field);
}

static Stream<Arguments> invalidOrders() {
    var base = anOrder().build();
    return Stream.of(
        Arguments.of(orderWithBlankCustomerId(base), "customerId"),
        Arguments.of(orderWithNoItems(base), "items"));
}
```

## 23. Subclass the code under test to feed a type you do not own

When a collaborator returns a third-party type that cannot be built or should
not be mocked, do not mock it. Subclass the code under test and override a
narrow, protected seam to return a canned instance. All of the subject's own
logic still runs.

```java
@Test
void treats_declined_response_as_a_failed_payment() {
    // given: override only the parse seam; the rest of the client runs for real
    var client = new PaymentClient(config) {
        @Override
        protected GatewayResponse parse(GatewayRequest request) {
            return GatewayResponse.declined("insufficient_funds");
        }
    };

    // when / then
    assertThatThrownBy(() -> client.charge(anOrder().build()))
        .isInstanceOf(PaymentFailedException.class)
        .hasMessageContaining("insufficient_funds");
}
```

Keep the seam as small as possible so the test exercises the real code around it.

## 24. Golden-file contract test

Pin a wire format with a checked-in fixture kept deliberately separate from the
production source, so a rename cannot be applied to both sides in one edit. The
test asserts the implementation round-trips the fixture and that every field the
format requires is present.

```java
@Test
void order_json_matches_the_golden_contract() throws Exception {
    // given
    var golden = Files.readString(Path.of("src/test/resources/contract/order.json"));

    // when
    var parsed = OrderCodec.fromJson(golden);
    var reSerialized = OrderCodec.toJson(parsed);

    // then
    assertThat(OrderCodec.fromJson(reSerialized)).isEqualTo(parsed);   // round-trips
    for (var field : List.of("id", "customerId", "status", "items")) {
        assertThat(reSerialized).as("required field %s", field).contains("\"" + field + "\"");
    }
}
```

## 25. Testing a custom validation constraint

Test a custom constraint against a real validator applied to a small annotated
holder, asserting on the resulting violation set. This needs no framework
context.

```java
private final Validator validator = Validation.buildDefaultValidatorFactory().getValidator();

record Holder(@ValidCurrency String currencyCode) {}

@Test
void rejects_an_unknown_currency_code() {
    // when
    var violations = validator.validate(new Holder("XYZ"));

    // then
    assertThat(violations)
        .singleElement()
        .satisfies(v -> {
            assertThat(v.getMessage()).isEqualTo("Invalid ISO 4217 currency code");
            assertThat(v.getPropertyPath()).hasToString("currencyCode");
        });
}
```
