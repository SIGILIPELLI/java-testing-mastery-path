# 04 · Selenium Grid & Parallel Execution

A suite of 200 UI tests at 5 seconds each is 16 minutes run one at a time.
Selenium Grid lets many browsers run those tests concurrently across
machines (or containers); TestNG's parallel execution lets your test runner
actually dispatch work to them at the same time instead of queuing.
Together they turn a coffee-break suite into a two-minute one.

## 1. Selenium Grid architecture

- **Hub** — receives test requests and routes them to an available node.
- **Node** — a machine (or container) that actually runs a browser.
- Since Selenium 4, "Grid" ships as a single jar with `standalone`, `hub`,
  and `node` roles, most commonly run via Docker.

```yaml
# docker-compose.yml
services:
  selenium-hub:
    image: selenium/hub:4.20
    ports: ["4442:4442", "4443:4443", "4444:4444"]

  chrome-node:
    image: selenium/node-chrome:4.20
    shm_size: 2gb
    depends_on: [selenium-hub]
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
      - SE_NODE_MAX_SESSIONS=4

  firefox-node:
    image: selenium/node-firefox:4.20
    shm_size: 2gb
    depends_on: [selenium-hub]
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
```

`shm_size: 2gb` matters more than it looks: Chrome's default `/dev/shm` in a
container is 64MB, and Chrome silently crashes under load without a bump —
one of the most common "works locally, dies in Docker" Selenium bugs.

## 2. Pointing tests at the Grid with RemoteWebDriver

```java
package com.example.grid;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.remote.RemoteWebDriver;
import java.net.URL;

public class RemoteDriverFactory {

    public static WebDriver create(String browser) throws Exception {
        URL hubUrl = new URL("http://localhost:4444/wd/hub");

        return switch (browser) {
            case "chrome" -> new RemoteWebDriver(hubUrl, new ChromeOptions());
            case "firefox" -> new RemoteWebDriver(hubUrl,
                    new org.openqa.selenium.firefox.FirefoxOptions());
            default -> throw new IllegalArgumentException("Unknown browser: " + browser);
        };
    }
}
```

The test code above this factory doesn't change at all versus a local
`ChromeDriver()` from Level 1 — `RemoteWebDriver` implements the same
`WebDriver` interface, so `driver.findElement(...)`, waits, and page objects
all work identically. Only the wiring at setup changes.

## 3. TestNG parallel execution

`testng.xml`:

```xml
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">
<suite name="RegressionSuite" parallel="classes" thread-count="4">
    <test name="ChromeTests">
        <parameter name="browser" value="chrome"/>
        <classes>
            <class name="com.example.grid.LoginFlowTest"/>
            <class name="com.example.grid.CheckoutFlowTest"/>
        </classes>
    </test>
</suite>
```

`parallel="classes"` runs each `<class>` on its own thread;
`parallel="methods"` goes finer-grained and runs each `@Test` method
concurrently. `thread-count` caps how many run at once — set it to (roughly)
the number of node sessions your Grid can actually serve, not higher, or
tests will queue at the Grid instead of your test runner.

```java
package com.example.grid;

import org.openqa.selenium.WebDriver;
import org.testng.annotations.*;

public class LoginFlowTest {

    private WebDriver driver;

    @BeforeMethod
    @Parameters("browser")
    public void setUp(String browser) throws Exception {
        driver = RemoteDriverFactory.create(browser);
    }

    @AfterMethod
    public void tearDown() {
        if (driver != null) driver.quit();
    }

    @Test
    public void validLoginReachesDashboard() {
        driver.get("https://example.test/login");
        // ... same Page Object calls as Level 2 Module 01
    }
}
```

Creating the `WebDriver` in `@BeforeMethod` — never as a shared static field
— is what makes this safe under `parallel="methods"`: each thread gets its
own driver instance and its own browser session, so one test's navigation
can't leak into another's.

I did not run an actual Selenium Grid or Docker containers in this
environment (no Docker daemon available headlessly here), so the
`docker-compose.yml`, `RemoteWebDriver` wiring, and TestNG config above are
reviewed against Selenium 4's and TestNG's documented behavior, not
executed. The parallel-safety pattern (per-thread driver via `@BeforeMethod`,
never static) is the same one used correctly in Level 2's Page Object
Model module, which *was* exercised there.

## 4. Cross-browser matrix without duplicating test code

```xml
<suite name="CrossBrowserSuite" parallel="tests" thread-count="2">
    <test name="Chrome">
        <parameter name="browser" value="chrome"/>
        <classes><class name="com.example.grid.LoginFlowTest"/></classes>
    </test>
    <test name="Firefox">
        <parameter name="browser" value="firefox"/>
        <classes><class name="com.example.grid.LoginFlowTest"/></classes>
    </test>
</suite>
```

One test class, two `<test>` blocks, `parallel="tests"` — the same
`LoginFlowTest` runs against both browsers concurrently, extending the
cross-browser approach from Level 2 Module 09 from sequential to parallel.

## 5. Testing traps

!!! warning "Trap 1 — shared mutable state across parallel threads"
    A `static Map<String,String> testData` written by one test and read by
    another is a race condition once `parallel="methods"` is on. Symptoms:
    tests that pass every time alone and fail unpredictably in the full
    suite. Fix: no shared mutable statics, ever, in a parallel suite.

!!! warning "Trap 2 — Grid node pool smaller than thread-count"
    `thread-count="10"` against a Grid with `SE_NODE_MAX_SESSIONS=4` doesn't
    fail loudly — the extra sessions just queue, and your "parallel" run is
    silently bottlenecked to 4-wide while looking like it should be 10-wide.
    Match thread-count to actual node capacity.

!!! warning "Trap 3 — flaky selectors multiply under parallel load"
    A selector relying on a fixed animation delay (a hard `Thread.sleep`)
    that barely worked under light single-threaded load will fail far more
    often once four sessions are competing for the same host's CPU.
    Parallel execution is the moment a marginal wait strategy becomes an
    unmissable one — replace sleeps with explicit `WebDriverWait` (Level 1
    Module 08) before scaling up.

!!! warning "Trap 4 — Chrome crashing silently from `/dev/shm`"
    Symptom: `SessionNotCreatedException` or the browser process disappearing
    mid-test with no clear Selenium-side error, specifically under
    Dockerized Grid nodes. Almost always the default 64MB `/dev/shm` — bump
    `shm_size`.

!!! warning "Trap 5 — one slow test holding up the whole parallel run"
    `parallel="classes"` with `thread-count="4"` still waits for the
    single slowest class if the other three finish early and there's no
    fifth class queued. Balance test classes by expected duration, not just
    count, when distributing across threads.

## Cheat sheet

| Task | Code / Config |
|---|---|
| Point WebDriver at a Grid | `new RemoteWebDriver(hubUrl, options)` |
| Grid hub URL (default) | `http://localhost:4444/wd/hub` |
| Parallel by class | `<suite parallel="classes" thread-count="N">` |
| Parallel by method | `<suite parallel="methods" thread-count="N">` |
| Parallel by `<test>` block (cross-browser) | `<suite parallel="tests">` |
| Pass a parameter per `<test>` | `<parameter name="browser" value="chrome"/>` |
| Read it in Java | `@Parameters("browser")` on a `@BeforeMethod` |
| Per-thread driver (required for safety) | create in `@BeforeMethod`, never `static` |
| Fix Chrome crashing in Docker | `shm_size: 2gb` on the node service |
| Match Grid capacity | `SE_NODE_MAX_SESSIONS` ≈ `thread-count` |

## Exercise

1. Bring up `selenium-hub` + `chrome-node` + `firefox-node` via the compose
   file above (or install Selenium Server standalone if Docker isn't
   available) and confirm the Grid console at `http://localhost:4444` shows
   both nodes registered.
2. Write `RemoteDriverFactory` and point `LoginFlowTest` at the Grid instead
   of a local driver; run it once against each browser and confirm both
   pass.
3. Convert `testng.xml` to `parallel="methods" thread-count="4"` across a
   suite of at least 4 test methods, and time the run versus a sequential
   (`parallel="none"`) run of the same suite.
4. Deliberately add a `static` field that one test writes and another reads,
   run the suite under `parallel="methods"` ten times, and record how many
   runs failed versus passed — this is Trap 1 made visible.
5. Set `SE_NODE_MAX_SESSIONS=1` on your Chrome node, set
   `thread-count="4"` in TestNG, and time the run. Explain the gap between
   this time and the Exercise 3 time in terms of Trap 2.
