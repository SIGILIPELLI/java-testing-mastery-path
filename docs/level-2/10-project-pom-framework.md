# 10 · Project — POM Framework with API Tests

This project assembles all nine modules into one deliverable: a Page Object
Model framework with a thread-safe driver factory, externalised
configuration, TestNG data providers, RestAssured API tests, cross-browser
execution and an Allure report. It is the artefact you show in an interview
when asked "what have you built?"

**Application under test:** `https://the-internet.herokuapp.com`
**API under test:** `https://jsonplaceholder.typicode.com`

## Project layout

```
qa-automation-framework/
├── pom.xml
├── README.md
└── src/test
    ├── java/com/example/
    │   ├── config/Config.java
    │   ├── driver/DriverFactory.java
    │   ├── listeners/ScreenshotListener.java
    │   ├── pages/       BasePage, LoginPage, SecurePage, DropdownPage
    │   ├── api/         PostsApiTest, UsersApiTest, PostRequest
    │   └── tests/       BaseTest, LoginIT, DropdownIT, LoginDataDrivenIT
    └── resources/
        ├── config.properties
        ├── logins.csv
        ├── schemas/user-schema.json
        ├── smoke.xml
        └── regression.xml
```

## The build file

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>qa-automation-framework</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <suiteXmlFile>src/test/resources/regression.xml</suiteXmlFile>
        <browser>chrome</browser>
        <headless>true</headless>
        <threads>3</threads>
        <aspectj.version>1.9.22</aspectj.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.seleniumhq.selenium</groupId>
            <artifactId>selenium-java</artifactId>
            <version>4.23.0</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testng</groupId>
            <artifactId>testng</artifactId>
            <version>7.10.2</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.rest-assured</groupId>
            <artifactId>rest-assured</artifactId>
            <version>5.4.0</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.rest-assured</groupId>
            <artifactId>json-schema-validator</artifactId>
            <version>5.4.0</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <version>3.25.3</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-testng</artifactId>
            <version>2.27.0</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-failsafe-plugin</artifactId>
                <version>3.2.5</version>
                <configuration>
                    <suiteXmlFiles>
                        <suiteXmlFile>${suiteXmlFile}</suiteXmlFile>
                    </suiteXmlFiles>
                    <trimStackTrace>false</trimStackTrace>
                    <argLine>
                        -javaagent:"${settings.localRepository}/org/aspectj/aspectjweaver/${aspectj.version}/aspectjweaver-${aspectj.version}.jar"
                    </argLine>
                    <systemPropertyVariables>
                        <browser>${browser}</browser>
                        <headless>${headless}</headless>
                        <allure.results.directory>${project.build.directory}/allure-results</allure.results.directory>
                    </systemPropertyVariables>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>integration-test</goal>
                            <goal>verify</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

    <profiles>
        <profile>
            <id>smoke</id>
            <properties>
                <suiteXmlFile>src/test/resources/smoke.xml</suiteXmlFile>
            </properties>
        </profile>
    </profiles>
</project>
```

## The base test

Everything the framework does per test lives here, so no test class ever
touches a driver directly.

```java
package com.example.tests;

import com.example.config.Config;
import com.example.driver.DriverFactory;
import org.openqa.selenium.WebDriver;
import org.testng.annotations.*;

@Listeners(com.example.listeners.ScreenshotListener.class)
public abstract class BaseTest {

    protected WebDriver driver;

    @Parameters({"browser"})
    @BeforeMethod(alwaysRun = true)
    public void startBrowser(@Optional String browser) {
        String chosen = (browser != null) ? browser : Config.browser();
        DriverFactory.start(chosen, Config.headless());
        driver = DriverFactory.get();
    }

    @AfterMethod(alwaysRun = true)
    public void stopBrowser() {
        DriverFactory.stop();
    }
}
```

```java
package com.example.tests;

import com.example.config.Config;
import com.example.pages.LoginPage;
import com.example.pages.SecurePage;
import io.qameta.allure.*;
import org.testng.annotations.Test;

import static org.assertj.core.api.Assertions.assertThat;

@Epic("Authentication")
@Feature("Login")
public class LoginIT extends BaseTest {

    @Test(groups = "smoke", description = "REQ-02/03/04 -- valid credentials reach the secure area")
    @Severity(SeverityLevel.BLOCKER)
    @Link(name = "REQ-02", url = "https://wiki.example.com/reqs/REQ-02")
    public void validLoginReachesSecureArea() {
        SecurePage secure = new LoginPage(driver, Config.baseUrl())
                .open()
                .loginAs(Config.username(), Config.password());

        assertThat(secure.getHeading()).isEqualTo("Secure Area");
        assertThat(secure.getFlashMessage()).contains("You logged into a secure area");
        assertThat(secure.isLogoutVisible()).as("REQ-04 logout button").isTrue();
    }

    @Test(groups = "regression", description = "REQ-09 -- logout returns to /login")
    public void logoutReturnsToLogin() {
        String flash = new LoginPage(driver, Config.baseUrl())
                .open()
                .loginAs(Config.username(), Config.password())
                .logout()
                .getFlashMessage();

        assertThat(flash).contains("You logged out of the secure area");
    }
}
```

```java
package com.example.tests;

import com.example.config.Config;
import com.example.pages.LoginPage;
import org.testng.annotations.*;

import static org.assertj.core.api.Assertions.assertThat;

public class LoginDataDrivenIT extends BaseTest {

    @DataProvider(name = "invalidLogins")
    public Object[][] invalidLogins() {
        return new Object[][] {
                { "wronguser", "SuperSecretPassword!", "Your username is invalid!" },
                { "tomsmith",  "wrongpassword",        "Your password is invalid!" },
                { "",          "",                     "Your username is invalid!" },
                { "TOMSMITH",  "SuperSecretPassword!", "Your username is invalid!" },
        };
    }

    @Test(dataProvider = "invalidLogins", groups = "regression",
          description = "REQ-05/06/07 -- invalid credentials are rejected")
    public void rejectsInvalidCredentials(String user, String pass, String expected) {
        String flash = new LoginPage(driver, Config.baseUrl())
                .open()
                .loginExpectingFailure(user, pass)
                .getFlashMessage();

        assertThat(flash)
                .as("user='%s' pass='%s'", user, pass)
                .contains(expected);
    }
}
```

## The API tests

```java
package com.example.api;

import io.restassured.builder.RequestSpecBuilder;
import io.restassured.http.ContentType;
import io.restassured.specification.RequestSpecification;
import org.testng.annotations.*;

import static io.restassured.RestAssured.given;
import static io.restassured.module.jsv.JsonSchemaValidator.matchesJsonSchemaInClasspath;
import static org.hamcrest.Matchers.*;

public class UsersApiTest {

    private RequestSpecification spec;

    @BeforeClass(alwaysRun = true)
    public void buildSpec() {
        spec = new RequestSpecBuilder()
                .setBaseUri("https://jsonplaceholder.typicode.com")
                .setContentType(ContentType.JSON)
                .build();
    }

    @Test(groups = "smoke", description = "API-01 -- GET /users returns 10 valid users")
    public void listUsers() {
        given().spec(spec)
        .when().get("/users")
        .then().statusCode(200)
               .body("size()", equalTo(10))
               .body("email", everyItem(containsString("@")))
               .time(lessThan(5000L));
    }

    @Test(groups = "regression", description = "API-02 -- a user matches the agreed schema")
    public void userMatchesSchema() {
        given().spec(spec)
        .when().get("/users/1")
        .then().statusCode(200)
               .body(matchesJsonSchemaInClasspath("schemas/user-schema.json"));
    }

    @Test(groups = "regression", description = "API-03 -- an unknown user returns 404")
    public void unknownUserReturns404() {
        given().spec(spec).when().get("/users/9999").then().statusCode(404);
    }
}
```

## Running it

```bash
# One-time: JDK 21 and Maven on PATH, Chrome and Firefox installed
mvn -version

# Full regression, headless Chrome, 3 threads
mvn clean verify

# Smoke only
mvn clean verify -Psmoke

# Watch it happen in a real browser
mvn clean verify -Dheadless=false

# A different browser and environment
mvn clean verify -Dbrowser=firefox -Dbase.url=https://staging.example.com

# One class, one method
mvn clean verify -Dit.test=LoginIT
mvn clean verify -Dit.test=LoginIT#validLoginReachesSecureArea

# The report
allure serve target/allure-results
```

```
===============================================
Regression Suite
Total tests run: 14, Passes: 14, Failures: 0, Skips: 0
===============================================
[INFO] BUILD SUCCESS
[INFO] Total time: 01:12 min
```

Your `README.md` must contain exactly this section. A framework a new
starter cannot run in ten minutes from the README is not finished.

## Acceptance checklist

- [ ] No `By` locator appears anywhere in `src/test/java/com/example/tests/`
- [ ] No `Thread.sleep` anywhere; implicit wait is zero
- [ ] No URL, username or password is hard-coded in a test class
- [ ] `DriverFactory` uses `ThreadLocal` and calls `remove()` in teardown
- [ ] Every teardown method has `alwaysRun = true`
- [ ] At least one `@DataProvider` drives four or more rows
- [ ] At least three RestAssured tests, one with schema validation
- [ ] `smoke.xml` and `regression.xml` both run green, smoke in under a minute
- [ ] The suite runs with `parallel="methods" thread-count="3"` without failures
- [ ] Failures produce a screenshot attached to the Allure report
- [ ] `mvn clean verify` is green from a fresh clone with no manual setup
- [ ] `README.md` documents every command in "Running it"

## Stretch goals

1. **Prove the parallelism is safe.** Set `thread-count="5"`, run twenty
   times, and record every failure. Any failure that appears in fewer than
   twenty runs is shared state — find it. This is the exercise that turns a
   framework from "works" into "trusted".
2. **Add API-driven UI setup.** Create a record through the API, then assert
   it in the UI. Measure how much faster it is than doing the setup through
   forms, and put the number in your README.
3. **Add cross-browser smoke.** A `cross-browser.xml` with Chrome, Firefox
   and Edge blocks under `parallel="tests"`. Report grouped by test, not by
   browser (Module 09, trap 5).
4. **Wire in CI.** A GitHub Actions workflow running `mvn clean verify` on
   every push, archiving `target/allure-results` and the screenshots as
   artifacts. Level 3 covers this properly — a first working version now is
   worth more than a perfect one later.
5. **Add a retry analyzer, then justify it.** Implement `IRetryAnalyzer`,
   log every retry, and write a paragraph on why a rising retry count is
   itself a defect (Module 03).
6. **Contract-test the API.** Extend `user-schema.json` to constrain formats
   (`email`, `uri`) as well as presence, and add a test that fails when a
   field's *type* changes rather than only when it disappears.

## Exercise

Build the framework above, then produce a `reflection.md` answering:

1. Which module's technique removed the most duplication from your Level 1
   code — page objects, the driver factory, data providers or the config
   class? Quantify it: lines before and after.
2. Run the suite serially and with three threads. Report both times, and name
   one thing that had to be thread-safe for the parallel run to work.
3. Point at one assertion in your suite whose failure message would let a
   developer diagnose the bug without opening your code, and one whose
   message would not. Fix the second.
4. Your API tests run against a fake backend that accepts every POST and
   stores nothing. Name two specific assertions that pass here but would fail
   against a real API, and how you would know.
5. Take the requirement list from the Level 1 project and map each
   requirement to the test that covers it. Any requirement with no test, and
   any test covering no requirement, is a finding — list both.
6. A colleague clones your repository on a Windows machine with only Firefox
   installed. Walk through exactly what happens when they run
   `mvn clean verify`, and fix whatever breaks.
