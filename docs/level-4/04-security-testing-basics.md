# 04 · Security Testing Basics for QA

Security testing isn't a separate discipline bolted onto functional testing
— it's the same skill (find the input the developer didn't consider) aimed
at a different question: not "does this work," but "can this be made to do
something it shouldn't." This module covers what a QA engineer, not a
dedicated security specialist, can reasonably own.

## 1. Input validation — the most common QA-owned security check

```java
package com.example.security;

import java.util.regex.Pattern;

public class InputValidator {

    private static final Pattern SAFE_USERNAME = Pattern.compile("^[a-zA-Z0-9_-]{3,20}$");

    public static boolean isValidUsername(String username) {
        return username != null && SAFE_USERNAME.matcher(username).matches();
    }

    public static String sanitizeForDisplay(String input) {
        if (input == null) return "";
        return input
                .replace("&", "&amp;")
                .replace("<", "&lt;")
                .replace(">", "&gt;")
                .replace("\"", "&quot;")
                .replace("'", "&#x27;");
    }
}
```

```java
package com.example.security;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import static org.junit.jupiter.api.Assertions.*;

class InputValidatorTest {

    @ParameterizedTest
    @ValueSource(strings = {
        "<script>alert('xss')</script>",
        "'; DROP TABLE users; --",
        "../../../etc/passwd",
        "admin' OR '1'='1"
    })
    void maliciousUsernamesAreRejected(String maliciousInput) {
        assertFalse(InputValidator.isValidUsername(maliciousInput),
                "should reject: " + maliciousInput);
    }

    @Test
    void legitimateUsernameIsAccepted() {
        assertTrue(InputValidator.isValidUsername("alice_92"));
    }

    @Test
    void scriptTagIsEscapedForDisplay() {
        String result = InputValidator.sanitizeForDisplay("<script>alert('xss')</script>");
        assertFalse(result.contains("<script>"));
        assertEquals("&lt;script&gt;alert(&#x27;xss&#x27;)&lt;/script&gt;", result);
    }

    @Test
    void sqlInjectionAttemptFailsUsernameFormat() {
        assertFalse(InputValidator.isValidUsername("admin'; DROP TABLE users; --"));
    }
}
```

I ran this exact validator and test class locally with Maven/JUnit 5 (pure
Java, no browser or live target involved) and all tests passed:

```
Tests run: 7, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

Testing the *allowlist* regex (only known-safe characters permitted) rather
than trying to enumerate every dangerous pattern is the pattern to prefer —
a denylist of "bad" substrings is a losing, endless game against creative
encodings; an allowlist of "known good" shapes is finite and provable.

## 2. SQL injection — testing that parameterization actually holds

```java
// The vulnerable version (never ship this -- shown to demonstrate the test)
public List<Employee> vulnerableFindByName(Connection conn, String name) throws SQLException {
    String sql = "SELECT * FROM employee WHERE name = '" + name + "'";   // string concatenation
    try (Statement st = conn.createStatement(); ResultSet rs = st.executeQuery(sql)) {
        // ...
    }
}

// The safe version, matching Level 3 Module 07's PreparedStatement pattern
public List<Employee> safeFindByName(Connection conn, String name) throws SQLException {
    String sql = "SELECT * FROM employee WHERE name = ?";
    try (PreparedStatement ps = conn.prepareStatement(sql)) {
        ps.setString(1, name);
        // ...
    }
    return List.of();
}
```

```java
@Test
void safeQueryIsNotVulnerableToInjection() throws SQLException {
    // reuses the H2 setup pattern from Level 3 Module 07
    repository.insert("Ada Lovelace", "Engineering", 95000.00);

    // classic injection payload: if concatenated, this becomes
    // WHERE name = '' OR '1'='1' -- returning every row
    List<Employee> result = repository.safeFindByName(connection, "' OR '1'='1");

    assertTrue(result.isEmpty(), "parameterized query correctly found no match for the injection payload");
}
```

The test's assertion is the point: a vulnerable implementation returns
*every* employee for this input (the injected `OR '1'='1'` always
evaluates true); the safe, parameterized version correctly treats the whole
string as a literal name nobody has, and returns nothing. This is a case
where the "wrong" (empty) result is the *secure* result — worth calling out
explicitly in test comments so a future reader doesn't "fix" it.

## 3. Authentication and session basics a QA suite should check

```java
@Test
void expiredTokenIsRejected() {
    String expiredToken = tokenService.generateWithExpiry(Duration.ofMillis(-1));
    assertThrows(UnauthorizedException.class, () -> authService.validate(expiredToken));
}

@Test
void tokenForOneUserCannotAccessAnotherUsersResource() {
    String userAToken = tokenService.generateFor("user-a");
    assertThrows(ForbiddenException.class,
            () -> orderService.getOrder(userAToken, "order-belonging-to-user-b"));
}

@Test
void passwordIsNeverReturnedInApiResponse() {
    UserResponse response = userService.getProfile("user-a");
    assertNull(response.password(), "password field must never be serialized");
}

@Test
void bruteForceLoginIsRateLimited() {
    for (int i = 0; i < 10; i++) {
        loginService.attempt("victim", "wrong-password-" + i);
    }
    assertThrows(TooManyRequestsException.class,
            () -> loginService.attempt("victim", "wrong-password-final"));
}
```

These four tests correspond to four of the most common real-world defects:
broken expiry, broken authorization (an authenticated user reaching
*someone else's* data — the single most common API security bug),
sensitive data leaking through a response, and no rate limiting on
authentication endpoints.

## 4. Dependency scanning — testing what you didn't write

```xml
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>9.2.0</version>
    <executions>
        <execution>
            <goals><goal>check</goal></goals>
        </execution>
    </executions>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>
    </configuration>
</plugin>
```

```bash
mvn verify
```

```
[INFO] One or more dependencies were identified with vulnerabilities:
[INFO] jackson-databind-2.9.8.jar: CVE-2019-12384 (CVSS 8.1)
[ERROR] Failed to execute goal ... Dependency-Check Failure
```

The vast majority of production security incidents trace back to a known,
already-patched vulnerability in a dependency, not a novel exploit against
custom code — scanning `pom.xml`/`package-lock.json`/etc. against CVE
databases on every build is a very high-leverage security test for very
low authoring effort.

I ran the `InputValidator`/`InputValidatorTest` code locally as shown
(section 1); the SQL injection test in section 2 is reviewed as an
extension of Level 3 Module 07's already-executed H2 repository pattern but
not re-run separately here; the authentication tests (section 3, no real
`tokenService`/`authService` implementation provided) and dependency-check
plugin (section 4, requires network access to a CVE database) are reviewed
against their documented APIs/behavior, not executed.

## 5. Testing traps

!!! warning "Trap 1 — denylist validation that misses an encoding"
    Blocking the literal string `<script>` misses `%3Cscript%3E` (URL
    encoded), `&#60;script&#62;` (HTML entity encoded), or mixed case
    `<ScRiPt>`. Validate structurally (an allowlist regex, section 1)
    rather than pattern-matching known-bad strings.

!!! warning "Trap 2 — testing authentication but not authorization"
    Confirming a user *can log in* says nothing about whether they can
    reach data that isn't theirs once logged in. `IDOR` (Insecure Direct
    Object Reference — guessing or incrementing another user's resource id)
    is one of the most common real vulnerabilities and needs its own
    explicit test, exactly like section 3's
    `tokenForOneUserCannotAccessAnotherUsersResource`.

!!! warning "Trap 3 — security tests that only run manually, occasionally"
    A security checklist run once before a big release, then forgotten, is
    a checklist that misses every regression introduced afterward. The
    tests in this module belong in the same CI suite as functional tests
    (Level 3 Module 03), not a separate, occasional audit.

!!! warning "Trap 4 — sensitive data logged during test debugging"
    A test that prints a real (even test-environment) password, token, or
    PII value to console output for debugging can end up captured in CI
    logs, which are often less access-controlled than the systems they're
    testing. Redact or use synthetic data-only fixtures, and check what a
    failed assertion's message actually prints.

!!! warning "Trap 5 — false confidence from one scan tool"
    A clean SpotBugs and dependency-check run (Level 4 Module 02 and this
    module) doesn't mean "secure" — static analysis and known-CVE scanning
    each catch specific categories and miss business-logic flaws (like a
    missing authorization check) entirely. Treat these as one layer among
    several, not a certification.

## Cheat sheet

| Risk category | What to test |
|---|---|
| Injection (SQL, script, path) | Allowlist input validation, parameterized queries |
| Broken authentication | Expired/invalid token rejection, rate limiting |
| Broken authorization (IDOR) | User A cannot access User B's resources |
| Sensitive data exposure | Passwords/secrets never serialized in responses |
| Vulnerable dependencies | `dependency-check-maven` in CI, fail on CVSS threshold |
| Output encoding | HTML-escape user content before rendering |

## Exercise

1. Build `InputValidator` and `InputValidatorTest` exactly as above, run
   them, and confirm all 7 pass.
2. Extend Level 3 Module 07's `EmployeeRepository` with `safeFindByName`
   (parameterized) and a deliberately vulnerable
   `vulnerableFindByName` (string concatenation), and write a test proving
   the injection payload behaves differently between the two — passing
   against the safe version, and (carefully, as a documented negative
   example) returning unintended rows against the vulnerable one.
3. Write the four authentication/authorization tests from section 3 against
   real or stubbed `tokenService`/`authService`/`orderService`/`loginService`
   implementations.
4. Add `dependency-check-maven` to a project's `pom.xml`, run
   `mvn verify`, and report what it finds (even zero findings is a valid,
   reportable result).
5. Take one form or API endpoint from earlier in this course (Level 1-2)
   and design (write test code for, even against a stub) three security
   tests for it: one input-validation test, one authorization test, and one
   sensitive-data-exposure test.
