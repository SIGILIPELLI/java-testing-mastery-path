# 05 · Test Data Management

Hard-coded test data is the second most common reason a suite rots (locators
being the first). `"tomsmith"` appears in twelve tests; the QA environment is
refreshed; eleven tests fail for a reason that has nothing to do with the
application. This module covers getting data *out* of test code — into
properties, CSV, JSON and Excel — generating it when it should be unique, and
building it in a way that stays readable.

## 1. The four kinds of test data

| Kind | Example | Where it belongs |
|---|---|---|
| **Environment config** | Base URL, timeouts, browser | `config.properties` + system properties |
| **Secrets** | Passwords, API keys | Environment variables — never in the repo |
| **Fixed reference data** | The tiered discount boundaries | A data provider or CSV, versioned with the tests |
| **Generated per-run data** | A new user's email, an order id | A faker or builder, created at runtime |

Confusing these is what produces both flaky suites and leaked credentials.
The rule of thumb: **if two runs must see the same value, externalise it; if
two runs must see different values, generate it.**

## 2. Configuration with properties

`src/test/resources/config.properties`:

```properties
base.url=https://the-internet.herokuapp.com
api.base.url=https://jsonplaceholder.typicode.com
browser=chrome
headless=true
timeout.seconds=10
```

```java
package com.example.config;

import java.io.InputStream;
import java.util.Properties;

public final class Config {

    private static final Properties PROPS = load();

    private static Properties load() {
        Properties props = new Properties();
        try (InputStream in = Config.class.getClassLoader()
                .getResourceAsStream("config.properties")) {
            if (in == null) {
                throw new IllegalStateException("config.properties not on the classpath");
            }
            props.load(in);
        } catch (Exception e) {
            throw new IllegalStateException("Could not load config.properties", e);
        }
        return props;
    }

    /** System property wins, then the file, then the supplied default. */
    public static String get(String key, String fallback) {
        String override = System.getProperty(key);
        if (override != null && !override.isBlank()) return override;
        return PROPS.getProperty(key, fallback);
    }

    public static String baseUrl()   { return get("base.url", "http://localhost:8080"); }
    public static String browser()   { return get("browser", "chrome"); }
    public static boolean headless() { return Boolean.parseBoolean(get("headless", "true")); }
    public static int timeout()      { return Integer.parseInt(get("timeout.seconds", "10")); }

    private Config() { }
}
```

```bash
mvn test                                    # uses config.properties
mvn test -Dbrowser=firefox -Dheadless=false # overridden for one run
mvn test -Dbase.url=https://staging.example.com
```

That precedence order — system property, then file, then default — is the
convention every CI pipeline expects. It means one command switches the whole
suite to a different environment.

!!! warning "Secrets are not configuration"
    Never put a password in `config.properties` — it is in the repository
    forever, including in every fork and every stale branch. Read it from the
    environment:
    ```java
    String password = System.getenv("QA_PASSWORD");
    Assumptions.assumeTrue(password != null, "QA_PASSWORD not set -- skipping");
    ```
    Skipping honestly beats failing mysteriously or hard-coding a secret.

## 3. CSV-driven tests

`src/test/resources/logins.csv`:

```csv
username,password,expectedMessage
tomsmith,SuperSecretPassword!,You logged into a secure area!
wronguser,SuperSecretPassword!,Your username is invalid!
tomsmith,wrongpassword,Your password is invalid!
,SuperSecretPassword!,Your username is invalid!
```

### JUnit 5 — built in, no library needed

```java
package com.example.data;

import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvFileSource;
import org.junit.jupiter.params.provider.CsvSource;

import static org.junit.jupiter.api.Assertions.*;

class CsvDataTest {

    @ParameterizedTest(name = "[{index}] {0}/{1} -> {2}")
    @CsvFileSource(resources = "/logins.csv", numLinesToSkip = 1)
    void loginOutcomes(String username, String password, String expected) {
        assertEquals(expected, new FakeAuth().login(username, password));
    }

    @ParameterizedTest(name = "{0} spends {1} -> {2}% discount")
    @CsvSource({
            "small order,  999.0,  0",
            "tier 1 edge, 1000.0,  5",
            "tier 2 edge, 5000.0, 10",
            "tier 3 edge,10000.0, 15",
    })
    void discountBoundaries(String label, double amount, int expectedPercent) {
        assertEquals(expectedPercent, new Discounts().percentFor(amount),
                label + " (amount " + amount + ")");
    }

    static class FakeAuth {
        String login(String user, String pass) {
            if (user == null || !user.equals("tomsmith")) return "Your username is invalid!";
            if (!"SuperSecretPassword!".equals(pass))     return "Your password is invalid!";
            return "You logged into a secure area!";
        }
    }

    static class Discounts {
        int percentFor(double amount) {
            if (amount >= 10000) return 15;
            if (amount >= 5000)  return 10;
            if (amount >= 1000)  return 5;
            return 0;
        }
    }
}
```

```
[1] tomsmith/SuperSecretPassword! -> You logged into a secure area! PASSED
[2] wronguser/SuperSecretPassword! -> Your username is invalid! PASSED
[3] tomsmith/wrongpassword -> Your password is invalid! PASSED
[4] null/SuperSecretPassword! -> Your username is invalid! PASSED
Tests run: 8, Failures: 0, Errors: 0, Skipped: 0
```

Note row 4: an empty CSV cell arrives as `null`, not `""`. Use
`emptyValue = ""` on the annotation if you need the empty string, and be
explicit about which one your requirement means.

### TestNG — read the file into a provider

```java
import com.opencsv.CSVReader;
import org.testng.annotations.DataProvider;

@DataProvider(name = "logins")
public Object[][] logins() throws Exception {
    try (CSVReader reader = new CSVReader(
            new InputStreamReader(getClass().getResourceAsStream("/logins.csv")))) {
        List<String[]> rows = reader.readAll();
        return rows.subList(1, rows.size()).stream()   // drop the header
                   .map(r -> new Object[] { r[0], r[1], r[2] })
                   .toArray(Object[][]::new);
    }
}
```

## 4. JSON test data

JSON handles nested data that CSV cannot.

`src/test/resources/users.json`:

```json
[
  { "name": "Standard user", "email": "std@example.com", "roles": ["viewer"] },
  { "name": "Admin user",    "email": "adm@example.com", "roles": ["viewer", "admin"] }
]
```

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.List;

public record TestUser(String name, String email, List<String> roles) { }

public static List<TestUser> loadUsers() throws Exception {
    ObjectMapper mapper = new ObjectMapper();
    try (InputStream in = TestData.class.getResourceAsStream("/users.json")) {
        return mapper.readValue(in, mapper.getTypeFactory()
                .constructCollectionType(List.class, TestUser.class));
    }
}
```

## 5. Excel, when the business owns the data

Business analysts maintain test cases in Excel. Apache POI reads it.

```java
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;

public static Object[][] readSheet(String path, String sheetName) throws Exception {
    try (Workbook workbook = new XSSFWorkbook(new FileInputStream(path))) {
        Sheet sheet = workbook.getSheet(sheetName);
        DataFormatter formatter = new DataFormatter();

        int rows = sheet.getLastRowNum();                       // excludes the header
        int cols = sheet.getRow(0).getLastCellNum();
        Object[][] data = new Object[rows][cols];

        for (int r = 1; r <= rows; r++) {
            for (int c = 0; c < cols; c++) {
                data[r - 1][c] = formatter.formatCellValue(sheet.getRow(r).getCell(c));
            }
        }
        return data;
    }
}
```

`DataFormatter` returns every cell as the string the user sees, which avoids
the classic Excel trap where `1000` arrives as `1000.0` and the assertion
fails on a formatting difference.

!!! warning "Excel is the least maintainable option"
    A binary file cannot be diffed or reviewed in a pull request, merges
    conflict destructively, and one dragged cell silently changes a test.
    Use it only when the business genuinely owns the data. CSV gives you 90%
    of the benefit and works with git.

## 6. Generated data

```java
import net.datafaker.Faker;

Faker faker = new Faker();

String name    = faker.name().fullName();
String email   = faker.internet().emailAddress();
String phone   = faker.phoneNumber().cellPhone();
String company = faker.company().name();
String street  = faker.address().streetAddress();
```

For a value that must be unique *and* traceable back to the run that made it:

```java
String email = "qa+" + System.currentTimeMillis() + "@example.com";
String email2 = "qa+" + UUID.randomUUID() + "@example.com";
```

!!! warning "Random data makes failures unreproducible"
    A test that fails once on a faker-generated name you can never see again
    is worthless. Log every generated value, or seed the faker
    (`new Faker(new Random(42))`) so the run is repeatable. Faker is for
    *uniqueness*, not for *coverage* — coverage comes from the deliberate
    boundary values you chose in Level 1.

## 7. The builder pattern

```java
public class UserBuilder {

    private String email = "default@example.com";
    private String role  = "viewer";
    private boolean active = true;

    public static UserBuilder aUser()          { return new UserBuilder(); }
    public UserBuilder withEmail(String e)     { this.email = e;   return this; }
    public UserBuilder withRole(String r)      { this.role = r;    return this; }
    public UserBuilder inactive()              { this.active = false; return this; }

    public TestUser build() {
        return new TestUser(email, role, active);
    }
}

// In a test -- only the relevant field is stated
TestUser admin    = UserBuilder.aUser().withRole("admin").build();
TestUser disabled = UserBuilder.aUser().inactive().build();
```

The point is not brevity. It is that `aUser().inactive()` tells the reader
that *inactive* is the only thing this test cares about, while
`new TestUser("x@y.com", "viewer", false)` hides that in a positional
argument list.

## 8. Cleaning up

| Strategy | How | Use when |
|---|---|---|
| Create per test | Fresh entity in setup, delete in teardown | Best default |
| Unique keys | Timestamped emails; never delete | Data volume is acceptable |
| Transaction rollback | Wrap in a transaction, roll back | Backend/API tests with DB access |
| Environment reset | Restore a DB snapshot before the run | Nightly runs on a private environment |

```java
@AfterEach
void cleanUp() {
    if (createdUserId != null) {
        given().spec(apiSpec).delete("/users/" + createdUserId);
    }
}
```

Deleting through the API in teardown is faster and more reliable than any UI
cleanup, and it runs even when the UI test failed halfway.

## 9. Testing traps

!!! warning "Trap 1 — order-dependent data"
    Test A creates the user that Test B logs in as. Run them in parallel, or
    alphabetically, or alone, and B fails. Every test must create what it
    needs.

!!! warning "Trap 2 — the shared 'test user' account"
    One account used by twenty tests means twenty tests fight over its cart,
    its preferences and its session. It is the single most common cause of
    "passes locally, fails on CI".

!!! warning "Trap 3 — data files outside `src/test/resources`"
    A path like `"./data/logins.csv"` resolves relative to the working
    directory, which differs between your IDE, Maven and CI. Put files on the
    classpath and load them with `getResourceAsStream("/logins.csv")`.

!!! warning "Trap 4 — production data in test environments"
    Copying a live customer table into QA is a data-protection breach in most
    jurisdictions, regardless of intent. Mask or synthesise it, and escalate
    if you are asked to do otherwise.

## Cheat sheet

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

## Exercise

1. Build the `Config` class from section 2 with `config.properties`, and
   prove the override order works: run the suite three times — default,
   `-Dbrowser=firefox`, and with a system property that is also in the file.
2. Move every hard-coded URL and credential out of your Module 01 page
   objects into `Config`. Then run against a URL that does not exist and
   confirm no test contains the old value.
3. Create `logins.csv` with at least six rows including an empty username and
   an over-long one, and drive your login tests from it with
   `@CsvFileSource`. Explain what the empty cell becomes in Java.
4. Convert the same data set to a TestNG `@DataProvider` reading the same
   CSV, so both frameworks share one file.
5. Add `users.json` with three users of different roles, load it with
   Jackson into a record, and write one test per user.
6. Write `UserBuilder` and use it in two tests. Then write the same two tests
   with a plain constructor and compare which version communicates intent.
7. Generate a unique registration email with Faker, log it, and add an
   `@AfterEach` that deletes the account through the API. Force a failure
   mid-test and confirm the cleanup still runs.
8. Find every hard-coded value left in your suite. For each one, decide which
   of the four categories in section 1 it belongs to, and move it there.
