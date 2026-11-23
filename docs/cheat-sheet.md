---
title: "Java Testing Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Java Testing cheat sheet: 35 quick-reference tables from every level of the Java Testing Mastery Path, each linked to its full lesson."
---

# Java Testing Cheat Sheet

Quick-reference tables from **35 lessons** of the free Java Testing Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [02 · Test Case Design &amp; Documentation](level-1/02-test-case-design.md)

| Artefact | One-line purpose |
|---|---|
| Test scenario | High-level *what* to test |
| Test case | Detailed, repeatable *how* to test it, with an exact expected result |
| Test suite | A grouped set of cases run together |
| Test data | The specific input values a case needs |
| Preconditions | State that must exist before step 1 |
| Test plan | The project-level strategy, scope and schedule document |
| RTM | Requirement ↔ test case ↔ defect mapping; proves coverage |
| Test summary report | End-of-cycle results and recommendation to release |

### [03 · Test Types &amp; Levels](level-1/03-test-types-levels.md)

| Term | One-liner |
|---|---|
| Unit test | One method, dependencies faked, milliseconds |
| Integration test | Components talking to each other |
| System test | Whole app, end to end, against the spec |
| UAT | Whole app, judged by the business |
| Smoke | Is this build worth testing? |
| Sanity | Does this one change work? |
| Regression | Did the change break anything else? |
| Re-test | Is this specific bug actually fixed? |
| Functional | Does it do the right thing? |
| Non-functional | Does it do it fast/securely/usably enough? |
| Black box | Test without seeing code |
| White box | Test the code paths |
| Static | Review without executing |
| Dynamic | Execute and observe |

### [04 · Defect Lifecycle &amp; Bug Reporting](level-1/04-defect-lifecycle.md)

| Concept | One-liner |
|---|---|
| Severity | How badly it breaks the product (tester decides) |
| Priority | How soon it must be fixed (business decides) |
| Reproducibility | Always / intermittent — say which |
| Minimal reproduction | Fewest steps that still show the bug — the highest-value thing you can provide |
| Verified vs Closed | Verified = fix confirmed; Closed = tracking finished |
| Reopened | Fix failed re-testing |
| Deferred | Real bug, postponed by the business |
| DRE | % of defects caught before production |

### [05 · Manual Testing in Practice](level-1/05-manual-testing-practice.md)

| Technique | Core idea | Test count |
|---|---|---|
| **EP** | One value per behaviour group | 1 per partition |
| **BVA** | Test at the edges: B−1, B, B+1 | 3 per boundary |
| **Decision table** | Every combination of conditions | 2ⁿ (collapsible) |
| **State transition** | Every valid and invalid state change | 1 per transition |
| **Exploratory** | Learn and test simultaneously, time-boxed with a charter | Session-based |

### [06 · Java Setup for Testers](level-1/06-java-setup-for-testers.md)

| Item | Value |
|---|---|
| JDK version | 17 or 21 (LTS) |
| Verify Java | `java -version` |
| Build tool | Maven — `mvn -version` |
| Project definition | `pom.xml` |
| Test code location | `src/test/java` |
| Test data location | `src/test/resources` |
| Run all tests | `mvn clean test` |
| Run one class | `mvn test -Dtest=ClassName` |
| Dependency scope for test libs | `<scope>test</scope>` |
| Reports | `target/surefire-reports/` |

### [07 · JUnit 5 Fundamentals](level-1/07-junit5-fundamentals.md)

| Annotation / method | Purpose |
|---|---|
| `@Test` | Marks a test method |
| `@BeforeAll` / `@AfterAll` | Once per class (static) |
| `@BeforeEach` / `@AfterEach` | Around every test |
| `@DisplayName("…")` | Readable name in reports |
| `@Disabled("reason")` | Skip with a documented reason |
| `@Tag("smoke")` | Group tests for selective runs |
| `@RepeatedTest(n)` | Run the same test n times |
| `@ParameterizedTest` + `@CsvSource` | Data-driven tests — automate EP/BVA |
| `assertEquals(exp, act, msg)` | Value equality |
| `assertTrue` / `assertFalse` | Boolean conditions |
| `assertNull` / `assertNotNull` | Null checks |
| `assertThrows(Type.class, () -> …)` | Negative tests |
| `assertAll(…)` | Report every failure in one run |
| `mvn test` | Run the suite |

### [08 · Selenium WebDriver Basics](level-1/08-selenium-webdriver-basics.md)

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

### [09 · TestNG Basics](level-1/09-testng-basics.md)

| Item | TestNG |
|---|---|
| Test method | `@Test public void name()` |
| Assertion | `Assert.assertEquals(actual, expected, "msg")` |
| Setup / teardown | `@BeforeMethod` / `@AfterMethod` (plus Class, Test, Suite) |
| Skip a test | `@Test(enabled = false)` or `throw new SkipException(...)` |
| Expected exception | `@Test(expectedExceptions = X.class)` |
| Order | `@Test(priority = 1)` — lower first |
| Grouping | `@Test(groups = {"smoke"})` |
| Dependency | `@Test(dependsOnMethods = "login")` |
| Timeout | `@Test(timeOut = 3000)` |
| Data-driven | `@DataProvider` + `@Test(dataProvider = "name")` |
| XML parameter | `@Parameters({"browser"})` |
| Soft assertions | `new SoftAssert()` … `softly.assertAll()` |
| Suite file | `testng.xml` |
| Parallel | `<suite parallel="methods" thread-count="4">` |
| Re-run failures | `test-output/testng-failed.xml` |

## Level 2 · Intermediate

### [01 · Page Object Model (POM)](level-2/01-page-object-model.md)

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

### [02 · Selenium Advanced](level-2/02-selenium-advanced.md)

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

### [03 · TestNG Advanced](level-2/03-testng-advanced.md)

| Task | Code |
|---|---|
| Data provider | `@DataProvider(name="x")` + `@Test(dataProvider="x")` |
| External provider | `@Test(dataProvider="x", dataProviderClass=TestData.class)` |
| Parallel data rows | `@DataProvider(parallel = true)` |
| Group a test | `@Test(groups = {"smoke"})` |
| Prerequisite | `@Test(dependsOnMethods = "login")` |
| Disable | `@Test(enabled = false)` |
| Time limit | `@Test(timeOut = 5000)` |
| Repeat | `@Test(invocationCount = 5)` |
| Expected exception | `@Test(expectedExceptions = IllegalArgumentException.class)` |
| Suite parallelism | `<suite parallel="methods" thread-count="4">` |
| XML parameter | `<parameter name="browser" value="chrome"/>` + `@Parameters` |
| Register listener | `@Listeners(X.class)` or `<listeners>` in XML |
| Thread-safe driver | `ThreadLocal<WebDriver>` + `remove()` in teardown |
| Run a suite file | `mvn test -DsuiteXmlFile=testng.xml` |

### [04 · API Testing with RestAssured](level-2/04-api-testing-restassured.md)

| Task | Code |
|---|---|
| Base URI | `RestAssured.baseURI = "https://api.example.com"` |
| GET | `given().when().get("/path").then()` |
| Query param | `.queryParam("userId", 3)` |
| Path param | `.pathParam("id", 2)` + `get("/users/{id}")` |
| Header | `.header("Authorization", "Bearer " + token)` |
| JSON body | `.contentType(ContentType.JSON).body(payload)` |
| Status | `.statusCode(200)` |
| Field | `.body("id", equalTo(1))` |
| Nested field | `.body("address.city", equalTo("Wisokyburgh"))` |
| List size | `.body("size()", equalTo(100))` |
| Every item | `.body("userId", everyItem(notNullValue()))` |
| Response time | `.time(lessThan(2000L))` |
| Extract value | `.extract().path("token")` |
| Extract list | `.extract().jsonPath().getList("email")` |
| Schema | `.body(matchesJsonSchemaInClasspath("schemas/user.json"))` |
| Shared config | `RequestSpecBuilder` + `given().spec(apiSpec)` |
| Debug logging | `.log().all()` / `.log().ifValidationFails()` |

### [05 · Test Data Management](level-2/05-test-data-management.md)

| Need | Tool |
|---|---|
| Environment config | `config.properties` + `-Dkey=value` override |
| Secrets | `System.getenv("QA_PASSWORD")` |
| Inline table of rows | `@CsvSource` (JUnit) / `@DataProvider` (TestNG) |
| External CSV | `@CsvFileSource(resources="/logins.csv", numLinesToSkip=1)` |
| Nested data | JSON + Jackson `ObjectMapper` |
| Business-owned data | Excel + Apache POI `DataFormatter` |
| Unique values | `net.datafaker.Faker`, or a timestamped email |
| Repeatable random | `new Faker(new Random(42))` |
| Readable construction | Builder — `aUser().withRole("admin").build()` |
| Load a resource | `getClass().getResourceAsStream("/file.csv")` |
| Cleanup | `@AfterEach` delete through the API |

### [06 · Maven/Gradle for Test Projects](level-2/06-maven-gradle-test-projects.md)

| Task | Command |
|---|---|
| Unit tests | `mvn test` |
| Unit + integration, with cleanup | `mvn verify` |
| One class | `mvn test -Dtest=LoginTest` |
| One method | `mvn test -Dtest=LoginTest#validLogin` |
| With a profile | `mvn verify -Psmoke` |
| Override a property | `mvn verify -Dbrowser=firefox -Dthreads=4` |
| Named TestNG suite | `mvn verify -DsuiteXmlFile=smoke.xml` |
| Skip *running* tests | `mvn install -DskipTests` (still compiles them) |
| Dependency tree | `mvn dependency:tree` |
| Unused/undeclared deps | `mvn dependency:analyze` |
| Which profiles are on | `mvn help:active-profiles` |
| Offline build | `mvn -o verify` |
| Reports | `target/surefire-reports/`, `target/failsafe-reports/` |

### [07 · Assertions &amp; Matchers](level-2/07-assertions-matchers.md)

| Check | AssertJ |
|---|---|
| Equality | `assertThat(x).isEqualTo(y)` |
| Null | `assertThat(x).isNull()` / `.isNotNull()` |
| Boolean | `assertThat(flag).isTrue()` |
| String contains | `assertThat(s).contains("abc")` |
| Case-insensitive | `assertThat(s).containsIgnoringCase("ABC")` |
| Regex | `assertThat(s).matches("id=\\d+")` |
| Blank | `assertThat(s).isNotBlank()` |
| Size | `assertThat(list).hasSize(3)` |
| Contains, order-free | `assertThat(list).containsExactlyInAnyOrder(a, b)` |
| Field projection | `assertThat(list).extracting(Product::name).contains("Mouse")` |
| Filter then assert | `assertThat(list).filteredOn(Product::inStock).hasSize(2)` |
| Every element | `assertThat(list).allSatisfy(p -> ...)` |
| Numeric tolerance | `assertThat(d).isCloseTo(250.0, within(0.05))` |
| Range | `assertThat(n).isBetween(1, 10)` |
| Map entry | `assertThat(map).containsEntry("k", "v")` |
| Exception | `assertThatThrownBy(...).isInstanceOf(X.class).hasMessageContaining("y")` |
| Add context | `.as("REQ-04 logout button")` |
| Soft (AssertJ) | `SoftAssertions softly = ...; softly.assertAll();` |
| Soft (TestNG) | `SoftAssert sa = new SoftAssert(); sa.assertAll();` |
| Group (JUnit) | `assertAll(() -> ..., () -> ...)` |

### [08 · Reporting (Allure, ExtentReports)](level-2/08-reporting.md)

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

### [09 · Cross-Browser Testing](level-2/09-cross-browser-testing.md)

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

## Level 3 · Advanced

### [01 · BDD with Cucumber](level-3/01-bdd-cucumber.md)

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

### [02 · Mocking with Mockito](level-3/02-mocking-mockito.md)

| Task | Code |
|---|---|
| Enable Mockito in JUnit 5 | `@ExtendWith(MockitoExtension.class)` |
| Create a mock | `@Mock PaymentGateway gateway;` |
| Auto-wire mocks into class under test | `@InjectMocks OrderService service;` |
| Stub a return value | `when(gateway.charge(...)).thenReturn(true)` |
| Stub an exception | `when(...).thenThrow(new RuntimeException(...))` |
| Match any value | `anyString()`, `anyInt()`, `any()` |
| Exact match (required alongside other matchers) | `eq(value)` |
| Assert a call happened | `verify(repository).save(order)` |
| Assert a call count | `verify(gateway, times(1)).charge(...)` |
| Assert never called | `verify(repository, never()).save(any())` |
| Capture an argument | `ArgumentCaptor.forClass(Order.class)` |
| Partial mock of a real object | `spy(realObject)` |
| Override one spy method | `doReturn(x).when(spy).method()` |

### [03 · CI/CD Integration](level-3/03-cicd-integration.md)

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

### [04 · Selenium Grid &amp; Parallel Execution](level-3/04-selenium-grid-parallel.md)

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

### [05 · Performance Testing Basics (JMeter)](level-3/05-performance-testing-jmeter.md)

| Concept | JMeter term |
|---|---|
| Concurrent virtual users | Thread Group → `num_threads` |
| Time to spin all users up | Thread Group → `ramp_time` |
| Repeats per user | Loop Controller → `loops` |
| The actual request | HTTP Request sampler |
| Pass/fail check | Response Assertion |
| Requests/sec | Throughput |
| 90th/95th/99th percentile latency | Aggregate Report percentile columns |
| Run headlessly | `jmeter -n -t plan.jmx -l results.jtl` |
| Generate HTML dashboard | `-e -o report/` |
| Error count/rate | `Err` column in summary output |

### [06 · Mobile Testing with Appium](level-3/06-mobile-testing-appium.md)

| Task | Code |
|---|---|
| Android capabilities | `UiAutomator2Options().setDeviceName(...).setApp(...)` |
| iOS capabilities | `XCUITestOptions().setDeviceName(...).setApp(...)` |
| Connect to Appium Server | `new AndroidDriver(new URL("http://127.0.0.1:4723"), options)` |
| Resource id locator (Android) | `AppiumBy.id("pkg:id/name")` |
| Cross-platform locator | `AppiumBy.accessibilityId("name")` |
| UiAutomator selector | `AppiumBy.androidUIAutomator("new UiSelector()...")` |
| Native back | `driver.pressKey(new KeyEvent(AndroidKey.BACK))` |
| Swipe/gesture | W3C `Sequence` + `PointerInput` |
| Auto-grant permissions | `autoGrantPermissions: true` capability |
| Always tear down | `driver.quit()` in `@AfterEach` |

### [07 · Database Testing from Java](level-3/07-database-testing.md)

| Task | Code |
|---|---|
| Fresh in-memory DB | `jdbc:h2:mem:<unique-name>;DB_CLOSE_DELAY=-1` |
| Parameterized insert (SQL-injection-safe) | `PreparedStatement` with `?` placeholders |
| Get generated key | `Statement.RETURN_GENERATED_KEYS` + `getGeneratedKeys()` |
| Wrap query results safely | try-with-resources on `Connection`/`Statement`/`ResultSet` |
| Assert doubles | `assertEquals(expected, actual, delta)` |
| Assert a constraint fires | `assertThrows(SQLException.class, () -> ...)` |
| Rollback-based isolation | `setAutoCommit(false)` + `rollback()` in `@AfterEach` |
| Row count from an UPDATE | return value of `executeUpdate()` |

### [08 · Framework Design Patterns](level-3/08-framework-design-patterns.md)

| Pattern | Use for | Example here |
|---|---|---|
| Factory | Creating one of several related objects behind one method | `DriverFactory.create(browser)` |
| Builder | Constructing objects with many optional fields, sensible defaults | `UserBuilder` |
| Singleton | One expensive, stateless, shared resource | `ConfigReader` |
| Strategy | Swappable algorithm behind one interface | `RetryingCheck` |
| Page Object (Level 2) | One class per UI page/component | `LoginPage` |
| Template method (via inheritance) | Shared setup/teardown, per-test specifics | `BaseTest` |

### [09 · Flaky Test Diagnosis &amp; Stabilization](level-3/09-flaky-test-diagnosis.md)

| Symptom | Likely cause | Fix direction |
|---|---|---|
| Fails ~X% of many identical runs | Timing / race | Poll for condition, not `Thread.sleep` |
| Passes alone, fails in full suite | Isolation | Remove shared/static mutable state |
| Only fails in CI, never locally | Environment or concurrency | Match CI's parallelism/resources locally |
| Fails only under `parallel="methods"` | Concurrency | Per-thread resources (Module 04) |
| Fails after another test runs first | Isolation, run-order dependency | `@TestMethodOrder` isolation test, then fix state leak |
| Diagnostic tool | JUnit 5 | `@RepeatedTest(N)` to measure real failure rate |

## Level 4 · Master

### [01 · Test Architecture at Scale](level-4/01-test-architecture-at-scale.md)

| Concept | Practice |
|---|---|
| Pyramid layers | Unit (many) → Integration (some) → E2E (few) |
| Fast feedback loop | `@Tag("unit")` + a Maven profile run on every save |
| Full/CI feedback loop | all tags, run on merge/nightly |
| Team ownership | `CODEOWNERS` + `@Tag("team-x")` |
| Cross-team coupling risk | Prefer contract tests at the boundary (Module 03) |
| Where to put a new test | Ask: fastest test that still proves the rule is *wired up*, not just correct in isolation |
| Anti-pattern to watch for | Ice-cream cone (E2E-heavy, unit-light) |

### [02 · Shift-Left &amp; Continuous Testing](level-4/02-shift-left-continuous-testing.md)

| Practice | Where it shifts the check to |
|---|---|
| IDE inline test run | While typing |
| Pre-commit hook (`unit` tests only) | Before the commit exists |
| TDD (test before implementation) | Before the code exists at all |
| Static analysis in `mvn verify` | Build time, no test execution needed |
| PR-triggered CI (Level 3 Module 03) | Before merge |
| Change-based test selection | Keeps PR feedback fast as the suite grows |
| Full suite on merge/nightly | Safety net for what fast paths might miss |

### [03 · Contract Testing (Pact)](level-4/03-contract-testing-pact.md)

| Concept | Where it lives |
|---|---|
| Consumer defines expected interaction | `@Pact` method + `PactDslWithProvider` |
| Generated contract file | `target/pacts/<consumer>-<provider>.json` |
| Provider maps state to real setup | `@State("...")` method |
| Provider replays and verifies | `@PactVerificationInvocationContextProvider` |
| Type/shape-only matcher | `like(...)`, `eachLike(...)` |
| Sharing contracts across teams | Pact Broker (or a shared artifact/folder) |
| Re-verify on every change | Broker webhook or CI job on provider deploy |

### [04 · Security Testing Basics for QA](level-4/04-security-testing-basics.md)

| Risk category | What to test |
|---|---|
| Injection (SQL, script, path) | Allowlist input validation, parameterized queries |
| Broken authentication | Expired/invalid token rejection, rate limiting |
| Broken authorization (IDOR) | User A cannot access User B's resources |
| Sensitive data exposure | Passwords/secrets never serialized in responses |
| Vulnerable dependencies | `dependency-check-maven` in CI, fail on CVSS threshold |
| Output encoding | HTML-escape user content before rendering |

### [05 · Test Metrics &amp; Quality Gates](level-4/05-test-metrics-quality-gates.md)

| Metric | Tool | What it actually tells you |
|---|---|---|
| Line/branch coverage | JaCoCo | What code *ran* during tests |
| Mutation coverage | PIT | Whether tests would *catch* an actual bug |
| Flaky rate | Custom, from CI history | Which tests erode trust in the suite |
| Pass/fail as a build gate | Surefire exit code (Level 3 Module 03) | Is the suite currently green |
| Dependency vulnerabilities | dependency-check (Level 4 Module 04) | Inherited risk, not authored code |
| Composite gate | `mvn verify` combining several plugins | One command, one pass/fail signal |

### [06 · Containerized Test Environments](level-4/06-containerized-test-environments.md)

| Task | Code |
|---|---|
| Postgres container | `new PostgreSQLContainer<>("postgres:16-alpine")` |
| Redis container | `new GenericContainer<>("redis:7-alpine").withExposedPorts(6379)` |
| Kafka container | `new KafkaContainer(DockerImageName.parse(...))` |
| Class-scoped container | `@Container static` |
| Per-test container | `@Container` (non-static) instance field |
| Get connection string | `postgres.getJdbcUrl()` / `.getUsername()` / `.getPassword()` |
| Wait for real readiness | `.waitingFor(Wait.forHttp("/health").forStatusCode(200))` |
| Tag as integration, not unit | `@Tag("integration")` (Level 4 Module 01) |

### [07 · AI-Assisted Testing](level-4/07-ai-assisted-testing.md)

| Task | AI-assist confidence | Human responsibility |
|---|---|---|
| Test class/method boilerplate | High | Fill in real assertions |
| Fuzz/edge-case input lists | High | Confirm expected behavior for each |
| Explaining a stack trace | Medium-high | Verify the hypothesis against the actual code |
| Writing the actual expected value | Low (without full context) | Always confirm manually |
| Business-rule correctness | Low | Domain knowledge stays human |
| Visual diff tooling | Tool-dependent | Confirm flagged diffs are real regressions |

### [08 · Building an In-House Test Framework](level-4/08-in-house-test-framework.md)

| Layer | Purpose | Example here |
|---|---|---|
| Base test class | Shared setup/teardown convention | `OrgBaseTest` |
| Data factory | Consistent, policy-compliant test data | `OrgTestData.uniqueEmail()` |
| Assertion helpers | Shared contract checks | `OrgApiAssertions.assertStandardErrorShape` |
| Distribution | Independently versioned artifact | Maven dependency, semantic version |
| Decision rule | "Would another team reuse this unchanged?" | Framework vs. team-local code |

### [09 · QA Leadership &amp; Strategy](level-4/09-qa-leadership-strategy.md)

| Leadership task | Grounded in |
|---|---|
| Prioritizing test investment | A stated model (section 1), not gut feel |
| Reporting to non-engineers | Business-translated metrics (section 2) |
| Writing a strategy doc | Current state → priorities → plan → metrics → non-goals |
| Interviewing QA candidates | Judgment on a flawed test, not tool trivia |
| Avoiding metric-gaming | Pair every target with a countermeasure check |
| Staying honest about the plan | A scheduled "what did we get wrong" review |

---

Go deeper with the full [Java Testing Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
