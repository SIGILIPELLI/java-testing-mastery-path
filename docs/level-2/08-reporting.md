# 08 · Reporting (Allure, ExtentReports)

A test run produces two audiences. The suite tells *you* what broke. A report
tells the release manager, the developer and the product owner what state the
build is in — with screenshots, timings, trends and a defect trail. In Level 1
you wrote an execution summary by hand; this module generates it, and makes
every failure carry the evidence needed to raise a bug report.

## 1. What you already have

Surefire and Failsafe write machine-readable XML into
`target/surefire-reports/` and `target/failsafe-reports/` on every run. That
JUnit XML format is the lingua franca: Jenkins, GitHub Actions, GitLab and
every reporting tool below consume it.

```bash
mvn clean verify
ls target/failsafe-reports/
# TEST-com.example.tests.LoginIT.xml   LoginIT.txt   failsafe-summary.xml
```

```bash
# A readable HTML view of exactly that XML, no extra dependencies
mvn surefire-report:report
open target/site/surefire-report.html
```

Start here. If your CI already ingests the XML, you may not need anything
else; add Allure or Extent when you actually need screenshots and steps.

## 2. Allure — the reporting standard

```xml
<dependency>
    <groupId>io.qameta.allure</groupId>
    <artifactId>allure-testng</artifactId>   <!-- or allure-junit5 -->
    <version>2.27.0</version>
    <scope>test</scope>
</dependency>
```

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <argLine>
            -javaagent:"${settings.localRepository}/org/aspectj/aspectjweaver/1.9.22/aspectjweaver-1.9.22.jar"
        </argLine>
        <systemPropertyVariables>
            <allure.results.directory>${project.build.directory}/allure-results</allure.results.directory>
        </systemPropertyVariables>
    </configuration>
</plugin>
```

Allure works in two stages: the run writes raw JSON into
`target/allure-results/`, and a separate command turns that into HTML.

```bash
mvn clean verify
allure serve target/allure-results          # opens a local report
allure generate target/allure-results -o target/allure-report --clean
```

### Annotating tests

```java
package com.example.tests;

import io.qameta.allure.*;
import org.testng.annotations.Test;

import static org.testng.Assert.assertEquals;

@Epic("Authentication")
@Feature("Login")
public class LoginAllureIT {

    @Test(description = "Valid credentials reach the secure area")
    @Story("A registered user can sign in")
    @Severity(SeverityLevel.BLOCKER)
    @Link(name = "REQ-02", url = "https://wiki.example.com/reqs/REQ-02")
    @Issue("BUG-142")
    public void validLogin() {
        openLoginPage();
        submitCredentials("tomsmith", "SuperSecretPassword!");
        assertEquals(currentHeading(), "Secure Area");
    }

    @Step("Open the login page")
    private void openLoginPage() { /* ... */ }

    @Step("Submit credentials for user {0}")
    private void submitCredentials(String user, String password) { /* ... */ }

    private String currentHeading() { return "Secure Area"; }
}
```

`@Step` is what makes an Allure report worth generating: each annotated
method becomes a collapsible line in the report, with its parameters, so a
failure shows exactly which step broke rather than a bare stack trace.

### Attaching screenshots and logs

```java
import io.qameta.allure.Allure;
import io.qameta.allure.Attachment;
import org.openqa.selenium.*;

import java.io.ByteArrayInputStream;

public final class AllureAttachments {

    @Attachment(value = "Screenshot", type = "image/png")
    public static byte[] screenshot(WebDriver driver) {
        return ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);
    }

    @Attachment(value = "Page source", type = "text/html")
    public static String pageSource(WebDriver driver) {
        return driver.getPageSource();
    }

    /** The non-annotation form, usable anywhere. */
    public static void attachPng(String name, byte[] png) {
        Allure.addAttachment(name, "image/png", new ByteArrayInputStream(png), ".png");
    }
}
```

Call them from the `ITestListener` you built in Module 03:

```java
@Override
public void onTestFailure(ITestResult result) {
    WebDriver driver = DriverFactory.get();
    if (driver != null) {
        AllureAttachments.screenshot(driver);
        AllureAttachments.pageSource(driver);
    }
}
```

### Environment and history

`target/allure-results/environment.properties`:

```properties
Browser=Chrome 126
Browser.Headless=true
OS=macOS 14.5
Base.URL=https://staging.example.com
Java=21.0.3
Build=1042
```

To get trend graphs, copy the previous report's `history/` folder into the
new `allure-results/` before generating. Without that step every report looks
like the first run ever — the most common Allure disappointment.

## 3. ExtentReports — self-contained HTML

Extent produces one HTML file with no separate CLI, which suits teams that
email reports around.

```xml
<dependency>
    <groupId>com.aventstack</groupId>
    <artifactId>extentreports</artifactId>
    <version>5.1.2</version>
    <scope>test</scope>
</dependency>
```

```java
package com.example.listeners;

import com.aventstack.extentreports.*;
import com.aventstack.extentreports.reporter.ExtentSparkReporter;
import org.openqa.selenium.*;
import org.testng.*;

public class ExtentListener implements ITestListener {

    private static ExtentReports extent;
    private static final ThreadLocal<ExtentTest> TEST = new ThreadLocal<>();

    @Override
    public void onStart(ITestContext context) {
        ExtentSparkReporter spark = new ExtentSparkReporter("target/extent-report.html");
        spark.config().setDocumentTitle("QA Automation Report");
        spark.config().setReportName("Regression Suite");

        extent = new ExtentReports();
        extent.attachReporter(spark);
        extent.setSystemInfo("Browser", System.getProperty("browser", "chrome"));
        extent.setSystemInfo("Environment", System.getProperty("base.url", "local"));
    }

    @Override
    public void onTestStart(ITestResult result) {
        TEST.set(extent.createTest(result.getMethod().getMethodName(),
                                   result.getMethod().getDescription()));
    }

    @Override
    public void onTestSuccess(ITestResult result) {
        TEST.get().pass("Passed in " + duration(result) + " ms");
    }

    @Override
    public void onTestFailure(ITestResult result) {
        ExtentTest test = TEST.get();
        test.fail(result.getThrowable());

        WebDriver driver = com.example.driver.DriverFactory.get();
        if (driver instanceof TakesScreenshot shooter) {
            String base64 = shooter.getScreenshotAs(OutputType.BASE64);
            test.addScreenCaptureFromBase64String(base64, "Failure screenshot");
        }
    }

    @Override
    public void onTestSkipped(ITestResult result) {
        TEST.get().skip("Skipped: " + result.getThrowable());
    }

    @Override
    public void onFinish(ITestContext context) {
        extent.flush();          // WITHOUT THIS THE FILE IS EMPTY
        TEST.remove();
    }
}
```

```
Regression Suite -- 42 tests, 40 passed, 1 failed, 1 skipped (4m 12s)
Report written to target/extent-report.html
```

!!! warning "`extent.flush()` and `ThreadLocal` are both mandatory"
    Extent buffers everything in memory; without `flush()` in `onFinish` the
    HTML file is created but empty. And a plain `ExtentTest` field is
    overwritten by whichever thread runs next under `parallel="methods"`,
    scrambling logs between tests. `ThreadLocal` — the same pattern as the
    driver in Module 03 — is the fix.

Embedding screenshots as base64 keeps the report to one file, at the cost of
size; a 200-test run with failures can produce a 30 MB HTML. Write PNGs to
disk and link them relatively when that becomes a problem.

## 4. Comparing the options

| | Surefire HTML | Allure | ExtentReports |
|---|---|---|---|
| Extra dependency | none | agent + CLI | one JAR |
| Output | HTML from XML | JSON → HTML | single HTML |
| Steps | no | yes (`@Step`) | yes (manual `log`) |
| Screenshots | no | yes | yes (base64 or file) |
| History/trends | no | yes (with `history/`) | no |
| Requirement links | no | `@Link`, `@Issue`, `@TmsLink` | manual |
| Best for | CI ingestion | team dashboards | emailing one file |

## 5. What a report must contain

Reporting tools make it easy to produce something pretty and useless. A
report that serves the Level 1 execution-summary purpose needs:

1. Pass/fail/skip counts **and the total planned**, so gaps are visible.
2. Environment: browser, version, base URL, build number, timestamp.
3. For every failure: the assertion message, a screenshot, and the step it
   failed at.
4. Duration per test, so the slowest ones are identifiable.
5. A link from each test to the requirement or ticket it covers.
6. Trend across recent runs — one failure is noise; four in a row is a defect.

If your report cannot answer "can we ship?", it is decoration.

## 6. Testing traps

!!! warning "Trap 1 — a report that hides skips"
    A suite reporting "40 passed" out of 60 written tests is a red build
    dressed as green. Always show planned vs executed.

!!! warning "Trap 2 — screenshots taken after teardown"
    If `@AfterMethod` quits the driver before the listener's
    `onTestFailure` runs, the screenshot call throws
    `NoSuchSessionException`. TestNG runs listeners around configuration
    methods; verify the ordering on your setup rather than assuming it.

!!! warning "Trap 3 — the report as the only artifact"
    HTML in `target/` is deleted by the next `mvn clean`. Archive reports as
    CI build artifacts, or the evidence for the release you shipped last
    Tuesday no longer exists.

!!! warning "Trap 4 — logging secrets into the report"
    `@Step("Log in as {0}/{1}")` prints the password into HTML that gets
    emailed around. Mask credentials in step names and attachments.

!!! warning "Trap 5 — the missing AspectJ weaver"
    Allure's `@Step` and `@Attachment` annotations do nothing without the
    `aspectjweaver` `-javaagent` in `argLine`. The symptom is a report that
    generates cleanly with no steps in it — and no error message anywhere.

## Cheat sheet

| Task | Command / code |
|---|---|
| Raw XML results | `target/surefire-reports/`, `target/failsafe-reports/` |
| Plain HTML report | `mvn surefire-report:report` |
| Allure: view | `allure serve target/allure-results` |
| Allure: generate | `allure generate target/allure-results -o target/allure-report --clean` |
| Allure step | `@Step("Submit credentials for {0}")` |
| Allure attachment | `@Attachment(value="Screenshot", type="image/png")` |
| Allure runtime attach | `Allure.addAttachment(name, "image/png", stream, ".png")` |
| Allure metadata | `@Epic`, `@Feature`, `@Story`, `@Severity`, `@Link`, `@Issue` |
| Allure environment | `target/allure-results/environment.properties` |
| Extent init | `new ExtentSparkReporter("target/extent-report.html")` |
| Extent per-test | `extent.createTest(name)` in a `ThreadLocal` |
| Extent screenshot | `test.addScreenCaptureFromBase64String(base64, "caption")` |
| Extent finalise | `extent.flush()` in `onFinish` |

## Exercise

1. Run `mvn clean verify` then `mvn surefire-report:report`, open the HTML,
   and note the three things it does *not* tell you that a release manager
   would ask.
2. Add Allure to your Module 03 TestNG suite, annotate one test with `@Epic`,
   `@Feature`, `@Story`, `@Severity` and `@Link`, and generate the report.
3. Convert three private helper methods into `@Step` methods with parameters
   in the step name. Break a test and confirm the report shows which step
   failed. Then remove the `aspectjweaver` `-javaagent` line, regenerate, and
   record what changes — this is trap 5, and recognising it saves an hour.
4. Attach a screenshot **and** the page source on failure, using the listener
   from Module 03.
5. Write `environment.properties` with browser, OS, base URL and build
   number, and confirm the Environment panel populates.
6. Run the suite twice, copying `history/` between runs, and confirm the
   trend graph appears.
7. Implement `ExtentListener` as in section 3 with a `ThreadLocal`. Then
   replace the `ThreadLocal` with a plain field, run with
   `parallel="methods" thread-count="3"`, and describe exactly how the report
   is wrong.
8. Using your generated report, write the same execution summary you wrote by
   hand in the Level 1 project. Note which numbers you could copy directly
   and which you still had to work out yourself.
