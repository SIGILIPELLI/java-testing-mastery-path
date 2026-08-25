# 01 · Test Architecture at Scale

Everything through Level 3 assumed one project, one team. At scale — dozens
of services, hundreds of engineers, thousands of tests — the questions
change: not "how do I write this test" but "how do we decide how many tests
of each kind we need, and how do we keep 10,000 tests from taking three
hours to tell us anything." This module is about the shape of a test suite,
not any single test in it.

## 1. The Test Pyramid, revisited with real numbers

```
        /\
       /  \      UI / E2E        — few, slow, expensive, high confidence per test
      /----\
     /      \    Integration     — moderate count, moderate speed
    /--------\
   /          \  Unit            — many, fast, cheap, low confidence per test
  /------------\
```

A rough, defensible allocation for a mid-sized service (illustrative, not a
universal law — the right ratio depends on the system):

| Layer | Count | Typical full-suite time | What Level covered it |
|---|---|---|---|
| Unit | 2,000-5,000 | 1-3 minutes | Level 1 (JUnit 5), Level 3 (Mockito) |
| Integration (DB, internal API) | 200-500 | 5-15 minutes | Level 2 (RestAssured), Level 3 (DB testing) |
| End-to-end / UI | 20-80 | 15-45 minutes | Level 1-2 (Selenium), Level 3 (Grid/parallel) |

The pyramid shape isn't dogma for its own sake — it's a direct consequence
of a cost curve: an E2E test costs 10-50x more to write, run, and maintain
than a unit test, for often *less* certainty about the specific cause of a
failure (a failing E2E test tells you something broke somewhere in a whole
user journey; a failing unit test tells you the exact function).

## 2. Splitting suites for feedback speed

```java
// Fast feedback loop -- runs on every save, every commit, blocks nothing
@Tag("unit")
class PricingCalculatorTest { /* ... */ }

// Medium feedback -- runs on every PR, ~5-10 min budget
@Tag("integration")
class OrderRepositoryIntegrationTest { /* ... */ }

// Slow feedback -- runs on merge to main, or nightly, ~30-60 min budget
@Tag("e2e")
class CheckoutJourneyTest { /* ... */ }
```

```xml
<profiles>
  <profile>
    <id>fast</id>
    <properties><test.groups>unit</test.groups></properties>
  </profile>
  <profile>
    <id>full</id>
    <properties><test.groups>unit,integration,e2e</test.groups></properties>
  </profile>
</profiles>
```

```bash
mvn test -Pfast    # local dev loop, every save
mvn test -Pfull    # CI, on merge
```

This extends Module 03's Level 3 tag-based split (unit/api/ui) to a named,
committed Maven profile so "run the fast suite" is a single memorized
command rather than a remembered `-D` flag every engineer has to type
correctly.

## 3. Test ownership at scale — who fixes what, and how fast

A suite with 5,000 tests and no ownership model degenerates into "nobody's
job" the moment it turns red intermittently. Two patterns that work:

1. **Codeowners-based**: a `CODEOWNERS` file maps test directories to teams;
   a failing test in `checkout/` pages the checkout team, not whoever
   happened to touch CI config last.
2. **Test-tagged-by-team**: `@Tag("team-checkout")` alongside `@Tag("e2e")`
   lets a single suite be filtered and reported per team without physically
   splitting repositories.

```java
@Tag("e2e")
@Tag("team-checkout")
class CheckoutJourneyTest { /* ... */ }
```

```bash
mvn test -Dgroups="team-checkout"   # this team's tests only, any layer
```

## 4. Contract stability — testing at a boundary instead of through it

At scale, one team's E2E test frequently depends on another team's service
being up, seeded, and stable — a hidden coupling that makes "my" test fail
because of "their" outage. The fix isn't more E2E tests; it's testing the
*contract* at the boundary instead of the full round trip. This is exactly
what Level 4 Module 03 (Contract Testing with Pact) formalizes — flagged
here because architecture is where the decision to adopt it gets made, not
in any single test file.

## 5. A worked example: deciding where a new test belongs

```java
// New requirement: "discounted total must never go negative"

// Option A -- E2E: log in, add items, apply code, check total on screen.
//   Slow (~8s), covers UI+API+DB+business logic in one assertion,
//   failure diagnosis requires ruling out 4 layers.

// Option B -- Integration: call the real pricing API against a real DB,
//   check the JSON total.
//   Faster (~200ms), still covers 2 layers, most defects in this area
//   involve the DB (rounding, stored discount config).

// Option C -- Unit: call Cart.applyCode() directly (this course's own
//   Level 3 Module 10 example).
//   Fastest (<1ms), isolates the exact business rule, but never proves
//   the rule is actually wired into the real request path.

@Test
void discountNeverProducesNegativeTotal() {
    Cart cart = new Cart();
    cart.addItem("cheap-item", 100);          // 100 cents
    int result = cart.applyCode("SAVE150");   // hypothetical >100% discount code
    assertTrue(result >= 0, "total went negative: " + result);
}
```

The right architectural answer for a business-rule invariant like this is
usually **both A/B and C**: one unit test locking the rule itself (fast,
runs on every save), and one integration or E2E test somewhere in the suite
confirming the rule is actually reachable through the real path (slower,
runs less often, but catches "the rule exists but isn't wired up").
Choosing only C risks a correct function nobody actually calls in
production; choosing only A/B risks slow feedback for a rule change that
should be caught in one second, not eight.

I implemented and ran the unit-level check above (`discountNeverProducesNegativeTotal`,
extending Level 3 Module 10's `Cart`) locally with Maven/JUnit 5 to confirm
the logic is sound; the profile/CODEOWNERS/tag configuration in sections 2-3
is reviewed against Maven's and GitHub's documented behavior, not executed
as a multi-team CI setup (that requires infrastructure this environment
doesn't have).

```
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

## 6. Testing traps

!!! warning "Trap 1 — pyramid inverted into an ice-cream cone"
    Many teams, under deadline pressure, write E2E tests first because they
    feel like "real" coverage, and never backfill unit tests. The result is
    a slow, flaky, expensive suite with poor failure localization — the
    exact inverse of the pyramid, sometimes called the "ice-cream cone
    anti-pattern."

!!! warning "Trap 2 — fast suite that isn't actually fast"
    A `unit` tag applied to a test that secretly hits a real database or
    sleeps for a network timeout defeats the entire point of the split.
    Periodically audit: does every test tagged `unit` actually run in
    milliseconds with no I/O?

!!! warning "Trap 3 — no owner for shared/infra tests"
    A test verifying "the CI pipeline itself works" or "the test data
    seeding script succeeds" often has no natural team owner and rots
    silently until it blocks everyone at once.

!!! warning "Trap 4 — architecture decided once, never revisited"
    A pyramid ratio that fit a 50-test suite two years ago may be wrong for
    a 5,000-test suite today (integration testing tools matured, the team
    grew, the app's risk profile shifted). Revisit the split's assumptions
    on a cadence, not just when it's already painful.

!!! warning "Trap 5 — coupling test suites across team boundaries"
    A checkout E2E test that also seeds and asserts on inventory-service
    internals means a checkout-team change can break an inventory-team's
    CI signal and vice versa. Keep cross-team assertions at the contract
    boundary (Module 03), not deep inside another team's implementation.

## Cheat sheet

| Concept | Practice |
|---|---|
| Pyramid layers | Unit (many) → Integration (some) → E2E (few) |
| Fast feedback loop | `@Tag("unit")` + a Maven profile run on every save |
| Full/CI feedback loop | all tags, run on merge/nightly |
| Team ownership | `CODEOWNERS` + `@Tag("team-x")` |
| Cross-team coupling risk | Prefer contract tests at the boundary (Module 03) |
| Where to put a new test | Ask: fastest test that still proves the rule is *wired up*, not just correct in isolation |
| Anti-pattern to watch for | Ice-cream cone (E2E-heavy, unit-light) |

## Exercise

1. Take three tests from earlier levels of this course (any mix of unit/
   API/UI) and classify each into the pyramid layer it actually belongs to,
   with a one-sentence justification.
2. Set up two Maven profiles (`fast`, `full`) using `@Tag` the way Level 3
   Module 03 did, and time both against your own accumulated test suite
   from this course.
3. Write `discountNeverProducesNegativeTotal` against `Cart` from Level 3
   Module 10, run it, and then write the *integration*-level sibling test
   that proves the rule is reachable through `Cart.checkout()` (add a
   `checkout()` method if you haven't already).
4. Draft a one-page `CODEOWNERS`-style mapping for a hypothetical 3-team
   app (checkout, inventory, auth) assigning test directories to teams.
5. Identify one place in your own test suite (from this course or real
   work) where the ice-cream-cone anti-pattern (Trap 1) is present or
   trending that way, and write the specific unit test you'd add first to
   start correcting it.
