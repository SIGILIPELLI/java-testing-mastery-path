# 08 · Selenium WebDriver Basics

Selenium WebDriver drives a real browser programmatically: it opens pages,
finds elements, clicks, types and reads text — exactly the actions a manual
tester performs, but scripted. It is the tool that turns your regression
suite from a three-day manual grind into a twenty-minute automated run.

## 1. How WebDriver works

```
Your Java test  →  WebDriver API  →  chromedriver (browser driver)  →  Chrome
```

Your code calls a method like `driver.findElement(...)`. Selenium sends it as
a W3C WebDriver protocol command over HTTP to a **driver binary**
(`chromedriver`, `geckodriver`), which is the browser vendor's bridge into
the real browser. The browser performs the action and the result travels back.

!!! info "You no longer need to download chromedriver"
    Selenium 4.6+ includes **Selenium Manager**, which detects your installed
    browser, downloads the matching driver, and caches it — automatically. If
    you read an older tutorial telling you to call
    `System.setProperty("webdriver.chrome.driver", "/path/to/chromedriver")`,
    that step is obsolete. Just make sure Chrome or Firefox is installed.

The Selenium dependency is already in your `pom.xml` from Module 06.

## 2. Your first browser test

```java
package com.example.tests;

import org.junit.jupiter.api.*;
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;

import java.time.Duration;

import static org.junit.jupiter.api.Assertions.*;

class FirstSeleniumTest {

    private WebDriver driver;

    @BeforeEach
    void setUp() {
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--headless=new");        // no visible window
        options.addArguments("--window-size=1920,1080");

        driver = new ChromeDriver(options);
        driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(10));
    }

    @Test
    @DisplayName("The Selenium documentation site loads with the expected title")
    void pageLoadsWithCorrectTitle() {
        driver.get("https://www.selenium.dev/");

        String title = driver.getTitle();
        System.out.println("Page title: " + title);

        assertTrue(title.contains("Selenium"),
                "Title should mention Selenium but was: " + title);
        assertEquals("https://www.selenium.dev/", driver.getCurrentUrl());
    }

    @AfterEach
    void tearDown() {
        if (driver != null) {
            driver.quit();
        }
    }
}
```

```
Page title: Selenium
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

Note the structure — it is the JUnit lifecycle from Module 07 doing exactly
the job it exists for: `@BeforeEach` starts a fresh browser, `@AfterEach`
shuts it down.

!!! warning "`driver.quit()` vs `driver.close()`"
    `close()` closes the current window. `quit()` closes **every** window and
    terminates the driver process. Always use `quit()` in teardown — using
    `close()` leaks chromedriver processes, and after a hundred test runs
    your machine has a hundred zombie browsers eating memory. Wrap it in a
    null check so a failed setup does not mask the real error with a
    `NullPointerException`.

## 3. Locators — finding elements

A locator tells Selenium how to find an element on the page. There are eight
strategies; you will use three of them 90% of the time.

```java
import org.openqa.selenium.By;

By.id("username")                          // #1 choice -- fastest, most stable
By.name("email")
By.className("btn-primary")
By.tagName("input")
By.linkText("Forgot password?")            // exact, full link text
By.partialLinkText("Forgot")               // substring of link text
By.cssSelector("input[name='email']")      // #2 choice -- flexible and fast
By.xpath("//button[text()='Log in']")      // #3 -- most powerful, most brittle
```

### Locator priority

| Rank | Strategy | Why |
|---|---|---|
| 1 | `id` | Unique by HTML spec, fastest lookup, rarely changes |
| 2 | `name` | Usually stable on form fields |
| 3 | `cssSelector` | Fast, readable, handles most cases |
| 4 | `linkText` | Natural for navigation links |
| 5 | `xpath` | Only when you need text matching or parent traversal |
| — | `className`, `tagName` | Too often non-unique |

### CSS selector syntax

```java
By.cssSelector("#login-btn")                  // id
By.cssSelector(".error-message")              // class
By.cssSelector("input[type='password']")      // attribute
By.cssSelector("input[name^='user']")         // attribute starts with
By.cssSelector("input[name$='name']")         // attribute ends with
By.cssSelector("a[href*='checkout']")         // attribute contains
By.cssSelector("form#login input.field")      // descendant
By.cssSelector("div.cart > span.total")       // direct child
By.cssSelector("li:nth-child(3)")             // third item in a list
```

### XPath syntax

```java
By.xpath("//input[@id='username']")                    // attribute match
By.xpath("//button[text()='Log in']")                  // exact text
By.xpath("//button[contains(text(),'Log')]")           // partial text
By.xpath("//div[@class='row'][2]")                     // second match
By.xpath("//label[text()='Email']/following-sibling::input")
By.xpath("//span[@class='price']/parent::div")         // walk upward
By.xpath("//input[@name='email' and @required]")       // multiple conditions
```

XPath's unique powers are **matching on visible text** and **traversing to a
parent**. CSS can do neither. Use XPath for exactly those cases.

!!! warning "Never use an absolute XPath"
    `/html/body/div[3]/div[2]/form/div[1]/input` breaks the moment a designer
    adds a wrapper div. Copying "Full XPath" from browser DevTools is the
    single most common source of unmaintainable test suites. Always write a
    relative locator anchored on something meaningful — an id, a name, a
    stable attribute, or visible text.

!!! info "Ask developers for test hooks"
    The most robust locator is one the application provides on purpose:
    `<button data-testid="checkout-submit">`. Then
    `By.cssSelector("[data-testid='checkout-submit']")` survives every CSS
    refactor. Requesting `data-testid` attributes on key elements is a
    reasonable and common QA ask — make it early in a project.

## 4. Interacting with elements

```java
package com.example.tests;

import org.junit.jupiter.api.*;
import org.openqa.selenium.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import java.time.Duration;
import static org.junit.jupiter.api.Assertions.*;

class InteractionTest {

    private WebDriver driver;

    @BeforeEach
    void setUp() {
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--headless=new", "--window-size=1920,1080");
        driver = new ChromeDriver(options);
        driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(10));
    }

    @Test
    @DisplayName("Login with valid credentials lands on the secure area")
    void loginWithValidCredentials() {
        driver.get("https://the-internet.herokuapp.com/login");

        WebElement username = driver.findElement(By.id("username"));
        WebElement password = driver.findElement(By.id("password"));
        WebElement loginBtn  = driver.findElement(By.cssSelector("button[type='submit']"));

        username.clear();
        username.sendKeys("tomsmith");
        password.sendKeys("SuperSecretPassword!");
        loginBtn.click();

        WebElement flash = driver.findElement(By.id("flash"));
        System.out.println("Flash message: " + flash.getText().trim());

        assertAll("successful login",
                () -> assertTrue(driver.getCurrentUrl().contains("/secure"),
                        "Should redirect to the secure area"),
                () -> assertTrue(flash.getText().contains("You logged into a secure area"),
                        "Success message should be displayed"),
                () -> assertTrue(driver.findElement(By.linkText("Logout")).isDisplayed(),
                        "Logout link should be visible"));
    }

    @Test
    @DisplayName("Login with an invalid username shows an error and stays on /login")
    void loginWithInvalidUsername() {
        driver.get("https://the-internet.herokuapp.com/login");

        driver.findElement(By.id("username")).sendKeys("wronguser");
        driver.findElement(By.id("password")).sendKeys("SuperSecretPassword!");
        driver.findElement(By.cssSelector("button[type='submit']")).click();

        String flash = driver.findElement(By.id("flash")).getText();

        assertAll("failed login",
                () -> assertTrue(driver.getCurrentUrl().endsWith("/login"),
                        "Should remain on the login page"),
                () -> assertTrue(flash.contains("Your username is invalid"),
                        "Expected an invalid-username error but got: " + flash));
    }

    @AfterEach
    void tearDown() {
        if (driver != null) driver.quit();
    }
}
```

```
Flash message: You logged into a secure area!
×
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

That second test is `TC_LOGIN_003` from Module 02 — the negative case,
automated, with a multi-part expected result. The manual test case and the
automated test are the same artefact expressed two ways.

### The element API

| Method | Does |
|---|---|
| `click()` | Clicks the element |
| `sendKeys("text")` | Types into it |
| `clear()` | Empties an input — always call before `sendKeys` on a pre-filled field |
| `getText()` | Visible inner text |
| `getAttribute("value")` | An attribute's value (use this for input contents, not `getText()`) |
| `getCssValue("color")` | A computed CSS property |
| `isDisplayed()` | Is it visible? |
| `isEnabled()` | Is it interactable (not disabled)? |
| `isSelected()` | Is a checkbox/radio checked? |
| `submit()` | Submits the containing form |

### The driver API

| Method | Does |
|---|---|
| `get(url)` / `navigate().to(url)` | Load a page |
| `getTitle()` | Page title |
| `getCurrentUrl()` | Current URL |
| `getPageSource()` | Full HTML |
| `findElement(By)` | First match — throws `NoSuchElementException` if none |
| `findElements(By)` | All matches as a `List` — **empty list if none, never throws** |
| `navigate().back()` / `.forward()` / `.refresh()` | Browser history controls |
| `manage().window().maximize()` | Maximize |
| `manage().deleteAllCookies()` | Clear cookies — useful in `@BeforeEach` |
| `quit()` | Close everything and end the session |

!!! info "Use `findElements` to assert absence"
    To verify an element is *not* on the page, `findElement` throws, which is
    awkward to assert on. Use `findElements` and check the list is empty:
    ```java
    assertTrue(driver.findElements(By.id("admin-panel")).isEmpty(),
            "Admin panel must not be visible to a standard user");
    ```

## 5. Dropdowns, checkboxes and radio buttons

```java
import org.openqa.selenium.support.ui.Select;

@Test
void dropdownSelection() {
    driver.get("https://the-internet.herokuapp.com/dropdown");

    Select dropdown = new Select(driver.findElement(By.id("dropdown")));

    dropdown.selectByVisibleText("Option 2");
    assertEquals("Option 2", dropdown.getFirstSelectedOption().getText());

    dropdown.selectByValue("1");
    assertEquals("Option 1", dropdown.getFirstSelectedOption().getText());

    dropdown.selectByIndex(2);
    assertEquals("Option 2", dropdown.getFirstSelectedOption().getText());

    System.out.println("Available options: " + dropdown.getOptions().size());
}

@Test
void checkboxes() {
    driver.get("https://the-internet.herokuapp.com/checkboxes");

    List<WebElement> boxes = driver.findElements(By.cssSelector("#checkboxes input"));

    assertFalse(boxes.get(0).isSelected(), "First checkbox starts unchecked");
    assertTrue(boxes.get(1).isSelected(),  "Second checkbox starts checked");

    boxes.get(0).click();
    assertTrue(boxes.get(0).isSelected(), "First checkbox should be checked after clicking");
}
```

```
Available options: 3
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
```

The `Select` class only works on real `<select>` elements. Modern
JavaScript "dropdowns" built from `<div>`s need ordinary clicks — a trap that
catches everyone once.

## 6. Waits — the most important concept in Selenium

Web pages load asynchronously. Your code runs in milliseconds; the browser
takes seconds. Without waits, tests fail randomly with
`NoSuchElementException` or `ElementNotInteractableException`.

```java
// ❌ NEVER do this
Thread.sleep(5000);
```

A hard sleep is always either too short (flaky) or too long (slow). A suite
built on `Thread.sleep` takes hours and still fails.

### Implicit wait

Set once; applies to every `findElement` call. The driver polls for up to the
given duration before throwing.

```java
driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(10));
```

Simple and useful as a baseline, but it only waits for *presence in the DOM* —
not visibility, not clickability.

### Explicit wait

Waits for a specific *condition* on a specific element. This is what you
should reach for whenever an implicit wait is not enough.

```java
import org.openqa.selenium.support.ui.WebDriverWait;
import org.openqa.selenium.support.ui.ExpectedConditions;

@Test
void explicitWaitForDynamicElement() {
    driver.get("https://the-internet.herokuapp.com/dynamic_loading/1");

    driver.findElement(By.cssSelector("#start button")).click();

    WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(15));
    WebElement finish = wait.until(
            ExpectedConditions.visibilityOfElementLocated(By.id("finish")));

    assertEquals("Hello World!", finish.getText());
}
```

```
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
```

Common `ExpectedConditions`:

| Condition | Waits until |
|---|---|
| `visibilityOfElementLocated(By)` | Present in DOM **and** visible |
| `presenceOfElementLocated(By)` | Present in DOM (may be hidden) |
| `elementToBeClickable(By)` | Visible and enabled |
| `invisibilityOfElementLocated(By)` | Gone or hidden — perfect for spinners |
| `textToBePresentInElementLocated(By, "text")` | The element contains that text |
| `titleContains("Dashboard")` | Page title contains a string |
| `urlContains("/secure")` | URL contains a string |
| `numberOfElementsToBe(By, 5)` | Exactly n matching elements exist |
| `alertIsPresent()` | A JavaScript alert appeared |

!!! warning "Do not mix implicit and explicit waits carelessly"
    Combining them can produce unpredictable, additive timeouts (an explicit
    wait of 10s plus an implicit wait of 10s can take far longer than either).
    The safest habit on a real project: set the implicit wait to zero and use
    explicit waits everywhere. Level 2, Module 02 covers wait strategy in full.

## 7. Screenshots on failure

When a test fails at 3 a.m. on CI, a screenshot is the difference between a
five-minute diagnosis and an hour.

```java
import org.openqa.selenium.OutputType;
import org.openqa.selenium.TakesScreenshot;
import java.io.File;
import java.nio.file.Files;
import java.nio.file.Path;

private void captureScreenshot(String name) {
    try {
        File src = ((TakesScreenshot) driver).getScreenshotAs(OutputType.FILE);
        Path dest = Path.of("target", "screenshots", name + ".png");
        Files.createDirectories(dest.getParent());
        Files.copy(src.toPath(), dest);
        System.out.println("Screenshot saved: " + dest.toAbsolutePath());
    } catch (Exception e) {
        System.out.println("Could not capture screenshot: " + e.getMessage());
    }
}
```

```
Screenshot saved: /Users/qa/java-testing-practice/target/screenshots/login-failure.png
```

Attach that file to the bug report you write in Module 04 — the workflow is
continuous.

## 8. Headless mode and browser options

```java
ChromeOptions options = new ChromeOptions();
options.addArguments("--headless=new");          // no visible window -- required on CI
options.addArguments("--window-size=1920,1080"); // headless has no real window size
options.addArguments("--disable-gpu");
options.addArguments("--no-sandbox");            // needed inside Docker containers
options.addArguments("--incognito");
options.addArguments("--lang=en-GB");
```

Headless is faster and mandatory on a CI server with no display. Develop with
a visible browser (drop the flag) so you can *see* what your test does, then
switch to headless for the automated runs.

For Firefox:

```java
import org.openqa.selenium.firefox.FirefoxDriver;
import org.openqa.selenium.firefox.FirefoxOptions;

FirefoxOptions ffOptions = new FirefoxOptions();
ffOptions.addArguments("-headless");
WebDriver driver = new FirefoxDriver(ffOptions);
```

## 9. Common exceptions and what they mean

| Exception | Cause | Fix |
|---|---|---|
| `NoSuchElementException` | Locator wrong, or element not yet rendered | Verify the locator in DevTools; add an explicit wait |
| `ElementNotInteractableException` | Element exists but is hidden or covered | Wait for `elementToBeClickable`; scroll it into view |
| `StaleElementReferenceException` | The page re-rendered after you found the element | Re-find the element after the change; do not cache `WebElement`s across navigations |
| `TimeoutException` | The wait condition never became true | The condition is wrong, or the app genuinely is broken — check manually first |
| `ElementClickInterceptedException` | A cookie banner/modal is over your element | Dismiss the overlay, or wait for it to disappear |
| `SessionNotCreatedException` | Browser/driver version mismatch | Update the browser; Selenium Manager handles the driver |
| `InvalidSelectorException` | Malformed CSS or XPath | Test the selector in DevTools console: `$$("css")` or `$x("xpath")` |

!!! info "Test your locators in the browser first"
    In Chrome DevTools console: `$$("input[name='email']")` evaluates a CSS
    selector, `$x("//button[text()='Log in']")` evaluates an XPath. Both
    return the matched elements. Confirming a locator here takes five seconds
    and saves a whole debug cycle.

## Cheat sheet

| Task | Code |
|---|---|
| Start Chrome | `WebDriver driver = new ChromeDriver(options);` |
| Headless | `options.addArguments("--headless=new");` |
| Open a URL | `driver.get("https://...")` |
| Find one element | `driver.findElement(By.id("username"))` |
| Find many | `driver.findElements(By.cssSelector(".item"))` |
| Type | `element.sendKeys("text")` |
| Clear then type | `element.clear(); element.sendKeys("text");` |
| Click | `element.click()` |
| Read text | `element.getText()` |
| Read an input's value | `element.getAttribute("value")` |
| Dropdown | `new Select(el).selectByVisibleText("Option 2")` |
| Explicit wait | `new WebDriverWait(driver, Duration.ofSeconds(10)).until(...)` |
| Screenshot | `((TakesScreenshot) driver).getScreenshotAs(OutputType.FILE)` |
| End the session | `driver.quit()` |

## Exercise

Use the practice site `https://the-internet.herokuapp.com/`, which is built
for exactly this purpose.

1. Write `LoginSeleniumTest` with `@BeforeEach`/`@AfterEach` lifecycle
   handling and headless Chrome, containing **four** tests against
   `/login`:
   - valid username + valid password → lands on `/secure`, success flash,
     Logout link visible
   - valid username + wrong password → error flash, still on `/login`
   - invalid username → error flash mentioning the username
   - both fields empty → error flash
   Use `assertAll` so each test reports every part of its expected result.
2. On `/dynamic_loading/2`, write a test that clicks **Start** and waits with
   an **explicit wait** for the "Hello World!" text. Then deliberately change
   the wait to 1 second and observe the `TimeoutException` — read the message
   carefully so you recognise it later.
3. On `/checkboxes`, write a test that asserts the initial checked state of
   both boxes, toggles both, and asserts the new state.
4. On `/dropdown`, write a test that asserts the dropdown has exactly 3
   options (including the disabled placeholder) and that selecting by visible
   text, by value and by index all produce the expected selection.
5. Add a `captureScreenshot` helper (section 7) and call it from
   `@AfterEach` whenever a test fails, naming the file after the test method.
   Break one test on purpose and confirm the PNG appears in
   `target/screenshots/`.
6. Rewrite every XPath you used as a CSS selector where possible, and write
   one sentence explaining the one case where you genuinely needed XPath.
