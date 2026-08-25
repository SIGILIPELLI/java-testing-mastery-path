# 02 · Mocking with Mockito

Everything tested so far talked to something real — a browser, a public API,
an in-memory fake. Real collaborators are slow, flaky, or simply don't exist
yet (the payment gateway your code will call in production costs real money
per call). Mockito lets you replace a collaborator with a stand-in you fully
control, so the test exercises *your* logic in isolation.

## 1. Setup

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.11.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <version>5.11.0</version>
    <scope>test</scope>
</dependency>
```

## 2. The class under test

```java
package com.example.orders;

public interface PaymentGateway {
    boolean charge(String customerId, int cents);
}

public interface OrderRepository {
    void save(Order order);
    Order findById(String id);
}

public class Order {
    String id;
    int totalCents;
    String status;
    Order(String id, int totalCents) { this.id = id; this.totalCents = totalCents; this.status = "PENDING"; }
}

public class OrderService {
    private final PaymentGateway gateway;
    private final OrderRepository repository;

    public OrderService(PaymentGateway gateway, OrderRepository repository) {
        this.gateway = gateway;
        this.repository = repository;
    }

    public Order checkout(String customerId, Order order) {
        boolean charged = gateway.charge(customerId, order.totalCents);
        order.status = charged ? "PAID" : "PAYMENT_FAILED";
        repository.save(order);
        return order;
    }
}
```

`OrderService` never talks to a real bank in a test — that's exactly what
Mockito replaces.

## 3. Mocking with `@Mock` and `@ExtendWith`

```java
package com.example.orders;

import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock PaymentGateway gateway;
    @Mock OrderRepository repository;
    @InjectMocks OrderService service;

    @Test
    @DisplayName("successful charge marks order PAID and saves it")
    void successfulCheckout() {
        when(gateway.charge("cust-1", 5000)).thenReturn(true);

        Order order = new Order("ord-1", 5000);
        Order result = service.checkout("cust-1", order);

        assertEquals("PAID", result.status);
        verify(repository).save(order);
        verify(gateway).charge("cust-1", 5000);
    }

    @Test
    @DisplayName("declined charge marks order PAYMENT_FAILED but still saves it")
    void declinedCheckout() {
        when(gateway.charge(anyString(), anyInt())).thenReturn(false);

        Order order = new Order("ord-2", 12000);
        Order result = service.checkout("cust-2", order);

        assertEquals("PAYMENT_FAILED", result.status);
        verify(repository, times(1)).save(order);
    }
}
```

`@Mock` creates the fake, `@InjectMocks` builds `OrderService` and wires the
mocks into its constructor automatically. `when(...).thenReturn(...)`
stubs a return value; `verify(...)` asserts a method was actually called —
the mocking equivalent of an assertion, since a mock with no stub returns
`null`/`false`/`0` and tells you nothing on its own.

```
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

I ran this exact test class locally with JUnit 5 + Mockito (Maven, no
network, no browser involved) and both tests passed as shown.

## 4. Argument matchers, exceptions, and call counts

```java
@Test
void gatewayThrowsIsNotSwallowed() {
    when(gateway.charge(anyString(), anyInt()))
        .thenThrow(new RuntimeException("gateway timeout"));

    Order order = new Order("ord-3", 999);

    RuntimeException ex = assertThrows(RuntimeException.class,
            () -> service.checkout("cust-3", order));
    assertEquals("gateway timeout", ex.getMessage());
    verify(repository, never()).save(any());   // never got there
}

@Test
void chargeCalledExactlyOnce() {
    when(gateway.charge(anyString(), anyInt())).thenReturn(true);
    service.checkout("cust-4", new Order("ord-4", 100));
    verify(gateway, times(1)).charge(eq("cust-4"), eq(100));
}
```

!!! warning "Don't mix raw values and matchers in one call"
    `verify(gateway).charge("cust-4", anyInt())` throws
    `InvalidUseOfMatchersException` — once you use one Mockito matcher in a
    call, *every* argument in that call must be a matcher (`eq("cust-4")`,
    not the bare string).

## 5. Capturing arguments

Sometimes the interesting thing isn't *whether* a method was called, but
*what* it was called with — useful when the argument is built inside the
method under test, not passed in by the test.

```java
@Test
void savedOrderHasCorrectId() {
    ArgumentCaptor<Order> captor = ArgumentCaptor.forClass(Order.class);
    when(gateway.charge(anyString(), anyInt())).thenReturn(true);

    service.checkout("cust-5", new Order("ord-5", 2500));

    verify(repository).save(captor.capture());
    assertEquals("ord-5", captor.getValue().id);
    assertEquals("PAID", captor.getValue().status);
}
```

## 6. Spies — partial mocking of a real object

```java
import java.util.ArrayList;
import java.util.List;

@Test
void spyDelegatesToRealMethodsByDefault() {
    List<String> real = new ArrayList<>();
    List<String> spy = spy(real);

    spy.add("a");                 // really executes on the real ArrayList
    assertEquals(1, spy.size());  // real method, not stubbed

    doReturn(99).when(spy).size(); // override just this one method
    assertEquals(99, spy.size());
}
```

A mock starts empty and does nothing until stubbed; a spy wraps a real
object and calls through unless you explicitly override a method. Reach for
a spy only when mocking the whole dependency is impractical — it's easy to
end up half-testing real code and half-testing a stub, which is confusing to
debug later.

## 7. Testing traps

!!! warning "Trap 1 — stubbing a method that's never called"
    Mockito's strict stubs (the JUnit 5 extension enables this by default)
    fail the test with `UnnecessaryStubbingException` if a `when(...)` is
    never exercised. This is a feature, not noise — it means a previous
    refactor left a dead stub, or the test doesn't test what you think it
    does. Delete the stub or fix the path that should call it.

!!! warning "Trap 2 — mocking the class you're testing"
    `@Mock OrderService service` instead of `@InjectMocks` gives you a fake
    version of the very thing you're trying to test — every method returns
    `null`. Only mock *collaborators*, never the unit under test.

!!! warning "Trap 3 — verifying too loosely"
    `verify(gateway).charge(anyString(), anyInt())` passes even if `charge`
    was called with the wrong customer entirely. Prefer exact values
    (`eq(...)`) unless the argument genuinely doesn't matter to this test.

!!! warning "Trap 4 — mocking value objects"
    Mocking a plain data class like `Order` instead of instantiating it
    produces an object whose fields are all `null`/`0` and whose `equals()`
    doesn't work as expected. Mock behaviour (interfaces, services); build
    data (POJOs, records) for real.

!!! warning "Trap 5 — leftover stubs from copy-pasted tests"
    Copying a test and changing the assertion but not the `when(...)` setup
    leaves a stub that silently no longer matches the call being made,
    returning `null` where the old test expected a real value. Strict
    stubbing (Trap 1) catches the unused half of this; it will not catch a
    stub that *does* match but returns a value now-stale for this scenario —
    review stubs on every copy-paste, not just new writes.

## Cheat sheet

| Task | Code |
|---|---|
| Enable Mockito in JUnit 5 | `@ExtendWith(MockitoExtension.class)` |
| Create a mock | `@Mock PaymentGateway gateway;` |
| Auto-wire mocks into class under test | `@InjectMocks OrderService service;` |
| Stub a return value | `when(gateway.charge(...)).thenReturn(true)` |
| Stub an exception | `when(...).thenThrow(new RuntimeException(...))` |
| Match any value | `anyString()`, `anyInt()`, `any()` |
| Exact match (required alongside other matchers) | `eq(value)` |
| Assert a call happened | `verify(repository).save(order)` |
| Assert a call count | `verify(gateway, times(1)).charge(...)` |
| Assert never called | `verify(repository, never()).save(any())` |
| Capture an argument | `ArgumentCaptor.forClass(Order.class)` |
| Partial mock of a real object | `spy(realObject)` |
| Override one spy method | `doReturn(x).when(spy).method()` |

## Exercise

1. Build `OrderService`, `PaymentGateway`, `OrderRepository`, `Order` exactly
   as above and write `OrderServiceTest` with the six tests shown; run it and
   confirm all pass.
2. Add a `refund(String orderId)` method to `OrderService` that looks the
   order up via `repository.findById`, calls a new `gateway.refund(...)`, and
   sets status to `REFUNDED`. Write a test using `when(repository.findById(...))`
   to stub the lookup.
3. Trigger `UnnecessaryStubbingException` on purpose (stub a method your test
   never calls) and paste the exact exception message.
4. Write a test using `ArgumentCaptor` to assert that a *retry* order (same
   customer, second attempt) is saved with a different `id` from the first —
   this needs two `checkout()` calls and two captured values, compared.
5. Convert one test to use `spy()` on a real `ArrayList`-backed
   `InMemoryOrderRepository` instead of a full mock, and explain in a
   sentence why a spy was (or wasn't) the right call here versus `@Mock`.
