# 09 · QA Leadership & Strategy

Every prior module answered a question inside a test suite. This one steps
outside it: as a QA lead, how do you decide where testing effort goes, how
do you justify that investment to people who don't write tests, and how do
you know — with evidence, not vibes — whether the strategy is working?

## 1. A defensible model for prioritizing test investment

Not every feature deserves equal test depth. A simple, tunable model:
`priority = (probability of defect) × (impact if it ships) / (cost to test
well)`. Code it once, apply it consistently instead of re-litigating the
argument every planning cycle.

```java
package com.example.strategy;

public record Feature(String name, double defectProbability, double impactScore, double testCostHours) {

    public double priorityScore() {
        if (testCostHours <= 0) throw new IllegalArgumentException("testCostHours must be positive");
        return (defectProbability * impactScore) / testCostHours;
    }
}
```

```java
package com.example.strategy;

import java.util.*;

public class TestInvestmentPlanner {

    public List<Feature> rankByPriority(List<Feature> features) {
        List<Feature> sorted = new ArrayList<>(features);
        sorted.sort(Comparator.comparingDouble(Feature::priorityScore).reversed());
        return sorted;
    }
}
```

```java
package com.example.strategy;

import org.junit.jupiter.api.Test;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class TestInvestmentPlannerTest {

    private final TestInvestmentPlanner planner = new TestInvestmentPlanner();

    @Test
    void highRiskCheapToTestFeatureRanksAboveLowRiskExpensiveOne() {
        Feature paymentRetry = new Feature("payment-retry-logic", 0.6, 9.0, 4.0);   // score 1.35
        Feature themeToggle = new Feature("dark-mode-toggle", 0.1, 2.0, 8.0);       // score 0.025

        List<Feature> ranked = planner.rankByPriority(List.of(themeToggle, paymentRetry));

        assertEquals("payment-retry-logic", ranked.get(0).name());
        assertEquals("dark-mode-toggle", ranked.get(1).name());
    }

    @Test
    void zeroCostFeatureThrowsRatherThanDivideByZero() {
        Feature broken = new Feature("misconfigured", 0.5, 5.0, 0.0);
        assertThrows(IllegalArgumentException.class, broken::priorityScore);
    }
}
```

I ran `Feature`, `TestInvestmentPlanner`, and this test class locally with
Maven/JUnit 5 and both tests passed:

```
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

This isn't meant as a precise formula to defend to three decimal places —
it's a structure for making the *inputs* to a prioritization argument
explicit and comparable, instead of an unstated gut feeling that's hard for
a skeptical stakeholder (or a new team member) to evaluate or challenge.

## 2. Translating test metrics into a business conversation

Level 4 Module 05's metrics (coverage, mutation score, flaky rate) are
engineering-facing. A leadership conversation needs a different framing:

| Engineering metric | Business translation |
|---|---|
| Flaky test rate rising | "Our release confidence signal is getting less trustworthy — we're at risk of either shipping a real bug that got lost in noise, or delaying a good release chasing a false alarm." |
| Mutation coverage 79% vs. line coverage 91% | "About 1 in 5 of our 'tested' code paths would still pass if the logic were subtly wrong — that's our real risk surface, not the 91% number." |
| E2E suite taking 45 minutes | "Every release candidate waits 45 minutes for a full signal — that's N releases/week we're capped at, structurally, regardless of how fast anyone codes." |
| Security test coverage (Level 4 Module 04) | "These are the categories of vulnerability we actively test for on every build vs. the ones we don't yet — a specific, auditable answer instead of 'we take security seriously.'" |

The pattern: every technical metric maps to a cost, a risk, or a constraint
a non-engineer can weigh against other priorities. A number without that
translation gets nodded at and forgotten.

## 3. Building a QA strategy document (structure, not prose to copy)

1. **Current state** — pyramid shape (Level 4 Module 01), suite runtime,
   flaky rate, coverage/mutation numbers, known gaps.
2. **Risk-based priorities** — using something like section 1's model,
   named and ranked, not implicit.
3. **Investment plan** — what changes in the next quarter (e.g. "add
   contract testing for the 3 highest-traffic service boundaries," Level 4
   Module 03), with effort estimates.
4. **Success metrics** — the specific numbers that will move, and by when,
   so "did the strategy work" has a factual answer in three months instead
   of a debate.
5. **Explicit non-goals** — what you're deliberately *not* investing in this
   cycle, and why — as important as the goals, because it pre-empts the
   "why didn't we also do X" conversation with a considered answer instead
   of an oversight.

## 4. Hiring and growing a QA team — what to actually screen for

Given everything this course has covered, the signal that predicts a strong
hire isn't "knows Selenium syntax" (that's learnable in weeks, as this
course itself demonstrates) — it's:

- Can they explain **why** a test is flaky, not just that it is (Level 3
  Module 09's diagnostic process)?
- Do they ask "what should this assert" before "how do I call this API,"
  reflecting real engagement with correctness, not just tool fluency?
- Can they push back on a coverage-number target with a mutation-testing-style
  argument (Level 4 Module 05) instead of either blindly complying or
  blindly refusing?
- Do they think about test ownership and maintenance cost (Level 4 Module
  08's litmus test), or only about getting today's test to pass?

A structured interview exercise that surfaces this: give a candidate a
small, deliberately flawed test (an `assertNotNull`-only test, Level 4
Module 07's AI-generated-test example) and ask them to critique it, not
write a new one from scratch. It's faster to run and reveals judgment more
directly than a from-scratch coding exercise does.

## 5. Testing traps (at the leadership level)

!!! warning "Trap 1 — a strategy with no falsifiable success metric"
    "Improve quality this quarter" can't fail, which means it can't
    meaningfully succeed either — there's no evidence either way. Section
    3's "success metrics" step exists specifically to prevent this.

!!! warning "Trap 2 — optimizing the metric instead of the goal it represents"
    A team told to "raise coverage to 80%" will raise coverage to 80% —
    sometimes via exactly the assertion-free tests Level 4 Module 05
    warned about. Pair every target metric with a countermeasure that
    would catch gaming it (mutation testing, code review, a spot-check).

!!! warning "Trap 3 — QA strategy decided without engineering input"
    A testing strategy set entirely top-down, without the engineers who'll
    execute it weighing in on feasibility, produces plans that look correct
    on a slide and fail on contact with the actual codebase's constraints.

!!! warning "Trap 4 — treating this course's toolset as the whole strategy"
    Cucumber, Mockito, Testcontainers, Pact, and everything else in this
    course are *capabilities* — none of them are a strategy on their own.
    Adopting a tool because it was covered in a course (including this one)
    without a specific problem it solves is cargo-culting, not strategy.

!!! warning "Trap 5 — no plan for when the strategy is wrong"
    A quarterly review that only ever confirms the existing plan was right
    (confirmation bias in metric selection, cherry-picked wins) never
    catches the moment reality diverged from the plan. Build in an explicit,
    scheduled "what did we get wrong" review, not just a "what did we
    achieve" one.

## Cheat sheet

| Leadership task | Grounded in |
|---|---|
| Prioritizing test investment | A stated model (section 1), not gut feel |
| Reporting to non-engineers | Business-translated metrics (section 2) |
| Writing a strategy doc | Current state → priorities → plan → metrics → non-goals |
| Interviewing QA candidates | Judgment on a flawed test, not tool trivia |
| Avoiding metric-gaming | Pair every target with a countermeasure check |
| Staying honest about the plan | A scheduled "what did we get wrong" review |

## Exercise

1. Implement `Feature`, `TestInvestmentPlanner`, and the two tests exactly
   as above; run them and confirm both pass.
2. Score three real features from your own work (or from this course's
   projects) using the formula, and write one sentence justifying each
   input number you chose.
3. Take one metric from Level 4 Module 05 (coverage, mutation score, or
   flaky rate) and write its business-translation sentence for a
   stakeholder who has never heard the technical term.
4. Draft the five-section strategy document structure from section 3 for a
   hypothetical small team, with at least one explicit non-goal and a
   reason for it.
5. Write the flawed test you'd hand a candidate in a hiring interview
   (per section 4), and a rubric of three things a strong answer would
   identify about it.
