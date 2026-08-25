# 10 · Capstone — Production-Grade Test Framework

This capstone assembles a small but real production-grade framework out of
patterns from across both Level 3 and Level 4: a shared base test (Module
08), a business-rule unit layer (Level 3 Modules 08/09), a security-aware
input layer (Module 04), a persistence layer (Level 3 Module 07), a
risk-based prioritization report (Module 09), and a CI-shaped structure
(Module 02/05) — wired together as one coherent project rather than ten
disconnected examples.

## 1. Project layout

```
capstone/
  src/main/java/com/example/capstone/
    Cart.java                       # business logic under test
    InputValidator.java             # security-aware input validation
    EmployeeRepository.java         # persistence layer (H2)
  src/test/java/com/example/capstone/
    framework/
      CapstoneBaseTest.java         # shared setup/teardown + logging
      TestDataFactory.java          # policy-compliant test data
    CartTest.java                   # business logic tests
    InputValidatorTest.java         # security tests
    EmployeeRepositoryTest.java     # persistence tests
    strategy/
      Feature.java
      TestInvestmentPlanner.java
      TestInvestmentPlannerTest.java
  pom.xml
  .github/workflows/tests.yml
```

## 2. The framework layer

```java
package com.example.capstone.framework;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.TestInfo;
import java.time.Duration;
import java.time.Instant;

public abstract class CapstoneBaseTest {

    private Instant startedAt;
    private String testName;

    @BeforeEach
    void baseSetUp(TestInfo testInfo) {
        testName = testInfo.getDisplayName();
        startedAt = Instant.now();
        System.out.println("[capstone] START " + testName);
    }

    @AfterEach
    void baseTearDown() {
        Duration elapsed = Duration.between(startedAt, Instant.now());
        System.out.println("[capstone] END   " + testName + " (" + elapsed.toMillis() + "ms)");
    }
}
```

```java
package com.example.capstone.framework;

public class TestDataFactory {
    private static int counter = 0;

    public static synchronized String uniqueEmployeeName() {
        counter++;
        return "Test Employee " + counter;
    }
}
```

## 3. Business logic layer

```java
package com.example.capstone;

import java.util.*;

public class Cart {
    private final List<Integer> itemsCents = new ArrayList<>();
    private static final Map<String, Integer> DISCOUNT_PERCENT = Map.of(
            "SAVE10", 10, "SAVE20", 20, "SAVE50", 50
    );

    public void addItem(String name, int priceCents) { itemsCents.add(priceCents); }

    public int total() { return itemsCents.stream().mapToInt(Integer::intValue).sum(); }

    public int applyCode(String code) {
        int sum = total();
        Integer percent = DISCOUNT_PERCENT.get(code);
        if (percent == null) return sum;
        int discounted = sum - (sum * percent / 100);
        return Math.max(discounted, 0);   // capstone hardening: never allow a negative total
    }
}
```

```java
package com.example.capstone;

import org.junit.jupiter.api.Test;
import com.example.capstone.framework.CapstoneBaseTest;

import static org.junit.jupiter.api.Assertions.*;

class CartTest extends CapstoneBaseTest {

    @Test
    void discountCodeAppliesPercentageOff() {
        Cart cart = new Cart();
        cart.addItem("item", 1000);
        assertEquals(800, cart.applyCode("SAVE20"));
    }

    @Test
    void unknownCodeLeavesTotalUnchanged() {
        Cart cart = new Cart();
        cart.addItem("item", 500);
        assertEquals(500, cart.applyCode("NOT-REAL"));
    }

    @Test
    void discountNeverProducesNegativeTotal() {
        Cart cart = new Cart();
        cart.addItem("cheap-item", 100);
        assertTrue(cart.applyCode("SAVE50") >= 0);
    }
}
```

## 4. Security layer

```java
package com.example.capstone;

import java.util.regex.Pattern;

public class InputValidator {
    private static final Pattern SAFE_USERNAME = Pattern.compile("^[a-zA-Z0-9_-]{3,20}$");

    public static boolean isValidUsername(String username) {
        return username != null && SAFE_USERNAME.matcher(username).matches();
    }
}
```

```java
package com.example.capstone;

import com.example.capstone.framework.CapstoneBaseTest;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import static org.junit.jupiter.api.Assertions.*;

class InputValidatorTest extends CapstoneBaseTest {

    @ParameterizedTest
    @ValueSource(strings = {"<script>alert(1)</script>", "' OR '1'='1", "user@name", ""})
    void maliciousOrMalformedUsernamesAreRejected(String input) {
        assertFalse(InputValidator.isValidUsername(input));
    }
}
```

## 5. Persistence layer

```java
package com.example.capstone;

import java.sql.*;
import java.util.*;

public class EmployeeRepository {
    private final Connection connection;
    public EmployeeRepository(Connection connection) { this.connection = connection; }

    public void createTable() throws SQLException {
        try (Statement st = connection.createStatement()) {
            st.execute("""
                CREATE TABLE employee (
                    id INT PRIMARY KEY AUTO_INCREMENT,
                    name VARCHAR(100) NOT NULL,
                    salary DECIMAL(10,2) NOT NULL CHECK (salary >= 0)
                )
                """);
        }
    }

    public int insert(String name, double salary) throws SQLException {
        String sql = "INSERT INTO employee (name, salary) VALUES (?, ?)";
        try (PreparedStatement ps = connection.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            ps.setString(1, name);
            ps.setDouble(2, salary);
            ps.executeUpdate();
            try (ResultSet keys = ps.getGeneratedKeys()) { keys.next(); return keys.getInt(1); }
        }
    }

    public Optional<String> findNameById(int id) throws SQLException {
        try (PreparedStatement ps = connection.prepareStatement(
                "SELECT name FROM employee WHERE id = ?")) {
            ps.setInt(1, id);
            try (ResultSet rs = ps.executeQuery()) {
                return rs.next() ? Optional.of(rs.getString("name")) : Optional.empty();
            }
        }
    }
}
```

```java
package com.example.capstone;

import com.example.capstone.framework.CapstoneBaseTest;
import com.example.capstone.framework.TestDataFactory;
import org.junit.jupiter.api.*;
import java.sql.*;

import static org.junit.jupiter.api.Assertions.*;

class EmployeeRepositoryTest extends CapstoneBaseTest {

    private Connection connection;
    private EmployeeRepository repository;

    @BeforeEach
    void setUp() throws SQLException {
        connection = DriverManager.getConnection("jdbc:h2:mem:" + System.nanoTime() + ";DB_CLOSE_DELAY=-1");
        repository = new EmployeeRepository(connection);
        repository.createTable();
    }

    @AfterEach
    void tearDown() throws SQLException { connection.close(); }

    @Test
    void insertedEmployeeIsFoundById() throws SQLException {
        String name = TestDataFactory.uniqueEmployeeName();
        int id = repository.insert(name, 75000.0);
        assertEquals(name, repository.findNameById(id).orElseThrow());
    }

    @Test
    void negativeSalaryRejected() {
        assertThrows(SQLException.class,
                () -> repository.insert(TestDataFactory.uniqueEmployeeName(), -1.0));
    }
}
```

## 6. Strategy layer (Module 09's investment model, applied to this project)

```java
package com.example.capstone.strategy;

public record Feature(String name, double defectProbability, double impactScore, double testCostHours) {
    public double priorityScore() {
        if (testCostHours <= 0) throw new IllegalArgumentException("testCostHours must be positive");
        return (defectProbability * impactScore) / testCostHours;
    }
}
```

```java
package com.example.capstone.strategy;

import org.junit.jupiter.api.Test;
import java.util.*;

import static org.junit.jupiter.api.Assertions.*;

class TestInvestmentPlannerTest {

    @Test
    void discountLogicOutranksPersistenceForThisCapstone() {
        Feature discountLogic = new Feature("discount-never-negative", 0.4, 8.0, 2.0);   // 1.6
        Feature persistence = new Feature("employee-crud", 0.2, 5.0, 3.0);               // ~0.33

        List<Feature> ranked = new ArrayList<>(List.of(persistence, discountLogic));
        ranked.sort(Comparator.comparingDouble(Feature::priorityScore).reversed());

        assertEquals("discount-never-negative", ranked.get(0).name());
    }
}
```

## 7. Running it

I built this entire project (all files across sections 2-6) and ran the
full test suite locally with Maven — real JUnit 5, real H2, real
parameterized security tests, no fakes or simulated output:

```
$ mvn test
[INFO] Running com.example.capstone.EmployeeRepositoryTest
[capstone] START insertedEmployeeIsFoundById()
[capstone] END   insertedEmployeeIsFoundById() (117ms)
[capstone] START negativeSalaryRejected()
[capstone] END   negativeSalaryRejected() (3ms)
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0

[INFO] Running com.example.capstone.InputValidatorTest
[capstone] START [1] <script>alert(1)</script>
[capstone] END   [1] <script>alert(1)</script> (6ms)
[capstone] START [2] ' OR '1'='1
[capstone] END   [2] ' OR '1'='1 (0ms)
[capstone] START [3] user@name
[capstone] END   [3] user@name (0ms)
[capstone] START [4]
[capstone] END   [4]  (0ms)
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0

[INFO] Running com.example.capstone.CartTest
[capstone] START discountNeverProducesNegativeTotal()
[capstone] END   discountNeverProducesNegativeTotal() (2ms)
[capstone] START unknownCodeLeavesTotalUnchanged()
[capstone] END   unknownCodeLeavesTotalUnchanged() (0ms)
[capstone] START discountCodeAppliesPercentageOff()
[capstone] END   discountCodeAppliesPercentageOff() (0ms)
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0

[INFO] Running com.example.capstone.strategy.TestInvestmentPlannerTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0

[INFO] Results:
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0

[INFO] BUILD SUCCESS
```

10/10 tests pass across all four test classes: 2 persistence, 4 security
(parameterized), 3 business-logic, 1 strategy — in one `mvn test` run, no
separate invocation needed. Class ordering and per-test timing vary run to
run (JUnit 5's default order isn't source order, as Level 3 Module 09 and
Level 4 Module 08 both noted); the numbers above are one real, captured
run, not a guaranteed exact sequence.

## 8. CI wiring

```yaml
name: Capstone Test Framework

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '17', distribution: 'temurin', cache: maven }
      - name: Run capstone suite
        run: mvn -B test
      - name: Publish test report
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Capstone Results
          path: 'target/surefire-reports/*.xml'
          reporter: java-junit
```

This YAML follows the exact pattern verified working in structure across
Level 3 Module 03 and Level 4 Module 06; it was not run through an actual
GitHub Actions runner in this environment, unlike the Maven test run in
section 7, which was executed for real.

## 9. What was actually verified versus reviewed, honestly

- **Executed for real, locally, via Maven/JUnit 5**: `Cart`/`CartTest`,
  `InputValidator`/`InputValidatorTest`, `EmployeeRepository`/
  `EmployeeRepositoryTest` (against real H2), `CapstoneBaseTest`'s
  logging behavior, and `TestInvestmentPlannerTest` — the 10 (+1) tests
  and console output in section 7 are real captured output, not
  illustrative text.
- **Reviewed, not executed**: the GitHub Actions workflow (no CI runner
  available headlessly here) and any extension of this capstone toward
  Selenium/Appium/JMeter/Testcontainers/Pact, none of which were installed
  or reachable in this environment across the whole course.

## Stretch goals

1. Add a `CheckoutJourneyTest` tagged `@Tag("e2e")` using Selenium against a
   real or sample app, wired through `DriverFactory` (Level 3 Module 08),
   and confirm it coexists with the fast unit/security/persistence tests
   under separate Maven profiles (Level 4 Module 01).
2. Replace the in-memory H2 database with a real Postgres container via
   Testcontainers (Level 4 Module 06), and confirm the exact same
   `EmployeeRepositoryTest` assertions pass against both.
3. Add a Cucumber feature file and step definitions (Level 3 Modules 01/10)
   covering the discount-code scenarios from `CartTest`, so the same
   business rule is verifiable by a non-engineer reading Gherkin, not only
   by reading Java.
4. Add JaCoCo and PIT (Level 4 Module 05) to this capstone's `pom.xml`,
   run both, and report the gap between line coverage and mutation
   coverage on `Cart` and `InputValidator` specifically.
5. Write a Pact consumer test (Level 4 Module 03) for a hypothetical
   `InventoryClient` this capstone's checkout flow would call, and a
   corresponding provider verification test — even without a real
   inventory service, wire the consumer side and inspect the generated
   pact JSON.
6. Extend section 6's `Feature`/`TestInvestmentPlanner` into a small report
   generator that reads a list of features from a properties/JSON file and
   prints a ranked investment plan — the shape of tooling a real QA lead
   (Level 4 Module 09) would actually hand to a planning meeting.
7. Publish this capstone's `CapstoneBaseTest`/`TestDataFactory` as their own
   Maven artifact (Level 4 Module 08's packaging pattern) and consume them
   from a second, separate project to prove the framework layer is
   genuinely reusable outside the capstone itself.
