# 05 · Test Metrics & Quality Gates

Every module so far answered "does this specific thing work?" This module
answers a different, organization-level question: "how do we know, in
aggregate, whether quality is improving, and how do we stop a release that
doesn't meet the bar?" Metrics without a gate are just a dashboard nobody
acts on; a gate without honest metrics blocks releases for the wrong
reasons. You need both.

## 1. Code coverage — measuring it honestly

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.12</version>
    <executions>
        <execution>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
        <execution>
            <id>check</id>
            <phase>verify</phase>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.70</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

```bash
mvn verify
```

```
[INFO] Analyzed bundle 'checkout-service' with 42 classes
[WARNING] Rule violated for bundle checkout-service: lines covered ratio is 0.64, but expected minimum is 0.70
[ERROR] Failed to execute goal ... Coverage Checks Have Not Been Met
```

`mvn verify` fails exactly like a failing test would — coverage becomes a
build gate, not a number someone checks manually once a sprint.

## 2. Why coverage percentage alone is a weak signal

```java
// 100% line coverage, genuinely useless test
@Test
void coversEveryLineButAssertsNothing() {
    Cart cart = new Cart();
    cart.addItem("widget", 1000);   // line covered
    cart.applyCode("SAVE10");       // line covered
    cart.total();                   // line covered
    // no assertions at all -- JaCoCo reports this as "covered," JUnit reports it as "passed"
}
```

I ran this exact test locally: it genuinely reports as `PASSED` under
JUnit 5 and every line it touches genuinely reports as `covered` under
JaCoCo, despite proving nothing.

```
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

Coverage measures *execution*, not *verification*. A 90% coverage target
with no complementary check (mutation testing, code review for real
assertions, or simply a policy against assertion-free tests) can be
satisfied by tests exactly like this one — worth stating explicitly to
anyone who treats a coverage number as a proxy for quality on its own.

## 3. Mutation testing — measuring whether assertions actually catch bugs

```xml
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.15.8</version>
    <configuration>
        <targetClasses><param>com.example.hybrid.*</param></targetClasses>
        <targetTests><param>com.example.hybrid.*</param></targetTests>
        <mutationThreshold>60</mutationThreshold>
    </configuration>
</plugin>
```

```bash
mvn org.pitest:pitest-maven:mutationCoverage
```

PIT deliberately mutates the code under test (flips a `>` to `>=`, changes
`+` to `-`, removes a line) and reruns the suite against each mutant. A
mutant that still passes means no test actually depends on that exact
logic — a much stronger signal than line coverage that a test *asserts*
something meaningful, not merely that it *runs* the line.

```
================================================================================
- Mutants
==================================================================================
> Generated 24 mutants
> Killed 19 (79%)
> Survived 5 (21%)

Line Coverage: 91%
Mutation Coverage: 79%
```

91% line coverage but only 79% mutation coverage is a common and telling
gap: it means roughly one in five lines that *look* tested would still pass
the suite even if the logic were subtly wrong.

## 4. Flaky test rate as a first-class metric

Following Level 3 Module 09, a test suite's flaky-test rate deserves its
own tracked number, not just ad-hoc firefighting:

```java
// A simple flaky-rate tracker, run against CI history
public class FlakyRateReport {
    public static double flakyRate(List<TestRun> last100Runs, String testName) {
        long passCount = last100Runs.stream()
                .filter(r -> r.testName().equals(testName))
                .filter(TestRun::passed).count();
        long totalCount = last100Runs.stream()
                .filter(r -> r.testName().equals(testName)).count();
        if (totalCount == 0) return 0.0;
        double passRate = (double) passCount / totalCount;
        // "flaky" here means neither reliably green nor reliably red
        return (passRate > 0.0 && passRate < 1.0) ? (1.0 - Math.abs(passRate - 0.5) * 2) : 0.0;
    }
}

public record TestRun(String testName, boolean passed) {}
```

```java
@Test
void consistentlyPassingTestHasZeroFlakyScore() {
    List<TestRun> runs = List.of(
            new TestRun("StableTest", true), new TestRun("StableTest", true),
            new TestRun("StableTest", true), new TestRun("StableTest", true));
    assertEquals(0.0, FlakyRateReport.flakyRate(runs, "StableTest"), 0.001);
}

@Test
void fiftyFiftyTestHasMaximumFlakyScore() {
    List<TestRun> runs = List.of(
            new TestRun("FlakyTest", true), new TestRun("FlakyTest", false),
            new TestRun("FlakyTest", true), new TestRun("FlakyTest", false));
    assertEquals(1.0, FlakyRateReport.flakyRate(runs, "FlakyTest"), 0.001);
}
```

I ran `FlakyRateReport` and both tests locally with plain JUnit 5 and both
passed, confirming the formula behaves as intended at its two extremes
(always-passing → 0, evenly split → 1).

```
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

Tracking this per-test over CI history (most CI systems expose historical
run data via API) surfaces exactly the tests Level 3 Module 09's checklist
should be applied to, ranked by how disruptive they actually are — a fully
red or fully green test isn't "flaky," even if it's failing, and this
formula deliberately scores those at 0.

## 5. A composite quality gate

```yaml
# CI job -- combines Level 3 Module 03's pattern with metrics gates
- name: Run tests with coverage
  run: mvn verify   # fails on: test failure, coverage < 70%, per section 1

- name: Mutation testing (nightly only, expensive)
  if: github.event_name == 'schedule'
  run: mvn org.pitest:pitest-maven:mutationCoverage

- name: Fail if mutation coverage regressed
  if: github.event_name == 'schedule'
  run: |
    SCORE=$(grep -oP 'Mutation Coverage: \K[0-9]+' target/pit-reports/index.html || echo 0)
    if [ "$SCORE" -lt 60 ]; then echo "Mutation coverage $SCORE% below 60% gate"; exit 1; fi
```

Running full mutation testing on every PR is usually too slow (it reruns
the suite once per mutant); gating it nightly or weekly, while gating line
coverage and test pass/fail on every PR, is the common compromise.

## 6. Testing traps

!!! warning "Trap 1 — chasing a coverage number instead of test quality"
    A team incentivized purely on hitting 80% coverage will find the
    fastest path there — often exactly the assertion-free tests from
    section 2. Pair a coverage gate with either mutation testing or
    mandatory review of new tests' assertions.

!!! warning "Trap 2 — a quality gate so strict it gets routinely overridden"
    A 90% coverage minimum on a legacy codebase that's genuinely at 40%
    gets bypassed via `-Dmaven.test.failure.ignore` or a manual override
    within the first week, training everyone that the gate is optional.
    Set the bar at a realistic, slightly-improving level and ratchet it up
    over time.

!!! warning "Trap 3 — mutation testing run on generated or trivial code"
    Including simple getters/setters or generated code (builders, records)
    in PIT's target classes produces a flood of "survived" mutants that add
    noise without signal — scope `targetClasses` to logic that actually
    matters.

!!! warning "Trap 4 — treating flaky-rate as a test-quality metric alone"
    A spike in flaky rate across many unrelated tests at once usually
    means shared infrastructure trouble (a slow CI runner, a degraded test
    environment), not a sudden wave of badly-written tests. Check the
    infrastructure explanation before assigning it to test authors.

!!! warning "Trap 5 — metrics reported but never reviewed"
    A dashboard nobody looks at in a recurring meeting or a bot post is not
    a quality gate — it's decoration. The value is entirely in the
    decision made when a number crosses a threshold, not the number's
    existence.

## Cheat sheet

| Metric | Tool | What it actually tells you |
|---|---|---|
| Line/branch coverage | JaCoCo | What code *ran* during tests |
| Mutation coverage | PIT | Whether tests would *catch* an actual bug |
| Flaky rate | Custom, from CI history | Which tests erode trust in the suite |
| Pass/fail as a build gate | Surefire exit code (Level 3 Module 03) | Is the suite currently green |
| Dependency vulnerabilities | dependency-check (Level 4 Module 04) | Inherited risk, not authored code |
| Composite gate | `mvn verify` combining several plugins | One command, one pass/fail signal |

## Exercise

1. Add JaCoCo to a project from this course, set a coverage minimum of
   70%, and confirm `mvn verify` fails when you comment out a test that
   pushes coverage below it.
2. Write `coversEveryLineButAssertsNothing` (or your own equivalent) and
   confirm it passes and reports as covered — then delete it, since it has
   no place in a real suite.
3. Add PIT to the same project, run mutation testing, and report the gap
   (if any) between line coverage and mutation coverage.
4. Implement `FlakyRateReport` and its two tests, then extend it with a
   third test for a fully-failing test (`passRate == 0.0`) and confirm it
   also scores 0 flaky, not 1.
5. Design a composite CI gate (in YAML, reviewed not necessarily run) for a
   real project combining test pass/fail, coverage minimum, and a nightly
   mutation-testing check — justify each threshold you choose in one
   sentence.
