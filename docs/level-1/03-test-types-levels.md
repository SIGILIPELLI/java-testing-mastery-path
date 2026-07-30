# 03 · Test Types & Levels

"We tested it" is meaningless without saying *at what level* and *of what
type*. A unit test and a UAT session both "test" the same feature but answer
completely different questions, cost different amounts, and are run by
different people. This module gives you the two axes that organize all
testing work: **levels** (how much of the system is under test) and **types**
(what property you are checking).

## 1. Test levels — the four stages

Levels describe *scope*: how much of the system is assembled when the test
runs. They map onto the V-Model, where each development phase has a matching
test phase.

| Level | Scope under test | Who usually writes it | Typical tool |
|---|---|---|---|
| **Unit** | One class or method in isolation | Developer | JUnit 5 (Module 07) |
| **Integration** | Two or more components talking to each other | Developer / SDET | JUnit + Spring Test, RestAssured |
| **System** | The whole assembled application, end to end | QA team | Selenium (Module 08), TestNG |
| **Acceptance (UAT)** | The whole app, judged against business need | Business users / client | Manual, sometimes Cucumber (Level 3) |

### Unit testing

Tests the smallest testable piece — usually one method — with all its
dependencies replaced by fakes. Fast (milliseconds), numerous (thousands),
and run on every commit.

> *Example:* `calculateDiscount(100, 10)` returns `90.0`.

### Integration testing

Tests that units work *together*. The classic bug it catches: two developers
each wrote a correct component, but one sends dates as `dd/MM/yyyy` and the
other parses `MM/dd/yyyy`.

Two strategies you should know by name:

| Approach | How it works | Trade-off |
|---|---|---|
| **Big Bang** | Integrate everything, then test | Simple to start; failures are hard to localize |
| **Top-Down** | Test high-level modules first, using **stubs** for the not-yet-built lower ones | Early demo of the main flow; needs many stubs |
| **Bottom-Up** | Test low-level modules first, using **drivers** to call them | Solid foundation; no working UI until late |
| **Sandwich/Hybrid** | Both directions meeting in the middle | Practical, but needs both stubs and drivers |

!!! info "Stub vs driver"
    A **stub** stands in for a component *called by* the code under test (a
    fake payment gateway that always returns "approved"). A **driver**
    stands in for a component that *calls* the code under test (a temporary
    `main` method that invokes a service you are building bottom-up). You
    will build real versions of these with Mockito in Level 3.

### System testing

The full application, deployed on a test environment that mirrors
production, tested end to end against the requirement specification. This is
where most manual QA effort lives, and what Selenium automates.

### User Acceptance Testing (UAT)

The business decides whether the software solves *their* problem. System
testing asks "does it match the spec?"; UAT asks "does it work for us in the
real world?" Variants:

- **Alpha testing** — by internal staff, at the developer's site.
- **Beta testing** — by real external users, in their own environment.
- **Contract/Regulatory acceptance** — against contractual or legal criteria.

## 2. Functional vs non-functional testing

Levels tell you *how much* is under test. Types tell you *what property* you
are checking.

| | Functional | Non-functional |
|---|---|---|
| Question | Does it do **what** it should? | **How well** does it do it? |
| Based on | Requirements / user stories | Quality attributes, SLAs |
| Example | "Transfer moves ₹500 from A to B" | "Transfer completes in under 2 seconds with 1,000 concurrent users" |
| Failure looks like | Wrong result | Correct result, but too slow / insecure / unusable |

### Functional test types

| Type | Verifies |
|---|---|
| **Smoke** | The build is stable enough to test at all |
| **Sanity** | A specific fix or new feature works, narrowly |
| **Regression** | Existing features still work after a change |
| **Re-testing (confirmation)** | A specific reported defect is actually fixed |
| **Integration** | Interfaces between modules |
| **UAT** | Real business workflows |

### Non-functional test types

| Type | Verifies | Typical tool |
|---|---|---|
| **Performance** | Response time and throughput under expected load | JMeter (Level 3) |
| **Load** | Behaviour at expected peak concurrent usage | JMeter, Gatling |
| **Stress** | Behaviour *beyond* capacity — does it fail gracefully? | JMeter |
| **Scalability** | Does adding resources actually increase capacity? | JMeter + infra |
| **Security** | Resistance to attack — injection, auth bypass, data exposure | OWASP ZAP (Level 4) |
| **Usability** | Can a real user complete the task without confusion? | Manual, user sessions |
| **Compatibility** | Works across browsers, OS, devices, screen sizes | BrowserStack, Selenium Grid |
| **Reliability** | Stays correct over long uptime | Endurance/soak runs |
| **Accessibility** | Usable with screen readers, keyboard only, sufficient contrast | axe, WAVE |
| **Recovery** | Restores correctly after a crash or network loss | Manual fault injection |

## 3. Smoke vs sanity vs regression vs re-testing

These four are confused constantly, including in interviews. Learn the
distinctions precisely.

| | **Smoke** | **Sanity** | **Regression** | **Re-testing** |
|---|---|---|---|---|
| Purpose | Is the build testable? | Does this specific change work? | Did the change break anything else? | Is *this* defect fixed? |
| Scope | Wide but shallow — critical paths only | Narrow and deep — one area | Wide and deep — the whole suite | Extremely narrow — one bug's steps |
| When | Immediately on receiving a new build | After a minor fix or small feature | Before release, or nightly | After a developer marks a bug Fixed |
| Documented? | Usually a scripted checklist | Often unscripted | Fully scripted, ideally automated | Follows the bug report's steps |
| If it fails | **Reject the build** — stop testing | Investigate that area | Log defects, continue | Reopen the defect |
| Automatable? | Yes, and should be | Sometimes | Yes — this is the #1 automation target | Yes, add as a new regression case |

A worked sequence on a real team:

1. Developer deploys build 4.2.1 to the QA environment.
2. QA runs the **smoke suite** (15 cases, 6 minutes): app loads, login works,
   dashboard renders, search returns results, checkout page opens. All pass →
   build accepted.
3. Build 4.2.1 contained a fix for BUG-114 (wrong error message on bad
   login). QA **re-tests** BUG-114 by following its exact reproduction steps.
   It now behaves correctly → mark Verified/Closed.
4. QA runs a **sanity check** around authentication generally — logout,
   remember-me, password reset — to see nothing adjacent broke.
5. Before the release, QA runs the full **regression suite** (600 cases,
   automated overnight) across the entire application.

!!! warning "Regression is why automation exists"
    A 600-case regression suite executed manually takes a team days, and by
    case 400 humans are skimming. Automated, it runs while everyone sleeps
    and reports identically every time. If you remember one reason to learn
    Selenium and TestNG in this course, it is this.

## 4. Black box, white box, grey box

A third axis: how much internal knowledge you use.

| | **Black box** | **White box** | **Grey box** |
|---|---|---|---|
| Knowledge of code | None — tests through the UI/API | Full — tests the code paths | Partial — knows the architecture/DB |
| Performed by | Testers | Developers | SDETs |
| Based on | Requirements | Code structure | Both |
| Techniques | Equivalence partitioning, BVA, decision tables (Module 05) | Statement/branch/path coverage | DB validation after a UI action |
| Finds | Missing or wrong functionality | Untested branches, dead code | Integration and data-layer issues |

## 5. Static vs dynamic testing

Not all testing runs the software.

- **Static testing** — reviewing artefacts without execution: requirement
  reviews, design walkthroughs, code reviews, static analysis tools
  (SonarQube, SpotBugs). Cheapest possible defect detection; this is
  verification.
- **Dynamic testing** — actually executing the software. Everything else in
  this module; this is validation.

A tester who reviews the requirement document and finds a contradiction
between REQ-04 and REQ-11 has done static testing worth more than a week of
execution.

## 6. The test pyramid

How many tests of each level should you have?

```
        /\          UAT / Manual exploratory  — few, slow, expensive
       /  \
      /----\        System / E2E (Selenium)   — some
     /      \
    /--------\      Integration (API tests)   — more
   /          \
  /------------\    Unit (JUnit)              — many, fast, cheap
```

The pyramid says: put most of your automated checks at the unit level where
they are fast and stable, fewer at the integration level, and only a thin
layer of slow end-to-end UI tests covering critical journeys.

!!! warning "The ice-cream cone anti-pattern"
    Teams that automate only through the UI end up with an inverted pyramid:
    hundreds of slow, flaky Selenium tests and almost no unit tests. The
    suite takes four hours, fails randomly, and everyone stops trusting it.
    You will diagnose and fix exactly this in Level 3, Module 09.

## Cheat sheet

| Term | One-liner |
|---|---|
| Unit test | One method, dependencies faked, milliseconds |
| Integration test | Components talking to each other |
| System test | Whole app, end to end, against the spec |
| UAT | Whole app, judged by the business |
| Smoke | Is this build worth testing? |
| Sanity | Does this one change work? |
| Regression | Did the change break anything else? |
| Re-test | Is this specific bug actually fixed? |
| Functional | Does it do the right thing? |
| Non-functional | Does it do it fast/securely/usably enough? |
| Black box | Test without seeing code |
| White box | Test the code paths |
| Static | Review without executing |
| Dynamic | Execute and observe |

## Exercise

An e-commerce team has just deployed build 7.3.0 to QA. It contains: (a) a
fix for a bug where the cart total ignored a coupon, and (b) a new "Save for
later" button on the cart page.

1. List **eight test cases** you would put in the **smoke suite** for this
   application. Justify in one line why each earns a place — remember smoke
   is wide but shallow.
2. Describe exactly what your **sanity check** for change (b) would cover,
   and how it differs from the regression suite.
3. Write the **re-test** steps for change (a), assuming the original bug
   report said "Cart total does not decrease when a valid 10% coupon is
   applied."
4. For the cart page, name **one test of each** of these types and state what
   it checks: performance, security, usability, compatibility, accessibility.
5. Sketch the **test pyramid** for this application: roughly how many unit,
   integration and end-to-end tests would you aim for, and name two specific
   checks that belong at each layer. Explain why "verify the cart total
   arithmetic" belongs at the unit layer rather than in Selenium.
