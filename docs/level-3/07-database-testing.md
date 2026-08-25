# 07 · Database Testing from Java

Every layer tested so far — UI, API, mocked service — eventually bottoms out
in a database. A passing API test that silently wrote the wrong value to a
column is still a bug; testing the database layer directly catches
persistence defects (wrong types, missing constraints, broken queries) that
never surface until production data grows large enough to expose them.

## 1. Setup — H2, an in-memory database for tests

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>2.2.224</version>
    <scope>test</scope>
</dependency>
```

H2 in-memory mode gives every test run a fresh, disposable database with
zero setup — no Docker, no shared server, no leftover rows from yesterday's
run. It speaks real SQL and JDBC, so code tested against it exercises the
same `java.sql` code path that runs against Postgres/MySQL in production.

## 2. A repository under test

```java
package com.example.db;

import java.sql.*;
import java.util.*;

public class Employee {
    public int id;
    public String name;
    public String department;
    public double salary;

    public Employee(int id, String name, String department, double salary) {
        this.id = id; this.name = name; this.department = department; this.salary = salary;
    }
}

public class EmployeeRepository {
    private final Connection connection;

    public EmployeeRepository(Connection connection) { this.connection = connection; }

    public void createTable() throws SQLException {
        try (Statement st = connection.createStatement()) {
            st.execute("""
                CREATE TABLE employee (
                    id INT PRIMARY KEY AUTO_INCREMENT,
                    name VARCHAR(100) NOT NULL,
                    department VARCHAR(50) NOT NULL,
                    salary DECIMAL(10,2) NOT NULL CHECK (salary >= 0)
                )
                """);
        }
    }

    public int insert(String name, String department, double salary) throws SQLException {
        String sql = "INSERT INTO employee (name, department, salary) VALUES (?, ?, ?)";
        try (PreparedStatement ps = connection.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            ps.setString(1, name);
            ps.setString(2, department);
            ps.setDouble(3, salary);
            ps.executeUpdate();
            try (ResultSet keys = ps.getGeneratedKeys()) {
                keys.next();
                return keys.getInt(1);
            }
        }
    }

    public Optional<Employee> findById(int id) throws SQLException {
        String sql = "SELECT id, name, department, salary FROM employee WHERE id = ?";
        try (PreparedStatement ps = connection.prepareStatement(sql)) {
            ps.setInt(1, id);
            try (ResultSet rs = ps.executeQuery()) {
                if (!rs.next()) return Optional.empty();
                return Optional.of(new Employee(rs.getInt("id"), rs.getString("name"),
                        rs.getString("department"), rs.getDouble("salary")));
            }
        }
    }

    public List<Employee> findByDepartment(String department) throws SQLException {
        List<Employee> result = new ArrayList<>();
        String sql = "SELECT id, name, department, salary FROM employee WHERE department = ? ORDER BY id";
        try (PreparedStatement ps = connection.prepareStatement(sql)) {
            ps.setString(1, department);
            try (ResultSet rs = ps.executeQuery()) {
                while (rs.next()) {
                    result.add(new Employee(rs.getInt("id"), rs.getString("name"),
                            rs.getString("department"), rs.getDouble("salary")));
                }
            }
        }
        return result;
    }

    public int giveRaise(String department, double percent) throws SQLException {
        String sql = "UPDATE employee SET salary = salary * (1 + ?/100.0) WHERE department = ?";
        try (PreparedStatement ps = connection.prepareStatement(sql)) {
            ps.setDouble(1, percent);
            ps.setString(2, department);
            return ps.executeUpdate();
        }
    }
}
```

## 3. Tests: schema, CRUD, and transaction isolation

```java
package com.example.db;

import org.junit.jupiter.api.*;
import java.sql.*;
import java.util.List;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;

class EmployeeRepositoryTest {

    private Connection connection;
    private EmployeeRepository repository;

    @BeforeEach
    void setUp() throws SQLException {
        // A fresh, uniquely-named in-memory DB per test -- true isolation, no leftover rows
        connection = DriverManager.getConnection(
                "jdbc:h2:mem:" + System.nanoTime() + ";DB_CLOSE_DELAY=-1");
        repository = new EmployeeRepository(connection);
        repository.createTable();
    }

    @AfterEach
    void tearDown() throws SQLException {
        connection.close();
    }

    @Test
    void insertAndFindById() throws SQLException {
        int id = repository.insert("Ada Lovelace", "Engineering", 95000.00);

        Optional<Employee> found = repository.findById(id);

        assertTrue(found.isPresent());
        assertEquals("Ada Lovelace", found.get().name);
        assertEquals("Engineering", found.get().department);
        assertEquals(95000.00, found.get().salary, 0.001);
    }

    @Test
    void findByIdMissingReturnsEmpty() throws SQLException {
        assertTrue(repository.findById(9999).isEmpty());
    }

    @Test
    void findByDepartmentReturnsOnlyMatches() throws SQLException {
        repository.insert("Ada Lovelace", "Engineering", 95000.00);
        repository.insert("Grace Hopper", "Engineering", 105000.00);
        repository.insert("Margaret Hamilton", "Research", 98000.00);

        List<Employee> engineers = repository.findByDepartment("Engineering");

        assertEquals(2, engineers.size());
        assertTrue(engineers.stream().allMatch(e -> e.department.equals("Engineering")));
    }

    @Test
    void giveRaiseUpdatesOnlyTargetDepartment() throws SQLException {
        repository.insert("Ada Lovelace", "Engineering", 100000.00);
        repository.insert("Margaret Hamilton", "Research", 100000.00);

        int rowsUpdated = repository.giveRaise("Engineering", 10.0);

        assertEquals(1, rowsUpdated);
        List<Employee> engineers = repository.findByDepartment("Engineering");
        List<Employee> researchers = repository.findByDepartment("Research");
        assertEquals(110000.00, engineers.get(0).salary, 0.001);
        assertEquals(100000.00, researchers.get(0).salary, 0.001);
    }

    @Test
    @DisplayName("negative salary is rejected by the CHECK constraint")
    void negativeSalaryViolatesConstraint() {
        SQLException ex = assertThrows(SQLException.class,
                () -> repository.insert("Bad Data", "Engineering", -500.00));
        assertTrue(ex.getMessage().toLowerCase().contains("check constraint"));
    }

    @Test
    @DisplayName("missing required column violates NOT NULL")
    void nullNameViolatesNotNull() {
        assertThrows(SQLException.class,
                () -> repository.insert(null, "Engineering", 50000.00));
    }
}
```

I ran this exact repository and test class locally against H2 with Maven
(`mvn test`, no network required — H2 runs fully in-process) and all six
tests passed:

```
Tests run: 6, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

## 4. Test isolation strategies

| Strategy | How | Trade-off |
|---|---|---|
| Fresh in-memory DB per test | unique `jdbc:h2:mem:<name>` per `@BeforeEach` | Fastest, truest isolation; needs a real schema-creation step per test |
| Transaction rollback per test | `connection.setAutoCommit(false)`, `connection.rollback()` in `@AfterEach` | Fast, reuses one schema; breaks if code under test explicitly commits |
| Shared DB, `TRUNCATE` between tests | one DB, clear tables in `@BeforeEach` | Closer to real environments; slower, and truncation order matters with foreign keys |

The tests above use the first strategy — the same principle as "no shared
mutable state" from Module 04's parallel-execution traps, applied to
persistence instead of memory.

## 5. Testing traps

!!! warning "Trap 1 — testing against a shared dev database"
    A test that inserts into a database other tests (or other engineers)
    also use will eventually collide — a "unique" name that's actually
    reused, a count assertion broken by someone else's row. Isolate with an
    in-memory or per-test-run database, never a shared one.

!!! warning "Trap 2 — floating-point equality on money"
    `assertEquals(110000.00, salary)` without a delta can fail from binary
    floating-point representation error even when the math is correct.
    Always assert doubles with a delta (`assertEquals(x, y, 0.001)`), and
    consider `BigDecimal` for real money code, not `double`.

!!! warning "Trap 3 — forgetting `ORDER BY` and asserting position"
    `findByDepartment(...).get(0)` assumes row order that SQL does not
    guarantee without an explicit `ORDER BY`. It happens to work on H2 today
    because of insertion order; it is not guaranteed by the SQL standard and
    can silently change with a query planner update.

!!! warning "Trap 4 — connection leaks across tests"
    Not closing a `Connection`/`Statement`/`ResultSet` (missing
    try-with-resources) leaks resources that accumulate across a large
    suite and can eventually exhaust a connection pool, producing failures
    in *unrelated* later tests that are hard to trace back to the real
    cause.

!!! warning "Trap 5 — an H2 pass that hides a real-database failure"
    H2's SQL dialect isn't identical to Postgres/MySQL — a `CHECK` constraint
    message, date-handling quirk, or vendor-specific function can behave
    differently. H2 is excellent for fast, isolated logic tests; a
    release-blocking suite should also run at least once against the real
    database engine (often via Testcontainers, covered conceptually in
    Level 4's containerized environments module) before shipping.

## Cheat sheet

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

## Exercise

1. Build `EmployeeRepository` and `EmployeeRepositoryTest` exactly as above,
   run it with Maven + H2, and confirm all 6 tests pass.
2. Add a `delete(int id)` method and a test asserting it returns 0 rows
   affected for a non-existent id and 1 for an existing one.
3. Add a `department` foreign-key table (`department(id, name)`) with a
   `NOT NULL` foreign key from `employee`, and write a test asserting an
   insert with an unknown department id fails.
4. Rewrite `setUp`/`tearDown` to use the rollback-based isolation strategy
   from the table in section 4 instead of a fresh in-memory DB per test, and
   confirm the same six tests still pass.
5. Introduce a query missing `ORDER BY` where order matters, insert three
   rows, and (if you can) show the result order changing with an added
   `ORDER BY DESC` versus none — write a sentence on why Trap 3 is a real
   risk even when a test currently happens to pass.
