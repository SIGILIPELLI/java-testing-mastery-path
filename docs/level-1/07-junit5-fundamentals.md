# 07 · JUnit 5 Fundamentals

JUnit 5 is the standard unit-testing framework for Java. It gives you three
things: a way to mark a method as a test (`@Test`), a way to check a result
(assertions), and a lifecycle for setup and teardown. Everything you learned
about equivalence partitioning and boundary values in Module 05 now becomes
executable.

## 1. Your first test

Create `src/test/java/com/example/tests/CalculatorTest.java`, testing the
`Calculator` from Module 06:

```java
package com.example.tests;

import com.example.Calculator;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertEquals;

class CalculatorTest {

    @Test
    void addsTwoPositiveNumbers() {
        Calculator calc = new Calculator();      // Arrange
        int result = calc.add(2, 3);             // Act
        assertEquals(5, result);                 // Assert
    }
}
```

```bash
mvn test
```

```
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.example.tests.CalculatorTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.041 s
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] BUILD SUCCESS
```

### The AAA pattern

Every test, in every framework, in every language, has the same three parts:

| Phase | Purpose |
|---|---|
| **Arrange** | Set up the object and data under test |
| **Act** | Perform the single action being tested |
| **Assert** | Verify the outcome matches the expectation |

Keep them visually separate. A test with assertions scattered between actions
is hard to read and usually testing more than one thing.

### What a failure looks like

Change the assertion to `assertEquals(6, result)`:

```
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0
[ERROR] CalculatorTest.addsTwoPositiveNumbers:14
       expected: <6> but was: <5>
[ERROR] BUILD FAILURE
```

`expected: <6> but was: <5>` — this is the machine equivalent of the
"Expected Result / Actual Result" fields from your bug report template in
Module 04. The parallel is exact.

## 2. Assertions

An assertion is the pass/fail decision. If it holds, execution continues; if
it fails, it throws and the test is marked failed immediately.

```java
package com.example.tests;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class AssertionDemoTest {

    @Test
    void coreAssertions() {
        assertEquals(5, 2 + 3);                        // exact equality
        assertNotEquals(4, 2 + 3);                     // inequality
        assertTrue("qa.user@example.com".contains("@"));
        assertFalse("".length() > 0);
        assertNull(System.getProperty("no.such.property"));
        assertNotNull("hello");
        assertSame("abc", "abc");                      // same object reference
        assertArrayEquals(new int[]{1, 2, 3}, new int[]{1, 2, 3});
        assertIterableEquals(List.of("a", "b"), List.of("a", "b"));
    }

    @Test
    void floatingPointNeedsATolerance() {
        double total = 0.1 + 0.2;             // = 0.30000000000000004
        // assertEquals(0.3, total);          // FAILS -- binary floating point
        assertEquals(0.3, total, 0.0001);     // passes: within delta
    }

    @Test
    void failureMessagesShouldExplainThemselves() {
        int cartItems = 3;
        assertEquals(3, cartItems,
                "Cart should contain 3 items after adding 3 products");
    }
}
```

!!! warning "Always add a message to a non-obvious assertion"
    `expected: <true> but was: <false>` tells you nothing at 2 a.m. when a CI
    run fails. `"Logout link should be visible after successful login ==>
    expected: <true> but was: <false>"` tells you exactly what broke. The
    third argument to any assertion is a message — use it.

### Testing that something throws

Negative test cases (Module 02) become `assertThrows`:

```java
@Test
void divideByZeroThrowsIllegalArgument() {
    Calculator calc = new Calculator();

    IllegalArgumentException ex = assertThrows(
            IllegalArgumentException.class,
            () -> calc.divide(10, 0));

    assertEquals("Cannot divide by zero", ex.getMessage());
}
```

Capturing the exception and asserting on its message matters — otherwise the
test passes if *any* `IllegalArgumentException` is thrown, including one from
an unrelated bug.

### Grouped assertions

`assertAll` runs every assertion even if earlier ones fail, then reports all
failures together — the equivalent of the multi-part expected result from
your Module 02 test case template.

```java
@Test
void discountTiersAreCorrect() {
    Calculator calc = new Calculator();

    assertAll("discount tiers",
            () -> assertEquals(0,  calc.discountFor(500),   "under 1000 -> 0%"),
            () -> assertEquals(5,  calc.discountFor(2500),  "1000-4999 -> 5%"),
            () -> assertEquals(10, calc.discountFor(7500),  "5000-9999 -> 10%"),
            () -> assertEquals(15, calc.discountFor(15000), "10000+ -> 15%"));
}
```

If two tiers are wrong, you see both failures in one run instead of fixing,
re-running, and discovering the next one.

## 3. The test lifecycle

```java
package com.example.tests;

import com.example.Calculator;
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.assertEquals;

class LifecycleTest {

    private Calculator calc;

    @BeforeAll
    static void setUpSuite() {
        System.out.println("@BeforeAll  -- once, before any test in this class");
    }

    @BeforeEach
    void setUp() {
        System.out.println("  @BeforeEach -- before EVERY test");
        calc = new Calculator();          // fresh object per test
    }

    @Test
    void firstTest() {
        System.out.println("    running firstTest");
        assertEquals(4, calc.add(2, 2));
    }

    @Test
    void secondTest() {
        System.out.println("    running secondTest");
        assertEquals(1, calc.subtract(3, 2));
    }

    @AfterEach
    void tearDown() {
        System.out.println("  @AfterEach  -- after EVERY test");
        calc = null;
    }

    @AfterAll
    static void tearDownSuite() {
        System.out.println("@AfterAll   -- once, after all tests in this class");
    }
}
```

```
@BeforeAll  -- once, before any test in this class
  @BeforeEach -- before EVERY test
    running firstTest
  @AfterEach  -- after EVERY test
  @BeforeEach -- before EVERY test
    running secondTest
  @AfterEach  -- after EVERY test
@AfterAll   -- once, after all tests in this class
```

| Annotation | Runs | Must be `static`? | Typical use in test automation |
|---|---|---|---|
| `@BeforeAll` | Once per class, first | Yes (by default) | Start a WebDriver, open a DB connection, load config |
| `@BeforeEach` | Before every test | No | Fresh object, navigate to the start page, reset test data |
| `@Test` | The test itself | No | One check |
| `@AfterEach` | After every test | No | Take a screenshot on failure, clear cookies, delete created records |
| `@AfterAll` | Once per class, last | Yes | Quit the driver, close the connection |

!!! info "Why `@BeforeEach` creates a new object every time"
    **Test independence** (Module 02, rule 5) is enforced here. Each test
    gets a clean `Calculator`, so no test can leave state that changes
    another test's result. This is the number-one cause of "passes alone,
    fails in the suite" — and of the flaky tests you will diagnose in Level 3.

## 4. Other annotations you will use

```java
package com.example.tests;

import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.assertEquals;

@DisplayName("Shopping cart behaviour")
class AnnotationDemoTest {

    @Test
    @DisplayName("Cart total updates when an item is removed")
    void cartTotalUpdatesOnRemoval() {
        assertEquals(2, 3 - 1);
    }

    @Test
    @Disabled("Blocked by BUG-114 -- re-enable when the fix ships")
    void loginShowsSpecificErrorMessage() {
        // Skipped, but visible in the report as skipped rather than deleted
    }

    @Test
    @Tag("smoke")
    void applicationHomePageLoads() {
        assertEquals(200, 200);
    }

    @Test
    @Tag("regression")
    void archivedOrdersAreSearchable() {
        assertEquals(1, 1);
    }

    @RepeatedTest(3)
    @DisplayName("Session token generation is stable across repeats")
    void repeatsThreeTimes(RepetitionInfo info) {
        System.out.println("Run " + info.getCurrentRepetition()
                + " of " + info.getTotalRepetitions());
        assertEquals(1, 1);
    }
}
```

```
Run 1 of 3
Run 2 of 3
Run 3 of 3
Tests run: 6, Failures: 0, Errors: 0, Skipped: 1
```

`@Tag` lets you run subsets — exactly the smoke/regression split from Module
03:

```bash
mvn test -Dgroups=smoke
```

`@Disabled` with a reason is far better than commenting a test out: the
report shows it as skipped, so nobody forgets it exists.

## 5. Parameterized tests — automating EP and BVA

This is where Module 05 pays off. Instead of one test method per boundary
value, you feed the values in.

Add to `pom.xml`:

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-params</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
```

```java
package com.example.tests;

import com.example.Calculator;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import org.junit.jupiter.params.provider.ValueSource;

import static org.junit.jupiter.api.Assertions.*;

class ParameterizedDemoTest {

    private final Calculator calc = new Calculator();

    // Boundary value analysis for the 1000/5000/10000 discount tiers
    @ParameterizedTest(name = "order {0} -> {1}% discount")
    @CsvSource({
            "999.99,   0",     // just below the first boundary
            "1000.00,  5",     // exactly at the boundary
            "1000.01,  5",     // just above
            "4999.99,  5",
            "5000.00, 10",
            "5000.01, 10",
            "9999.99, 10",
            "10000.00, 15",
            "10000.01, 15",
            "0.00,     0"      // zero -- the edge from Module 05
    })
    void discountBoundaries(double amount, int expectedDiscount) {
        assertEquals(expectedDiscount, calc.discountFor(amount));
    }

    // Equivalence partition: every negative value is rejected identically
    @ParameterizedTest(name = "amount {0} is rejected")
    @ValueSource(doubles = {-0.01, -1, -100, -99999})
    void negativeAmountsAreRejected(double amount) {
        assertThrows(IllegalArgumentException.class,
                () -> calc.discountFor(amount));
    }
}
```

```
[INFO] Running com.example.tests.ParameterizedDemoTest
order 999.99 -> 0% discount           PASSED
order 1000.0 -> 5% discount           PASSED
order 1000.01 -> 5% discount          PASSED
order 4999.99 -> 5% discount          PASSED
order 5000.0 -> 10% discount          PASSED
order 5000.01 -> 10% discount         PASSED
order 9999.99 -> 10% discount         PASSED
order 10000.0 -> 15% discount         PASSED
order 10000.01 -> 15% discount        PASSED
order 0.0 -> 0% discount              PASSED
amount -0.01 is rejected              PASSED
amount -1.0 is rejected               PASSED
amount -100.0 is rejected             PASSED
amount -99999.0 is rejected           PASSED
[INFO] Tests run: 14, Failures: 0, Errors: 0, Skipped: 0
```

**Fourteen boundary and partition checks from two methods.** That is the
whole point: your manual test design technique becomes a data table, and the
framework does the repetition.

Other data sources:

| Annotation | Source |
|---|---|
| `@ValueSource` | A simple array of one-argument values |
| `@CsvSource` | Inline rows of comma-separated arguments |
| `@CsvFileSource(resources = "/testdata.csv")` | A CSV in `src/test/resources` |
| `@MethodSource("methodName")` | A static method returning a `Stream` of arguments |
| `@EnumSource(Status.class)` | Every value of an enum |
| `@NullAndEmptySource` | `null` and `""` — the two inputs everyone forgets |

## 6. Naming tests

The test name is documentation. It appears in the report when it fails, so it
should state the expectation.

| ❌ Weak | ✅ Strong |
|---|---|
| `test1()` | `addsTwoPositiveNumbers()` |
| `testLogin()` | `loginWithInvalidPasswordShowsGenericError()` |
| `testDiscount()` | `orderOfExactly5000GetsTenPercentDiscount()` |
| `checkCart()` | `removingLastItemSetsCartTotalToZero()` |

A widely used convention is
`methodUnderTest_condition_expectedOutcome`, e.g.
`discountFor_amountBelow1000_returnsZero()`. Pick one style and be consistent.

## 7. Reading the report

After `mvn test`, look in `target/surefire-reports/`:

```
target/surefire-reports/
├── com.example.tests.CalculatorTest.txt      ← human-readable summary
└── TEST-com.example.tests.CalculatorTest.xml ← machine-readable (JUnit XML)
```

The XML format is the universal interchange for test results — Jenkins,
GitHub Actions, Allure and every other tool reads it. That is how your local
run becomes a CI dashboard in Level 3.

## Cheat sheet

| Annotation / method | Purpose |
|---|---|
| `@Test` | Marks a test method |
| `@BeforeAll` / `@AfterAll` | Once per class (static) |
| `@BeforeEach` / `@AfterEach` | Around every test |
| `@DisplayName("…")` | Readable name in reports |
| `@Disabled("reason")` | Skip with a documented reason |
| `@Tag("smoke")` | Group tests for selective runs |
| `@RepeatedTest(n)` | Run the same test n times |
| `@ParameterizedTest` + `@CsvSource` | Data-driven tests — automate EP/BVA |
| `assertEquals(exp, act, msg)` | Value equality |
| `assertTrue` / `assertFalse` | Boolean conditions |
| `assertNull` / `assertNotNull` | Null checks |
| `assertThrows(Type.class, () -> …)` | Negative tests |
| `assertAll(…)` | Report every failure in one run |
| `mvn test` | Run the suite |

## Exercise

Working in the `java-testing-practice` project from Module 06:

1. Create `LoginValidator` in `src/main/java/com/example/` with a method
   `boolean isValidPassword(String password)` implementing the rule from
   Module 05: 8–20 characters, at least one uppercase letter, one digit and
   one special character. Throw `IllegalArgumentException` for `null`.
2. Write `LoginValidatorTest` covering it. Include:
   - A `@ParameterizedTest` with `@CsvSource` covering the **length
     boundaries** — 7, 8, 9, 19, 20 and 21 characters — with the expected
     boolean for each.
   - Separate tests for each missing character class (no uppercase, no digit,
     no special).
   - An `assertThrows` test for `null`, asserting the exception message.
   - `@NullAndEmptySource` on a parameterized test.
3. Use `@DisplayName` on the class and on at least three methods so the
   report reads like specification sentences.
4. Add `@BeforeEach` that prints the test's `TestInfo` display name, and
   confirm from the output that it runs before each test.
5. Deliberately introduce an off-by-one bug in `isValidPassword` (use `> 8`
   instead of `>= 8`). Run `mvn test`, and confirm that exactly the boundary
   test for 8 characters fails — and that nothing else does. This
   demonstrates why BVA has the highest yield per test case.
