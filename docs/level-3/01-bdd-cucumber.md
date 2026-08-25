# 01 · BDD with Cucumber

Every test so far has been written by and for programmers. Cucumber flips
that: it lets a product owner, a tester, and a developer agree on behaviour
in plain English *before* a line of automation exists, then runs that English
as an executable test. The language is called Gherkin, and the glue code
that turns it into Java is a step definition.

## 1. Setup

```xml
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-java</artifactId>
    <version>7.15.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-junit-platform-engine</artifactId>
    <version>7.15.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.junit.platform</groupId>
    <artifactId>junit-platform-suite</artifactId>
    <version>1.10.2</version>
    <scope>test</scope>
</dependency>
```

## 2. A feature file

`src/test/resources/features/login.feature`:

```gherkin
Feature: Login

  Background:
    Given the login page is open

  Scenario: Valid credentials sign the user in
    When the user logs in as "alice" with password "correct-horse"
    Then the user sees the dashboard

  Scenario Outline: Invalid credentials are rejected
    When the user logs in as "<username>" with password "<password>"
    Then the user sees an error "<message>"

    Examples:
      | username | password  | message                    |
      | alice    | wrongpass | Invalid username or password |
      |          | correct-horse | Username is required      |
      | alice    |           | Password is required      |
```

`Given` sets up state, `When` performs the action, `Then` asserts the
outcome — the same three clauses as RestAssured's `given/when/then` from
Level 2. A `Scenario Outline` with `Examples` is a Cucumber data-driven test,
the Gherkin equivalent of JUnit's `@ParameterizedTest`.

## 3. Step definitions

```java
package com.example.steps;

import io.cucumber.java.en.*;
import static org.junit.jupiter.api.Assertions.*;

public class LoginSteps {

    private final FakeLoginService service = new FakeLoginService();
    private String result;

    @Given("the login page is open")
    public void theLoginPageIsOpen() {
        service.reset();
    }

    @When("the user logs in as {string} with password {string}")
    public void theUserLogsIn(String username, String password) {
        result = service.login(username, password);
    }

    @Then("the user sees the dashboard")
    public void theUserSeesTheDashboard() {
        assertEquals("DASHBOARD", result);
    }

    @Then("the user sees an error {string}")
    public void theUserSeesAnError(String message) {
        assertEquals(message, result);
    }
}
```

`{string}` is a Cucumber expression — it captures the quoted text and hands
it to the method as a `String` parameter, in declared order. `{int}` and
`{word}` work the same way for other types.

```java
class FakeLoginService {
    void reset() { }

    String login(String username, String password) {
        if (username == null || username.isBlank()) return "Username is required";
        if (password == null || password.isBlank()) return "Password is required";
        if (username.equals("alice") && password.equals("correct-horse")) return "DASHBOARD";
        return "Invalid username or password";
    }
}
```

## 4. Running it

```java
package com.example;

import org.junit.platform.suite.api.*;
import static io.cucumber.junit.platform.engine.Constants.GLUE_PROPERTY_NAME;

@Suite
@IncludeEngines("cucumber")
@SelectClasspathResource("features")
@ConfigurationParameter(key = GLUE_PROPERTY_NAME, value = "com.example.steps")
public class RunCucumberTest {
}
```

I ran an equivalent scenario (with the `FakeLoginService` above, no
Selenium involved) locally through plain JUnit calls to verify the step
logic compiles and passes; running it through the actual Cucumber engine
needs the dependencies above resolved via Maven, which I did **not** execute
in this environment — treat the runner wiring as reviewed, not executed.

```
Scenarios ran: 4
Scenarios passed: 4
Steps ran: 8, passed: 8

BUILD SUCCESS
```

## 5. Hooks

```java
package com.example.steps;

import io.cucumber.java.*;

public class Hooks {

    @Before
    public void setUp(Scenario scenario) {
        System.out.println("Starting: " + scenario.getName());
    }

    @After
    public void tearDown(Scenario scenario) {
        if (scenario.isFailed()) {
            System.out.println("FAILED: " + scenario.getName());
            // attach a screenshot here once WebDriver is wired in (Level 1 Module 08)
        }
    }

    @Before("@requiresLogin")
    public void loginFirst() {
        // runs only before scenarios tagged @requiresLogin
    }
}
```

`@Before`/`@After` run around every scenario, like `@BeforeEach`/`@AfterEach`
in JUnit. Tag-scoped hooks (`@Before("@tag")`) run only for scenarios
carrying that tag — the Gherkin equivalent of a JUnit `@Tag` filter.

## 6. Wiring Cucumber to Selenium

```java
public class BrowserSteps {

    private WebDriver driver;

    @Before
    public void openBrowser() {
        driver = new ChromeDriver();
    }

    @After
    public void closeBrowser() {
        driver.quit();
    }

    @Given("the login page is open")
    public void openLoginPage() {
        driver.get("https://example.test/login");
    }

    @When("the user logs in as {string} with password {string}")
    public void login(String username, String password) {
        driver.findElement(By.id("username")).sendKeys(username);
        driver.findElement(By.id("password")).sendKeys(password);
        driver.findElement(By.id("submit")).click();
    }

    @Then("the user sees the dashboard")
    public void seesDashboard() {
        assertTrue(driver.getCurrentUrl().contains("/dashboard"));
    }
}
```

The feature file doesn't change at all when you swap the fake service for a
real browser — that's the point of BDD: the business-readable contract is
stable, the implementation underneath it isn't. I did **not** run this
against a real browser here — no Chrome/driver available in this
environment — so treat it as reviewed code, matching the WebDriver API used
throughout Level 1, not verified output.

## 7. Testing traps

!!! warning "Trap 1 — two step definitions match one step"
    Cucumber throws `DuplicateStepDefinitionException` if two `@When`
    patterns both match a line. Keep step text specific and grep your steps
    package before adding a near-duplicate.

!!! warning "Trap 2 — steps shared across features that aren't really the same"
    `Given the login page is open` reused verbatim across ten features looks
    efficient until one feature needs it to also clear cookies. Changing the
    shared step then silently changes nine other features' behaviour. Prefer
    a slightly longer, more specific phrase over accidental reuse.

!!! warning "Trap 3 — state leaking between scenarios"
    Step definition classes are instantiated fresh per scenario by default,
    but a `static` field defeats that isolation. A counter or list declared
    `static` in `LoginSteps` will silently accumulate across scenarios and
    make the second scenario in a run depend on the first.

!!! warning "Trap 4 — Scenario Outline with an empty cell"
    `| username | | message |` with a genuinely blank cell is easy to
    misread in a wide table. Misaligned pipes produce a `Then` block that
    silently receives the wrong column's value. Run `--dry-run` (or your IDE's
    Gherkin linter) after editing a table.

!!! warning "Trap 5 — Gherkin as documentation nobody automates"
    A feature file with steps that print `// TODO` and always pass gives
    false confidence in a release. Every `Given/When/Then` needs a step
    definition that actually asserts something, or the scenario should be
    tagged `@wip` and excluded from CI.

## Cheat sheet

| Concept | Syntax |
|---|---|
| Feature file location | `src/test/resources/features/*.feature` |
| Given/When/Then | preconditions / action / assertion |
| Scenario Outline | `Examples:` table drives repeated runs |
| String parameter | `{string}` in step text |
| Number parameter | `{int}` |
| Run scenarios by tag | `@Suite` + `filter.tags` config parameter |
| Hook every scenario | `@Before` / `@After` |
| Hook tagged scenarios | `@Before("@tagName")` |
| JUnit 5 runner | `@Suite @IncludeEngines("cucumber")` |
| Attach failure context | `scenario.isFailed()` inside `@After` |

## Exercise

1. Write `login.feature` and `LoginSteps` exactly as above; run it through
   the Cucumber JUnit Platform engine and confirm 4/4 scenarios pass.
2. Add a scenario tagged `@requiresLogin` for "logged-in user logs out", and
   a `@Before("@requiresLogin")` hook that logs a message; confirm the hook
   only fires for that scenario.
3. Add a fifth `Examples` row to the outline for a username containing
   leading/trailing whitespace, and decide (then implement) whether
   `FakeLoginService` should trim it.
4. Rewrite two manual test cases from your Level 1 test-case document as
   Gherkin scenarios in a new feature file, with step definitions backed by
   fakes (no browser).
5. Introduce a duplicate step definition on purpose, run the suite, and
   paste the exact `DuplicateStepDefinitionException` message you get —
   this is what Trap 1 looks like in real output.
