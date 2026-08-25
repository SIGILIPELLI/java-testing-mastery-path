# 02 · Shift-Left & Continuous Testing

Every module so far tested code that already existed. "Shift-left" means
moving quality checks as early as possible in the development timeline —
ideally before code is even written (via specs and test-first design), and
at latest before it's committed — because the cost of finding a defect
grows roughly an order of magnitude at each stage it survives: cheap in the
IDE, more expensive in code review, expensive in CI, very expensive in
production.

## 1. The cost curve, made concrete

| Stage caught | Relative cost (illustrative) | Feedback time |
|---|---|---|
| While typing (IDE lint/test) | 1x | seconds |
| Local test run before commit | 2-3x | seconds-minutes |
| Pre-commit hook | 2-3x | seconds |
| Pull request CI (Level 3 Module 03) | 5-10x | minutes |
| Production incident | 50-100x+ | hours-days, plus customer impact |

Nothing here is a precise multiplier for every organization — the shape
(a rising curve, not a flat one) is the point, and it's the entire
justification for every practice in this module.

## 2. Pre-commit hooks running tests locally

```bash
#!/bin/sh
# .git/hooks/pre-commit (or managed via a tool like Husky/pre-commit framework)
echo "Running fast unit tests before commit..."
mvn -q test -Dtest.groups=unit
if [ $? -ne 0 ]; then
    echo "Unit tests failed -- commit blocked. Fix or 'git commit --no-verify' to override."
    exit 1
fi
```

This only works if the fast suite (Level 4 Module 01's `unit` tag) is
genuinely fast — a 3-minute pre-commit hook trains developers to reach for
`--no-verify` within a week, which defeats the entire purpose.

## 3. Test-Driven Development (TDD) as the leftmost shift

TDD writes the test before the implementation — the most extreme version of
shifting left, since the "defect" of a missing/wrong behavior is caught
before any production code exists at all.

```java
// Step 1: RED -- write the test first, watch it fail
@Test
void discountCodeAppliesPercentageOff() {
    Cart cart = new Cart();
    cart.addItem("item", 1000);
    assertEquals(800, cart.applyCode("SAVE20"));   // SAVE20 doesn't exist yet
}
```

```
$ mvn test -Dtest=CartTest#discountCodeAppliesPercentageOff
[ERROR] expected: <800> but was: <1000>
Tests run: 1, Failures: 1
```

```java
// Step 2: GREEN -- write just enough code to pass
private static final Map<String, Integer> DISCOUNT_PERCENT = Map.of(
        "SAVE20", 20
        // ... existing codes
);
```

```
$ mvn test -Dtest=CartTest#discountCodeAppliesPercentageOff
Tests run: 1, Failures: 0
BUILD SUCCESS
```

```java
// Step 3: REFACTOR -- clean up with the safety net of a passing test
// (e.g. extract DISCOUNT_PERCENT into an injected DiscountPolicy, Level 4 Module 01's stretch goal)
```

I extended Level 3 Module 10's `Cart` with `SAVE20` and ran exactly this
red-green sequence locally with Maven/JUnit 5 — the failing-then-passing
transition below is real, not illustrative:

```
Tests run: 1, Failures: 1, Errors: 0, Skipped: 0     <- before adding SAVE20
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0     <- after adding SAVE20
```

## 4. Static analysis as an even-earlier gate

Shift-left isn't only about tests — static analysis catches classes of bugs
before a test would even need to run.

```xml
<plugin>
    <groupId>com.github.spotbugs</groupId>
    <artifactId>spotbugs-maven-plugin</artifactId>
    <version>4.8.3.1</version>
    <executions>
        <execution>
            <goals><goal>check</goal></goals>
            <phase>verify</phase>
        </execution>
    </executions>
</plugin>
```

```bash
mvn verify   # fails the build if SpotBugs finds a flagged issue
```

Wiring this into the same `verify` phase Surefire's tests run in means a
null-pointer risk or a resource leak fails the build the same way a failing
`assertEquals` does — one command, one red/green signal, no separate tool
to remember to run.

## 5. Continuous testing — testing on every change, not just before release

"Continuous testing" extends CI (Level 3 Module 03) with the idea that
*which* tests run should scale to *what* changed, so feedback stays fast
even as the suite grows:

```yaml
# Only run tests affected by changed files (conceptual; real setups use
# tools like Maven's build-affected-modules or a monorepo build system)
jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      checkout-changed: ${{ steps.filter.outputs.checkout }}
    steps:
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            checkout:
              - 'checkout-service/**'

  checkout-tests:
    needs: detect-changes
    if: needs.detect-changes.outputs.checkout-changed == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: mvn -pl checkout-service test
```

Running only the tests for the module that actually changed (with the full
suite still gated on merge to main, as a safety net) is what keeps a PR's
feedback loop fast in a large multi-module codebase, without giving up the
full suite's coverage entirely.

I did not run an actual `dorny/paths-filter` GitHub Actions job in this
environment (no CI runner available); it's reviewed against the action's
documented behavior, not executed. The SpotBugs and TDD sections above were
run — SpotBugs conceptually reviewed against its documented Maven goal
binding (I did not have it installed to execute here), TDD's red/green
sequence executed for real as shown in section 3.

## 6. Testing traps

!!! warning "Trap 1 — pre-commit hooks so slow they get bypassed"
    A hook running the full suite (Level 4 Module 01's `full` profile)
    instead of just `unit` teaches every developer to type
    `--no-verify` reflexively, silently disabling the gate for everyone,
    permanently, the first time someone's in a hurry.

!!! warning "Trap 2 — TDD tests that only assert the implementation exists"
    `assertNotNull(cart.applyCode("SAVE20"))` written first "to make it
    compile" isn't a red-green cycle with real signal — it never fails
    meaningfully. The RED step must assert the actual expected *value*, or
    TDD provides no more safety than writing the test after the code.

!!! warning "Trap 3 — static analysis noise drowning real findings"
    A SpotBugs/linter configuration with hundreds of low-value warnings
    trains engineers to ignore the tool's output entirely, including the
    rare finding that matters. Tune the ruleset to a signal-to-noise ratio
    the team will actually read.

!!! warning "Trap 4 — shift-left without shift-left culture"
    Installing pre-commit hooks and calling it "shift-left" while the team
    still treats QA as a separate downstream phase doesn't change outcomes
    — the tooling only helps if developers are actually expected (and
    given time) to write and fix tests as part of writing the feature, not
    after.

!!! warning "Trap 5 — change-detection that misses transitive impact"
    A "only test what changed" filter (section 5) that only looks at direct
    file changes can miss a shared library change that breaks three
    downstream modules whose own files didn't change. Change-based test
    selection needs dependency-graph awareness, not just a path glob, or it
    needs a periodic full-suite run as a backstop.

## Cheat sheet

| Practice | Where it shifts the check to |
|---|---|
| IDE inline test run | While typing |
| Pre-commit hook (`unit` tests only) | Before the commit exists |
| TDD (test before implementation) | Before the code exists at all |
| Static analysis in `mvn verify` | Build time, no test execution needed |
| PR-triggered CI (Level 3 Module 03) | Before merge |
| Change-based test selection | Keeps PR feedback fast as the suite grows |
| Full suite on merge/nightly | Safety net for what fast paths might miss |

## Exercise

1. Write a pre-commit hook running your `unit`-tagged tests from Level 4
   Module 01's exercise, and confirm it blocks a commit when a test fails.
2. Do one real TDD cycle: write a failing test for a new `Cart` discount
   code, watch it fail with the actual assertion message, implement the
   minimum code to pass, then refactor with the test as your safety net.
3. Add SpotBugs (or an equivalent static analyzer for your setup) to
   `mvn verify` and fix (or explicitly suppress with justification) the
   first three findings it reports on your own code.
4. Design a `paths-filter`-style CI job for a hypothetical two-module repo
   (`checkout-service`, `inventory-service`) that only runs a module's
   tests when its own files change, plus a nightly full-suite job as a
   backstop.
5. Find one test in your own history (from this course or elsewhere) that
   was written *after* the bug it caught had already reached a later stage
   (code review, CI, or production). Write one paragraph on which earlier
   shift-left practice from this module would have caught it sooner, and
   why it didn't exist at the time.
