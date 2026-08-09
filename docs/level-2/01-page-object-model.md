# 01 · Page Object Model (POM)

Level 1 ended with a suite where every test knew `By.id("username")`. That
works for four tests. At forty tests, a single renamed field means forty
edits, and you will miss three of them. The Page Object Model fixes this by
giving every page of the application one Java class that owns its locators
and its behaviour — so a UI change becomes a one-file change.

## 1. The problem, made concrete

```java
// Test 1
driver.findElement(By.id("username")).sendKeys("tomsmith");
driver.findElement(By.id("password")).sendKeys("SuperSecretPassword!");
driver.findElement(By.cssSelector("button[type='submit']")).click();

// Test 2 ... and 3, and 4, and 40
driver.findElement(By.id("username")).sendKeys("wronguser");
driver.findElement(By.id("password")).sendKeys("SuperSecretPassword!");
driver.findElement(By.cssSelector("button[type='submit']")).click();
```

Three defects live in that code: the locator is duplicated, the *intent*
("log in") is buried in mechanics, and a reader cannot tell what is being
tested without decoding CSS.

## 2. The rules of a page object

| Rule | Why |
|---|---|
| One class per page (or per significant component) | The class maps to something the user recognises |
| Locators are `private` fields | Tests must never see a `By` |
| Methods are named in the user's language — `login()`, `openCart()` | Tests read as scenarios, not scripts |
| A method that navigates away returns the **next** page object | The compiler documents the flow |
| **No assertions inside a page object** | Pages describe the app; tests decide what is correct |
| Expose state via getters — `getErrorMessage()`, `isLogoutVisible()` | Tests assert on returned values |

That fifth rule is the one people break. A page object with assertions in it
cannot be reused by a negative test that expects the opposite.

## 3. A base page

Every page needs a driver and a wait. Put that once in a superclass.

```java
package com.example.pages;

import org.openqa.selenium.*;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;

import java.time.Duration;

public abstract class BasePage {

    protected final WebDriver driver;
    protected final WebDriverWait wait;

    protected BasePage(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    protected WebElement visible(By locator) {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(locator));
    }

    protected WebElement clickable(By locator) {
        return wait.until(ExpectedConditions.elementToBeClickable(locator));
    }

    protected void type(By locator, String text) {
        WebElement field = visible(locator);
        field.clear();
        field.sendKeys(text);
    }

    protected boolean isPresent(By locator) {
        return !driver.findElements(locator).isEmpty();
    }

    public String getPageTitle() {
        return driver.getTitle();
    }
}
```

The wait is baked into `visible()` and `clickable()`. Every page object
inherits correct waiting for free — the single biggest reduction in flakiness
a framework can deliver.

## 4. Two page objects

```java
package com.example.pages;

import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;

public class LoginPage extends BasePage {

    private static final String URL = "https://the-internet.herokuapp.com/login";

    private final By usernameField = By.id("username");
    private final By passwordField = By.id("password");
    private final By loginButton   = By.cssSelector("button[type='submit']");
    private final By flashMessage  = By.id("flash");

    public LoginPage(WebDriver driver) {
        super(driver);
    }

    public LoginPage open() {
        driver.get(URL);
        visible(usernameField);
        return this;
    }

    /** Submits credentials and returns the page the user lands on. */
    public SecurePage loginAs(String username, String password) {
        type(usernameField, username);
        type(passwordField, password);
        clickable(loginButton).click();
        return new SecurePage(driver);
    }

    /** For negative cases: submit, stay here, and read the error. */
    public LoginPage loginExpectingFailure(String username, String password) {
        type(usernameField, username);
        type(passwordField, password);
        clickable(loginButton).click();
        return this;
    }

    public String getFlashMessage() {
        return visible(flashMessage).getText().replace("×", "").trim();
    }

    public boolean isPasswordMasked() {
        return "password".equals(visible(passwordField).getAttribute("type"));
    }
}
```

```java
package com.example.pages;

import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;

public class SecurePage extends BasePage {

    private final By flashMessage = By.id("flash");
    private final By logoutButton = By.linkText("Logout");
    private final By heading      = By.tagName("h2");

    public SecurePage(WebDriver driver) {
        super(driver);
    }

    public String getFlashMessage() {
        return visible(flashMessage).getText().replace("×", "").trim();
    }

    public String getHeading() {
        return visible(heading).getText();
    }

    public boolean isLogoutVisible() {
        return isPresent(logoutButton) && visible(logoutButton).isDisplayed();
    }

    public LoginPage logout() {
        clickable(logoutButton).click();
        return new LoginPage(driver);
    }
}
```

## 5. The tests that use them

```java
package com.example.tests;

import com.example.pages.LoginPage;
import com.example.pages.SecurePage;
import org.junit.jupiter.api.*;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;

import static org.junit.jupiter.api.Assertions.*;

class LoginPageObjectTest {

    private WebDriver driver;
    private LoginPage loginPage;

    @BeforeEach
    void setUp() {
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--headless=new", "--window-size=1920,1080");
        driver = new ChromeDriver(options);
        loginPage = new LoginPage(driver).open();
    }

    @Test
    @DisplayName("Valid credentials reach the secure area")
    void validLogin() {
        SecurePage secure = loginPage.loginAs("tomsmith", "SuperSecretPassword!");

        assertAll("successful login",
                () -> assertTrue(secure.getFlashMessage().contains("You logged into a secure area")),
                () -> assertEquals("Secure Area", secure.getHeading()),
                () -> assertTrue(secure.isLogoutVisible(), "Logout button should be visible"));
    }

    @Test
    @DisplayName("An unknown username is rejected")
    void invalidUsername() {
        String flash = loginPage
                .loginExpectingFailure("nosuchuser", "SuperSecretPassword!")
                .getFlashMessage();

        assertTrue(flash.contains("Your username is invalid"),
                "Expected an invalid-username error but got: " + flash);
    }

    @Test
    @DisplayName("Logout returns the user to the login page")
    void logoutRoundTrip() {
        String flash = loginPage
                .loginAs("tomsmith", "SuperSecretPassword!")
                .logout()
                .getFlashMessage();

        assertTrue(flash.contains("You logged out of the secure area"));
    }

    @AfterEach
    void tearDown() {
        if (driver != null) driver.quit();
    }
}
```

```
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

Read `logoutRoundTrip` aloud: "log in as tomsmith, log out, read the flash
message." That is a Level 1 Module 02 test case, expressed in Java. No
locator in sight.

## 6. `@FindBy` and PageFactory — and why it is optional

Selenium ships an annotation-driven variant:

```java
import org.openqa.selenium.support.FindBy;
import org.openqa.selenium.support.PageFactory;

public class LoginPageFactory {

    @FindBy(id = "username")
    private WebElement username;

    @FindBy(css = "button[type='submit']")
    private WebElement loginButton;

    public LoginPageFactory(WebDriver driver) {
        PageFactory.initElements(driver, this);   // wires the proxies
    }
}
```

!!! warning "PageFactory is not a wait"
    `@FindBy` fields are lazy proxies re-located on each use, which helps with
    staleness but does **nothing** about timing — you still need explicit
    waits. Selenium's own documentation now discourages PageFactory, and
    `AjaxElementLocatorFactory` (its "implicit wait" variant) interacts badly
    with `WebDriverWait`. Plain `By` constants, as in section 4, are the
    current recommendation. Learn `@FindBy` because you will inherit
    codebases that use it — do not start a new framework with it.

## 7. Testing traps

!!! warning "Trap 1 — assertions inside page objects"
    A `login()` that asserts success cannot be reused by the four negative
    tests that need it to fail. Page objects report; tests judge.

!!! warning "Trap 2 — returning `void` from a navigating method"
    If `loginAs()` returns nothing, the test must construct `SecurePage`
    itself, and nothing stops it doing so when login actually failed.
    Returning the next page object makes the flow type-checked.

!!! warning "Trap 3 — caching `WebElement` fields"
    `private WebElement loginButton = driver.findElement(...)` in a
    constructor produces `StaleElementReferenceException` after any
    re-render. Store the `By`; find the element at the moment of use.

!!! warning "Trap 4 — the god page object"
    A 900-line `HomePage` with methods for the header, the nav, the footer
    and the cart is not a page object, it is a junk drawer. Split shared
    regions into **component objects** (`HeaderComponent`, `CartWidget`) that
    take the driver the same way and are exposed from the page.

!!! warning "Trap 5 — leaking `WebDriver` out of the page"
    A `getDriver()` getter invites tests to reach around the abstraction.
    Within a week half the suite calls `page.getDriver().findElement(...)`
    and you are back where you started.

## Cheat sheet

| Concern | Convention |
|---|---|
| One class per | Page or reusable component |
| Locator visibility | `private final By` |
| Shared driver/wait | `BasePage` superclass |
| Navigation method | Returns the next page object |
| Same-page action | Returns `this` (fluent chaining) |
| State exposure | `getErrorMessage()`, `isLogoutVisible()` |
| Assertions | In the test class only |
| Waiting | Inside `BasePage` helpers, never in tests |
| Element storage | Store `By`, never a `WebElement` field |
| Package layout | `com.example.pages` / `com.example.tests` |

## Exercise

Work against `https://the-internet.herokuapp.com/`.

1. Build `BasePage`, `LoginPage` and `SecurePage` exactly as above in a
   `com.example.pages` package, and port your Level 1 login tests onto them.
   Delete every `By` from the test classes.
2. Add `isPasswordMasked()` coverage: a test asserting the password field has
   `type="password"` (REQ-08 from the Level 1 project).
3. Add a `DropdownPage` for `/dropdown` with `selectByText(String)`,
   `getSelectedOption()` and `getOptionCount()`. Write three tests using it,
   and note how much shorter they are than the Level 1 versions.
4. Extract a `FlashMessageComponent` that owns the `#flash` locator and the
   `×`-stripping logic. Have both `LoginPage` and `SecurePage` delegate to
   it. Explain in one sentence why this beats the duplicated
   `getFlashMessage()` in section 4.
5. Break the app on purpose: change `usernameField` to `By.id("user")`. Run
   the suite, confirm that **one file** needed fixing, and time the fix
   against the same change in the Level 1 suite.
6. Rewrite `LoginPage` a second time using `@FindBy` and `PageFactory`, then
   write three sentences comparing the two — which one makes the waiting
   strategy more obvious to a new reader?
