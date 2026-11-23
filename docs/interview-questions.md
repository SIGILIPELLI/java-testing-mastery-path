---
title: "Java Testing Interview Questions and Answers (179 Questions)"
description: "179 Java Testing interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Java Testing Mastery Path."
---

# Java Testing Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Java Testing Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What Is Software Testing?](level-1/01-what-is-software-testing.md)

**Q1.** What is software testing?

**Answer:** Evaluating a program against expectations to find defects and provide quality information

Disciplined and repeatable.

**Q2.** Why can't you prove a program has no bugs by testing?

**Answer:** Exhaustive testing is impossible; tests show presence, not absence, of defects

Choose tests wisely.

**Q3.** What does an oracle provide?

**Answer:** The expected result against which actual output is compared

No oracle, no test.

**Q4.** What's the cost of late defects?

**Answer:** Much higher than early ones

Shift left.

### [02 · Test Case Design &amp; Documentation](level-1/02-test-case-design.md)

**Q5.** What's the difference between a test scenario and a test case?

**Answer:** A scenario is a high-level thing to test; a case has precise steps, data and expected results

Specificity.

**Q6.** What does boundary value analysis suggest for a valid age range 18..65?

```java
public class Main {

public static void main(String[] args) throws Exception {
int[] v = {17, 18, 19, 64, 65, 66};
for (int a : v) System.out.print((a >= 18 && a <= 65 ? "Y" : "N"));
}
}
```

**Answer:** NYYYYN

Test just inside and outside each boundary. Verified by compiling and running it.

**Q7.** What's equivalence partitioning?

**Answer:** Grouping inputs expected to behave the same and testing one per group

Reduce tests.

**Q8.** What makes a good expected result?

**Answer:** Specific and observable

Unambiguous.

### [03 · Test Types &amp; Levels](level-1/03-test-types-levels.md)

**Q9.** List the test levels from small to large.

**Answer:** Unit, integration, system, acceptance

Scope.

**Q10.** What does regression testing verify?

**Answer:** Previously working behavior still works after changes

Automate it.

**Q11.** Which is non-functional?

**Answer:** Performance, security and usability testing

How well it works.

**Q12.** What's a smoke test?

**Answer:** A quick check that critical functions work before deeper testing

Gate builds.

### [04 · Defect Lifecycle &amp; Bug Reporting](level-1/04-defect-lifecycle.md)

**Q13.** What's the typical defect lifecycle?

**Answer:** New, assigned, fixed, retested/verified, closed (or reopened)

State machine.

**Q14.** What belongs in a good bug report?

**Answer:** Steps to reproduce, expected vs actual, environment and evidence

Reproducible.

**Q15.** Severity vs priority?

**Answer:** Severity is impact; priority is urgency to fix

Different axes.

**Q16.** What if a bug can't be reproduced?

**Answer:** Gather logs/environment details and add precise steps or mark needs info

Collaboration.

### [05 · Manual Testing in Practice](level-1/05-manual-testing-practice.md)

**Q17.** When is manual testing most valuable?

**Answer:** Exploratory, usability and one-off checks not worth automating

Human judgment.

**Q18.** What's exploratory testing?

**Answer:** Simultaneous learning, test design and execution guided by a charter

Time-boxed.

**Q19.** Why take notes during sessions?

**Answer:** To reproduce findings and show coverage

Evidence.

**Q20.** What's a test charter?

**Answer:** A short mission guiding an exploratory session

Focus.

### [06 · Java Setup for Testers](level-1/06-java-setup-for-testers.md)

**Q21.** Which JDK/tool builds Java test projects?

**Answer:** JDK plus Maven or Gradle

Dependency management.

**Q22.** What does `mvn test` do?

**Answer:** Compiles and runs unit tests

Lifecycle.

**Q23.** Where do Maven test sources live?

**Answer:** src/test/java

Conventions.

**Q24.** Why use an IDE?

**Answer:** Run/debug tests quickly with navigation and refactoring

Productivity.

### [07 · JUnit 5 Fundamentals](level-1/07-junit5-fundamentals.md)

**Q25.** What does this print (a tiny assertion helper in the style of JUnit)?

```java
public class Main {

public static void main(String[] args) throws Exception {
int pass = 0, fail = 0;
int[][] cases = {{2, 3, 5}, {0, 0, 0}, {-1, 1, 0}, {2, 2, 5}};
for (int[] c : cases) { if (c[0] + c[1] == c[2]) pass++; else fail++; }
System.out.println(pass + " passed, " + fail + " failed");
}
}
```

**Answer:** 3 passed, 1 failed

A test is input, action and assertion. Verified by compiling and running it.

**Q26.** Which annotation marks a JUnit 5 test?

**Answer:** @Test

org.junit.jupiter.api.Test.

**Q27.** What does assertEquals(expected, actual) do?

**Answer:** Fails the test when the values differ

Order matters for messages.

**Q28.** What do @BeforeEach/@AfterEach do?

**Answer:** Run setup/cleanup around every test

Fresh state.

### [08 · Selenium WebDriver Basics](level-1/08-selenium-webdriver-basics.md)

**Q29.** What is Selenium WebDriver?

**Answer:** A browser automation API for testing web apps

Drives real browsers.

**Q30.** Which locator is usually most stable?

**Answer:** IDs or dedicated test attributes (data-testid)

Resilient.

**Q31.** Why use explicit waits?

**Answer:** Wait for a condition instead of fixed sleeps, reducing flakiness

WebDriverWait.

**Q32.** What does driver.quit() do?

**Answer:** Closes all windows and ends the session

Always in teardown.

### [09 · TestNG Basics](level-1/09-testng-basics.md)

**Q33.** What does TestNG add over basic JUnit 4?

**Answer:** Groups, data providers, dependencies and flexible suites (XML)

Common in Selenium.

**Q34.** What's @DataProvider?

**Answer:** Supplies multiple data sets to a parameterized test

Data-driven.

**Q35.** What's testng.xml?

**Answer:** Suite configuration selecting classes, groups and parallelism

Run control.

**Q36.** What does `dependsOnMethods` do?

**Answer:** Runs a test only after another passes

Use sparingly.

### [10 · Project — Manual Test Plan + Automated Suite](level-1/10-project-test-plan-suite.md)

**Q37.** What does the test plan + suite project combine?

**Answer:** A written test plan and automated JUnit/Selenium tests

Level 1 project.

**Q38.** What should a test plan define?

**Answer:** Scope, approach, risks, environments and exit criteria

Guidance.

**Q39.** What's traceability?

**Answer:** Linking tests to requirements

Coverage.

**Q40.** What do you automate first?

**Answer:** Stable, repetitive, high-value checks

ROI.

## Level 2 · Intermediate

### [01 · Page Object Model (POM)](level-2/01-page-object-model.md)

**Q41.** What is the Page Object Model?

**Answer:** A class per page encapsulating locators and actions

Maintainability.

**Q42.** What should page objects avoid?

**Answer:** Putting assertions that hide test intent inside them

Keep assertions in tests.

**Q43.** What do page methods typically return?

**Answer:** The next page object (fluent navigation)

Readable.

**Q44.** Why is POM valuable?

**Answer:** UI changes are fixed in one place

DRY.

### [02 · Selenium Advanced](level-2/02-selenium-advanced.md)

**Q45.** What are fluent waits?

**Answer:** Waits with custom polling and ignored exceptions

More control.

**Q46.** Why avoid mixing implicit and explicit waits?

**Answer:** They interact unpredictably, causing long/flaky waits

Prefer explicit.

**Q47.** How handle iframes?

**Answer:** switchTo().frame(...) then switch back

Context.

**Q48.** What does JavascriptExecutor allow?

**Answer:** Running JS in the page for edge cases

Use sparingly.

### [03 · TestNG Advanced](level-2/03-testng-advanced.md)

**Q49.** What does TestNG's retryAnalyzer do?

**Answer:** Re-runs failed tests to detect flakiness (use carefully)

Don't mask bugs.

**Q50.** How run tests in parallel in TestNG?

**Answer:** parallel/thread-count in the suite XML

Thread-safe drivers needed.

**Q51.** What's ThreadLocal&lt;WebDriver&gt; for?

**Answer:** Giving each thread its own driver

Parallel safety.

**Q52.** What are listeners?

**Answer:** Hooks reacting to test events (screenshots on failure)

ITestListener.

### [04 · API Testing with RestAssured](level-2/04-api-testing-restassured.md)

**Q53.** What does REST Assured do?

**Answer:** A Java DSL for testing REST APIs

given/when/then.

**Q54.** What should API tests verify?

**Answer:** Status code, headers, body and schema

Contracts.

**Q55.** Why test APIs rather than only UI?

**Answer:** Faster, more stable and closer to the logic

Pyramid.

**Q56.** What does JSON schema validation check?

**Answer:** Response structure and types

Contract.

### [05 · Test Data Management](level-2/05-test-data-management.md)

**Q57.** What's good test data practice?

**Answer:** Create data per test and clean up, avoiding shared state

Isolation.

**Q58.** Where store data sets?

**Answer:** External files (JSON/CSV) or generators, not hard-coded

Data-driven.

**Q59.** What's data masking for tests?

**Answer:** Obfuscating real data to protect privacy

Compliance.

**Q60.** What does a factory/builder give tests?

**Answer:** Readable object creation with sensible defaults

Test data builders.

### [06 · Maven/Gradle for Test Projects](level-2/06-maven-gradle-test-projects.md)

**Q61.** What does Maven Surefire do?

**Answer:** Runs unit tests in the test phase

Reports in target/.

**Q62.** What's Failsafe for?

**Answer:** Integration tests in the verify phase

Separation.

**Q63.** What do profiles do?

**Answer:** Switch configuration/environments (e.g. -P staging)

Parameterize.

**Q64.** What's a BOM?

**Answer:** A bill of materials aligning dependency versions

Consistency.

### [07 · Assertions &amp; Matchers](level-2/07-assertions-matchers.md)

**Q65.** What does this print (hand-rolled matchers like Hamcrest)?

```java
import java.util.*;
public class Main {

public static void main(String[] args) throws Exception {
List<Integer> l = List.of(3, 1, 2);
System.out.println(l.contains(2) + " " + (l.size() == 3) + " " + l.stream().allMatch(x -> x > 0) + " " + l.stream().anyMatch(x -> x > 5));
}
}
```

**Answer:** true true true false

Matchers express intent. Verified by compiling and running it.

**Q66.** What does assertThrows verify?

**Answer:** That a given exception type is thrown

Error paths.

**Q67.** Why use assertAll?

**Answer:** Report all failures in a group, not just the first

Soft assertions.

**Q68.** Why prefer descriptive failure messages?

**Answer:** Faster diagnosis without a debugger

Readable.

### [08 · Reporting (Allure, ExtentReports)](level-2/08-reporting.md)

**Q69.** What do Allure/Extent reports provide?

**Answer:** Rich HTML reports with steps, attachments and trends

Stakeholder-friendly.

**Q70.** What should failed UI tests attach?

**Answer:** Screenshots, page source and logs

Evidence.

**Q71.** What do trends show?

**Answer:** Pass rate, duration and flakiness over time

Health.

**Q72.** Why publish reports from CI?

**Answer:** Everyone can see results without running tests

Transparency.

### [09 · Cross-Browser Testing](level-2/09-cross-browser-testing.md)

**Q73.** How run on multiple browsers?

**Answer:** Parameterize the browser in the driver factory

Config.

**Q74.** What do cloud grids (BrowserStack/Sauce) offer?

**Answer:** Many browser/OS combinations on demand

Coverage.

**Q75.** What should cross-browser tests prioritize?

**Answer:** Critical flows on the browsers your users actually use

Analytics-driven.

**Q76.** What's a common cross-browser issue?

**Answer:** Timing/rendering differences needing robust waits

Avoid sleeps.

### [10 · Project — POM Framework with API Tests](level-2/10-project-pom-framework.md)

**Q77.** What does the POM framework project combine?

**Answer:** Page objects, TestNG, data-driven tests, reports and CI

Level 2 project.

**Q78.** How keep tests independent?

**Answer:** Fresh state per test and no order dependence

Reliability.

**Q79.** How configure environments?

**Answer:** Properties/env vars loaded by a config class

Flexibility.

**Q80.** What proves framework quality?

**Answer:** Stable runs, readable tests and quick failure diagnosis

Maintainable.

## Level 3 · Advanced

### [01 · BDD with Cucumber](level-3/01-bdd-cucumber.md)

**Q81.** What is BDD?

**Answer:** Behavior-driven development: specifying behavior in shared, readable language

Collaboration.

**Q82.** What's Gherkin?

**Answer:** Given/When/Then syntax for scenarios

Feature files.

**Q83.** What do step definitions do?

**Answer:** Bind Gherkin steps to executable code

Cucumber.

**Q84.** What's a BDD smell?

**Answer:** Scenarios full of UI details instead of business behavior

Declarative style.

### [02 · Mocking with Mockito](level-3/02-mocking-mockito.md)

**Q85.** What does this hand-written mock record (the idea behind Mockito)?

```java
public class Main {
static class Recorder<T> implements Notifier { java.util.List<String> calls = new java.util.ArrayList<>(); public void send(String m) { calls.add("sent:" + m); } }
interface Notifier { void send(String m); }
static class OrderService { Notifier n; OrderService(Notifier n) { this.n = n; } void place(String id) { n.send(id); } }
public static void main(String[] args) throws Exception {
Recorder<String> mailer = new Recorder<>();
OrderService svc = new OrderService(mailer);
svc.place("A1"); svc.place("A2");
System.out.println(mailer.calls + " " + mailer.calls.size());
}
}
```

**Answer:** [sent:A1, sent:A2] 2

Mocks verify interactions with collaborators. Verified by compiling and running it.

**Q86.** What's the difference between a stub and a mock?

**Answer:** Stubs supply canned answers; mocks verify interactions

Mockito does both.

**Q87.** What does `verify(mock).method()` check?

**Answer:** That the method was called as expected

Interaction testing.

**Q88.** What's over-mocking?

**Answer:** Mocking so much that tests check implementation details

Brittle.

### [03 · CI/CD Integration](level-3/03-cicd-integration.md)

**Q89.** What should CI run for tests?

**Answer:** Build, unit and integration tests with reports on every change

Fast feedback.

**Q90.** Why run tests headless in CI?

**Answer:** No display is needed and runs are faster

Chrome headless.

**Q91.** What speeds CI?

**Answer:** Caching dependencies and parallelizing suites

Efficiency.

**Q92.** What should fail the build?

**Answer:** Any test failure or quality gate violation

Gate.

### [04 · Selenium Grid &amp; Parallel Execution](level-3/04-selenium-grid-parallel.md)

**Q93.** What is Selenium Grid?

**Answer:** A hub that distributes tests across nodes/browsers for parallel runs

Scale.

**Q94.** What does RemoteWebDriver need?

**Answer:** The Grid URL and desired capabilities/options

Remote.

**Q95.** What's a challenge of parallel UI tests?

**Answer:** Shared state and non-thread-safe drivers

Isolate.

**Q96.** How size a grid?

**Answer:** Match nodes to the number of parallel sessions

Capacity.

### [05 · Performance Testing Basics (JMeter)](level-3/05-performance-testing-jmeter.md)

**Q97.** What does JMeter test?

**Answer:** Performance: load, stress and endurance of services

Thread groups.

**Q98.** What's a thread group?

**Answer:** Simulated users running a test plan

Concurrency.

**Q99.** Which metrics matter?

**Answer:** Response time percentiles, throughput and error rate

p95/p99.

**Q100.** Why run performance tests in a prod-like environment?

**Answer:** Results depend on realistic capacity and data

Representative.

### [06 · Mobile Testing with Appium](level-3/06-mobile-testing-appium.md)

**Q101.** What's Appium?

**Answer:** Automation for mobile apps using the WebDriver protocol

iOS/Android.

**Q102.** What are desired capabilities?

**Answer:** Settings describing the device/app/platform to automate

Session config.

**Q103.** What's a mobile test challenge?

**Answer:** Device fragmentation and flaky gestures

Real devices/emulators.

**Q104.** What's an emulator vs real device?

**Answer:** Emulators are fast and cheap; real devices catch hardware quirks

Use both.

### [07 · Database Testing from Java](level-3/07-database-testing.md)

**Q105.** What should database tests verify?

**Answer:** Data changes made by the app and constraints/migrations

Integrity.

**Q106.** How isolate DB tests?

**Answer:** Transactions rolled back or containerized throwaway databases

Repeatable.

**Q107.** What's a safe way to query test DBs?

**Answer:** Parameterized queries via JDBC

Avoid injection.

**Q108.** What do migration tests ensure?

**Answer:** Schema changes apply cleanly and keep data intact

Flyway/Liquibase.

### [08 · Framework Design Patterns](level-3/08-framework-design-patterns.md)

**Q109.** Which pattern helps test frameworks?

**Answer:** Factory (drivers), Builder (data), Singleton-free config and Strategy (browsers)

Design.

**Q110.** What's a fluent interface?

**Answer:** Method chaining for readable setup/actions

Page objects.

**Q111.** What's the Single Responsibility principle for tests?

**Answer:** Each class/test has one reason to change

Cohesion.

**Q112.** What's a framework smell?

**Answer:** Static state and sleeps scattered throughout

Refactor.

### [09 · Flaky Test Diagnosis &amp; Stabilization](level-3/09-flaky-test-diagnosis.md)

**Q113.** What's the first step to diagnose a flaky test?

**Answer:** Run it repeatedly and capture failures with evidence

Data.

**Q114.** Common flakiness causes?

**Answer:** Async timing, shared state, order dependence and environment

Fix root cause.

**Q115.** Is a retry a fix?

**Answer:** No, it masks the issue; quarantine and fix

Track flake rate.

**Q116.** What reduces timing flakes?

**Answer:** Waiting on conditions instead of fixed delays

Explicit waits.

### [10 · Project — Hybrid BDD Framework with CI](level-3/10-project-bdd-framework-ci.md)

**Q117.** What does the BDD + CI project combine?

**Answer:** Cucumber features, page objects, reports and a CI pipeline

Level 3 project.

**Q118.** How keep feature files maintainable?

**Answer:** Business-readable steps reused across scenarios

Declarative.

**Q119.** How publish results?

**Answer:** CI artifacts and HTML/Allure reports

Visible.

**Q120.** What proves reliability?

**Answer:** Repeated green runs and low flake rate

Trend.

## Level 4 · Master

### [01 · Test Architecture at Scale](level-4/01-test-architecture-at-scale.md)

**Q121.** What does test architecture at scale require?

**Answer:** Layered, modular frameworks with shared libraries and clear ownership

Scalable.

**Q122.** What's the test pyramid's advice?

**Answer:** Many fast unit tests, fewer integration, fewest end-to-end

Cost and speed.

**Q123.** What reduces duplication across teams?

**Answer:** A shared test library with versioned APIs

Reuse.

**Q124.** What governs a big suite?

**Answer:** Ownership, tagging and running policies

Maintainability.

### [02 · Shift-Left &amp; Continuous Testing](level-4/02-shift-left-continuous-testing.md)

**Q125.** What does shift-left mean?

**Answer:** Testing earlier in the lifecycle (requirements, design, dev)

Cheaper fixes.

**Q126.** What's continuous testing?

**Answer:** Automated tests running throughout the pipeline for fast feedback

CI/CD.

**Q127.** What helps shift-left?

**Answer:** Unit/API tests, static analysis and test-first practices

Earlier.

**Q128.** What are quality gates?

**Answer:** Automated criteria blocking progression when unmet

Enforced.

### [03 · Contract Testing (Pact)](level-4/03-contract-testing-pact.md)

**Q129.** What problem does contract testing solve?

**Answer:** Verifying service interfaces between consumers and providers without full integration

Pact.

**Q130.** Who defines the contract in consumer-driven testing?

**Answer:** The consumer's expectations

CDC.

**Q131.** What's a Pact broker?

**Answer:** A server sharing and versioning contracts

Can-I-deploy.

**Q132.** Why not rely only on end-to-end tests?

**Answer:** They're slow and flaky for service interactions

Contracts localize failures.

### [04 · Security Testing Basics for QA](level-4/04-security-testing-basics.md)

**Q133.** What basic security tests can QA do?

**Answer:** Input validation, auth/authorization checks and dependency scanning

Basics.

**Q134.** What does OWASP ZAP do?

**Answer:** Scans web apps for common vulnerabilities (authorized targets)

DAST.

**Q135.** What does SCA find?

**Answer:** Known vulnerabilities in dependencies

OWASP Dependency-Check.

**Q136.** Where integrate security tests?

**Answer:** In CI so issues are caught early

Shift left.

### [05 · Test Metrics &amp; Quality Gates](level-4/05-test-metrics-quality-gates.md)

**Q137.** Which metrics are meaningful?

**Answer:** Defect escape rate, flake rate and time to feedback

Outcomes.

**Q138.** What's wrong with chasing coverage numbers?

**Answer:** They can be gamed and don't measure assertion quality

Goodhart.

**Q139.** What's a quality gate example?

**Answer:** No new critical bugs and coverage not decreasing

Ratchet.

**Q140.** What does mutation testing assess?

**Answer:** Whether tests detect small code changes

PIT.

### [06 · Containerized Test Environments](level-4/06-containerized-test-environments.md)

**Q141.** Why use Docker for test environments?

**Answer:** Reproducible, isolated and disposable environments

Consistency.

**Q142.** What's Testcontainers?

**Answer:** A library starting throwaway containers (DB, Kafka) for tests

Realistic.

**Q143.** What's docker-compose used for in testing?

**Answer:** Bringing up multi-service environments

Integration.

**Q144.** What's a downside?

**Answer:** Startup time and resource use

Reuse containers.

### [07 · AI-Assisted Testing](level-4/07-ai-assisted-testing.md)

**Q145.** How can AI assist testing?

**Answer:** Generating test ideas/data and summarizing failures, with human review

Assistive.

**Q146.** What's a risk of AI-generated tests?

**Answer:** Plausible but wrong assertions and false confidence

Verify.

**Q147.** What data shouldn't go into public AI tools?

**Answer:** Confidential or personal data

Privacy.

**Q148.** How validate AI output?

**Answer:** Review, run and mutation-check generated tests

Evidence.

### [08 · Building an In-House Test Framework](level-4/08-in-house-test-framework.md)

**Q149.** When build an in-house framework?

**Answer:** When constraints exceed existing tools and you can maintain it

Cost.

**Q150.** What are core components?

**Answer:** Runner, assertions, reporting and utilities

Minimal.

**Q151.** What's the cost?

**Answer:** Ongoing maintenance and missing community features

Prefer standard.

**Q152.** How design for adoption?

**Answer:** Simple API, docs and examples

Developer experience.

### [09 · QA Leadership &amp; Strategy](level-4/09-qa-leadership-strategy.md)

**Q153.** What does QA leadership focus on?

**Answer:** Strategy, risk-based testing, people and quality culture

Enablement.

**Q154.** What's risk-based testing?

**Answer:** Prioritizing effort by likelihood and impact of failure

Focus.

**Q155.** How grow a team?

**Answer:** Mentoring, clear standards and learning time

People.

**Q156.** How communicate quality?

**Answer:** Dashboards with actionable metrics and trends

Transparency.

### [10 · Capstone — Production-Grade Test Framework](level-4/10-capstone-test-framework.md)

**Q157.** What does the capstone combine?

**Answer:** Architecture, CI/CD, contracts, security, metrics and containers

Production framework.

**Q158.** How prove it works?

**Answer:** Stable runs, fast feedback and documented onboarding

Evidence.

**Q159.** What documentation helps?

**Answer:** Architecture, how to run/extend and troubleshooting

Adoption.

**Q160.** What sustains it?

**Answer:** Ownership, refactoring and flake budgets

Care.

## Mixed questions across all levels

**Q161.** What is the best definition of testing?

**Answer:** Evaluating a program against expectations to find defects and give quality information

It can't prove absence of defects.

**Q162.** Which inputs does boundary value analysis pick for a valid range of 18 to 65? (Y = accepted)

```java
public class Main {

public static void main(String[] args) throws Exception {
int[] v = {17, 18, 19, 64, 65, 66};
for (int a : v) System.out.print(a >= 18 && a <= 65 ? "Y" : "N");
}
}
```

**Answer:** NYYYYN

Test both sides of each boundary. Verified by compiling and running it.

**Q163.** What's the difference between severity and priority of a defect?

**Answer:** Severity is the impact; priority is how urgently to fix it

Triage uses both.

**Q164.** A test needs to wait for a button to become clickable. What do you use in Selenium?

**Answer:** An explicit wait on the condition (WebDriverWait)

Condition-based waits reduce flakiness.

**Q165.** Which locator is usually most stable?

**Answer:** A dedicated test attribute or ID

Locators should survive layout changes.

**Q166.** What is the Page Object Model's main benefit?

**Answer:** UI changes are fixed in one place instead of every test

Keep assertions in tests.

**Q167.** What does this hand-written mock/spy record?

```java
import java.util.*;
public class Main {
interface Notifier { void send(String m); }
static class Svc { Notifier n; Svc(Notifier n) { this.n = n; } void place(String id) { n.send(id); } }
public static void main(String[] args) throws Exception {
List<String> sent = new ArrayList<>();
Notifier n = sent::add;
new Svc(n).place("A1"); new Svc(n).place("A2");
System.out.println(sent);
}
}
```

**Answer:** [A1, A2]

Test doubles let you verify interactions. Verified by compiling and running it.

**Q168.** What's the difference between a stub and a mock?

**Answer:** Stubs return canned answers; mocks verify interactions

Mockito supports both.

**Q169.** A test passes on retry but fails first time. What should you do?

**Answer:** Treat it as flaky: find the root cause (timing, shared state), quarantine and fix

Retries mask problems.

**Q170.** Where should test data come from for independent tests?

**Answer:** Created per test (builders/factories) and cleaned up, not shared mutable data

Isolation.

**Q171.** What does REST Assured let you do in Java?

**Answer:** Write readable given/when/then tests for HTTP APIs

Verify status, headers and body.

**Q172.** Why test at the API level more than the UI level?

**Answer:** It's faster and more stable while covering the same logic

The test pyramid.

**Q173.** What does a Cucumber step definition do?

**Answer:** Binds a Gherkin step to executable code

BDD.

**Q174.** What does Selenium Grid add?

**Answer:** Parallel execution across browsers/nodes

Drivers must be per-thread.

**Q175.** Why give each parallel test thread its own WebDriver?

**Answer:** WebDriver isn't thread-safe

ThreadLocal&lt;WebDriver&gt;.

**Q176.** What does contract testing (Pact) verify?

**Answer:** That a consumer and provider agree on an API without a full integration environment

Consumer-driven.

**Q177.** A service dependency is slow to start in CI. What helps?

**Answer:** Containerized throwaway environments (Testcontainers/docker-compose)

Reproducible.

**Q178.** Why is a 90% coverage gate not enough?

**Answer:** Coverage doesn't measure assertion quality; use mutation testing and meaningful gates

Metrics can be gamed.

**Q179.** An AI tool generates tests for you. What must you do?

**Answer:** Review and run them; verify assertions are meaningful and no sensitive data was shared

AI is an assistant.

## Practical coding questions

### A tiny JUnit-style runner

Complete run(): discover @Test methods by reflection (sorted by name), create a fresh instance per test, run @BeforeEach methods, and report PASS/FAIL, honoring @Test(expected=...). Finish with a summary line.

Starter code:

```
import java.lang.annotation.*;
import java.lang.reflect.*;
import java.util.*;

public class Main {
    @Retention(RetentionPolicy.RUNTIME) @interface Test { Class<? extends Throwable> expected() default None.class; }
    @Retention(RetentionPolicy.RUNTIME) @interface BeforeEach {}
    static class None extends Throwable {}

    static class CalcTests {
        int acc;
        @BeforeEach void reset() { acc = 10; }
        @Test void adds() { acc += 5; check(acc == 15, "acc should be 15"); }
        @Test void freshStatePerTest() { check(acc == 10, "setup must run before each test"); }
        @Test void fails() { check(acc == 99, "expected 99"); }
        @Test(expected = ArithmeticException.class) void throwsExpected() { int z = 0; int r = 1 / z; }
        @Test(expected = IllegalStateException.class) void wrongException() { throw new IllegalArgumentException("x"); }
    }

    static void check(boolean ok, String msg) { if (!ok) throw new AssertionError(msg); }

    static void run(Class<?> cls) throws Exception {
        // for every @Test method (sorted by name): create a fresh instance, run @BeforeEach methods, invoke it;
        // PASS when it returns normally (and no exception was expected) or throws the expected exception;
        // otherwise print FAIL with the reason; finish with "<n> passed, <m> failed"
    }

    public static void main(String[] args) throws Exception { run(CalcTests.class); }
}
```

Expected output:

```text
PASS adds
FAIL fails: AssertionError expected 99
PASS freshStatePerTest
PASS throwsExpected
FAIL wrongException: IllegalArgumentException x
3 passed, 2 failed
```

??? success "Show a solution"

    ```
    import java.lang.annotation.*;
    import java.lang.reflect.*;
    import java.util.*;

    public class Main {
        @Retention(RetentionPolicy.RUNTIME) @interface Test { Class<? extends Throwable> expected() default None.class; }
        @Retention(RetentionPolicy.RUNTIME) @interface BeforeEach {}
        static class None extends Throwable {}

        static class CalcTests {
            int acc;
            @BeforeEach void reset() { acc = 10; }
            @Test void adds() { acc += 5; check(acc == 15, "acc should be 15"); }
            @Test void freshStatePerTest() { check(acc == 10, "setup must run before each test"); }
            @Test void fails() { check(acc == 99, "expected 99"); }
            @Test(expected = ArithmeticException.class) void throwsExpected() { int z = 0; int r = 1 / z; }
            @Test(expected = IllegalStateException.class) void wrongException() { throw new IllegalArgumentException("x"); }
        }

        static void check(boolean ok, String msg) { if (!ok) throw new AssertionError(msg); }

        static void run(Class<?> cls) throws Exception {
            int pass = 0, fail = 0;
            List<Method> tests = new ArrayList<>();
            for (Method m : cls.getDeclaredMethods()) if (m.isAnnotationPresent(Test.class)) tests.add(m);
            tests.sort(Comparator.comparing(Method::getName));
            for (Method t : tests) {
                Object inst = cls.getDeclaredConstructor().newInstance();
                for (Method m : cls.getDeclaredMethods()) if (m.isAnnotationPresent(BeforeEach.class)) { m.setAccessible(true); m.invoke(inst); }
                Class<? extends Throwable> expected = t.getAnnotation(Test.class).expected();
                t.setAccessible(true);
                try {
                    t.invoke(inst);
                    if (expected != None.class) { fail++; System.out.println("FAIL " + t.getName() + ": expected " + expected.getSimpleName()); }
                    else { pass++; System.out.println("PASS " + t.getName()); }
                } catch (InvocationTargetException e) {
                    Throwable c = e.getCause();
                    if (expected.isInstance(c)) { pass++; System.out.println("PASS " + t.getName()); }
                    else { fail++; System.out.println("FAIL " + t.getName() + ": " + c.getClass().getSimpleName() + (c.getMessage() != null ? " " + c.getMessage() : "")); }
                }
            }
            System.out.println(pass + " passed, " + fail + " failed");
        }

        public static void main(String[] args) throws Exception { run(CalcTests.class); }
    }
    ```

    This is how JUnit works conceptually: annotations mark tests, a fresh instance per test gives isolation, and an exception either fails the test or satisfies `expected`. Verified by running the solution with java Main.java.

### Test doubles for a checkout

Implement Checkout.pay: reject non-positive amounts before touching the gateway, charge, email a receipt on success and return PAID, otherwise DECLINED. The fake gateway and spy mailer record interactions so the assertions can verify them.

Starter code:

```
import java.util.*;

public class Main {
    interface PaymentGateway { boolean charge(String customer, int cents); }
    interface Mailer { void send(String to, String body); }

    static class FakeGateway implements PaymentGateway {
        final List<String> calls = new ArrayList<>();
        final Set<String> declines = new HashSet<>();
        public boolean charge(String customer, int cents) { calls.add(customer + ":" + cents); return !declines.contains(customer); }
    }
    static class SpyMailer implements Mailer {
        final List<String> sent = new ArrayList<>();
        public void send(String to, String body) { sent.add(to + "|" + body); }
    }

    static class Checkout {
        final PaymentGateway gw; final Mailer mailer;
        Checkout(PaymentGateway gw, Mailer mailer) { this.gw = gw; this.mailer = mailer; }
        String pay(String customer, int cents) {
            // throw IllegalArgumentException("amount must be positive") BEFORE touching the gateway when cents <= 0;
            // charge; on success email a receipt ("receipt <cents>") and return "PAID"; otherwise return "DECLINED"
            return "";
        }
    }

    public static void main(String[] args) {
        FakeGateway gw = new FakeGateway(); SpyMailer mail = new SpyMailer();
        gw.declines.add("bo");
        Checkout c = new Checkout(gw, mail);
        System.out.println(c.pay("ada", 500));
        System.out.println(c.pay("bo", 700));
        try { c.pay("cy", 0); } catch (IllegalArgumentException e) { System.out.println("rejected: " + e.getMessage()); }
        System.out.println("gateway calls: " + gw.calls);
        System.out.println("emails: " + mail.sent);
        System.out.println("no email for declined: " + mail.sent.stream().noneMatch(s -> s.startsWith("bo")));
        System.out.println("gateway untouched for invalid amount: " + !gw.calls.contains("cy:0"));
    }
}
```

Expected output:

```text
PAID
DECLINED
rejected: amount must be positive
gateway calls: [ada:500, bo:700]
emails: [ada|receipt 500]
no email for declined: true
gateway untouched for invalid amount: true
```

??? success "Show a solution"

    ```
    import java.util.*;

    public class Main {
        interface PaymentGateway { boolean charge(String customer, int cents); }
        interface Mailer { void send(String to, String body); }

        static class FakeGateway implements PaymentGateway {
            final List<String> calls = new ArrayList<>();
            final Set<String> declines = new HashSet<>();
            public boolean charge(String customer, int cents) { calls.add(customer + ":" + cents); return !declines.contains(customer); }
        }
        static class SpyMailer implements Mailer {
            final List<String> sent = new ArrayList<>();
            public void send(String to, String body) { sent.add(to + "|" + body); }
        }

        static class Checkout {
            final PaymentGateway gw; final Mailer mailer;
            Checkout(PaymentGateway gw, Mailer mailer) { this.gw = gw; this.mailer = mailer; }
            String pay(String customer, int cents) {
                if (cents <= 0) throw new IllegalArgumentException("amount must be positive");
                if (gw.charge(customer, cents)) { mailer.send(customer, "receipt " + cents); return "PAID"; }
                return "DECLINED";
            }
        }

        public static void main(String[] args) {
            FakeGateway gw = new FakeGateway(); SpyMailer mail = new SpyMailer();
            gw.declines.add("bo");
            Checkout c = new Checkout(gw, mail);
            System.out.println(c.pay("ada", 500));
            System.out.println(c.pay("bo", 700));
            try { c.pay("cy", 0); } catch (IllegalArgumentException e) { System.out.println("rejected: " + e.getMessage()); }
            System.out.println("gateway calls: " + gw.calls);
            System.out.println("emails: " + mail.sent);
            System.out.println("no email for declined: " + mail.sent.stream().noneMatch(s -> s.startsWith("bo")));
            System.out.println("gateway untouched for invalid amount: " + !gw.calls.contains("cy:0"));
        }
    }
    ```

    A fake gateway controls outcomes and a spy mailer records calls, so behavior and interactions can be asserted without real services. Verified by running the solution with java Main.java.

---

More free practice: the [Java Testing Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
