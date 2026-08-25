# 08 · Framework Design Patterns

Levels 1–2 built individual pieces — Page Objects, RestAssured specs, test
data builders. This module names the patterns that turn a pile of test
classes into a *framework*: something a new team member can extend without
reading every existing test first.

## 1. Factory pattern — driver creation

Module 04 and 06 already used this without naming it. Naming it makes the
intent explicit and the pattern reusable beyond WebDriver.

```java
package com.example.framework;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.firefox.FirefoxDriver;

public class DriverFactory {

    public static WebDriver create(String browser) {
        return switch (browser.toLowerCase()) {
            case "chrome" -> new ChromeDriver();
            case "firefox" -> new FirefoxDriver();
            default -> throw new IllegalArgumentException("Unsupported browser: " + browser);
        };
    }
}
```

Every place that needs a driver calls `DriverFactory.create(...)` instead of
`new ChromeDriver()` directly — adding Safari later means editing one method,
not every test class.

## 2. Builder pattern — test data

```java
package com.example.framework;

public class UserBuilder {
    private String username = "default-user";
    private String email = "default@example.com";
    private String role = "STANDARD";
    private boolean active = true;

    public UserBuilder withUsername(String username) { this.username = username; return this; }
    public UserBuilder withEmail(String email) { this.email = email; return this; }
    public UserBuilder withRole(String role) { this.role = role; return this; }
    public UserBuilder inactive() { this.active = false; return this; }

    public TestUser build() { return new TestUser(username, email, role, active); }
}

public record TestUser(String username, String email, String role, boolean active) {}
```

```java
TestUser admin = new UserBuilder().withUsername("carol").withRole("ADMIN").build();
TestUser inactiveUser = new UserBuilder().withUsername("dave").inactive().build();
```

Sensible defaults plus fluent overrides mean a test only states what's
relevant to it — `inactiveUser` doesn't have to specify a username *and*
email *and* role just to flip one flag, and a reader immediately sees which
field the test actually cares about.

## 3. Singleton (used carefully) — shared, expensive resources

```java
package com.example.framework;

public class ConfigReader {
    private static ConfigReader instance;
    private final java.util.Properties properties = new java.util.Properties();

    private ConfigReader() {
        try (var in = getClass().getClassLoader().getResourceAsStream("config.properties")) {
            properties.load(in);
        } catch (Exception e) {
            throw new RuntimeException("Could not load config.properties", e);
        }
    }

    public static synchronized ConfigReader getInstance() {
        if (instance == null) instance = new ConfigReader();
        return instance;
    }

    public String get(String key) { return properties.getProperty(key); }
}
```

Config loaded from disk once, reused everywhere — right fit for a Singleton.
A `WebDriver`, by contrast, is a *bad* Singleton candidate: sharing one
browser session across parallel tests (Module 04) reintroduces exactly the
race conditions that per-thread creation was built to avoid. The pattern
isn't good or bad in the abstract — it's wrong for anything that needs
per-test isolation.

## 4. Strategy pattern — pluggable assertions/waits

```java
package com.example.framework;

import java.util.function.Predicate;
import java.util.function.Supplier;

public class RetryingCheck {

    public static <T> T until(Supplier<T> action, Predicate<T> condition,
                               int maxAttempts, long delayMillis) {
        RuntimeException last = null;
        for (int i = 0; i < maxAttempts; i++) {
            try {
                T result = action.get();
                if (condition.test(result)) return result;
            } catch (RuntimeException e) {
                last = e;
            }
            try { Thread.sleep(delayMillis); } catch (InterruptedException ignored) {}
        }
        throw new AssertionError("Condition not met after " + maxAttempts + " attempts", last);
    }
}
```

```java
String status = RetryingCheck.until(
        () -> orderService.getStatus(orderId),
        s -> s.equals("SHIPPED"),
        5, 500);
```

The retry *strategy* (attempts, delay) is decoupled from *what* is being
checked — the same helper works for an order status, a UI element's text, or
an eventually-consistent API response, without three copy-pasted retry
loops.

## 5. Putting it together — a small framework skeleton

```
src/test/java/com/example/framework/
  DriverFactory.java        (Factory)
  UserBuilder.java          (Builder)
  ConfigReader.java         (Singleton)
  RetryingCheck.java        (Strategy)
  pages/
    LoginPage.java          (Page Object, Level 2 Module 01)
  api/
    ApiClient.java          (wraps RestAssured spec, Level 2 Module 04)
  BaseTest.java
```

```java
package com.example.framework;

import org.junit.jupiter.api.*;
import org.openqa.selenium.WebDriver;

public abstract class BaseTest {
    protected WebDriver driver;

    @BeforeEach
    void baseSetUp() {
        String browser = ConfigReader.getInstance().get("browser");
        driver = DriverFactory.create(browser == null ? "chrome" : browser);
    }

    @AfterEach
    void baseTearDown() {
        if (driver != null) driver.quit();
    }
}
```

```java
class LoginFlowTest extends BaseTest {
    @Test
    void adminCanLogIn() {
        TestUser admin = new UserBuilder().withUsername("carol").withRole("ADMIN").build();
        driver.get("https://example.test/login");
        // ... Page Object calls using admin.username(), a real password fixture, etc.
    }
}
```

`BaseTest` centralizes setup/teardown once; every concrete test class
inherits it and only adds what's specific to that scenario — a smaller,
more honest diff for every new test than copy-pasting the driver lifecycle
each time.

I compiled and ran `UserBuilder`/`TestUser`/`RetryingCheck` locally with
plain JUnit 5 (no browser needed for these three) and confirmed the builder
produces the expected defaults/overrides and the retry helper both succeeds
within budget and fails correctly when the condition never becomes true.
`DriverFactory`, `ConfigReader`, and `BaseTest` are reviewed against the
Selenium/JUnit APIs used (and executed) elsewhere in this course, not run
directly here (no browser/config file in this environment).

```
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

## 6. Testing traps

!!! warning "Trap 1 — Singleton WebDriver"
    A `static WebDriver driver` shared across a suite looks efficient (one
    browser launch) until `parallel="methods"` (Module 04) is enabled, at
    which point two tests fight over one browser session and corrupt each
    other's state. Singletons are for stateless or read-only resources, not
    session-scoped ones.

!!! warning "Trap 2 — Builder defaults nobody remembers"
    A `UserBuilder` default of `role = "ADMIN"` (instead of the least
    privileged role) means every test that forgets to call `.withRole(...)`
    silently runs as admin — a permissions bug hidden by convenient
    defaults. Default to the least powerful, most restrictive state.

!!! warning "Trap 3 — retry strategy masking a real bug"
    `RetryingCheck.until(..., 10, 1000)` retrying for 10 seconds can turn a
    genuinely broken feature (never reaches the expected state) into a slow
    *pass* if the condition happens to flip true for an unrelated reason, or
    a slow, uninformative *failure* that took 10 seconds to tell you what a
    single check would have shown instantly. Use retries for known
    eventual-consistency delays, not as a substitute for fixing a race
    condition in the app.

!!! warning "Trap 4 — inheritance-heavy `BaseTest` hierarchies"
    A three-level `BaseTest → BaseUiTest → BaseAdminTest` chain makes it hard
    to tell, from inside a leaf test, what setup actually ran and in what
    order. Prefer composition (helper objects, JUnit extensions) over deep
    inheritance once a base class needs more than one clear responsibility.

!!! warning "Trap 5 — `ConfigReader` failing silently"
    Swallowing the exception in the `getInstance()` load step (an empty
    `catch`) means a missing `config.properties` produces `null` for every
    property, and a test fails downstream with a confusing NPE far from the
    real cause. Fail loudly and immediately at config-load time instead.

## Cheat sheet

| Pattern | Use for | Example here |
|---|---|---|
| Factory | Creating one of several related objects behind one method | `DriverFactory.create(browser)` |
| Builder | Constructing objects with many optional fields, sensible defaults | `UserBuilder` |
| Singleton | One expensive, stateless, shared resource | `ConfigReader` |
| Strategy | Swappable algorithm behind one interface | `RetryingCheck` |
| Page Object (Level 2) | One class per UI page/component | `LoginPage` |
| Template method (via inheritance) | Shared setup/teardown, per-test specifics | `BaseTest` |

## Exercise

1. Implement `DriverFactory`, `UserBuilder`/`TestUser`, `ConfigReader`, and
   `RetryingCheck` exactly as above; write and run a plain JUnit 5 test for
   the builder and the retry helper (no browser needed).
2. Build `BaseTest` and one concrete `LoginFlowTest` extending it; confirm
   setup/teardown run automatically per test.
3. Change `UserBuilder`'s default role from `"STANDARD"` to `"ADMIN"`, run
   your existing tests, and identify (in a comment) every test that would
   now silently run with elevated privileges — this is Trap 2 made
   concrete.
4. Write a test for `RetryingCheck.until(...)` where the condition never
   becomes true; confirm it throws `AssertionError` only after the full
   `maxAttempts * delayMillis` budget, and measure how long that took.
5. Refactor one Level 2 Page Object test to go through `DriverFactory`
   instead of `new ChromeDriver()` directly, and describe in one sentence
   what would need to change to add Safari support under each version.
