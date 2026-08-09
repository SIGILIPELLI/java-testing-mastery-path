# 03 · TestNG Advanced

TestNG earns its place in automation frameworks — over JUnit — for three
things: data providers that turn one method into fifty test cases, XML-driven
suite configuration you can change without recompiling, and listeners that
hook the whole run for screenshots and reporting. This module covers all
three, plus the parallel execution that makes a 40-minute suite finish in 8.

## 1. Data providers

A `@DataProvider` supplies rows; TestNG runs the method once per row and
reports each row as a separate test result.

```java
package com.example.tests;

import org.testng.annotations.*;

import static org.testng.Assert.*;

public class LoginDataDrivenTest {

    @DataProvider(name = "invalidCredentials")
    public Object[][] invalidCredentials() {
        return new Object[][] {
                { "wronguser", "SuperSecretPassword!", "Your username is invalid!" },
                { "tomsmith",  "wrongpassword",        "Your password is invalid!" },
                { "",          "SuperSecretPassword!", "Your username is invalid!" },
                { "tomsmith",  "",                     "Your password is invalid!" },
                { "TOMSMITH",  "SuperSecretPassword!", "Your username is invalid!" },
        };
    }

    @Test(dataProvider = "invalidCredentials")
    public void rejectsInvalidCredentials(String user, String pass, String expected) {
        String actual = new FakeAuth().login(user, pass);

        assertTrue(actual.contains(expected),
                String.format("user='%s' pass='%s' -> expected '%s' but got '%s'",
                        user, pass, expected, actual));
    }

    /** Stand-in for the real page object, so this example runs without a browser. */
    static class FakeAuth {
        String login(String user, String pass) {
            if (!"tomsmith".equals(user)) return "Your username is invalid!";
            if (!"SuperSecretPassword!".equals(pass)) return "Your password is invalid!";
            return "You logged into a secure area!";
        }
    }
}
```

```
===============================================
Default Suite
Total tests run: 5, Passes: 5, Failures: 0, Skips: 0
===============================================
```

Five test cases, one method. The last row — `TOMSMITH` in capitals — is the
case-sensitivity check from Level 1's equivalence partitioning, and adding it
cost one line.

### Data providers in a separate class

```java
public class TestData {

    @DataProvider(name = "checkoutTotals")
    public static Object[][] checkoutTotals() {
        return new Object[][] {
                { 999.0,   0 },     // just below the first tier
                { 1000.0,  5 },     // boundary
                { 5000.0,  10 },    // boundary
                { 10000.0, 15 },    // boundary
        };
    }
}

@Test(dataProvider = "checkoutTotals", dataProviderClass = TestData.class)
public void appliesCorrectDiscount(double amount, int expectedPercent) {
    assertEquals(new Calculator().discountFor(amount), expectedPercent);
}
```

`dataProviderClass` is what lets one data set feed many test classes.

### `ITestContext` and `Method` injection

```java
@DataProvider(name = "browsers")
public Object[][] browsers(ITestContext context) {
    String only = context.getCurrentXmlTest().getParameter("browser");
    return only == null
            ? new Object[][] { {"chrome"}, {"firefox"} }
            : new Object[][] { {only} };
}
```

TestNG injects `ITestContext` or `java.lang.reflect.Method` into a provider if
you declare the parameter — that is how a provider can vary by suite file or
by the test method it is feeding.

## 2. testng.xml — configuration without recompiling

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">
<suite name="Regression Suite" parallel="classes" thread-count="4" verbose="1">

    <parameter name="baseUrl" value="https://the-internet.herokuapp.com"/>
    <parameter name="browser" value="chrome"/>

    <listeners>
        <listener class-name="com.example.listeners.ScreenshotListener"/>
    </listeners>

    <test name="Smoke">
        <groups>
            <run>
                <include name="smoke"/>
                <exclude name="wip"/>
            </run>
        </groups>
        <classes>
            <class name="com.example.tests.LoginDataDrivenTest"/>
        </classes>
    </test>

    <test name="Full Regression">
        <packages>
            <package name="com.example.tests.*"/>
        </packages>
    </test>
</suite>
```

Run it with `mvn test -DsuiteXmlFile=testng.xml`, or wire it into Surefire
(Module 06). The value of XML config is that a release manager can flip
`thread-count` or swap `browser` without touching Java.

Reading parameters:

```java
@Parameters({"baseUrl", "browser"})
@BeforeClass
public void setUp(String baseUrl, @Optional("chrome") String browser) {
    this.baseUrl = baseUrl;
    this.driver  = DriverFactory.create(browser);
}
```

`@Optional` supplies a default so the class still runs when launched from the
IDE without an XML file.

## 3. Groups, dependencies and priority

```java
@Test(groups = {"smoke", "auth"})
public void loginPageLoads() { }

@Test(groups = "regression", dependsOnMethods = "loginPageLoads")
public void userCanLogIn() { }

@Test(groups = "regression", dependsOnGroups = "auth", alwaysRun = false)
public void userCanUpdateProfile() { }

@Test(priority = 1)      // lower numbers run first; default is 0
public void runsSecond() { }

@Test(enabled = false, description = "Blocked by BUG-142")
public void knownBroken() { }

@Test(invocationCount = 5, successPercentage = 80)
public void flakyEndpointToleratesOneFailure() { }

@Test(timeOut = 5000)    // milliseconds -- fails if it takes longer
public void mustRespondQuickly() { }
```

When a `dependsOnMethods` prerequisite fails, dependent tests are reported
**Skipped**, not Failed — a genuinely useful distinction in a report, because
skipped means "not evaluated", not "broken".

!!! warning "Dependencies are not a substitute for independent tests"
    A chain of eight `dependsOnMethods` tests is a single test with eight
    checkpoints: it cannot run in parallel, cannot be run individually, and
    one early failure blinds you to the other seven. Use dependencies for
    genuine prerequisites (log in before profile tests), not to sequence
    steps that should have been one test.

## 4. Parallel execution

| `parallel` value | Unit of parallelism |
|---|---|
| `methods` | Every `@Test` method, across all classes |
| `classes` | Each class on one thread; methods within it sequential |
| `tests` | Each `<test>` block |
| `instances` | Each instance of a factory-created class |

The one thing that must change when you go parallel is the driver field.

```java
package com.example.driver;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;

public final class DriverFactory {

    private static final ThreadLocal<WebDriver> DRIVER = new ThreadLocal<>();

    public static WebDriver get() {
        return DRIVER.get();
    }

    public static void start() {
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--headless=new", "--window-size=1920,1080");
        DRIVER.set(new ChromeDriver(options));
    }

    public static void stop() {
        WebDriver driver = DRIVER.get();
        if (driver != null) {
            driver.quit();
            DRIVER.remove();      // MUST remove, or the thread pool leaks drivers
        }
    }
}
```

```java
public class ParallelBaseTest {

    @BeforeMethod(alwaysRun = true)
    public void startBrowser() { DriverFactory.start(); }

    @AfterMethod(alwaysRun = true)
    public void stopBrowser()  { DriverFactory.stop(); }
}
```

!!! warning "`ThreadLocal.remove()` is not optional"
    TestNG reuses threads from a pool. Without `remove()`, the next test on
    that thread inherits a quit driver and fails with
    `NoSuchSessionException` — and because the failure lands on a random
    later test, you will spend a day blaming the wrong code.

## 5. Listeners

`ITestListener` gives you a callback on every result — the standard place for
failure screenshots.

```java
package com.example.listeners;

import com.example.driver.DriverFactory;
import org.openqa.selenium.*;
import org.testng.ITestListener;
import org.testng.ITestResult;

import java.nio.file.*;

public class ScreenshotListener implements ITestListener {

    @Override
    public void onTestFailure(ITestResult result) {
        WebDriver driver = DriverFactory.get();
        if (!(driver instanceof TakesScreenshot shooter)) return;

        try {
            byte[] png = shooter.getScreenshotAs(OutputType.BYTES);
            Path dest = Path.of("target", "screenshots", result.getName() + ".png");
            Files.createDirectories(dest.getParent());
            Files.write(dest, png);
            System.out.println("Screenshot: " + dest.toAbsolutePath());
        } catch (Exception e) {
            System.out.println("Screenshot failed: " + e.getMessage());
        }
    }

    @Override
    public void onTestSkipped(ITestResult result) {
        System.out.println("SKIPPED " + result.getName()
                + " -- cause: " + result.getThrowable());
    }
}
```

Register it in `testng.xml` (section 2) or with
`@Listeners(ScreenshotListener.class)` on the class.

| Interface | Hooks |
|---|---|
| `ITestListener` | Start/success/failure/skip of each test |
| `ISuiteListener` | Suite start and finish |
| `IRetryAnalyzer` | Decide whether to re-run a failed test |
| `IAnnotationTransformer` | Rewrite `@Test` attributes at runtime |
| `IInvokedMethodListener` | Before/after every method including configuration |

### Retry analyzer

```java
public class RetryOnce implements IRetryAnalyzer {
    private int attempts = 0;

    @Override
    public boolean retry(ITestResult result) {
        return attempts++ < 1;      // one extra attempt
    }
}

@Test(retryAnalyzer = RetryOnce.class)
public void occasionallyFlaky() { }
```

!!! warning "Retries hide bugs"
    A retried test that passes on attempt two is reported green, and the
    intermittent race condition your customer will hit stays in production.
    If you must retry, log every retry and treat a rising retry count as a
    defect in its own right. Fixing the wait is almost always the right
    answer instead.

## 6. Testing traps

!!! warning "Trap 1 — instance fields shared across parallel methods"
    With `parallel="methods"`, one instance is shared. A
    `private WebDriver driver` field is then written by several threads.
    `ThreadLocal` or nothing.

!!! warning "Trap 2 — `assertEquals` argument order is reversed"
    TestNG is `assertEquals(actual, expected)`; JUnit is
    `assertEquals(expected, actual)`. Mixing them produces failure messages
    that accuse the wrong value. Never import both in one class.

!!! warning "Trap 3 — `@BeforeClass` with `parallel="methods"`"
    It runs once for the class, but the methods sharing it run on different
    threads. Anything it creates is shared state. Use `@BeforeMethod` for
    per-test resources like drivers.

!!! warning "Trap 4 — forgetting `alwaysRun = true` on teardown"
    If `@BeforeMethod` fails, `@AfterMethod` is skipped by default and the
    browser is never closed. Every configuration method that releases a
    resource needs `alwaysRun = true`.

## Cheat sheet

| Task | Code |
|---|---|
| Data provider | `@DataProvider(name="x")` + `@Test(dataProvider="x")` |
| External provider | `@Test(dataProvider="x", dataProviderClass=TestData.class)` |
| Parallel data rows | `@DataProvider(parallel = true)` |
| Group a test | `@Test(groups = {"smoke"})` |
| Prerequisite | `@Test(dependsOnMethods = "login")` |
| Disable | `@Test(enabled = false)` |
| Time limit | `@Test(timeOut = 5000)` |
| Repeat | `@Test(invocationCount = 5)` |
| Expected exception | `@Test(expectedExceptions = IllegalArgumentException.class)` |
| Suite parallelism | `<suite parallel="methods" thread-count="4">` |
| XML parameter | `<parameter name="browser" value="chrome"/>` + `@Parameters` |
| Register listener | `@Listeners(X.class)` or `<listeners>` in XML |
| Thread-safe driver | `ThreadLocal<WebDriver>` + `remove()` in teardown |
| Run a suite file | `mvn test -DsuiteXmlFile=testng.xml` |

## Exercise

1. Convert your Level 1 login tests to a single `@DataProvider`-driven method
   with at least six rows, including both boundary cases and the
   case-sensitivity row. Confirm the report shows six results, not one.
2. Move the provider into a separate `TestData` class and feed it to two
   different test classes via `dataProviderClass`.
3. Write a `testng.xml` with two `<test>` blocks — Smoke (group `smoke`
   only) and Full Regression (whole package) — and a `baseUrl` parameter read
   through `@Parameters`. Run each block on its own.
4. Implement `DriverFactory` with `ThreadLocal` exactly as in section 4, set
   `parallel="methods" thread-count="3"`, and confirm the suite time drops.
   Then delete the `DRIVER.remove()` call, re-run, and record the exception.
5. Add `ScreenshotListener`, break one test deliberately, and confirm the PNG
   lands in `target/screenshots/` with the test method's name.
6. Add a `dependsOnMethods` chain of three tests, fail the first, and note
   how the other two are reported. Then write two sentences on why "Skipped"
   is more honest than "Failed" here.
7. Add `RetryOnce` to one test and write three sentences on when your team
   should allow retries and when it must not.
