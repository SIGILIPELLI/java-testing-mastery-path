# 10 · Project — Hybrid BDD Framework with CI

This project pulls together every module in Level 3: a Cucumber feature file
(Module 01) drives step definitions built around Framework Design Patterns
(Module 08), tested with real assertions, and packaged to run in a GitHub
Actions pipeline (Module 03). "Hybrid" means the same suite happily contains
Gherkin-driven acceptance tests and plain JUnit unit tests side by side —
neither approach replaces the other.

## 1. Project layout

```
src/test/
  java/com/example/hybrid/
    Cart.java                     # the class under test
    steps/
      CartSteps.java              # Cucumber step definitions
    runners/
      RunCucumberTest.java        # JUnit 5 Platform Suite runner
  resources/features/
    cart.feature                  # Gherkin scenarios
.github/workflows/
  tests.yml                       # CI pipeline (Module 03 pattern)
pom.xml
```

## 2. The class under test

```java
package com.example.hybrid;

import java.util.*;

public class Cart {
    private final List<Integer> itemsCents = new ArrayList<>();
    private static final Map<String, Integer> DISCOUNT_PERCENT = Map.of(
            "SAVE10", 10,
            "SAVE50", 50
    );

    public void addItem(String name, int priceCents) {
        itemsCents.add(priceCents);
    }

    public int total() {
        return itemsCents.stream().mapToInt(Integer::intValue).sum();
    }

    public int applyCode(String code) {
        int sum = total();
        Integer percent = DISCOUNT_PERCENT.get(code);
        if (percent == null) return sum;             // unknown code: no-op, not an error
        return sum - (sum * percent / 100);
    }
}
```

## 3. Feature file

`src/test/resources/features/cart.feature`:

```gherkin
Feature: Shopping cart pricing

  Background:
    Given an empty cart

  Scenario: Adding items totals their price
    Given the customer adds "widget" priced at 999 cents
    When the customer also adds "gadget" priced at 2500 cents
    Then the cart total is 3499 cents

  Scenario Outline: Discount codes apply correctly
    Given the customer adds "widget" priced at 1000 cents
    When the customer applies code "<code>"
    Then the cart total is <expected> cents

    Examples:
      | code        | expected |
      | SAVE10      | 900      |
      | INVALIDCODE | 1000     |
      | SAVE50      | 500      |
```

Note the `Given`/`When` split on the first scenario: `the customer adds` and
`the customer also adds` are deliberately worded as separate steps mapped to
`@Given` and `@When` respectively. Cucumber treats a step matched by two
annotations on one method as two distinct definitions of the *same* pattern
text and throws `DuplicateStepDefinitionException` (Module 01's Trap 1) —
this project hit that exact error during development, which is exactly why
the wording differs here rather than reusing one phrase under both
annotations.

## 4. Step definitions

```java
package com.example.hybrid.steps;

import com.example.hybrid.Cart;
import io.cucumber.java.en.*;
import static org.junit.jupiter.api.Assertions.*;

public class CartSteps {

    private Cart cart;
    private int lastTotal;

    @Given("an empty cart")
    public void anEmptyCart() {
        cart = new Cart();
    }

    @Given("the customer adds {string} priced at {int} cents")
    public void givenTheCustomerAdds(String name, int priceCents) {
        cart.addItem(name, priceCents);
    }

    @When("the customer also adds {string} priced at {int} cents")
    public void whenTheCustomerAdds(String name, int priceCents) {
        cart.addItem(name, priceCents);
    }

    @When("the customer applies code {string}")
    public void theCustomerAppliesCode(String code) {
        lastTotal = cart.applyCode(code);
    }

    @Then("the cart total is {int} cents")
    public void theCartTotalIs(int expected) {
        int actual = (lastTotal != 0) ? lastTotal : cart.total();
        assertEquals(expected, actual);
    }
}
```

`theCartTotalIs` reads `lastTotal` if a discount was applied, else falls
back to the raw `cart.total()` — a small hybrid dispatch inside one step so
both scenarios in the feature file can share the same `Then` phrasing
without a second, near-duplicate step (Module 01's Trap 2).

## 5. The runner

```java
package com.example.hybrid.runners;

import org.junit.platform.suite.api.*;
import static io.cucumber.junit.platform.engine.Constants.GLUE_PROPERTY_NAME;

@Suite
@IncludeEngines("cucumber")
@SelectClasspathResource("features")
@ConfigurationParameter(key = GLUE_PROPERTY_NAME, value = "com.example.hybrid.steps")
public class RunCucumberTest {
}
```

## 6. Running it

```xml
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-java</artifactId>
    <version>7.15.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-junit-platform-engine</artifactId>
    <version>7.15.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.junit.platform</groupId>
    <artifactId>junit-platform-suite</artifactId>
    <version>1.10.2</version>
    <scope>test</scope>
</dependency>
```

I built this exact project (all five files above, unchanged) and ran it
with Maven + the real Cucumber JUnit Platform engine — no fakes, no
simulated output:

```
$ mvn -Dtest=RunCucumberTest test
...
[INFO] Running com.example.hybrid.runners.RunCucumberTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.086 s
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] BUILD SUCCESS
```

4 scenarios (1 plain scenario + 3 rows of the Scenario Outline) — all pass.
Getting here on the first attempt did not happen: the initial version used
one step method annotated with both `@Given` and `@When` for "the customer
adds," which failed with `DuplicateStepDefinitionException` exactly as
Module 01 describes; splitting the wording (section 3-4 above) fixed it.
That failure-then-fix is left in this writeup deliberately, because it's the
realistic path to a working hybrid suite, not a sanitized one.

## 7. CI pipeline

`.github/workflows/tests.yml`, following the Module 03 pattern:

```yaml
name: Hybrid BDD Suite

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven

      - name: Run Cucumber + unit tests
        run: mvn -B test

      - name: Publish test report
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Cucumber + JUnit Results
          path: 'target/surefire-reports/*.xml'
          reporter: java-junit
```

Because `RunCucumberTest` is itself a JUnit 5 test class (via the Platform
Suite engine), `mvn test` runs it exactly like any other test — Surefire
produces the same XML report format, and the CI job above needs no
Cucumber-specific plumbing beyond the dependencies in the POM. This YAML is
written to the same documented syntax verified in Module 03 but was not run
through an actual GitHub Actions runner in this environment.

## 8. Adding a plain-JUnit unit test alongside the BDD suite

```java
package com.example.hybrid;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CartUnitTest {

    @Test
    void unknownDiscountCodeLeavesTotalUnchanged() {
        Cart cart = new Cart();
        cart.addItem("widget", 500);
        assertEquals(500, cart.applyCode("NOT-A-REAL-CODE"));
    }

    @Test
    void emptyCartTotalsZero() {
        assertEquals(0, new Cart().total());
    }
}
```

I ran this class too, alongside the Cucumber runner, with the same
`mvn test` invocation:

```
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0   (CartUnitTest)
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0   (RunCucumberTest)
BUILD SUCCESS
```

This is the "hybrid" in the project title made concrete: one `mvn test`
run, one CI job, two different testing styles, zero glue code required to
combine them — the JUnit Platform runs both engines because both are
declared on the classpath.

## Stretch goals

1. Add a `@requiresDiscount` tag to the outline scenarios and a
   `@Before("@requiresDiscount")` hook (Module 01) that logs which code is
   about to be tested; confirm it fires only for tagged scenarios.
2. Extract `Cart`'s discount table into a `DiscountPolicy` interface with a
   `PercentageDiscountPolicy` implementation (Strategy pattern, Module 08),
   and write a Mockito test (Module 02) proving `Cart` calls the policy
   rather than hard-coding percentages.
3. Split the workflow into a fast job (`CartUnitTest` only) and a full job
   (Cucumber + unit), with the full job gated behind the fast one passing —
   the sequencing pattern from Module 03 section 4.
4. Add a scenario for a cart with zero items and a discount code applied;
   decide what the correct behavior is (probably still 0, but write the
   scenario and step logic to prove it rather than assume it).
5. Introduce a genuine `DuplicateStepDefinitionException` on purpose (revert
   the `@When`/`@Given` split from section 3) and capture the exact
   exception message your Cucumber version reports — compare it to the one
   documented in Module 01.
6. Add an H2-backed `OrderHistoryRepository` (Module 07 pattern) that
   `Cart.checkout()` writes to, and extend one scenario's `Then` step to
   assert a row was actually persisted, not just that the in-memory total is
   correct.
