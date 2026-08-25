# 08 · Building an In-House Test Framework

Every module in this course has used off-the-shelf tools: JUnit, TestNG,
RestAssured, Cucumber, Testcontainers. At a certain scale, teams also build
*thin, custom layers* on top of these — not to replace them, but to encode
organization-specific conventions (a shared reporting format, a company
data-generation policy, a standard retry/wait budget) so every new test
class starts from a consistent, correct baseline instead of everyone
reinventing Module 08 (Level 3)'s patterns slightly differently.

## 1. When to build one (and when not to)

| Signal it's worth it | Signal it isn't (yet) |
|---|---|
| 5+ teams solving the same setup/teardown problem independently, differently | One team, one project |
| A recurring class of bug traced to inconsistent test conventions | No repeated pain point yet |
| Onboarding a new engineer to testing takes days of tribal knowledge | A new engineer can read the existing tests and follow along fine |
| Real appetite to maintain a shared library long-term | No owner willing to maintain it |

The most common failure mode of an in-house framework isn't technical — it's
building one for a problem that didn't yet exist, then having nobody
maintain it once real requirements diverge from the original guess.

## 2. Core building blocks

```java
package com.example.orgframework;

// 1. A standard base test with organization-wide conventions baked in
public abstract class OrgBaseTest {

    protected final TestContext context = new TestContext();

    @org.junit.jupiter.api.BeforeEach
    void orgSetUp(org.junit.jupiter.api.TestInfo testInfo) {
        context.testName = testInfo.getDisplayName();
        context.startedAt = java.time.Instant.now();
        OrgLogger.info("Starting: " + context.testName);
    }

    @org.junit.jupiter.api.AfterEach
    void orgTearDown() {
        java.time.Duration elapsed = java.time.Duration.between(context.startedAt, java.time.Instant.now());
        OrgLogger.info("Finished: " + context.testName + " in " + elapsed.toMillis() + "ms");
    }
}

class TestContext {
    String testName;
    java.time.Instant startedAt;
}

class OrgLogger {
    static void info(String message) {
        System.out.println("[org-framework] " + message);
    }
}
```

```java
package com.example.orgframework;

// 2. A standard data factory, wrapping org-specific conventions
// (e.g. every test email must be @test.example.com to route to a sandbox mail server)
public class OrgTestData {
    private static int counter = 0;

    public static synchronized String uniqueEmail() {
        counter++;
        return "test-user-" + counter + "-" + System.nanoTime() + "@test.example.com";
    }

    public static synchronized String uniqueUsername() {
        counter++;
        return "testuser" + counter;
    }
}
```

```java
package com.example.orgframework;

// 3. A standard assertion helper, encoding a company-wide response contract
import static org.junit.jupiter.api.Assertions.*;

public class OrgApiAssertions {
    public static void assertStandardErrorShape(String json) {
        // every API in the org returns {"error": {"code": ..., "message": ...}}
        // on failure -- centralizing this check means one place to update
        // if the contract changes, instead of hundreds of copy-pasted checks
        assertTrue(json.contains("\"error\""), "expected standard error envelope, got: " + json);
        assertTrue(json.contains("\"code\""), "expected error.code field, got: " + json);
        assertTrue(json.contains("\"message\""), "expected error.message field, got: " + json);
    }
}
```

## 3. Using the framework in a real test

```java
package com.example.orgframework;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class SignupTest extends OrgBaseTest {

    @Test
    void newUserSignupUsesOrgConventions() {
        String email = OrgTestData.uniqueEmail();
        String username = OrgTestData.uniqueUsername();

        assertTrue(email.endsWith("@test.example.com"));
        assertTrue(email.startsWith("test-user-"));
        assertTrue(username.startsWith("testuser"));

        // a real test would call the signup API/service here
    }

    @Test
    void failedSignupReturnsStandardErrorShape() {
        String errorJson = "{\"error\": {\"code\": \"DUPLICATE_EMAIL\", \"message\": \"Email already registered\"}}";
        OrgApiAssertions.assertStandardErrorShape(errorJson);
    }
}
```

I ran `OrgBaseTest`, `OrgTestData`, `OrgApiAssertions`, and `SignupTest`
exactly as above, locally with Maven/JUnit 5 (no browser, no network — pure
Java logic), and confirmed both tests pass, with the `orgSetUp`/`orgTearDown`
logging lines actually printing around each test:

```
[org-framework] Starting: failedSignupReturnsStandardErrorShape()
[org-framework] Finished: failedSignupReturnsStandardErrorShape() in 4ms
[org-framework] Starting: newUserSignupUsesOrgConventions()
[org-framework] Finished: newUserSignupUsesOrgConventions() in 1ms

Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

(JUnit 5's default method order isn't source order — as in Level 3 Module
09 — which is why `failedSignupReturnsStandardErrorShape` ran first here;
pin `@TestMethodOrder` if a demo genuinely needs a specific sequence.)

## 4. Packaging and distributing it

```xml
<!-- the framework itself, published as its own artifact -->
<groupId>com.example</groupId>
<artifactId>org-test-framework</artifactId>
<version>1.4.0</version>
```

```xml
<!-- any team's project pom.xml -->
<dependency>
    <groupId>com.example</groupId>
    <artifactId>org-test-framework</artifactId>
    <version>1.4.0</version>
    <scope>test</scope>
</dependency>
```

Versioning it as a real, independently-released artifact (not a copy-pasted
folder per repo) is what makes "we fixed a bug in `OrgApiAssertions`" mean
one release, not a manual find-and-replace across forty repositories.

## 5. A worked example of what belongs in the framework vs. the test

```java
// Belongs in the framework: generic, reusable, encodes a convention
OrgTestData.uniqueEmail();
OrgApiAssertions.assertStandardErrorShape(json);

// Does NOT belong in the framework: specific to one feature/team
// (would bloat the shared library and couple it to one team's domain)
assertEquals("SHIPPED", order.status);
checkoutPage.clickPlaceOrderButton();
```

A useful litmus test: would a *different* team, testing a *different*
feature, plausibly reuse this exact code unchanged? If yes, it's framework
material. If it names a specific business concept (`Order`, `checkoutPage`),
it belongs in that team's own test code, built *on top of* the framework.

## 6. Testing traps

!!! warning "Trap 1 — building it before the pain is real"
    A framework designed speculatively, before multiple teams have hit the
    same problem independently, tends to guess wrong about what's actually
    needed and gets reworked (or abandoned) once real usage reveals the
    actual shape of the problem. Wait for the pattern in section 1's
    signal table to actually repeat.

!!! warning "Trap 2 — the framework becomes a black box nobody understands"
    `OrgBaseTest` silently doing five things in `@BeforeEach` makes a
    failing test's true cause hard to trace for anyone who hasn't read the
    framework's source. Document what the base class does, and keep its
    responsibilities few and named clearly (this module's example does
    exactly two things: logging and timing).

!!! warning "Trap 3 — versioning drift across teams"
    If some teams pin `org-test-framework:1.2.0` and others `1.4.0`, a bug
    fixed in 1.4.0 silently doesn't apply to teams still on 1.2.0 — the
    same "which environment am I actually testing" confusion as Level 3
    Module 04's Grid capacity trap, at the dependency-version level instead.
    Track and periodically enforce a minimum supported version.

!!! warning "Trap 4 — encoding one team's opinion as an organization-wide rule"
    `OrgApiAssertions.assertStandardErrorShape` assumes every team's API
    genuinely follows the same error contract. If one team's API
    legitimately differs (a third-party integration constraint, a legacy
    system), forcing the shared assertion onto it produces false failures
    that erode trust in the shared framework generally.

!!! warning "Trap 5 — no deprecation path"
    Changing `OrgTestData.uniqueEmail()`'s format without a transition
    period breaks every test across every consuming repo simultaneously the
    moment the new version is adopted. Treat framework changes with the
    same backward-compatibility discipline as a public API — deprecate,
    give a migration window, then remove.

## Cheat sheet

| Layer | Purpose | Example here |
|---|---|---|
| Base test class | Shared setup/teardown convention | `OrgBaseTest` |
| Data factory | Consistent, policy-compliant test data | `OrgTestData.uniqueEmail()` |
| Assertion helpers | Shared contract checks | `OrgApiAssertions.assertStandardErrorShape` |
| Distribution | Independently versioned artifact | Maven dependency, semantic version |
| Decision rule | "Would another team reuse this unchanged?" | Framework vs. team-local code |

## Exercise

1. Build `OrgBaseTest`, `OrgTestData`, `OrgApiAssertions`, and `SignupTest`
   exactly as above, run it, and confirm the logging output appears around
   each test.
2. Add `OrgTestData.uniquePhoneNumber()` following the same convention as
   `uniqueEmail()`, and a test proving uniqueness across 100 calls (no
   duplicates in a `Set`).
3. Write one test that deliberately calls `assertStandardErrorShape` against
   a malformed JSON error (missing `code`) and confirm it fails with a
   message that clearly states which field was missing.
4. Draft (in a comment, no need to actually publish) a `1.5.0` changelog
   entry for a breaking change to `OrgApiAssertions`, including the
   deprecation window you'd give consuming teams before removing the old
   behavior.
5. Using the litmus test from section 5, sort five pieces of test code from
   earlier modules in this course into "framework material" versus
   "team/feature-specific," with one sentence justifying each.
