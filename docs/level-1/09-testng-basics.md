# 09 · TestNG Basics

TestNG ("Next Generation") is the other major Java test framework, and it
dominates UI test automation. Where JUnit was designed for developers writing
unit tests, TestNG was designed with *suites* in mind: grouping, ordering,
dependencies, parallel execution and XML-driven configuration — all things a
regression suite of 800 Selenium tests actually needs.

Most real automation projects use TestNG. This module gets you fluent in it.

## 1. Your first TestNG test

The dependency is already in the `pom.xml` from Module 06. Create
`src/test/java/com/example/tests/CalculatorTestNG.java`:

```java
package com.example.tests;

import com.example.Calculator;
import org.testng.annotations.Test;
import static org.testng.Assert.assertEquals;

public class CalculatorTestNG {

    @Test
    public void addsTwoPositiveNumbers() {
        Calculator calc = new Calculator();
        int result = calc.add(2, 3);
        assertEquals(result, 5, "2 + 3 should equal 5");
    }
}
```

```bash
mvn test
```

```
===============================================
Default Suite
Total tests run: 1, Passes: 1, Failures: 0, Skips: 0
===============================================

[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

!!! warning "TestNG's assertion argument order is REVERSED"
    JUnit: `assertEquals(expected, actual)`
    TestNG: `assertEquals(actual, expected)`

    This trips up everyone who moves between the two. Get it backwards and
    the test still passes or fails correctly — but the failure message reads
    `expected [5] but found [7]` with the values swapped, and you debug the
    wrong side. Fix the order in your muscle memory now.

    Also note: TestNG classes and methods must be `public`; JUnit 5 allows
    package-private.

## 2. The lifecycle annotations

TestNG has five levels of setup/teardown, compared to JUnit's two.

```java
package com.example.tests;

import org.testng.annotations.*;

public class LifecycleTestNG {

    @BeforeSuite
    public void beforeSuite()   { System.out.println("@BeforeSuite  -- once for the whole suite"); }

    @BeforeTest
    public void beforeTest()    { System.out.println(" @BeforeTest   -- once per <test> tag in testng.xml"); }

    @BeforeClass
    public void beforeClass()   { System.out.println("  @BeforeClass  -- once per class"); }

    @BeforeMethod
    public void beforeMethod()  { System.out.println("   @BeforeMethod -- before EVERY test method"); }

    @Test
    public void testOne()       { System.out.println("     >> testOne"); }

    @Test
    public void testTwo()       { System.out.println("     >> testTwo"); }

    @AfterMethod
    public void afterMethod()   { System.out.println("   @AfterMethod  -- after EVERY test method"); }

    @AfterClass
    public void afterClass()    { System.out.println("  @AfterClass"); }

    @AfterTest
    public void afterTest()     { System.out.println(" @AfterTest"); }

    @AfterSuite
    public void afterSuite()    { System.out.println("@AfterSuite"); }
}
```

```
@BeforeSuite  -- once for the whole suite
 @BeforeTest   -- once per <test> tag in testng.xml
  @BeforeClass  -- once per class
   @BeforeMethod -- before EVERY test method
     >> testOne
   @AfterMethod  -- after EVERY test method
   @BeforeMethod -- before EVERY test method
     >> testTwo
   @AfterMethod  -- after EVERY test method
  @AfterClass
 @AfterTest
@AfterSuite
```

| Annotation | Scope | Typical use in a Selenium suite |
|---|---|---|
| `@BeforeSuite` | Once, entire suite | Load config, start a Selenium Grid, set up a report |
| `@BeforeTest` | Once per `<test>` in `testng.xml` | Set the browser for this `<test>` block |
| `@BeforeClass` | Once per class | Launch the WebDriver, log in once |
| `@BeforeGroups` | Before the first method of a group | Group-specific data setup |
| `@BeforeMethod` | Before every test method | Navigate to the start page, clear cookies |
| `@AfterMethod` | After every test method | Screenshot on failure, log out |
| `@AfterClass` | Once per class | `driver.quit()` |
| `@AfterTest` / `@AfterSuite` | Closing scopes | Flush reports, stop the grid |

Note that TestNG lifecycle methods do **not** need to be `static`, unlike
JUnit's `@BeforeAll`/`@AfterAll` — one small quality-of-life advantage.

## 3. `@Test` attributes

TestNG packs a lot of behaviour into attributes on the `@Test` annotation
itself.

```java
package com.example.tests;

import org.testng.annotations.Test;
import org.testng.SkipException;
import static org.testng.Assert.*;

public class TestAttributesDemo {

    @Test(description = "Cart total recalculates when an item is removed",
          priority = 1,
          groups = {"smoke", "cart"})
    public void cartTotalUpdates() {
        assertEquals(3 - 1, 2);
    }

    @Test(priority = 2, groups = {"regression"})
    public void archivedOrdersAreSearchable() {
        assertTrue(true);
    }

    @Test(enabled = false, description = "Blocked by BUG-114")
    public void loginErrorMessage() {
        // Disabled -- appears in the report, not deleted from the codebase
    }

    @Test(expectedExceptions = IllegalArgumentException.class,
          expectedExceptionsMessageRegExp = "Cannot divide by zero")
    public void divideByZeroThrows() {
        new com.example.Calculator().divide(10, 0);
    }

    @Test(timeOut = 3000)   // milliseconds -- fails if it takes longer
    public void searchRespondsWithinThreeSeconds() throws InterruptedException {
        Thread.sleep(100);
        assertTrue(true);
    }

    @Test(invocationCount = 3)
    public void runsThreeTimes() {
        assertTrue(true);
    }

    @Test(dependsOnMethods = {"cartTotalUpdates"})
    public void checkoutRequiresACart() {
        assertTrue(true);
    }

    @Test
    public void skippedWhenPreconditionMissing() {
        boolean paymentSandboxAvailable = false;
        if (!paymentSandboxAvailable) {
            throw new SkipException("Payment sandbox unavailable -- reported as Blocked");
        }
        assertTrue(true);
    }
}
```

```
Total tests run: 8, Passes: 6, Failures: 0, Skips: 2
```

| Attribute | Effect |
|---|---|
| `description` | Human-readable name in reports |
| `priority` | Execution order — **lower runs first**; default is 0 |
| `groups` | Tag for selective execution (smoke/regression, Module 03) |
| `enabled = false` | Skip this test |
| `expectedExceptions` | The test passes only if it throws that exception |
| `timeOut` | Fail if it exceeds n milliseconds |
| `invocationCount` | Run it n times |
| `dependsOnMethods` | Run only after the named method **passes** |
| `alwaysRun = true` | Run even if a dependency failed (used on teardown methods) |

!!! info "`dependsOnMethods` and the Blocked status"
    If `cartTotalUpdates` fails, `checkoutRequiresACart` is reported as
    **Skipped**, not Failed. This is TestNG expressing exactly the
    *Blocked* status from your Module 02 manual test template — a test that
    could not run because its precondition failed. It is one of the clearest
    advantages over JUnit for end-to-end suites.

`SkipException` gives you the same thing programmatically: when the
environment is not ready, report Blocked rather than a misleading Failure.

## 4. `testng.xml` — the suite file

This is TestNG's signature feature. An XML file defines which classes,
methods and groups run, in what order, with what parameters, and with how
much parallelism — all without touching Java code.

Create `testng.xml` in the project root:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">

<suite name="Regression Suite" verbose="1">

    <!-- Suite-wide parameter, readable with @Parameters -->
    <parameter name="baseUrl" value="https://the-internet.herokuapp.com"/>

    <test name="Smoke Tests">
        <parameter name="browser" value="chrome"/>
        <groups>
            <run>
                <include name="smoke"/>
                <exclude name="wip"/>
            </run>
        </groups>
        <classes>
            <class name="com.example.tests.TestAttributesDemo"/>
        </classes>
    </test>

    <test name="Login Tests">
        <parameter name="browser" value="chrome"/>
        <classes>
            <class name="com.example.tests.LoginTestNG">
                <methods>
                    <include name="loginWithValidCredentials"/>
                    <include name="loginWithInvalidPassword"/>
                    <exclude name="loginWithLockedAccount"/>
                </methods>
            </class>
        </classes>
    </test>

</suite>
```

Wire it into Maven so `mvn test` uses it:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <suiteXmlFiles>
            <suiteXmlFile>testng.xml</suiteXmlFile>
        </suiteXmlFiles>
    </configuration>
</plugin>
```

```bash
mvn clean test
```

```
===============================================
Regression Suite
Total tests run: 5, Passes: 4, Failures: 0, Skips: 1
===============================================
```

The power here is that a release manager can change *what runs* by editing
XML — no recompilation, no Java knowledge. Nightly full regression and
per-commit smoke can be two XML files pointing at the same test code.

## 5. Parameters from XML

```java
package com.example.tests;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.firefox.FirefoxDriver;
import org.openqa.selenium.firefox.FirefoxOptions;
import org.testng.annotations.*;
import static org.testng.Assert.assertTrue;

public class ParameterizedByXmlTest {

    private WebDriver driver;
    private String baseUrl;

    @Parameters({"browser", "baseUrl"})
    @BeforeClass
    public void setUp(String browser, String baseUrl) {
        this.baseUrl = baseUrl;
        System.out.println("Launching " + browser + " against " + baseUrl);

        if ("firefox".equalsIgnoreCase(browser)) {
            FirefoxOptions o = new FirefoxOptions();
            o.addArguments("-headless");
            driver = new FirefoxDriver(o);
        } else {
            ChromeOptions o = new ChromeOptions();
            o.addArguments("--headless=new", "--window-size=1920,1080");
            driver = new ChromeDriver(o);
        }
    }

    @Test(groups = "smoke")
    public void homePageLoads() {
        driver.get(baseUrl);
        assertTrue(driver.getTitle().length() > 0, "Page should have a title");
    }

    @AfterClass(alwaysRun = true)
    public void tearDown() {
        if (driver != null) driver.quit();
    }
}
```

```
Launching chrome against https://the-internet.herokuapp.com
Total tests run: 1, Passes: 1, Failures: 0, Skips: 0
```

Now the same test class runs against Chrome or Firefox purely by editing XML
— the foundation of the cross-browser testing you will build out in Level 2,
Module 09.

## 6. TestNG assertions

```java
import org.testng.Assert;
import org.testng.asserts.SoftAssert;

// Hard assertions -- stop the test on first failure
Assert.assertEquals(actual, expected);
Assert.assertEquals(actual, expected, "message");
Assert.assertNotEquals(actual, expected);
Assert.assertTrue(condition, "message");
Assert.assertFalse(condition, "message");
Assert.assertNull(object);
Assert.assertNotNull(object);
Assert.assertSame(actual, expected);
Assert.fail("Reached an unreachable branch");
```

### Soft assertions

TestNG's answer to JUnit's `assertAll` — collect every failure, then report
them all at the end.

```java
package com.example.tests;

import org.testng.annotations.Test;
import org.testng.asserts.SoftAssert;

public class SoftAssertDemo {

    @Test
    public void validateDashboardWidgets() {
        SoftAssert softly = new SoftAssert();

        String welcome = "Welcome, QA User";
        int cartCount  = 0;
        boolean logoutVisible = true;

        softly.assertTrue(welcome.startsWith("Welcome"),  "Welcome banner text");
        softly.assertEquals(cartCount, 3, "Cart badge count");
        softly.assertTrue(logoutVisible,  "Logout link visibility");

        softly.assertAll();   // MUST be called, or failures are silently swallowed
    }
}
```

```
java.lang.AssertionError: The following asserts failed:
	Cart badge count expected [3] but found [0]
```

!!! warning "Forgetting `assertAll()` makes soft assertions useless"
    Without the final `softly.assertAll()`, every soft assertion failure is
    discarded and the test reports as passed. This is a genuinely dangerous
    silent failure — a suite full of green ticks verifying nothing. Make
    `assertAll()` the last line of every method that uses `SoftAssert`, and
    create a new `SoftAssert` instance per test method (never share one).

Use **hard** assertions for preconditions (if login failed, there is no point
continuing) and **soft** assertions for validating many independent things on
one page.

## 7. Data providers

TestNG's equivalent of `@ParameterizedTest` — and considerably more powerful,
because the provider is ordinary Java code that can read a CSV, an Excel file
or a database.

```java
package com.example.tests;

import com.example.Calculator;
import org.testng.annotations.DataProvider;
import org.testng.annotations.Test;
import static org.testng.Assert.assertEquals;

public class DataProviderDemo {

    @DataProvider(name = "discountBoundaries")
    public Object[][] discountBoundaries() {
        return new Object[][] {
                { 999.99,    0 },
                { 1000.00,   5 },
                { 1000.01,   5 },
                { 4999.99,   5 },
                { 5000.00,  10 },
                { 9999.99,  10 },
                { 10000.00, 15 },
                { 0.00,      0 }
        };
    }

    @Test(dataProvider = "discountBoundaries")
    public void discountTierIsCorrect(double amount, int expected) {
        assertEquals(new Calculator().discountFor(amount), expected,
                "Discount for order amount " + amount);
    }
}
```

```
Total tests run: 8, Passes: 8, Failures: 0, Skips: 0
```

Eight boundary value tests (Module 05) from one method. Level 2, Module 03
extends this to CSV and Excel sources.

## 8. Parallel execution

One line of XML turns a 40-minute suite into a 10-minute one:

```xml
<suite name="Regression Suite" parallel="methods" thread-count="4">
```

| `parallel` value | Runs in parallel |
|---|---|
| `methods` | Individual `@Test` methods |
| `classes` | Test classes |
| `tests` | `<test>` blocks in the XML |
| `instances` | Instances of the same class |

!!! warning "Parallel execution requires thread-safe tests"
    If your `WebDriver` is a plain instance field shared across threads, four
    tests will fight over one browser and fail chaotically. Parallel Selenium
    needs a `ThreadLocal<WebDriver>` — covered properly in Level 2, Module 03
    and Level 3, Module 04. Do not switch on parallelism until your tests are
    genuinely independent (Module 02, rule 5).

## 9. TestNG vs JUnit 5

| Feature | JUnit 5 | TestNG |
|---|---|---|
| Assertion order | `assertEquals(expected, actual)` | `assertEquals(actual, expected)` |
| Lifecycle levels | 2 (`All`, `Each`) | 5 (`Suite`, `Test`, `Class`, `Groups`, `Method`) |
| Static setup required | Yes for `@BeforeAll` | No |
| Grouping | `@Tag` | `groups` attribute — richer include/exclude |
| Suite configuration | Code / build config | `testng.xml` — no recompilation needed |
| Data-driven | `@ParameterizedTest` + sources | `@DataProvider` — arbitrary Java |
| Test dependencies | Not supported | `dependsOnMethods` / `dependsOnGroups` |
| Parallel execution | Experimental config | First-class, one XML attribute |
| Soft assertions | `assertAll` | `SoftAssert` class |
| Built-in retry of failures | No | `IRetryAnalyzer` + `testng-failed.xml` |
| Typical use | Unit tests, developer-written | UI/E2E suites, QA-written |

**Which should you learn?** Both, which is why this course covers both. In
practice: JUnit 5 for unit and API tests inside a developer codebase, TestNG
for Selenium regression suites. If a job ad says "Selenium + Java", it almost
certainly means TestNG.

## 10. Re-running only the failures

After a run, TestNG writes `test-output/testng-failed.xml` containing exactly
the tests that failed (plus their dependencies). Run it directly:

```bash
mvn test -DsuiteXmlFile=test-output/testng-failed.xml
```

On a 600-test overnight suite with 4 failures, this re-verifies the fixes in
seconds instead of re-running everything.

## Cheat sheet

| Item | TestNG |
|---|---|
| Test method | `@Test public void name()` |
| Assertion | `Assert.assertEquals(actual, expected, "msg")` |
| Setup / teardown | `@BeforeMethod` / `@AfterMethod` (plus Class, Test, Suite) |
| Skip a test | `@Test(enabled = false)` or `throw new SkipException(...)` |
| Expected exception | `@Test(expectedExceptions = X.class)` |
| Order | `@Test(priority = 1)` — lower first |
| Grouping | `@Test(groups = {"smoke"})` |
| Dependency | `@Test(dependsOnMethods = "login")` |
| Timeout | `@Test(timeOut = 3000)` |
| Data-driven | `@DataProvider` + `@Test(dataProvider = "name")` |
| XML parameter | `@Parameters({"browser"})` |
| Soft assertions | `new SoftAssert()` … `softly.assertAll()` |
| Suite file | `testng.xml` |
| Parallel | `<suite parallel="methods" thread-count="4">` |
| Re-run failures | `test-output/testng-failed.xml` |

## Exercise

1. Convert your `LoginSeleniumTest` from Module 08 to TestNG. Use
   `@BeforeClass` to launch the driver once, `@BeforeMethod` to navigate to
   `/login` and clear cookies, and `@AfterClass(alwaysRun = true)` to quit.
   Watch the assertion argument order.
2. Tag the valid-credentials test `groups = {"smoke"}` and the three negative
   tests `groups = {"regression"}`.
3. Write a `testng.xml` with **two** `<test>` blocks: "Smoke" (includes only
   the `smoke` group) and "Full Regression" (includes both groups). Pass
   `browser` and `baseUrl` as parameters and read them with `@Parameters`.
   Wire it into Surefire and confirm `mvn clean test` runs your suite.
4. Add a `@DataProvider` named `loginCredentials` returning four rows —
   username, password, expected message fragment — covering valid, wrong
   password, invalid username and empty fields. Rewrite the four login tests
   as **one** data-driven test method.
5. Use `SoftAssert` in the successful-login test to check the URL, the flash
   message and the Logout link visibility in a single method. Then comment
   out `assertAll()`, break one of the checks, and confirm the test still
   reports as passed — so you never forget that trap.
6. Make the invalid-password test `dependsOnMethods` the valid-login test.
   Deliberately break the valid-login test and confirm the dependent test is
   reported as **Skipped**, not Failed. Explain in one sentence how that maps
   to the *Blocked* status from Module 02.
