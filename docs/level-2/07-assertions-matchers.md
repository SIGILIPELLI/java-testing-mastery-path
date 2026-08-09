# 07 · Assertions & Matchers

An assertion has two jobs: decide pass or fail, and — when it fails — tell
whoever reads the CI log what went wrong without them opening the code.
`assertTrue(list.size() == 3)` does the first job and fails the second: the
report says only "expected true but was false". This module is about writing
the second kind of assertion, with Hamcrest, AssertJ and soft assertions.

## 1. Why the default assertion is not enough

```java
// The report says: expected: <true> but was: <false>
assertTrue(order.getItems().size() == 3);

// The report says: expected: <3> but was: <2>
assertEquals(3, order.getItems().size());

// The report says: Expected size: 3 but was: 2 in:
//                  ["Keyboard", "Mouse"]
assertThat(order.getItems()).hasSize(3);
```

Only the third told you *what was actually in the list*. Over a year of CI
failures that difference is measured in days of engineer time.

## 2. JUnit 5 built-ins

```java
package com.example.assertions;

import org.junit.jupiter.api.*;

import java.time.Duration;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class JUnitAssertionsTest {

    @Test
    void coreAssertions() {
        assertEquals(15, new Discounts().percentFor(10000),
                "Orders of 10000 and above get the top tier");
        assertNotEquals(0, new Discounts().percentFor(1000));
        assertTrue("qa@example.com".contains("@"), "Email must contain @");
        assertNull(null);
        assertNotNull(List.of());
        assertSame(Boolean.TRUE, Boolean.TRUE);
        assertArrayEquals(new int[]{1, 2, 3}, new int[]{1, 2, 3});
        assertIterableEquals(List.of("a", "b"), List.of("a", "b"));
        assertLinesMatch(List.of("id=\\d+"), List.of("id=42"));   // regex per line
    }

    @Test
    void groupedAssertions() {
        Discounts discounts = new Discounts();

        assertAll("discount tiers",
                () -> assertEquals(0,  discounts.percentFor(999.0)),
                () -> assertEquals(5,  discounts.percentFor(1000.0)),
                () -> assertEquals(10, discounts.percentFor(5000.0)),
                () -> assertEquals(15, discounts.percentFor(10000.0)));
    }

    @Test
    void exceptions() {
        IllegalArgumentException thrown = assertThrows(
                IllegalArgumentException.class,
                () -> new Discounts().percentFor(-1));

        assertEquals("Order amount cannot be negative", thrown.getMessage());
        assertDoesNotThrow(() -> new Discounts().percentFor(0));
    }

    @Test
    void timing() {
        assertTimeout(Duration.ofSeconds(2), () -> new Discounts().percentFor(500));
    }

    static class Discounts {
        int percentFor(double amount) {
            if (amount < 0) throw new IllegalArgumentException("Order amount cannot be negative");
            if (amount >= 10000) return 15;
            if (amount >= 5000)  return 10;
            if (amount >= 1000)  return 5;
            return 0;
        }
    }
}
```

```
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0
```

`assertAll` is the one to internalise. Without it, the first failing
boundary hides the other three, so you fix one, re-run for three minutes,
and discover the next — four times over.

## 3. Hamcrest

Hamcrest reads as a sentence and produces "Expected: … but: …" messages. It
is also what RestAssured's `.body(...)` uses, so you already know it.

```java
import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.*;

assertThat(price, is(closeTo(249.99, 0.001)));
assertThat(title, containsStringIgnoringCase("selenium"));
assertThat(username, allOf(notNullValue(), not(emptyString()), hasLength(8)));
assertThat(statusCode, anyOf(is(200), is(201), is(204)));
assertThat(browsers, hasItems("chrome", "firefox"));
assertThat(browsers, hasSize(3));
assertThat(results, everyItem(greaterThan(0)));
assertThat(config, hasEntry("browser", "chrome"));
assertThat(errorMessage, startsWith("Your username"));
```

A failure reads:

```
java.lang.AssertionError:
Expected: a collection containing "edge"
     but: was <[chrome, firefox]>
```

## 4. AssertJ — the modern default

One entry point, `assertThat`, and the IDE autocompletes every assertion
valid for that type.

```java
package com.example.assertions;

import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

class AssertJTest {

    record Product(String name, double price, boolean inStock) { }

    @Test
    void strings() {
        String flash = "You logged into a secure area!";

        assertThat(flash)
                .isNotBlank()
                .startsWith("You logged")
                .containsIgnoringCase("SECURE")
                .doesNotContain("error")
                .hasSizeGreaterThan(10)
                .matches(".*area!$");
    }

    @Test
    void collections() {
        List<Product> cart = List.of(
                new Product("Keyboard", 49.99, true),
                new Product("Mouse", 19.99, true),
                new Product("Monitor", 199.99, false));

        assertThat(cart)
                .hasSize(3)
                .extracting(Product::name)
                .containsExactly("Keyboard", "Mouse", "Monitor");

        assertThat(cart)
                .filteredOn(Product::inStock)
                .hasSize(2)
                .extracting(Product::name)
                .containsExactlyInAnyOrder("Mouse", "Keyboard");

        assertThat(cart)
                .allSatisfy(p -> assertThat(p.price()).isPositive())
                .anyMatch(p -> p.price() > 100);

        assertThat(cart)
                .extracting(Product::name, Product::inStock)
                .contains(tuple("Monitor", false));
    }

    @Test
    void numbersAndMaps() {
        assertThat(249.99).isCloseTo(250.0, within(0.05));
        assertThat(15).isBetween(0, 20).isNotNegative();

        Map<String, String> config = Map.of("browser", "chrome", "headless", "true");
        assertThat(config)
                .containsEntry("browser", "chrome")
                .containsKeys("headless")
                .doesNotContainKey("password");
    }

    @Test
    void exceptions() {
        assertThatThrownBy(() -> { throw new IllegalStateException("driver not started"); })
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("not started");

        assertThatNoException().isThrownBy(() -> Integer.parseInt("42"));
    }

    @Test
    void withContext() {
        int actual = 3;
        assertThat(actual)
                .as("cart should hold %d items after adding the bundle", 5)
                .isEqualTo(5);
    }
}
```

```
[withContext] expected:
  [cart should hold 5 items after adding the bundle]
  expected: 5
   but was: 3
```

`.as(...)` is how you attach the business meaning. Write it for any
assertion whose failure would leave a colleague guessing.

## 5. Soft assertions

A hard assertion stops the test at the first failure. When you are verifying
one page against six requirements, you want all six results.

```java
// AssertJ
import org.assertj.core.api.SoftAssertions;

@Test
void secureAreaMeetsAllRequirements() {
    SoftAssertions softly = new SoftAssertions();

    softly.assertThat(page.getHeading()).as("REQ-03 heading").isEqualTo("Secure Area");
    softly.assertThat(page.getFlashMessage()).as("REQ-03 flash").contains("You logged into");
    softly.assertThat(page.isLogoutVisible()).as("REQ-04 logout button").isTrue();
    softly.assertThat(page.getPageTitle()).as("page title").isEqualTo("The Internet");

    softly.assertAll();     // reports EVERY failure, then fails the test
}
```

```
org.assertj.core.error.AssertJMultipleFailuresError:
Multiple Failures (2 failures)
-- failure 1 --
[REQ-03 heading] expected: "Secure Area" but was: "Secure area"
-- failure 2 --
[REQ-04 logout button] expected: true but was: false
```

```java
// TestNG
import org.testng.asserts.SoftAssert;

SoftAssert softAssert = new SoftAssert();
softAssert.assertEquals(page.getHeading(), "Secure Area", "REQ-03 heading");
softAssert.assertTrue(page.isLogoutVisible(), "REQ-04 logout button");
softAssert.assertAll();
```

!!! warning "A `SoftAssert` without `assertAll()` always passes"
    Forget the final call and the test reports green no matter how many
    checks failed. This is the most dangerous defect a test suite can have,
    because it is invisible. Make "every `SoftAssert` gets an `assertAll()`"
    a review checklist item, or use AssertJ's
    `@ExtendWith(SoftAssertionsExtension.class)` with an injected
    `SoftAssertions` parameter, which calls it for you.

Use soft assertions for **independent checks on one state**. Use hard
assertions when a failure invalidates everything after it — there is no point
checking the cart total when the page did not load.

## 6. Custom assertions

When a check appears in six tests, give it a name.

```java
import org.assertj.core.api.AbstractAssert;

public class OrderAssert extends AbstractAssert<OrderAssert, Order> {

    public OrderAssert(Order actual) { super(actual, OrderAssert.class); }

    public static OrderAssert assertThat(Order actual) { return new OrderAssert(actual); }

    public OrderAssert isShippable() {
        isNotNull();
        if (!actual.isPaid() || actual.getItems().isEmpty()) {
            failWithMessage("Expected order <%s> to be shippable but paid=<%s>, items=<%d>",
                    actual.getId(), actual.isPaid(), actual.getItems().size());
        }
        return this;
    }
}

// In a test
OrderAssert.assertThat(order).isShippable();
```

The failure message now states the domain rule and every value that mattered
— which is what a defect report needs anyway.

## 7. Testing traps

!!! warning "Trap 1 — reversed argument order between frameworks"
    JUnit: `assertEquals(expected, actual)`. TestNG:
    `assertEquals(actual, expected)`. Mixed up, every failure message accuses
    the wrong side. AssertJ and Hamcrest have no such ambiguity — one more
    reason to prefer them.

!!! warning "Trap 2 — `assertEquals` on doubles"
    `assertEquals(0.1 + 0.2, 0.3)` fails: binary floating point gives
    `0.30000000000000004`. Always use a delta —
    `assertEquals(0.3, sum, 0.0001)` or `isCloseTo(0.3, within(0.0001))`.

!!! warning "Trap 3 — assertion-free tests"
    A Selenium test that navigates, clicks and ends passes as long as nothing
    throws. It verifies nothing. Grep your suite for test methods with no
    `assert` in them; there are usually more than you expect.

!!! warning "Trap 4 — over-specified assertions"
    `assertEquals("Showing 1-10 of 47 results", banner)` fails every time the
    fixture data changes, for no defect. Assert the part the requirement
    actually constrains: `assertThat(banner).matches("Showing 1-10 of \\d+ results")`.

!!! warning "Trap 5 — `assertTrue(a.equals(b))`"
    Passes and fails identically to `assertEquals(a, b)` but prints no
    values. Never wrap a comparison in `assertTrue`.

## Cheat sheet

| Check | AssertJ |
|---|---|
| Equality | `assertThat(x).isEqualTo(y)` |
| Null | `assertThat(x).isNull()` / `.isNotNull()` |
| Boolean | `assertThat(flag).isTrue()` |
| String contains | `assertThat(s).contains("abc")` |
| Case-insensitive | `assertThat(s).containsIgnoringCase("ABC")` |
| Regex | `assertThat(s).matches("id=\\d+")` |
| Blank | `assertThat(s).isNotBlank()` |
| Size | `assertThat(list).hasSize(3)` |
| Contains, order-free | `assertThat(list).containsExactlyInAnyOrder(a, b)` |
| Field projection | `assertThat(list).extracting(Product::name).contains("Mouse")` |
| Filter then assert | `assertThat(list).filteredOn(Product::inStock).hasSize(2)` |
| Every element | `assertThat(list).allSatisfy(p -> ...)` |
| Numeric tolerance | `assertThat(d).isCloseTo(250.0, within(0.05))` |
| Range | `assertThat(n).isBetween(1, 10)` |
| Map entry | `assertThat(map).containsEntry("k", "v")` |
| Exception | `assertThatThrownBy(...).isInstanceOf(X.class).hasMessageContaining("y")` |
| Add context | `.as("REQ-04 logout button")` |
| Soft (AssertJ) | `SoftAssertions softly = ...; softly.assertAll();` |
| Soft (TestNG) | `SoftAssert sa = new SoftAssert(); sa.assertAll();` |
| Group (JUnit) | `assertAll(() -> ..., () -> ...)` |

## Exercise

1. Add AssertJ to your `pom.xml` and rewrite every `assertTrue` in your suite
   as a typed AssertJ assertion. For each one, note whether the failure
   message improved.
2. Take the four discount boundaries and write them three ways: four separate
   `@Test` methods, one method with `assertAll`, and one with
   `SoftAssertions`. Break the 5000 boundary and compare the three reports.
3. Delete the `softly.assertAll()` from your soft-assertion test, keep the
   deliberate break, and confirm the test passes. Write two sentences on how
   you would stop this reaching your main branch.
4. Write an assertion on a `List<Product>` using `extracting`, `filteredOn`
   and `tuple` that would have caught a bug where out-of-stock items are
   still shown as purchasable.
5. Assert `0.1 + 0.2 == 0.3` with `assertEquals` and watch it fail. Fix it
   two ways (JUnit delta, AssertJ `within`) and note the exact value printed.
6. Write a custom `PageAssert` with `hasFlashMessageContaining(String)` and
   `showsLogoutButton()`, then rewrite one Module 01 test with it.
7. Audit your whole suite: list every test method containing no assertion,
   and either add one or explain in a comment why the test is legitimate
   (a smoke navigation test may be — say so explicitly).
