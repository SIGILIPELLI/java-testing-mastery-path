# 06 · Containerized Test Environments

Level 3 Module 07 flagged a real limitation of H2: it's fast and isolated,
but its SQL dialect isn't identical to production databases, so a passing
H2 suite can still hide a real-database bug. Testcontainers solves this by
running the *actual* database (or queue, or cache) in a throwaway Docker
container for the duration of the test run — real Postgres, real Kafka,
real Redis, torn down automatically when the JVM exits.

## 1. Why not just install Postgres locally?

| Approach | Problem |
|---|---|
| Shared dev/staging Postgres | Module 04's (Level 3) Trap 1 — collisions between tests/engineers |
| Locally installed Postgres | Version drift between machines; manual setup; not in CI by default |
| H2 in "Postgres mode" | Closer, but still not the real engine — some features/quirks differ |
| Testcontainers | Real Postgres, exact pinned version, fresh per run, zero manual setup |

## 2. Setup

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.19.7</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <version>1.19.7</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>1.19.7</version>
    <scope>test</scope>
</dependency>
```

Testcontainers needs a Docker daemon reachable from wherever the tests run
— the local machine's Docker Desktop/Engine, or Docker-in-Docker on a CI
runner. Without one, every test in this module fails at container startup,
not at an assertion — worth checking `docker info` first when a
Testcontainers suite mysteriously won't even begin.

## 3. A real Postgres container per test class

```java
package com.example.containers;

import org.junit.jupiter.api.*;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.sql.*;
import java.util.*;

import static org.junit.jupiter.api.Assertions.*;

@Testcontainers
class EmployeeRepositoryPostgresTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    private Connection connection;
    private EmployeeRepository repository;   // same class from Level 3 Module 07

    @BeforeEach
    void setUp() throws SQLException {
        connection = DriverManager.getConnection(
                postgres.getJdbcUrl(), postgres.getUsername(), postgres.getPassword());
        repository = new EmployeeRepository(connection);
        repository.createTable();
    }

    @AfterEach
    void tearDown() throws SQLException {
        // real Postgres, unlike per-test H2, needs explicit cleanup between tests
        try (Statement st = connection.createStatement()) {
            st.execute("DROP TABLE IF EXISTS employee");
        }
        connection.close();
    }

    @Test
    void insertAndFindByIdAgainstRealPostgres() throws SQLException {
        int id = repository.insert("Ada Lovelace", "Engineering", 95000.00);
        Optional<Employee> found = repository.findById(id);
        assertTrue(found.isPresent());
        assertEquals("Ada Lovelace", found.get().name);
    }

    @Test
    void checkConstraintBehavesIdenticallyToH2() {
        assertThrows(SQLException.class,
                () -> repository.insert("Bad Data", "Engineering", -500.00));
    }
}
```

`@Container static` scopes one Postgres instance to the whole test class
(started once, reused across `@Test` methods, torn down when the class
finishes) — a deliberate trade-off: faster than one container per test, at
the cost of needing the explicit `DROP TABLE`/re-`createTable` cleanup in
`@BeforeEach`/`@AfterEach` that Level 3 Module 07's per-test H2 database got
for free.

I did not run this against real Testcontainers/Docker in this environment
— no Docker daemon available headlessly here. The `EmployeeRepository`
class and its SQL are identical to the version verified working against H2
in Level 3 Module 07 (6/6 passing there); this module's contribution is the
Testcontainers wiring around it, reviewed against Testcontainers' documented
JUnit 5 extension API, not executed.

## 4. Comparable output to what H2 produced (Level 3 Module 07), for contrast

```
# Level 3 Module 07, H2, actually run:
Tests run: 6, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS

# This module, real Postgres via Testcontainers, expected shape (not executed here):
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

The point of running both isn't redundancy — H2 gives near-instant feedback
for everyday development; a smaller, targeted Testcontainers suite (often
tagged `@Tag("integration")`, per Level 4 Module 01's split) runs less often
and catches the narrower class of bugs that are specific to the real
database engine's actual behavior.

## 5. Testcontainers for other dependencies

```java
// Redis
@Container
static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

// Kafka
@Container
static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"));

// A custom app image, e.g. testing against the real inventory-service from Module 03's contract testing
@Container
static GenericContainer<?> inventoryService = new GenericContainer<>("inventory-service:test")
        .withExposedPorts(8080)
        .waitingFor(Wait.forHttp("/health").forStatusCode(200));
```

The `waitingFor(...)` strategy matters: a container reporting "started" to
Docker doesn't mean the application inside it has finished booting.
Testcontainers' wait strategies (HTTP health check, log message match, port
open) prevent the classic "connection refused" flake of a test starting a
fraction of a second before its dependency is actually ready — the
container equivalent of Level 3 Module 09's "wait for the condition, not a
duration" lesson.

## 6. Wiring into CI

```yaml
jobs:
  integration-tests:
    runs-on: ubuntu-latest   # GitHub-hosted runners include Docker by default
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '17', distribution: 'temurin' }
      - name: Run Testcontainers-backed integration tests
        run: mvn test -Dtest.groups=integration
```

No extra setup needed on GitHub-hosted runners specifically because Docker
is already present; a self-hosted runner needs Docker installed and the
runner's user given permission to use it.

## 7. Testing traps

!!! warning "Trap 1 — forgetting Docker isn't available everywhere"
    A Testcontainers-backed suite silently included in the *default*
    `mvn test` run breaks for any environment without Docker (some
    corporate laptops, some minimal CI images). Tag these tests separately
    (`@Tag("integration")`, per Level 4 Module 01) and document the Docker
    requirement clearly.

!!! warning "Trap 2 — no cleanup between tests in a class-scoped container"
    Reusing one container across a test class (section 3) means table/row
    state from one test leaks into the next unless explicitly cleaned up —
    exactly Level 3 Module 07's isolation lesson, just at the container
    level. Decide deliberately between class-scoped (faster, needs cleanup)
    and per-test containers (slower, free isolation).

!!! warning "Trap 3 — first-run container pull adds surprising latency"
    The first execution of a Testcontainers suite on a fresh machine or CI
    cache pulls the image from a registry — potentially tens of seconds to
    minutes — which can look like the test itself is hanging. Pre-pulling
    images in a CI cache step avoids this surprise on every cold run.

!!! warning "Trap 4 — treating a container's readiness as the app's readiness"
    Without an explicit `waitingFor(...)` health check (section 5),
    "container started" and "server ready to accept requests" are treated
    as the same event when they aren't — the exact race condition this
    module exists to prevent, reintroduced by skipping the one line that
    prevents it.

!!! warning "Trap 5 — running full Testcontainers suites on every save"
    Container startup (seconds) is far slower than an in-memory H2
    connection (milliseconds). Running this suite as part of the fast local
    loop (Level 4 Module 02's pre-commit hook) defeats the fast-feedback
    goal — keep it in the `integration`/`full` tier, not `unit`.

## Cheat sheet

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

## Exercise

1. Confirm Docker is available (`docker info`) and build
   `EmployeeRepositoryPostgresTest` exactly as above against Level 3 Module
   07's `EmployeeRepository`; run it and compare the output to the
   H2-backed version.
2. Add a test that exists specifically because it might behave differently
   on real Postgres versus H2 (a date/time edge case, or a specific SQL
   function) and confirm both versions actually agree — or find a genuine
   divergence and document it.
3. Convert the container from class-scoped (`static`) to per-test
   (non-static) and measure the difference in total suite run time across
   5 tests.
4. Add a `GenericContainer` for Redis, write one test caching a value and
   reading it back, and add an explicit `waitingFor(...)` check.
5. Tag your Testcontainers-backed tests `@Tag("integration")` and confirm
   they're excluded from the `fast` Maven profile built in Level 4 Module
   01, but included in `full`.
