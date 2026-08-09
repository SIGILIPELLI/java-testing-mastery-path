# 02 · Selenium Advanced

Level 1 gave you `findElement`, `click` and `sendKeys` — enough for a form.
Real applications add spinners, iframes, native alerts, drag-and-drop, new
tabs and elements that only appear on hover. This module covers the Selenium
APIs for all of it, and settles the wait strategy question properly, because
timing is the cause of roughly every flaky UI test ever written.

## 1. Wait strategy — pick one and commit

| Wait | Scope | Waits for | Verdict |
|---|---|---|---|
| `Thread.sleep` | One line | Nothing — it just stops | Never |
| Implicit | Every `findElement`, globally | Presence in the DOM only | Baseline only |
| Explicit (`WebDriverWait`) | One call | A named condition | **Default choice** |
| `FluentWait` | One call | A condition, with tuning | For polling odd things |

!!! warning "Do not mix implicit and explicit waits"
    The implicit wait applies inside `WebDriverWait`'s polling loop, so a 10s
    explicit wait layered on a 10s implicit wait can take far longer than 10
    seconds — and in some driver versions produces genuinely unpredictable
    timeouts. The professional setting is **implicit wait = 0**, explicit
    waits everywhere:
    ```java
    driver.manage().timeouts().implicitlyWait(Duration.ZERO);
    ```

### FluentWait

`WebDriverWait` is a `FluentWait<WebDriver>` with defaults. Build your own
when you need a different polling interval or extra ignored exceptions.

```java
package com.example.tests;

import org.openqa.selenium.*;
import org.openqa.selenium.support.ui.FluentWait;
import org.openqa.selenium.support.ui.Wait;

import java.time.Duration;
import java.util.function.Function;

public final class Waits {

    public static WebElement waitForElement(WebDriver driver, By locator) {
        Wait<WebDriver> wait = new FluentWait<>(driver)
                .withTimeout(Duration.ofSeconds(30))
                .pollingEvery(Duration.ofMillis(500))
                .ignoring(NoSuchElementException.class)
                .ignoring(StaleElementReferenceException.class)
                .withMessage("Element never became visible: " + locator);

        return wait.until((Function<WebDriver, WebElement>) d -> {
            WebElement el = d.findElement(locator);
            return el.isDisplayed() ? el : null;
        });
    }
}
```

A `FluentWait` condition succeeds when the function returns non-`null` (or
`true`). Returning `null` means "not yet, poll again".

### Custom expected conditions

```java
import org.openqa.selenium.support.ui.ExpectedCondition;

public static ExpectedCondition<Boolean> spinnerGone(By spinner) {
    return driver -> driver.findElements(spinner).stream()
            .noneMatch(WebElement::isDisplayed);
}

// usage
new WebDriverWait(driver, Duration.ofSeconds(20)).until(spinnerGone(By.cssSelector(".loading")));
```

Wait for the *application's* signal that it is ready — a spinner
disappearing, a row count settling — not for a fixed number of seconds.

## 2. Alerts, confirms and prompts

JavaScript dialogs are browser chrome, not DOM. `findElement` cannot see them.

```java
package com.example.tests;

import org.junit.jupiter.api.*;
import org.openqa.selenium.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;

import java.time.Duration;

import static org.junit.jupiter.api.Assertions.*;

class AlertTest {

    private WebDriver driver;
    private WebDriverWait wait;

    @BeforeEach
    void setUp() {
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--headless=new", "--window-size=1920,1080");
        driver = new ChromeDriver(options);
        driver.manage().timeouts().implicitlyWait(Duration.ZERO);
        wait = new WebDriverWait(driver, Duration.ofSeconds(10));
        driver.get("https://the-internet.herokuapp.com/javascript_alerts");
    }

    @Test
    @DisplayName("Accepting a confirm dialog is reported on the page")
    void acceptConfirm() {
        driver.findElement(By.cssSelector("button[onclick='jsConfirm()']")).click();

        Alert alert = wait.until(ExpectedConditions.alertIsPresent());
        assertEquals("I am a JS Confirm", alert.getText());
        alert.accept();

        assertEquals("You clicked: Ok",
                driver.findElement(By.id("result")).getText());
    }

    @Test
    @DisplayName("A prompt returns the typed text")
    void answerPrompt() {
        driver.findElement(By.cssSelector("button[onclick='jsPrompt()']")).click();

        Alert alert = wait.until(ExpectedConditions.alertIsPresent());
        alert.sendKeys("QA Engineer");
        alert.accept();

        assertEquals("You entered: QA Engineer",
                driver.findElement(By.id("result")).getText());
    }

    @AfterEach
    void tearDown() {
        if (driver != null) driver.quit();
    }
}
```

```
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
```

| Method | Effect |
|---|---|
| `alert.accept()` | OK |
| `alert.dismiss()` | Cancel |
| `alert.getText()` | The dialog message |
| `alert.sendKeys(text)` | Types into a prompt only |

An unhandled alert makes the *next* command throw
`UnhandledAlertException` — a confusing failure far from its cause.

## 3. Frames

An iframe is a separate document. You must switch into it and back out.

```java
@Test
void typeInsideAnIframe() {
    driver.get("https://the-internet.herokuapp.com/iframe");

    driver.switchTo().frame("mce_0_ifr");            // by name or id
    WebElement body = driver.findElement(By.id("tinymce"));
    body.clear();
    body.sendKeys("Text typed inside the editor frame");
    assertEquals("Text typed inside the editor frame", body.getText());

    driver.switchTo().defaultContent();              // back to the main document
    assertTrue(driver.findElement(By.id("page-footer")).isDisplayed());
}
```

| Call | Switches to |
|---|---|
| `switchTo().frame("name-or-id")` | Named frame |
| `switchTo().frame(0)` | Frame by index |
| `switchTo().frame(webElement)` | Frame located as an element |
| `switchTo().parentFrame()` | One level up |
| `switchTo().defaultContent()` | The top document |

## 4. Windows and tabs

```java
@Test
void handleNewWindow() {
    driver.get("https://the-internet.herokuapp.com/windows");
    String original = driver.getWindowHandle();

    driver.findElement(By.linkText("Click Here")).click();
    wait.until(ExpectedConditions.numberOfWindowsToBe(2));

    for (String handle : driver.getWindowHandles()) {
        if (!handle.equals(original)) {
            driver.switchTo().window(handle);
            break;
        }
    }

    assertEquals("New Window", driver.getTitle());
    driver.close();                       // close the new window only
    driver.switchTo().window(original);   // ALWAYS switch back after close()
    assertEquals("The Internet", driver.getTitle());
}
```

Selenium 4 can also open one itself:

```java
driver.switchTo().newWindow(WindowType.TAB);
driver.switchTo().newWindow(WindowType.WINDOW);
```

!!! warning "After `close()` the driver points at a dead window"
    Every subsequent command throws `NoSuchWindowException` until you call
    `switchTo().window(...)`. Make "close then switch" a single habit.

## 5. The Actions API

For hovers, drag-and-drop, right-clicks, double-clicks and key combinations.

```java
import org.openqa.selenium.interactions.Actions;

@Test
void hoverRevealsCaption() {
    driver.get("https://the-internet.herokuapp.com/hovers");
    WebElement avatar = driver.findElement(By.cssSelector(".figure:nth-child(3) img"));

    new Actions(driver).moveToElement(avatar).perform();

    WebElement caption = driver.findElement(By.cssSelector(".figure:nth-child(3) h5"));
    assertTrue(caption.isDisplayed(), "Caption should appear on hover");
    assertEquals("name: user1", caption.getText());
}

@Test
void dragAndDrop() {
    driver.get("https://the-internet.herokuapp.com/drag_and_drop");
    WebElement columnA = driver.findElement(By.id("column-a"));
    WebElement columnB = driver.findElement(By.id("column-b"));

    new Actions(driver).dragAndDrop(columnA, columnB).perform();

    assertEquals("B", driver.findElement(By.id("column-a")).getText());
}
```

| Action | Code |
|---|---|
| Hover | `.moveToElement(el)` |
| Right-click | `.contextClick(el)` |
| Double-click | `.doubleClick(el)` |
| Drag and drop | `.dragAndDrop(src, target)` |
| Manual drag | `.clickAndHold(src).moveToElement(t).release()` |
| Ctrl+A | `.keyDown(Keys.CONTROL).sendKeys("a").keyUp(Keys.CONTROL)` |
| Scroll to element | `.scrollToElement(el)` (Selenium 4) |

!!! warning "HTML5 drag-and-drop often ignores `dragAndDrop`"
    Selenium's synthetic events do not always trigger HTML5 `dragstart`
    listeners. When `dragAndDrop` silently does nothing, fall back to
    `clickAndHold(...).moveByOffset(10, 0).moveToElement(target).release()`,
    or dispatch the drag events with a JavaScript snippet. This is a known
    limitation, not a bug in your test.

## 6. JavaScriptExecutor — the escape hatch

```java
import org.openqa.selenium.JavascriptExecutor;

JavascriptExecutor js = (JavascriptExecutor) driver;

js.executeScript("arguments[0].scrollIntoView({block:'center'});", element);
js.executeScript("window.scrollTo(0, document.body.scrollHeight);");
js.executeScript("arguments[0].click();", element);                    // last resort
js.executeScript("arguments[0].value='typed';", inputElement);         // last resort
String state = (String) js.executeScript("return document.readyState;");
```

!!! warning "A JS click is a test that no longer tests the user"
    `js.executeScript("arguments[0].click()")` fires the handler even when
    the element is invisible, disabled or covered by a modal — exactly the
    conditions a real user could not click through. Using it to "fix" a
    failing test converts a genuine bug into a green build. Use it only for
    scrolling and diagnostics; if a normal `click()` fails, find out why.

## 7. Other things you will meet

```java
// File upload -- send the absolute path to the <input type="file">
driver.findElement(By.id("file-upload"))
      .sendKeys(Path.of("src/test/resources/upload.txt").toAbsolutePath().toString());

// Shadow DOM (Selenium 4)
SearchContext shadow = driver.findElement(By.id("host")).getShadowRoot();
shadow.findElement(By.cssSelector(".inner")).click();

// Relative locators (Selenium 4)
import static org.openqa.selenium.support.locators.RelativeLocator.with;
driver.findElement(with(By.tagName("input")).below(By.id("email-label")));

// Element screenshot, not whole page
File png = driver.findElement(By.id("chart")).getScreenshotAs(OutputType.FILE);

// Cookies -- skip the login UI entirely
driver.manage().addCookie(new Cookie("session", "abc123"));
```

## 8. Testing traps

!!! warning "Trap 1 — `isDisplayed()` on a missing element throws"
    It throws `NoSuchElementException`, not `false`. Use
    `!driver.findElements(by).isEmpty()` to test for absence.

!!! warning "Trap 2 — stale elements after a re-render"
    A `WebElement` is a handle to one DOM node. Any framework re-render
    invalidates it. Re-find after every action that changes the page, and
    never hold an element across a navigation.

!!! warning "Trap 3 — window size changes what exists"
    Headless Chrome defaults to a small viewport where responsive sites hide
    the desktop nav entirely. Always set `--window-size=1920,1080` so headless
    and headed runs see the same DOM.

!!! warning "Trap 4 — waiting for presence when you need interactability"
    `presenceOfElementLocated` returns while the element is still hidden
    behind a fade-in, and the click then throws
    `ElementClickInterceptedException`. For clicks, wait for
    `elementToBeClickable`.

## Cheat sheet

| Task | Code |
|---|---|
| Disable implicit wait | `driver.manage().timeouts().implicitlyWait(Duration.ZERO)` |
| Explicit wait | `new WebDriverWait(driver, Duration.ofSeconds(10)).until(...)` |
| Fluent wait | `new FluentWait<>(driver).withTimeout(...).pollingEvery(...).ignoring(...)` |
| Alert | `wait.until(ExpectedConditions.alertIsPresent()).accept()` |
| Enter frame | `driver.switchTo().frame("id")` |
| Leave frame | `driver.switchTo().defaultContent()` |
| Switch window | `driver.switchTo().window(handle)` |
| New tab | `driver.switchTo().newWindow(WindowType.TAB)` |
| Hover | `new Actions(driver).moveToElement(el).perform()` |
| Drag | `new Actions(driver).dragAndDrop(a, b).perform()` |
| Scroll into view | `js.executeScript("arguments[0].scrollIntoView(true);", el)` |
| Upload | `input.sendKeys(absolutePath)` |
| Shadow root | `el.getShadowRoot()` |

## Exercise

All targets are on `https://the-internet.herokuapp.com/`.

1. Set the implicit wait to zero across your suite and fix everything that
   breaks with explicit waits. Record how many tests needed a change.
2. On `/dynamic_loading/2`, replace `WebDriverWait` with a `FluentWait`
   polling every 250 ms for 20 s, ignoring `NoSuchElementException`. Add a
   `withMessage(...)`, force a timeout, and confirm your message appears in
   the failure.
3. On `/javascript_alerts`, write three tests: accept an alert, dismiss a
   confirm, and answer a prompt — asserting `#result` each time.
4. On `/iframe`, type into the TinyMCE body, switch back out, and assert the
   footer is visible. Then deliberately omit `defaultContent()` and record
   the exception you get.
5. On `/windows`, open the new window, assert its title, close it, switch
   back, and assert the original title. Then omit the switch-back and record
   the exception.
6. On `/hovers`, assert that all three captions are hidden initially and that
   hovering the second one reveals `name: user2`.
7. On `/dynamic_controls`, click **Remove**, wait for the checkbox to
   disappear, assert the "It's gone!" message, then click **Add** and wait
   for it to return. Write a custom `ExpectedCondition` for one of the two
   directions.
8. Write two sentences on when a `JavascriptExecutor` click is legitimate,
   and one example where using it would have hidden a real accessibility bug.
