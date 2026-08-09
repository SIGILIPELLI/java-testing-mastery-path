# 06 · Maven/Gradle for Test Projects

Level 1 used Maven as a way to download Selenium. In a real framework the
build file does much more: it separates fast unit tests from slow browser
tests, switches environments with a flag, controls parallelism, and produces
the reports CI consumes. A build a colleague can run with one command, on a
machine they have never configured, is a deliverable in its own right.

## 1. A framework-grade `pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>qa-automation</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <selenium.version>4.23.0</selenium.version>
        <testng.version>7.10.2</testng.version>
        <restassured.version>5.4.0</restassured.version>
        <suiteXmlFile>src/test/resources/testng.xml</suiteXmlFile>
        <browser>chrome</browser>
        <threads>1</threads>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.junit</groupId>
                <artifactId>junit-bom</artifactId>
                <version>5.10.2</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.seleniumhq.selenium</groupId>
            <artifactId>selenium-java</artifactId>
            <version>${selenium.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testng</groupId>
            <artifactId>testng</artifactId>
            <version>${testng.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.rest-assured</groupId>
            <artifactId>rest-assured</artifactId>
            <version>${restassured.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <version>3.25.3</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Fast tests: *Test.java -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.5</version>
                <configuration>
                    <includes>
                        <include>**/*Test.java</include>
                    </includes>
                    <systemPropertyVariables>
                        <browser>${browser}</browser>
                    </systemPropertyVariables>
                </configuration>
            </plugin>

            <!-- Slow tests: *IT.java -- run in `verify`, failures reported after teardown -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-failsafe-plugin</artifactId>
                <version>3.2.5</version>
                <configuration>
                    <suiteXmlFiles>
                        <suiteXmlFile>${suiteXmlFile}</suiteXmlFile>
                    </suiteXmlFiles>
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
</project>
```

## 2. Surefire vs Failsafe — the distinction that matters

| | Surefire | Failsafe |
|---|---|---|
| Phase | `test` | `integration-test` + `verify` |
| Default naming | `*Test`, `Test*`, `*Tests`, `*TestCase` | `*IT`, `IT*`, `*ITCase` |
| On failure | Fails the build **immediately** | Records it, runs `post-integration-test`, fails at `verify` |
| Meant for | Unit tests — fast, no external dependency | UI and API tests — slow, need an environment |

That "on failure" row is the whole reason Failsafe exists. If a Selenium test
fails under Surefire, the build stops and your `post-integration-test`
teardown — stopping a server, tearing down a container, closing browsers —
never runs. Failsafe guarantees cleanup happens first.

```bash
mvn test            # unit tests only (Surefire)
mvn verify          # unit + integration tests, with cleanup (Failsafe)
mvn clean verify -Dbrowser=firefox -Dthreads=4
```

## 3. Profiles — one build, many environments

```xml
<profiles>
    <profile>
        <id>smoke</id>
        <properties>
            <suiteXmlFile>src/test/resources/smoke.xml</suiteXmlFile>
            <threads>2</threads>
        </properties>
    </profile>

    <profile>
        <id>regression</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <properties>
            <suiteXmlFile>src/test/resources/regression.xml</suiteXmlFile>
            <threads>4</threads>
        </properties>
    </profile>

    <profile>
        <id>staging</id>
        <properties>
            <base.url>https://staging.example.com</base.url>
        </properties>
    </profile>
</profiles>
```

```bash
mvn verify -Psmoke
mvn verify -Pregression,staging
mvn help:active-profiles          # which profiles are actually on?
```

That last command answers the question "why is it running the wrong suite?",
which will otherwise cost you an hour.

## 4. Dependency scopes and conflicts

| Scope | On compile classpath | On test classpath | Packaged |
|---|---|---|---|
| `compile` (default) | yes | yes | yes |
| `test` | no | yes | no |
| `provided` | yes | yes | no |
| `runtime` | no | yes | yes |
| `import` | — (BOM only) | — | no |

Every testing library should be `test`. If Selenium is `compile` scope, it
ships to production inside your application JAR.

```bash
mvn dependency:tree
mvn dependency:tree -Dincludes=com.google.guava
mvn dependency:analyze          # declared-but-unused, used-but-undeclared
```

!!! warning "Version conflicts are resolved by depth, not by version"
    Maven's "nearest wins" rule picks the version closest to your `pom.xml`
    in the tree — which can be an *older* one. Selenium and RestAssured both
    pull Guava and Jackson, and a mismatch surfaces as
    `NoSuchMethodError` at runtime, not as a build error. Fix it by pinning
    the version in `<dependencyManagement>`, or by `<exclusions>` on the
    offending dependency. A BOM (as with `junit-bom` above) is the cleanest
    prevention.

## 5. Multi-module layout

Once a framework is shared across teams, split it:

```
qa-automation/
├── pom.xml                     ← <packaging>pom</packaging>, lists modules
├── framework-core/             ← DriverFactory, BasePage, Config, listeners
│   └── pom.xml
├── ui-tests/                   ← page objects + UI tests
│   └── pom.xml
└── api-tests/                  ← RestAssured tests
    └── pom.xml
```

```xml
<!-- parent pom.xml -->
<packaging>pom</packaging>
<modules>
    <module>framework-core</module>
    <module>ui-tests</module>
    <module>api-tests</module>
</modules>
```

!!! info "`framework-core` uses `compile` scope"
    Reusable framework code lives in `src/main/java` of `framework-core` with
    `compile`-scoped Selenium — otherwise the downstream test modules cannot
    see it. Only the *test* modules use `test` scope. This trips people up
    the first time.

## 6. The Gradle equivalent

```kotlin
// build.gradle.kts
plugins { `java` }

repositories { mavenCentral() }

dependencies {
    testImplementation(platform("org.junit:junit-bom:5.10.2"))
    testImplementation("org.junit.jupiter:junit-jupiter")
    testImplementation("org.seleniumhq.selenium:selenium-java:4.23.0")
    testImplementation("org.testng:testng:7.10.2")
    testImplementation("io.rest-assured:rest-assured:5.4.0")
}

java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }

tasks.test {
    useJUnitPlatform()
    maxParallelForks = 4
    systemProperty("browser", System.getProperty("browser", "chrome"))
    testLogging { events("passed", "skipped", "failed") }
}

tasks.register<Test>("uiTests") {
    useTestNG { suites("src/test/resources/testng.xml") }
    shouldRunAfter(tasks.test)
}
```

```bash
./gradlew test
./gradlew uiTests -Dbrowser=firefox
```

| Maven | Gradle |
|---|---|
| `<scope>test</scope>` | `testImplementation` |
| `<scope>compile</scope>` | `implementation` |
| Surefire / Failsafe | `test` task / a custom `Test` task |
| Profiles | Project properties + task configuration |
| `mvn clean test` | `./gradlew clean test` |

Gradle is faster (incremental builds, a build cache, a warm daemon); Maven is
more predictable and far more common in enterprise QA. Both are fine — being
fluent in whichever your team uses is what matters.

## 7. Useful Surefire/Failsafe configuration

```xml
<configuration>
    <parallel>methods</parallel>
    <threadCount>${threads}</threadCount>
    <reuseForks>false</reuseForks>              <!-- fresh JVM per fork -->
    <forkCount>2</forkCount>
    <skipAfterFailureCount>5</skipAfterFailureCount>  <!-- fail fast -->
    <argLine>-Xmx1024m -Duser.language=en</argLine>
    <groups>smoke</groups>                       <!-- JUnit 5 @Tag / TestNG group -->
    <excludedGroups>slow</excludedGroups>
    <trimStackTrace>false</trimStackTrace>       <!-- keep the full trace -->
    <systemPropertyVariables>
        <base.url>${base.url}</base.url>
    </systemPropertyVariables>
</configuration>
```

`trimStackTrace` defaults to true and cuts exactly the frames you need when
diagnosing a CI-only failure. Turn it off on day one.

## 8. Testing traps

!!! warning "Trap 1 — `No tests were executed!`"
    Almost always a naming mismatch: your class is `LoginTests` (plural, not
    matched by Failsafe) or `TestLogin` under Failsafe rather than Surefire.
    Check the plugin's default includes before changing anything else.

!!! warning "Trap 2 — a green build that ran nothing"
    `mvn test` on a project whose tests are all `*IT` reports BUILD SUCCESS
    having executed zero tests. Always read the "Tests run:" line, and
    consider `<failIfNoTests>true</failIfNoTests>`.

!!! warning "Trap 3 — `SNAPSHOT` or ranged versions"
    `<version>4.+</version>` or a SNAPSHOT dependency means today's green
    build says nothing about tomorrow's. Pin every version; upgrade
    deliberately, in a commit of its own.

!!! warning "Trap 4 — `-DskipTests` in the CI pipeline"
    It happens, usually as a "temporary" unblock. Grep your pipeline files
    for it; a build that skips tests is a build that reports nothing.

## Cheat sheet

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

## Exercise

1. Restructure your Level 2 project with the `pom.xml` from section 1. Rename
   your Selenium classes to `*IT` and your non-browser tests to `*Test`, then
   confirm `mvn test` runs only the fast ones and `mvn verify` runs both.
2. Add `smoke` and `regression` profiles pointing at two different
   `testng.xml` files. Run each, and use `mvn help:active-profiles` to prove
   which one was applied.
3. Run `mvn dependency:tree -Dincludes=com.fasterxml.jackson.core` and report
   which libraries pull Jackson in and at which versions. If they differ,
   pin one version in `<dependencyManagement>` and re-run to confirm.
4. Set `<trimStackTrace>false</trimStackTrace>`, force a failure, and compare
   the stack trace with the trimmed default.
5. Add `-Dbrowser` plumbing end to end: property in `pom.xml` →
   `systemPropertyVariables` → your `Config` class → `DriverFactory`. Prove
   it with `mvn verify -Dbrowser=firefox`.
6. Split `framework-core` into its own module and make `ui-tests` depend on
   it. Note what had to change about the Selenium dependency's scope, and
   why.
7. Write the Gradle equivalent of your Maven build and get `./gradlew test`
   green. Then write three sentences on which build file a new team member
   would understand faster, and why.
