# 09 · Cross-Browser Testing

"Works on my machine" has a browser-shaped variant: works in Chrome. Firefox
renders a date input differently, Safari handles cookies more strictly, and
Edge inherits Chromium's engine but not its defaults. Cross-browser testing
is how you find those differences before a customer does — and the framework
work is mostly one class: a driver factory that takes a browser name.

## 1. Decide what actually needs a second browser

Running the whole suite on four browsers quadruples your runtime and your
maintenance for very little extra defect-finding. A sane split:

| Layer | Where it should run |
|---|---|
| Business logic, API tests | One browser — or none |
| Full functional regression | One primary browser (usually Chrome) |
| Smoke / critical user journeys | Every supported browser |
| Rendering and layout | Every supported browser, plus mobile viewports |

Get the supported list from analytics, not from opinion. "Chrome, Firefox,
Safari, Edge, current and previous major version" is a typical policy; write
it into the test plan so the scope is a decision, not an accident.

## 2. A driver factory

```java
package com.example.driver;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.*;
import org.openqa.selenium.edge.*;
import org.openqa.selenium.firefox.*;
import org.openqa.selenium.safari.SafariDriver;

import java.time.Duration;
import java.util.Locale;

public final class DriverFactory {

    private static final ThreadLocal<WebDriver> DRIVER = new ThreadLocal<>();

    public static WebDriver get() {
        WebDriver driver = DRIVER.get();
        if (driver == null) {
            throw new IllegalStateException("No driver on this thread -- call start() first");
        }
        return driver;
    }

    public static void start(String browser, boolean headless) {
        WebDriver driver = switch (browser.toLowerCase(Locale.ROOT)) {
            case "chrome"  -> new ChromeDriver(chromeOptions(headless));
            case "firefox" -> new FirefoxDriver(firefoxOptions(headless));
            case "edge"    -> new EdgeDriver(edgeOptions(headless));
            case "safari"  -> new SafariDriver();          // macOS only, no headless mode
            default -> throw new IllegalArgumentException("Unsupported browser: " + browser);
        };

        driver.manage().timeouts().implicitlyWait(Duration.ZERO);
        driver.manage().timeouts().pageLoadTimeout(Duration.ofSeconds(30));
        DRIVER.set(driver);
    }

    private static ChromeOptions chromeOptions(boolean headless) {
        ChromeOptions options = new ChromeOptions();
        if (headless) options.addArguments("--headless=new");
        options.addArguments("--window-size=1920,1080", "--disable-gpu",
                             "--no-sandbox", "--disable-dev-shm-usage",
                             "--lang=en-GB");
        return options;
    }

    private static FirefoxOptions firefoxOptions(boolean headless) {
        FirefoxOptions options = new FirefoxOptions();
        if (headless) options.addArguments("-headless");
        options.addArguments("--width=1920", "--height=1080");
        options.addPreference("intl.accept_languages", "en-GB");
        return options;
    }

    private static EdgeOptions edgeOptions(boolean headless) {
        EdgeOptions options = new EdgeOptions();
        if (headless) options.addArguments("--headless=new");
        options.addArguments("--window-size=1920,1080");
        return options;
    }

    public static void stop() {
        WebDriver driver = DRIVER.get();
        if (driver != null) {
            driver.quit();
            DRIVER.remove();
        }
    }

    private DriverFactory() { }
}
```

Note the differences the factory hides: Chrome and Edge take
`--headless=new`, Firefox takes `-headless` with a single dash, Firefox sizes
via `--width`/`--height`, and Safari has no headless mode at all. Every one
of those is a trap someone hits on their first cross-browser day.

!!! info "Selenium Manager handles the driver binaries"
    Selenium 4.6+ downloads and caches the matching `chromedriver`,
    `geckodriver` or `msedgedriver` automatically. You do not need
    WebDriverManager or `System.setProperty` any more — but the *browser*
    itself must be installed, and Safari needs
    `safaridriver --enable` run once.

## 3. Parameterising the suite

### TestNG — one `<test>` block per browser

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">
<suite name="Cross-Browser Smoke" parallel="tests" thread-count="3">

    <test name="Chrome">
        <parameter name="browser" value="chrome"/>
        <classes><class name="com.example.tests.LoginSmokeIT"/></classes>
    </test>

    <test name="Firefox">
        <parameter name="browser" value="firefox"/>
        <classes><class name="com.example.tests.LoginSmokeIT"/></classes>
    </test>

    <test name="Edge">
        <parameter name="browser" value="edge"/>
        <classes><class name="com.example.tests.LoginSmokeIT"/></classes>
    </test>
</suite>
```

```java
package com.example.tests;

import com.example.driver.DriverFactory;
import org.testng.annotations.*;

import static org.testng.Assert.assertTrue;

public class LoginSmokeIT {

    @Parameters({"browser"})
    @BeforeMethod(alwaysRun = true)
    public void startBrowser(@Optional("chrome") String browser) {
        DriverFactory.start(browser, Boolean.parseBoolean(
                System.getProperty("headless", "true")));
    }

    @Test(groups = "smoke", description = "The login page renders on every supported browser")
    public void loginPageRenders() {
        DriverFactory.get().get("https://the-internet.herokuapp.com/login");
        assertTrue(DriverFactory.get().getTitle().contains("The Internet"),
                "Unexpected title: " + DriverFactory.get().getTitle());
    }

    @AfterMethod(alwaysRun = true)
    public void stopBrowser() {
        DriverFactory.stop();
    }
}
```

```
===============================================
Cross-Browser Smoke
Total tests run: 3, Passes: 3, Failures: 0, Skips: 0
===============================================
```

`parallel="tests"` runs the three browsers simultaneously, so three browsers
cost roughly the wall-clock time of one.

### JUnit 5 — a parameterised test

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

@ParameterizedTest(name = "login page renders in {0}")
@ValueSource(strings = {"chrome", "firefox"})
void loginPageRenders(String browser) {
    DriverFactory.start(browser, true);
    try {
        DriverFactory.get().get("https://the-internet.herokuapp.com/login");
        assertThat(DriverFactory.get().getTitle()).contains("The Internet");
    } finally {
        DriverFactory.stop();
    }
}
```

## 4. Selenium Grid and cloud providers

One machine cannot host Safari and Edge and three Chrome versions. A Grid
(or a cloud vendor) runs browsers remotely; your test connects over HTTP.

```bash
# The whole grid, one container
docker run -d -p 4444:4444 -p 7900:7900 --shm-size=2g selenium/standalone-chrome:latest
```

```java
import org.openqa.selenium.remote.RemoteWebDriver;
import java.net.URL;

public static WebDriver remote(String gridUrl, String browser) throws Exception {
    Capabilities caps = switch (browser) {
        case "firefox" -> new FirefoxOptions();
        case "edge"    -> new EdgeOptions();
        default        -> new ChromeOptions();
    };
    return new RemoteWebDriver(new URL(gridUrl), caps);
}
```

```java
// Cloud vendors take extra capabilities in a vendor-prefixed map
ChromeOptions options = new ChromeOptions();
options.setCapability("browserVersion", "126");
options.setCapability("platformName", "Windows 11");
options.setCapability("cloud:options", Map.of(
        "build",   System.getenv("BUILD_NUMBER"),
        "name",    "Login smoke",
        "video",   true));
```

Make the factory return a `RemoteWebDriver` when a `grid.url` property is
set, and a local driver otherwise. Tests then run unchanged locally and on
CI — the only property that changes is a URL.

!!! warning "`--shm-size=2g` is not optional in Docker"
    The default 64 MB of shared memory makes Chrome crash intermittently
    inside containers with `session deleted because of page crash`. It looks
    exactly like a flaky test and it is not.

## 5. Responsive and viewport testing

```java
driver.manage().window().setSize(new Dimension(375, 812));   // iPhone-ish
driver.manage().window().setSize(new Dimension(768, 1024));  // tablet
driver.manage().window().maximize();                          // desktop

// Chrome's device emulation
ChromeOptions options = new ChromeOptions();
options.setExperimentalOption("mobileEmulation", Map.of("deviceName", "Pixel 7"));
```

Emulation resizes the viewport and spoofs the user agent. It does **not**
reproduce a real mobile browser engine, touch behaviour or device
performance — good for catching layout breakpoints, not a substitute for a
real device when the defect is behavioural.

## 6. Testing traps

!!! warning "Trap 1 — locators that only work in one browser"
    `By.xpath("//div[@class='x']/text()")` and CSS pseudo-elements behave
    differently across engines. If a locator fails only in Firefox, suspect
    the locator before the application.

!!! warning "Trap 2 — timing differences read as bugs"
    Firefox is often slower to fire load events than Chrome. A test that
    passes in Chrome with a 5-second wait and fails in Firefox usually has an
    inadequate wait, not a Firefox defect. Fix the wait; do not add a
    browser-specific sleep.

!!! warning "Trap 3 — headless behaves differently from headed"
    Headless has no real window size, no GPU compositing, and in some
    versions different font rendering. A layout bug that only appears
    headless may not exist for users. Reproduce headed before raising it.

!!! warning "Trap 4 — `click()` intercepted by different scroll positions"
    Browsers scroll elements into view differently, so an element clickable
    in Chrome can be under a sticky header in Firefox. Scroll deliberately
    (`scrollIntoView({block:'center'})`) instead of relying on the default.

!!! warning "Trap 5 — one failure across three browsers is three failures"
    A cross-browser report showing 3 failed of 90 hides that it is the *same*
    test failing everywhere. Group your report by test, not by browser, or
    you will triage the same defect three times.

## Cheat sheet

| Task | Code |
|---|---|
| Chrome headless | `options.addArguments("--headless=new")` |
| Firefox headless | `options.addArguments("-headless")` |
| Edge headless | `options.addArguments("--headless=new")` |
| Safari | `new SafariDriver()` — macOS only, no headless |
| Enable safaridriver | `safaridriver --enable` (once, as admin) |
| Window size, Chrome/Edge | `--window-size=1920,1080` |
| Window size, Firefox | `--width=1920 --height=1080` |
| Resize at runtime | `driver.manage().window().setSize(new Dimension(375, 812))` |
| Mobile emulation | `options.setExperimentalOption("mobileEmulation", ...)` |
| Remote driver | `new RemoteWebDriver(new URL(gridUrl), options)` |
| Grid in Docker | `docker run -p 4444:4444 --shm-size=2g selenium/standalone-chrome` |
| Watch a Grid session | VNC on port 7900 |
| Browser per TestNG block | `<test><parameter name="browser" value="firefox"/></test>` |
| Browsers in parallel | `<suite parallel="tests" thread-count="3">` |
| Thread safety | `ThreadLocal<WebDriver>` + `remove()` |

## Exercise

1. Build `DriverFactory` exactly as in section 2 and get one test passing on
   Chrome and Firefox locally. Record every difference you had to encode in
   the options methods.
2. Write a `cross-browser.xml` with three `<test>` blocks and
   `parallel="tests"`. Compare total runtime against running them
   sequentially.
3. Take one Module 01 page object test and run it on both browsers. If it
   passes on one and fails on the other, decide — with evidence — whether it
   is a locator problem, a wait problem, or a genuine application defect, and
   write the reasoning down.
4. Run the same suite headed and headless on Chrome. Note any difference in
   runtime and in results, and explain trap 3 in your own words.
5. Start `selenium/standalone-chrome` in Docker, point a `RemoteWebDriver` at
   `http://localhost:4444`, and run your smoke test against it. Watch it via
   VNC on port 7900.
6. Extend the factory so a `-Dgrid.url=...` property switches between local
   and remote with no change to any test class.
7. Resize the viewport to 375x812, run your login test, and note whether the
   navigation is still reachable. If a hamburger menu appeared, your page
   object now needs a responsive strategy — describe it in three sentences.
8. Write the browser-support section of a test plan: which browsers, which
   versions, which subset of tests runs on each, and the justification.
