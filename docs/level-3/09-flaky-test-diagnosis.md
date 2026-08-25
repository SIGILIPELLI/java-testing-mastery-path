# 09 · Flaky Test Diagnosis & Stabilization

A flaky test — one that passes and fails on identical code, with no
external change — is worse than no test at all. A missing test is an honest
gap; a flaky one trains the team to click "rerun" instead of investigating,
which is exactly the reflex that lets a real regression slip through the day
it actually matters. This module is a diagnostic process, not a single fix,
because flakiness has several distinct root causes that look identical from
the outside ("sometimes it just fails").

## 1. The four usual suspects

1. **Timing** — the test checks state before the system under test has
   actually reached it (animation not finished, async call not resolved,
   eventual-consistency write not visible yet).
2. **Isolation** — one test's leftover state (a static field, a database row,
   a file on disk) leaks into another test, so outcome depends on run order.
3. **Environment** — resolution/locale/timezone/network variance between
   machines or CI runs.
4. **Concurrency** — parallel execution (Module 04) exposing a genuine race
   condition, either in the test or in the app itself.

## 2. Reproducing a suspected timing flake

The single most useful tool for confirming timing flakiness is running the
test many times in a tight loop and looking at the failure rate.

```java
package com.example.flaky;

import org.junit.jupiter.api.RepeatedTest;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class SuspectedFlakeTest {

    @RepeatedTest(50)
    void asyncResultEventuallyArrives() throws InterruptedException {
        AsyncCounter counter = new AsyncCounter();
        counter.incrementAsync();
        // BAD -- fixed sleep, guesses at timing instead of waiting for the real condition
        Thread.sleep(5);
        assertEquals(1, counter.get());
    }
}

class AsyncCounter {
    private volatile int value = 0;
    void incrementAsync() {
        new Thread(() -> {
            try { Thread.sleep((long) (Math.random() * 20)); } catch (InterruptedException ignored) {}
            value++;
        }).start();
    }
    int get() { return value; }
}
```

I ran `@RepeatedTest(50)` against this exact class with JUnit 5 locally:

```
Tests run: 50, Failures: 41, Errors: 0, Skipped: 0
BUILD FAILURE
```

41/50 (82%) failed — a hard fixed `Thread.sleep(5)` racing against a
random 0-20ms background delay, plus real thread-spawn overhead, means the
assertion usually runs before the increment lands. The exact failure rate
depends on machine load and JVM warm-up (I got 82% on this run; yours may
differ), but the shape is the same either way: not "always fails," not
"never fails," a consistent partial-failure rate over many runs. A single
run would have shown either a lucky pass or an unlucky failure, telling you
nothing about the real frequency.

## 3. Fixing timing flakiness — wait for the condition, not a duration

```java
package com.example.flaky;

import org.junit.jupiter.api.RepeatedTest;
import java.time.Duration;
import java.time.Instant;
import static org.junit.jupiter.api.Assertions.*;

class FixedAsyncTest {

    @RepeatedTest(50)
    void asyncResultEventuallyArrives() {
        AsyncCounter counter = new AsyncCounter();
        counter.incrementAsync();

        Instant deadline = Instant.now().plus(Duration.ofMillis(200));
        while (counter.get() != 1 && Instant.now().isBefore(deadline)) {
            try { Thread.sleep(2); } catch (InterruptedException ignored) {}
        }

        assertEquals(1, counter.get(), "value never reached 1 within 200ms");
    }
}
```

```
Tests run: 50, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

Same 50 repetitions, same random 0-20ms delay, zero failures — because the
assertion now polls for the actual condition with a generous ceiling instead
of gambling on a fixed sleep shorter than the slowest possible real delay.
`WebDriverWait` from Level 1 Module 08 is this exact pattern, pre-built,
for the browser.

## 4. Diagnosing isolation flakiness — run order matters

```java
package com.example.flaky;

import org.junit.jupiter.api.*;
import java.util.*;

import static org.junit.jupiter.api.Assertions.*;

@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class OrderDependentTest {

    static List<String> sharedLog = new ArrayList<>();   // the bug: static + mutable

    @Test
    @Order(1)
    void firstTestAddsAnEntry() {
        sharedLog.add("first");
        assertEquals(1, sharedLog.size());
    }

    @Test
    @Order(2)
    void secondTestAssumesEmptyLog() {
        assertTrue(sharedLog.isEmpty(), "expected a clean log, found: " + sharedLog);
    }
}
```

JUnit 5's *default* method order is deterministic but deliberately not tied
to source order — which is itself a lesson: don't assume declaration order
without `@TestMethodOrder`. Pinning the order explicitly with
`@Order(1)`/`@Order(2)` is what actually reproduces the leak here: run the
full class and `secondTestAssumesEmptyLog` fails, `sharedLog` contains
`[first]`. Run `secondTestAssumesEmptyLog` alone (filtered with
`-Dtest=OrderDependentTest#secondTestAssumesEmptyLog`), and it passes — I
confirmed both outcomes locally, exactly this divergence. This is what
"flaky in the full suite, passes when I run it alone" almost always means.

```java
// The fix: no shared static mutable state
class IsolatedTest {
    @Test
    void firstTest() {
        List<String> log = new ArrayList<>();
        log.add("first");
        assertEquals(1, log.size());
    }

    @Test
    void secondTest() {
        List<String> log = new ArrayList<>();   // fresh, not shared
        assertTrue(log.isEmpty());
    }
}
```

## 5. A checklist for a flaky test report

When a test is reported flaky, work through these in order rather than
guessing:

1. **Reproduce the failure rate.** `@RepeatedTest(50)` (or your CI's "run N
   times" feature) locally and on CI both. If it never fails locally but
   fails on CI, suspect environment or concurrency, not raw timing.
2. **Check for `Thread.sleep`, fixed waits, or race-prone async code** in
   the test or the code path it exercises.
3. **Check for shared mutable state** — `static` fields, singletons holding
   session data (Module 08's Trap 1), a database/file not reset between
   runs.
4. **Run the single test in isolation** vs. the full suite. A pass alone and
   a fail in the suite points strongly at isolation, not the test's own
   logic.
5. **Check CI's parallelism settings** against what you assumed locally —
   Module 04's Trap 1 (shared state under parallel threads) is often
   invisible until CI's thread count differs from your laptop's.
6. **Quarantine, don't delete**, a test you can't fix immediately — tag it
   (`@Tag("flaky")`), exclude it from the blocking pipeline, and track it as
   a real defect with an owner and a deadline. A silently-deleted test is a
   silently-reduced safety net.

## 6. Testing traps

!!! warning "Trap 1 — 'just add a retry' as the fix"
    A test-runner-level retry (rerun on failure, keep the pass) hides the
    exact failure rate that would tell you whether this is a minor timing
    issue or a serious race condition. Retries can be a legitimate stopgap
    for a known, understood, low-risk timing gap — never a substitute for
    diagnosis on a newly-reported flake.

!!! warning "Trap 2 — fixing the symptom in one test, leaving the pattern everywhere"
    Replacing one `Thread.sleep(5)` with a proper wait fixes that test but
    not the four other tests copy-pasted from it. Grep for the anti-pattern
    (`Thread.sleep` in test code) across the whole suite once you've found
    one instance.

!!! warning "Trap 3 — blaming the test when the app has a real race condition"
    Sometimes the "flaky test" is correctly detecting a genuine
    intermittent bug in the application under test — a double-submit
    button, an unhandled race in the backend. Confirm the app's actual
    behavior manually before assuming the test is the problem.

!!! warning "Trap 4 — CI-only flakiness dismissed as 'CI is just flaky'"
    CI environments are usually more resource-constrained and more
    parallel than a developer laptop, which surfaces real timing and
    concurrency bugs that a fast, lightly-loaded local machine never
    triggers. "Only fails in CI" is a clue pointing at load/timing, not
    evidence the failure is meaningless.

!!! warning "Trap 5 — no record of flaky-test history"
    Without tracking which tests have been flagged flaky and why, the same
    root cause (a shared static field pattern, a common async helper) gets
    independently "fixed" in five different tests by five different people,
    none of whom realize it's one systemic issue.

## Cheat sheet

| Symptom | Likely cause | Fix direction |
|---|---|---|
| Fails ~X% of many identical runs | Timing / race | Poll for condition, not `Thread.sleep` |
| Passes alone, fails in full suite | Isolation | Remove shared/static mutable state |
| Only fails in CI, never locally | Environment or concurrency | Match CI's parallelism/resources locally |
| Fails only under `parallel="methods"` | Concurrency | Per-thread resources (Module 04) |
| Fails after another test runs first | Isolation, run-order dependency | `@TestMethodOrder` isolation test, then fix state leak |
| Diagnostic tool | JUnit 5 | `@RepeatedTest(N)` to measure real failure rate |

## Exercise

1. Build `AsyncCounter`/`SuspectedFlakeTest` exactly as above and run it
   with `@RepeatedTest(50)`; record the actual failure count you get (it
   will vary run to run since the delay is random — that variability *is*
   the point).
2. Apply the polling fix from section 3 and rerun `@RepeatedTest(50)`;
   confirm 0 failures.
3. Build `OrderDependentTest` and reproduce the order-dependent failure by
   running the full class, then running only
   `secondTestAssumesEmptyLog` filtered on its own; record both outcomes.
4. Fix it using the isolation pattern in section 4, and rerun both ways to
   confirm the fix holds regardless of run order.
5. Pick one test from an earlier module in this course (yours, not this
   file's) that uses `Thread.sleep` for any reason, replace it with a
   condition-based wait, and run it 20 times before and after to compare
   failure rates.
