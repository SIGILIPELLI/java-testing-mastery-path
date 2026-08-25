# 03 · Contract Testing (Pact)

Level 4 Module 01 flagged the problem: at scale, E2E tests that span team
boundaries couple teams together in ways that make one team's outage into
another team's test failure. Contract testing solves this by testing the
*agreement* between a consumer and a provider — independently, without
either side needing the other running — and catching a broken agreement
before it ever reaches a shared environment.

## 1. The core idea

- **Consumer** — the service that calls another service (e.g. a checkout
  service calling an inventory service).
- **Provider** — the service being called.
- **Contract (pact file)** — a JSON document, generated *from the consumer's
  own tests*, describing every request it makes and the response shape it
  expects.
- **Verification** — the provider replays every recorded request against
  its real implementation and confirms the real response still matches the
  contract.

The consumer never needs the real provider to write its test; the provider
never needs the real consumer to verify compliance. Each side runs
independently, in its own CI pipeline, and Pact's broker (or shared files)
is the only thing connecting them.

## 2. Setup

```xml
<dependency>
    <groupId>au.com.dius.pact.consumer</groupId>
    <artifactId>junit5</artifactId>
    <version>4.6.14</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>au.com.dius.pact.provider</groupId>
    <artifactId>junit5</artifactId>
    <version>4.6.14</version>
    <scope>test</scope>
</dependency>
```

## 3. Consumer side — defining the expected interaction

```java
package com.example.checkout;

import au.com.dius.pact.consumer.dsl.PactDslWithProvider;
import au.com.dius.pact.consumer.junit5.PactConsumerTestExt;
import au.com.dius.pact.consumer.junit5.PactTestFor;
import au.com.dius.pact.core.model.RequestResponsePact;
import au.com.dius.pact.core.model.annotations.Pact;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;

import java.io.IOException;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

@ExtendWith(PactConsumerTestExt.class)
class InventoryClientPactTest {

    @Pact(consumer = "checkout-service", provider = "inventory-service")
    RequestResponsePact stockLevelPact(PactDslWithProvider builder) {
        return builder
            .given("item widget-1 has 5 units in stock")
            .uponReceiving("a request for stock level of widget-1")
                .path("/stock/widget-1")
                .method("GET")
            .willRespondWith()
                .status(200)
                .headers(Map.of("Content-Type", "application/json"))
                .body("{\"itemId\": \"widget-1\", \"quantity\": 5}")
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "stockLevelPact")
    void checkoutClientParsesStockResponse(au.com.dius.pact.core.model.messaging.Message pact) {
        // Pact spins up a mock provider on a random port matching the interaction above
    }

    @Test
    @PactTestFor(pactMethod = "stockLevelPact")
    void clientHandlesRealisticStockResponse() throws IOException {
        InventoryClient client = new InventoryClient("http://localhost:8080");
        StockLevel stock = client.getStock("widget-1");
        assertEquals("widget-1", stock.itemId());
        assertEquals(5, stock.quantity());
    }
}
```

```java
package com.example.checkout;

public record StockLevel(String itemId, int quantity) {}

public class InventoryClient {
    private final String baseUrl;
    public InventoryClient(String baseUrl) { this.baseUrl = baseUrl; }

    public StockLevel getStock(String itemId) throws java.io.IOException {
        // real implementation: RestAssured or HttpClient call to baseUrl + "/stock/" + itemId,
        // parsed into StockLevel -- same shape as Level 2's RestAssured clients
        throw new UnsupportedOperationException("wire to a real HTTP client");
    }
}
```

Running the consumer test produces `checkout-service-inventory-service.json`
in `target/pacts/` — the contract, generated as a *side effect* of a test
that also proves the consumer's own parsing code works against exactly that
shape.

## 4. Provider side — verifying the real service honors the contract

```java
package com.example.inventory;

import au.com.dius.pact.provider.junit5.PactVerificationInvocationContextProvider;
import au.com.dius.pact.provider.junit5.HttpTestTarget;
import au.com.dius.pact.provider.junitsupport.Provider;
import au.com.dius.pact.provider.junitsupport.State;
import au.com.dius.pact.provider.junitsupport.loader.PactFolder;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.TestTemplate;
import org.junit.jupiter.api.extension.ExtendWith;

@Provider("inventory-service")
@PactFolder("../checkout-service/target/pacts")   // or a Pact Broker URL in real setups
class InventoryProviderPactTest {

    @BeforeEach
    void setUp(au.com.dius.pact.provider.junitsupport.target.TestTarget target) {
        ((HttpTestTarget) target).setPort(8080);
        // real inventory-service running on 8080 for this test run
    }

    @State("item widget-1 has 5 units in stock")
    void widgetHasFiveInStock() {
        // seed the real (or test) database so the provider actually returns quantity: 5
        // e.g. inventoryRepository.setStock("widget-1", 5);
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verifyPact() {
        // Pact replays every recorded interaction and asserts the real response matches
    }
}
```

`@State` is the crucial piece: it maps the consumer's `.given(...)` string
to actual setup code on the provider side, so the provider is tested in the
exact state the consumer assumed when writing its contract — not whatever
data happens to be lying around.

## 5. What breaks the contract, and what CI does about it

```
Verifying a pact between checkout-service and inventory-service
  Given item widget-1 has 5 units in stock
  a request for stock level of widget-1
    returns a response which
      has status code 200 (OK)
      includes headers "Content-Type" with value "application/json" (OK)
      has a matching body (FAILED)

Failures:
1) Body: $.quantity: Expected 5 but received "5" (type mismatch: Integer vs String)
```

This is the exact class of bug contract testing exists to catch: the
provider changed `quantity` from a number to a string (perhaps to add units
like `"5 units"` later), silently breaking every consumer that expects an
integer, with zero E2E environment required to detect it.

I did not run an actual Pact broker, mock provider server, or real
provider verification in this environment (no Pact CLI/broker available
headlessly here). This entire module — consumer DSL, `@State` mapping, and
the sample verification failure — is reviewed against Pact JVM's
documented API and typical output format, not executed. The `record
StockLevel`/`InventoryClient` shapes follow the same client patterns
verified working with real RestAssured HTTP calls in Level 2 Module 04.

## 6. Where contract testing fits versus E2E

| Question | Contract test | E2E test |
|---|---|---|
| Does my code handle the response shape correctly? | Yes | Yes, indirectly |
| Does the provider actually return that shape today? | Yes (via verification) | Yes |
| Does the whole user journey work end-to-end? | No | Yes |
| Needs both services running together? | No | Yes |
| Runs in each team's own CI, independently? | Yes | Usually no |
| Catches a UI regression? | No | Yes |

Contract tests replace *some* E2E tests (the ones purely checking an API
boundary) — they don't replace the smaller number of true user-journey E2E
tests that verify the whole system genuinely works together.

## 7. Testing traps

!!! warning "Trap 1 — writing the pact by hand instead of generating it"
    Hand-writing the JSON contract (instead of letting the consumer's test
    generate it via the DSL) disconnects the contract from any real
    consumer code — nothing proves the consumer actually parses that shape
    correctly, defeating half the value of the practice.

!!! warning "Trap 2 — provider states that don't match production reality"
    A `@State` setup that seeds unrealistic data (e.g. always exactly one
    hardcoded record) can pass verification while the real code path — the
    one with pagination, nulls, or a second matching record — is never
    exercised.

!!! warning "Trap 3 — contracts that never get re-verified"
    A pact file generated once and never re-run against the provider in CI
    is a snapshot, not a contract — it stops catching drift the moment
    either side changes. Verification must run continuously, ideally via a
    Pact Broker webhook triggering on every provider deploy.

!!! warning "Trap 4 — over-specifying the contract"
    Asserting on every field's exact value (including ones the consumer
    doesn't actually use) makes the contract fail on harmless provider
    additions. Use Pact's matchers (`like(...)`, `eachLike(...)`) to assert
    type/shape for fields the consumer only reads structurally, and exact
    values only for fields whose specific value the consumer's logic
    depends on.

!!! warning "Trap 5 — treating contract tests as a replacement for all integration testing"
    A contract test proves the *shape* is honored; it does not prove the
    *business logic* connecting two real, running services under real load
    behaves correctly. Keep a small number of true integration/E2E tests
    for that — Module 01's "both, not either" answer applies here too.

## Cheat sheet

| Concept | Where it lives |
|---|---|
| Consumer defines expected interaction | `@Pact` method + `PactDslWithProvider` |
| Generated contract file | `target/pacts/<consumer>-<provider>.json` |
| Provider maps state to real setup | `@State("...")` method |
| Provider replays and verifies | `@PactVerificationInvocationContextProvider` |
| Type/shape-only matcher | `like(...)`, `eachLike(...)` |
| Sharing contracts across teams | Pact Broker (or a shared artifact/folder) |
| Re-verify on every change | Broker webhook or CI job on provider deploy |

## Exercise

1. Write `InventoryClientPactTest` and a minimal `InventoryClient`
   implementation, generate the pact file, and inspect the resulting JSON
   in `target/pacts/`.
2. Write `InventoryProviderPactTest` against a real (even minimal, in-memory)
   inventory service implementation and confirm verification passes.
3. Deliberately change the provider's response field name from `quantity`
   to `qty`, rerun verification, and capture the exact failure Pact
   reports.
4. Rewrite the contract to use `like(5)` instead of a literal `5` for
   `quantity`, and explain in a sentence what class of provider change this
   now tolerates versus the earlier, exact-value version.
5. For a real (or hypothetical) two-service relationship you know, list
   three interactions you'd write contract tests for, and one interaction
   you'd deliberately leave to a true E2E test instead — justify the split
   using the table in section 6.
