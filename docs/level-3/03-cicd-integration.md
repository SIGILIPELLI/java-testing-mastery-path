# 03 · CI/CD Integration

A test suite that only runs on your laptop protects nobody but you. The
point of everything built in Levels 1–2 is to run automatically on every
push, block a bad merge, and report back before a human even looks. This
module wires a Maven test suite into GitHub Actions.

## 1. What CI actually needs from your project

1. A build tool that can run headlessly with one command (`mvn test`).
2. A pinned JDK and dependency versions (no "works on my machine").
3. A machine-readable test report (Surefire XML) CI can parse for a
   pass/fail badge and per-test detail.
4. A way to fail the whole pipeline the moment a test fails — CI is only
   useful if a red suite actually blocks the merge.

## 2. Surefire configuration

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>3.2.5</version>
      <configuration>
        <includes>
          <include>**/*Test.java</include>
        </includes>
        <testFailureIgnore>false</testFailureIgnore>
      </configuration>
    </plugin>
  </plugins>
</build>
```

`testFailureIgnore=false` (the default) is the setting that makes CI
meaningful: `mvn test` exits non-zero on any failure, and every CI runner
treats a non-zero exit as a failed job.

```
$ mvn test
...
Tests run: 24, Failures: 1, Errors: 0, Skipped: 0

[INFO] BUILD FAILURE
$ echo $?
1
```

## 3. A GitHub Actions workflow

`.github/workflows/tests.yml`:

```yaml
name: Test Suite

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven

      - name: Run unit + API tests
        run: mvn -B test -Dgroups="unit,api"

      - name: Publish test report
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Surefire Results
          path: 'target/surefire-reports/*.xml'
          reporter: java-junit

      - name: Upload failure screenshots
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: failure-screenshots
          path: target/screenshots/
```

`cache: maven` alone typically cuts a cold `mvn test` from a couple of
minutes to a few seconds on repeat runs — dependency downloads are the
biggest fixed cost in a Java CI job. `if: always()` on the report step
matters: without it, a failed test run skips the very step that would show
you *which* test failed.

## 4. Splitting fast and slow tests

UI tests are 10-100x slower than unit tests; running both in one blocking
job means a one-line typo waits behind a five-minute Selenium suite. TestNG
groups (Level 1 Module 09) or JUnit 5 tags solve this:

```java
import org.junit.jupiter.api.Tag;

@Tag("unit")
class OrderServiceTest { /* ... */ }

@Tag("ui")
class LoginFlowTest { /* ... */ }
```

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-surefire-plugin</artifactId>
  <configuration>
    <groups>${test.groups}</groups>
  </configuration>
</plugin>
```

```yaml
jobs:
  fast-tests:
    runs-on: ubuntu-latest
    steps:
      - run: mvn test -Dtest.groups="unit,api"

  ui-tests:
    needs: fast-tests
    runs-on: ubuntu-latest
    steps:
      - run: mvn test -Dtest.groups="ui"
```

`needs: fast-tests` makes the slow job wait for the fast one, so a broken
unit test fails the pipeline in seconds instead of after a five-minute
Selenium run has already spun up.

## 5. Running Selenium headlessly in CI

CI runners have no display. Headless Chrome plus `xvfb` (or headless mode
directly) is the standard fix:

```java
ChromeOptions options = new ChromeOptions();
if (System.getenv("CI") != null) {
    options.addArguments("--headless=new", "--no-sandbox", "--disable-dev-shm-usage");
}
WebDriver driver = new ChromeDriver(options);
```

Reading `CI` (a variable every major CI provider sets automatically) means
the same test class runs headed on your laptop for debugging and headless
in the pipeline, with zero duplicated code.

## 6. Failing fast and reporting clearly

```java
@AfterEach
void screenshotOnFailure(TestInfo testInfo, TestReporter reporter) {
    // JUnit 5 doesn't expose pass/fail directly here without an extension;
    // TestWatcher is the supported hook for this.
}
```

```java
public class ScreenshotOnFailureExtension implements TestWatcher {
    @Override
    public void testFailed(ExtensionContext context, Throwable cause) {
        WebDriver driver = DriverHolder.get();
        if (driver instanceof TakesScreenshot ts) {
            File shot = ts.getScreenshotAs(OutputType.FILE);
            File dest = new File("target/screenshots/" + context.getDisplayName() + ".png");
            dest.getParentFile().mkdirs();
            try { Files.copy(shot.toPath(), dest.toPath()); } catch (IOException ignored) {}
        }
    }
}
```

```java
@ExtendWith(ScreenshotOnFailureExtension.class)
class LoginFlowTest { /* ... */ }
```

`TestWatcher` is the JUnit 5 extension point built exactly for this — it's
notified of the outcome (`testSuccessful`, `testFailed`, `testAborted`)
without you having to track pass/fail manually in every test.

I did not run an actual GitHub Actions job or headless Chrome in this
environment (no CI runner, no browser available here); the YAML and Java
above are reviewed against GitHub Actions' documented syntax and the
`maven-surefire-plugin`/JUnit 5 APIs used elsewhere in this course, not
executed.

## 7. Testing traps

!!! warning "Trap 1 — green locally, red in CI"
    A test that reads a local file by absolute path, depends on your
    machine's timezone, or relies on a locally-installed browser version
    will pass for you and fail in CI. Parameterize paths, pin the browser
    version in CI, and set `TZ=UTC` explicitly rather than assuming.

!!! warning "Trap 2 — `testFailureIgnore=true` left on"
    Someone sets this while debugging a flaky suite and forgets to revert
    it. The build reports success with failing tests inside it — worse than
    no CI at all, because it looks trustworthy. Grep for it before every
    release.

!!! warning "Trap 3 — no timeout on the CI job itself"
    A hung WebDriver session (Trap 5 from Module 02's mocking discussion,
    now at the infrastructure level) can leave a CI job running for hours,
    burning minutes/credits. Set `timeout-minutes:` on the job.

!!! warning "Trap 4 — order-dependent tests that only fail in CI's parallelism"
    CI runners often execute more tests in parallel than your laptop does by
    default. A test relying on shared static state (Module 01's Trap 3) may
    pass every time locally and fail intermittently in CI purely from
    scheduling differences.

!!! warning "Trap 5 — secrets committed to make CI 'just work'"
    Hard-coding an API key into the workflow YAML "to unblock the build" is
    a public leak the moment the repo is public (or ever becomes public).
    Use the CI provider's encrypted secrets store and inject via environment
    variables, exactly as flagged for RestAssured tokens in Level 2.

## Cheat sheet

| Task | Config |
|---|---|
| Fail build on test failure | `testFailureIgnore=false` (Surefire default) |
| Run only a tag/group | `mvn test -Dgroups=unit` |
| Cache Maven deps in Actions | `cache: maven` on `setup-java` |
| Publish JUnit XML as a report | `dorny/test-reporter` + `surefire-reports/*.xml` |
| Detect CI environment in code | `System.getenv("CI") != null` |
| Headless Chrome flags | `--headless=new --no-sandbox --disable-dev-shm-usage` |
| Sequence slow jobs after fast ones | `needs: fast-tests` |
| Hook pass/fail in JUnit 5 | `implements TestWatcher` |
| Upload artifacts on failure | `if: failure()` + `upload-artifact` |
| Cap a hung job | `timeout-minutes: 15` on the job |

## Exercise

1. Write a `tests.yml` workflow that runs `mvn test` on every push and pull
   request, using JDK 17 and Maven dependency caching.
2. Split your suite into `unit`/`api` tags and a `ui` tag using JUnit 5
   `@Tag`, and create two sequential jobs so a unit-test failure blocks the
   UI job from ever starting.
3. Add `timeout-minutes: 10` to the UI job and explain, in your own words,
   what happens to a hung Selenium session with and without it.
4. Implement `ScreenshotOnFailureExtension` and prove it works by writing a
   test that intentionally fails and checking a screenshot file appears
   under `target/screenshots/`.
5. Deliberately set `testFailureIgnore=true`, introduce a failing test, run
   `mvn test`, and record the exit code. Then revert the setting, rerun, and
   record the new exit code — write one sentence on why the difference
   matters for CI.
