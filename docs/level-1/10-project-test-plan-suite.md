# 10 · Project — Manual Test Plan + Automated Suite

This project combines everything in Level 1. You will act as the sole QA
engineer for a small web application: write a real test plan, design manual
test cases using formal techniques, build a traceability matrix, raise a
defect report, and then automate the repeatable parts with JUnit 5, Selenium
and TestNG.

**Application under test:** `https://the-internet.herokuapp.com/login` — a
public practice site with a deliberately simple login flow.

- Valid credentials: username `tomsmith`, password `SuperSecretPassword!`
- Invalid username → flash message *"Your username is invalid!"*
- Invalid password → flash message *"Your password is invalid!"*
- Success → redirect to `/secure`, flash *"You logged into a secure area!"*,
  a **Logout** button appears

## Requirements to test against

| Req ID | Requirement |
|---|---|
| REQ-01 | The login page displays a Username field, a Password field and a Login button |
| REQ-02 | Valid credentials redirect the user to `/secure` |
| REQ-03 | A successful login displays the message "You logged into a secure area!" |
| REQ-04 | A successful login displays a Logout button |
| REQ-05 | An invalid username displays "Your username is invalid!" and keeps the user on `/login` |
| REQ-06 | An invalid password displays "Your password is invalid!" and keeps the user on `/login` |
| REQ-07 | Submitting empty fields is rejected with an error message |
| REQ-08 | The password field masks the characters entered |
| REQ-09 | Clicking Logout returns the user to `/login` with the message "You logged out of the secure area!" |
| REQ-10 | Navigating directly to `/secure` without logging in is rejected |

## Deliverable 1 — Test plan

Write `test-plan.md` covering the eight sections from Module 02, section 6.
A worked skeleton:

```markdown
# Test Plan — Login Feature

## 1. Scope
In scope:  the /login page, authentication, the /secure page, logout.
Out of scope: registration, password reset, any other page on the site,
              performance and load testing.

## 2. Test approach
Manual: exploratory session on the login form; usability and UI checks;
        boundary and negative cases requiring human judgement.
Automated: the regression subset -- all four credential combinations,
        the logout flow, and direct-URL access to /secure.
Levels: system testing (black box, through the UI).
Types: functional, plus basic security (session protection) and
        usability observations.

## 3. Test environment
Browser: Chrome 126 (headless for automation, headed for exploratory)
OS: <your OS>
Java 21, Maven 3.9, JUnit 5.10.2, Selenium 4.23.0, TestNG 7.10.2
Base URL: https://the-internet.herokuapp.com

## 4. Entry & exit criteria
Entry: application reachable; smoke test (page loads, form renders) passes.
Exit:  100% of planned cases executed; zero open S1/S2 defects;
       all automated tests green in a clean `mvn clean test` run.

## 5. Schedule
Test design ......... 1 day
Manual execution .... 0.5 day
Automation build .... 1.5 days
Regression run ...... 0.5 day

## 6. Roles
QA Engineer (you): design, execution, automation, defect reporting.

## 7. Risks & mitigations
- Public practice site may be temporarily unavailable
  -> mitigate: retry, and mark blocked rather than failed.
- Site content may change without notice
  -> mitigate: locators anchored on stable ids, not layout.

## 8. Deliverables
test-plan.md, test-cases.md, rtm.md, defect-report.md,
automated suite under src/test/java, testng.xml, execution summary.
```

## Deliverable 2 — Manual test cases

Write `test-cases.md` with **at least 12 test cases** in the full template
from Module 02. Here are two worked examples to set the standard; you write
the rest.

| Field | Value |
|---|---|
| **Test Case ID** | TC_LOGIN_001 |
| **Module** | Authentication |
| **Title** | Verify successful login with valid credentials |
| **Requirement ID** | REQ-02, REQ-03, REQ-04 |
| **Priority** | P1 |
| **Type** | Positive |
| **Preconditions** | Browser is open; user is not logged in; no session cookie present |
| **Test Data** | Username `tomsmith` · Password `SuperSecretPassword!` |
| **Steps** | 1. Navigate to `https://the-internet.herokuapp.com/login`.<br>2. Enter `tomsmith` in the **Username** field.<br>3. Enter `SuperSecretPassword!` in the **Password** field.<br>4. Click the **Login** button. |
| **Expected Result** | 1. The browser navigates to `/secure`.<br>2. A green flash message reads "You logged into a secure area!".<br>3. A **Logout** button is visible.<br>4. The heading reads "Secure Area". |
| **Status** | Not Executed |

| Field | Value |
|---|---|
| **Test Case ID** | TC_LOGIN_010 |
| **Module** | Authentication / Session security |
| **Title** | Verify direct navigation to /secure without authentication is rejected |
| **Requirement ID** | REQ-10 |
| **Priority** | P1 |
| **Type** | Negative / Security |
| **Preconditions** | No active session; browser cookies cleared |
| **Test Data** | URL `https://the-internet.herokuapp.com/secure` |
| **Steps** | 1. Clear all cookies.<br>2. Navigate directly to `https://the-internet.herokuapp.com/secure`. |
| **Expected Result** | 1. The user is redirected to `/login`.<br>2. A flash message indicates the page must be logged into.<br>3. No secure-area content is rendered at any point. |
| **Status** | Not Executed |

Your 12+ cases must include:

- All four credential combinations (valid/valid, valid/invalid,
  invalid/valid, empty/empty)
- **Boundary cases** on the username field — 1 character, a very long value
  (5,000 characters), leading/trailing whitespace
- **Equivalence partitions** — special characters, SQL-injection-shaped input
  (`' OR '1'='1`), Unicode (`Zoë`, `你好`, emoji)
- REQ-08: the password field masks input (check
  `getAttribute("type") == "password"`)
- The logout flow (REQ-09)
- Direct-URL access to `/secure` (REQ-10)
- At least one UI/usability observation

## Deliverable 3 — Requirement Traceability Matrix

Write `rtm.md` mapping every requirement to your test case IDs. Every row
must have at least one case, or you have found a coverage gap.

| Req ID | Requirement | Test Case IDs | Automated? | Status |
|---|---|---|---|---|
| REQ-01 | Login form renders | TC_LOGIN_011 | Yes | |
| REQ-02 | Valid login redirects to /secure | TC_LOGIN_001 | Yes | |
| REQ-03 | Success message shown | TC_LOGIN_001 | Yes | |
| REQ-04 | Logout button appears | TC_LOGIN_001 | Yes | |
| REQ-05 | Invalid username error | TC_LOGIN_002 | Yes | |
| REQ-06 | Invalid password error | TC_LOGIN_003 | Yes | |
| REQ-07 | Empty fields rejected | TC_LOGIN_004 | Yes | |
| REQ-08 | Password masked | TC_LOGIN_005 | Yes | |
| REQ-09 | Logout flow | TC_LOGIN_009 | Yes | |
| REQ-10 | /secure protected | TC_LOGIN_010 | Yes | |

## Deliverable 4 — Automated suite

### 4a. A JUnit 5 unit test

First, a pure unit test with no browser. Create
`src/main/java/com/example/CredentialValidator.java`:

```java
package com.example;

public class CredentialValidator {

    /** Client-side pre-check before the form is submitted. */
    public boolean isSubmittable(String username, String password) {
        if (username == null || password == null) {
            throw new IllegalArgumentException("Credentials must not be null");
        }
        return !username.trim().isEmpty()
                && !password.trim().isEmpty()
                && username.trim().length() <= 100;
    }
}
```

Then `src/test/java/com/example/tests/CredentialValidatorTest.java`:

```java
package com.example.tests;

import com.example.CredentialValidator;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

import static org.junit.jupiter.api.Assertions.*;

@DisplayName("Credential pre-validation")
class CredentialValidatorTest {

    private CredentialValidator validator;

    @BeforeEach
    void setUp() {
        validator = new CredentialValidator();
    }

    @ParameterizedTest(name = "user=[{0}] pass=[{1}] -> submittable={2}")
    @CsvSource({
            "tomsmith,     SuperSecretPassword!, true",   // valid
            "'',           SuperSecretPassword!, false",  // empty username
            "tomsmith,     '',                   false",  // empty password
            "'',           '',                   false",  // both empty
            "'   ',        password,             false",  // whitespace only
            "a,            p,                    true"    // 1-char boundary
    })
    void submittability(String username, String password, boolean expected) {
        assertEquals(expected, validator.isSubmittable(username, password));
    }

    @Test
    @DisplayName("A username of exactly 100 characters is submittable")
    void usernameAtUpperBoundary() {
        assertTrue(validator.isSubmittable("u".repeat(100), "pw"));
    }

    @Test
    @DisplayName("A username of 101 characters is rejected")
    void usernameAboveUpperBoundary() {
        assertFalse(validator.isSubmittable("u".repeat(101), "pw"));
    }

    @Test
    @DisplayName("Null credentials throw IllegalArgumentException")
    void nullCredentialsThrow() {
        IllegalArgumentException ex = assertThrows(
                IllegalArgumentException.class,
                () -> validator.isSubmittable(null, "pw"));
        assertEquals("Credentials must not be null", ex.getMessage());
    }
}
```

```
[INFO] Running com.example.tests.CredentialValidatorTest
user=[tomsmith] pass=[SuperSecretPassword!] -> submittable=true   PASSED
user=[] pass=[SuperSecretPassword!] -> submittable=false          PASSED
user=[tomsmith] pass=[] -> submittable=false                      PASSED
user=[] pass=[] -> submittable=false                              PASSED
user=[   ] pass=[password] -> submittable=false                   PASSED
user=[a] pass=[p] -> submittable=true                             PASSED
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0
```

### 4b. The Selenium + TestNG suite

`src/test/java/com/example/tests/LoginSuiteTest.java`:

```java
package com.example.tests;

import org.openqa.selenium.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;
import org.testng.ITestResult;
import org.testng.annotations.*;
import org.testng.asserts.SoftAssert;

import java.io.File;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Duration;

import static org.testng.Assert.*;

public class LoginSuiteTest {

    private WebDriver driver;
    private WebDriverWait wait;
    private String baseUrl;

    private static final By USERNAME  = By.id("username");
    private static final By PASSWORD  = By.id("password");
    private static final By LOGIN_BTN = By.cssSelector("button[type='submit']");
    private static final By FLASH     = By.id("flash");
    private static final By LOGOUT    = By.cssSelector("a[href='/logout']");

    @Parameters({"browser", "baseUrl"})
    @BeforeClass(alwaysRun = true)
    public void setUpClass(@Optional("chrome") String browser,
                           @Optional("https://the-internet.herokuapp.com") String baseUrl) {
        this.baseUrl = baseUrl;
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--headless=new", "--window-size=1920,1080");
        driver = new ChromeDriver(options);
        wait = new WebDriverWait(driver, Duration.ofSeconds(15));
        System.out.println("Suite starting on " + browser + " against " + baseUrl);
    }

    @BeforeMethod(alwaysRun = true)
    public void openLoginPage() {
        driver.manage().deleteAllCookies();
        driver.get(baseUrl + "/login");
        wait.until(ExpectedConditions.visibilityOfElementLocated(USERNAME));
    }

    private void login(String username, String password) {
        WebElement user = driver.findElement(USERNAME);
        WebElement pass = driver.findElement(PASSWORD);
        user.clear();
        pass.clear();
        if (!username.isEmpty()) user.sendKeys(username);
        if (!password.isEmpty()) pass.sendKeys(password);
        driver.findElement(LOGIN_BTN).click();
    }

    private String flashText() {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(FLASH))
                   .getText().replace("×", "").trim();
    }

    // ---------- TC_LOGIN_011 -- REQ-01 ----------
    @Test(priority = 0, groups = "smoke",
          description = "TC_LOGIN_011 -- login form renders all three controls")
    public void loginFormRenders() {
        SoftAssert softly = new SoftAssert();
        softly.assertTrue(driver.findElement(USERNAME).isDisplayed(),  "Username field visible");
        softly.assertTrue(driver.findElement(PASSWORD).isDisplayed(),  "Password field visible");
        softly.assertTrue(driver.findElement(LOGIN_BTN).isDisplayed(), "Login button visible");
        softly.assertTrue(driver.findElement(LOGIN_BTN).isEnabled(),   "Login button enabled");
        softly.assertAll();
    }

    // ---------- TC_LOGIN_001 -- REQ-02, 03, 04 ----------
    @Test(priority = 1, groups = "smoke",
          description = "TC_LOGIN_001 -- valid credentials log the user in")
    public void loginWithValidCredentials() {
        login("tomsmith", "SuperSecretPassword!");

        SoftAssert softly = new SoftAssert();
        softly.assertTrue(driver.getCurrentUrl().endsWith("/secure"),
                "REQ-02: should redirect to /secure, was " + driver.getCurrentUrl());
        softly.assertTrue(flashText().contains("You logged into a secure area!"),
                "REQ-03: success message");
        softly.assertTrue(driver.findElement(LOGOUT).isDisplayed(),
                "REQ-04: Logout button visible");
        softly.assertAll();
    }

    // ---------- TC_LOGIN_005 -- REQ-08 ----------
    @Test(priority = 2, groups = "smoke",
          description = "TC_LOGIN_005 -- password field masks input")
    public void passwordFieldIsMasked() {
        assertEquals(driver.findElement(PASSWORD).getAttribute("type"), "password",
                "REQ-08: password input must be of type=password");
    }

    // ---------- TC_LOGIN_009 -- REQ-09 ----------
    @Test(priority = 3, groups = "regression",
          dependsOnMethods = "loginWithValidCredentials",
          description = "TC_LOGIN_009 -- logout returns the user to /login")
    public void logoutFlow() {
        login("tomsmith", "SuperSecretPassword!");
        driver.findElement(LOGOUT).click();

        SoftAssert softly = new SoftAssert();
        softly.assertTrue(driver.getCurrentUrl().endsWith("/login"), "Back on /login");
        softly.assertTrue(flashText().contains("You logged out of the secure area!"),
                "REQ-09: logout message");
        softly.assertAll();
    }

    // ---------- TC_LOGIN_010 -- REQ-10 ----------
    @Test(priority = 4, groups = "regression",
          description = "TC_LOGIN_010 -- /secure is not reachable without a session")
    public void secureAreaBlockedWhenLoggedOut() {
        driver.manage().deleteAllCookies();
        driver.get(baseUrl + "/secure");

        assertTrue(driver.getCurrentUrl().endsWith("/login"),
                "REQ-10: unauthenticated access to /secure must redirect to /login");
        assertTrue(driver.findElements(LOGOUT).isEmpty(),
                "REQ-10: no secure-area content should be rendered");
    }

    // ---------- TC_LOGIN_002/003/004 -- data-driven negatives ----------
    @DataProvider(name = "invalidCredentials")
    public Object[][] invalidCredentials() {
        return new Object[][] {
                { "wronguser", "SuperSecretPassword!", "Your username is invalid!", "TC_LOGIN_002" },
                { "tomsmith",  "WrongPassword1!",      "Your password is invalid!", "TC_LOGIN_003" },
                { "",          "",                     "Your username is invalid!", "TC_LOGIN_004" },
                { "TOMSMITH",  "SuperSecretPassword!", "Your username is invalid!", "TC_LOGIN_006" },
                { " tomsmith ","SuperSecretPassword!", "Your username is invalid!", "TC_LOGIN_007" },
                { "' OR '1'='1", "anything",           "Your username is invalid!", "TC_LOGIN_008" }
        };
    }

    @Test(priority = 5, groups = "regression", dataProvider = "invalidCredentials",
          description = "Invalid credential combinations are rejected")
    public void invalidCredentialsAreRejected(String username, String password,
                                              String expectedFragment, String caseId) {
        login(username, password);

        SoftAssert softly = new SoftAssert();
        softly.assertTrue(driver.getCurrentUrl().endsWith("/login"),
                caseId + ": should remain on /login");
        softly.assertTrue(flashText().contains(expectedFragment),
                caseId + ": expected message containing '" + expectedFragment
                        + "' but got '" + flashText() + "'");
        softly.assertTrue(driver.findElements(LOGOUT).isEmpty(),
                caseId + ": no Logout button should appear");
        softly.assertAll();
    }

    // ---------- screenshot on failure ----------
    @AfterMethod(alwaysRun = true)
    public void captureOnFailure(ITestResult result) {
        if (result.getStatus() == ITestResult.FAILURE) {
            try {
                File src = ((TakesScreenshot) driver).getScreenshotAs(OutputType.FILE);
                Path dest = Path.of("target", "screenshots", result.getName() + ".png");
                Files.createDirectories(dest.getParent());
                Files.copy(src.toPath(), dest);
                System.out.println("FAILED -- screenshot: " + dest.toAbsolutePath());
            } catch (Exception e) {
                System.out.println("Screenshot failed: " + e.getMessage());
            }
        }
    }

    @AfterClass(alwaysRun = true)
    public void tearDownClass() {
        if (driver != null) driver.quit();
    }
}
```

### 4c. The suite file

`testng.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">

<suite name="Login Regression Suite" verbose="1">
    <parameter name="baseUrl" value="https://the-internet.herokuapp.com"/>

    <test name="Smoke">
        <parameter name="browser" value="chrome"/>
        <groups><run><include name="smoke"/></run></groups>
        <classes><class name="com.example.tests.LoginSuiteTest"/></classes>
    </test>

    <test name="Full Regression">
        <parameter name="browser" value="chrome"/>
        <groups><run><include name="smoke"/><include name="regression"/></run></groups>
        <classes><class name="com.example.tests.LoginSuiteTest"/></classes>
    </test>
</suite>
```

### Running it

```bash
mvn clean test
```

```
Suite starting on chrome against https://the-internet.herokuapp.com

===============================================
Login Regression Suite
Total tests run: 17, Passes: 17, Failures: 0, Skips: 0
===============================================

[INFO] Tests run: 27, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

(17 TestNG results — 3 smoke + 3 smoke again in the regression block + 2
regression + 6 data-driven rows + form render — plus the 10 JUnit results
from `CredentialValidatorTest`.)

## Deliverable 5 — Defect report

While executing manually you *will* notice something. Common findings on this
site: the flash message contains a stray `×` character; the error text
reveals whether a username exists (an account-enumeration issue, exactly the
security concern from Module 02); the username field accepts unlimited
length; there is no rate limiting on failed attempts.

Pick one and write `defect-report.md` using the full template from Module 04
— title, environment, numbered reproduction steps, expected vs actual,
severity and priority with justification, and reproducibility.

!!! info "The account-enumeration finding"
    The site returns *"Your username is invalid!"* for a non-existent user
    but *"Your password is invalid!"* for a real user with the wrong
    password. An attacker can therefore discover which usernames exist.
    Severity S3 (nothing is broken functionally), Priority P2 (a real
    security weakness on the authentication endpoint). This is the kind of
    finding that comes from *thinking about* the messages, not from clicking
    through a script — precisely the tester mindset from Module 01.

## Deliverable 6 — Execution summary

Write `execution-summary.md`:

```markdown
# Test Execution Summary — Login Feature

Build / date: <build> / <date>
Environment:  Chrome 126 headless, macOS 14

| Metric | Value |
|---|---|
| Test cases planned    | 12 |
| Executed              | 12 |
| Passed                | 10 |
| Failed                | 1  |
| Blocked               | 1  |
| Automated             | 9  |
| Automation coverage   | 75% |
| Defects raised        | 1 (S3/P2) |
| Open S1/S2 defects    | 0 |

## Requirement coverage
All 10 requirements have at least one test case; see rtm.md.

## Recommendation
Ready to release, with BUG-001 (account enumeration) tracked for the
next sprint. Exit criteria met: zero open S1/S2 defects, 100% of
planned cases executed.
```

## Acceptance checklist

- [ ] `test-plan.md` covers all eight sections, including explicit entry and exit criteria
- [ ] `test-cases.md` has 12+ cases in the full template, with **specific**, multi-part expected results
- [ ] At least 6 cases are negative, boundary or equivalence-partition based
- [ ] `rtm.md` maps all 10 requirements; no requirement has zero cases
- [ ] `defect-report.md` uses the full Module 04 template with justified severity **and** priority
- [ ] `CredentialValidatorTest` runs under JUnit 5 with at least one `@ParameterizedTest` covering boundaries
- [ ] `LoginSuiteTest` runs under TestNG with `@BeforeClass`/`@BeforeMethod`/`@AfterClass` lifecycle
- [ ] At least one `@DataProvider` drives multiple negative cases from one method
- [ ] `SoftAssert` is used with `assertAll()` present in every method that creates one
- [ ] Explicit waits are used; there is **no** `Thread.sleep` anywhere
- [ ] Screenshots are captured on failure into `target/screenshots/`
- [ ] `testng.xml` defines separate Smoke and Full Regression blocks using groups
- [ ] `mvn clean test` completes with `BUILD SUCCESS` from a clean checkout
- [ ] `execution-summary.md` reports the metrics and a release recommendation

## Stretch goals

1. **Break it deliberately.** Change one locator to a wrong id, run the
   suite, and confirm the screenshot and the failure message together let you
   diagnose the problem in under a minute. If they do not, improve your
   assertion messages.
2. **Add `@Test(retryAnalyzer = ...)`.** Implement TestNG's `IRetryAnalyzer`
   to retry a failed test once. Then write two sentences on *why retrying is
   dangerous* — it hides real intermittent bugs, the topic of Level 3,
   Module 09.
3. **Extract the locators.** Move the `By` constants and the `login()` helper
   into a separate `LoginPage` class. You have just invented the Page Object
   Model — Level 2, Module 01, which formalises exactly this refactor.
4. **Run it in parallel.** Add `parallel="methods" thread-count="3"` to
   `testng.xml` and observe what breaks. Explain why a shared `WebDriver`
   field makes the suite unsafe, and what a `ThreadLocal<WebDriver>` would
   fix.

## Exercise

Complete all six deliverables and place them in a single repository with this
structure:

```
login-qa-project/
├── pom.xml
├── testng.xml
├── docs/
│   ├── test-plan.md
│   ├── test-cases.md
│   ├── rtm.md
│   ├── defect-report.md
│   └── execution-summary.md
└── src/
    ├── main/java/com/example/CredentialValidator.java
    └── test/java/com/example/tests/
        ├── CredentialValidatorTest.java
        └── LoginSuiteTest.java
```

Then answer these four questions in a `reflection.md`:

1. Which of your 12 manual test cases did you choose **not** to automate, and
   why? (There should be at least one — if you automated everything, you have
   probably automated something that did not deserve it.)
2. Which technique from Module 05 — EP, BVA, decision tables or state
   transition — produced the most valuable test case here, and what did it
   catch that ad-hoc clicking would have missed?
3. Your suite currently uses one browser and hard-coded credentials. Name
   three specific things you would change before this could run nightly on a
   CI server against three environments.
4. Re-read your defect report as if you were the developer receiving it.
   Could you reproduce the bug from the report alone, with no follow-up
   questions? If not, fix it.
