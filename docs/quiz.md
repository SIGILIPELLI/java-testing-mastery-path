---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Java Testing Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What is the best definition of testing?",
    "options": [
      "Evaluating a program against expectations to find defects and give quality information",
      "Writing code",
      "Clicking around",
      "Proving no bugs"
    ],
    "correctIndex": 0,
    "explanation": "It can't prove absence of defects."
  },
  {
    "question": "Which inputs does boundary value analysis pick for a valid range of 18 to 65? (Y = accepted)\n```java\npublic class Main {\n\npublic static void main(String[] args) throws Exception {\nint[] v = {17, 18, 19, 64, 65, 66};\nfor (int a : v) System.out.print(a >= 18 && a <= 65 ? \"Y\" : \"N\");\n}\n}\n```\n",
    "options": [
      "NNNNNN",
      "YYYYYY",
      "NYYYYN",
      "YNNNNY"
    ],
    "correctIndex": 2,
    "explanation": "Test both sides of each boundary. Verified by compiling and running it."
  },
  {
    "question": "What's the difference between severity and priority of a defect?",
    "options": [
      "Reverse",
      "None",
      "Severity is the impact; priority is how urgently to fix it",
      "Same"
    ],
    "correctIndex": 2,
    "explanation": "Triage uses both."
  },
  {
    "question": "A test needs to wait for a button to become clickable. What do you use in Selenium?",
    "options": [
      "Thread.sleep(5000)",
      "An explicit wait on the condition (WebDriverWait)",
      "Implicit wait only",
      "Retry the test"
    ],
    "correctIndex": 1,
    "explanation": "Condition-based waits reduce flakiness."
  },
  {
    "question": "Which locator is usually most stable?",
    "options": [
      "Visible text of a long paragraph",
      "Absolute XPath",
      "A dedicated test attribute or ID",
      "Index-based XPath"
    ],
    "correctIndex": 2,
    "explanation": "Locators should survive layout changes."
  },
  {
    "question": "What is the Page Object Model's main benefit?",
    "options": [
      "No waits needed",
      "Faster browsers",
      "UI changes are fixed in one place instead of every test",
      "Fewer assertions"
    ],
    "correctIndex": 2,
    "explanation": "Keep assertions in tests."
  },
  {
    "question": "What does this hand-written mock/spy record?\n```java\nimport java.util.*;\npublic class Main {\ninterface Notifier { void send(String m); }\nstatic class Svc { Notifier n; Svc(Notifier n) { this.n = n; } void place(String id) { n.send(id); } }\npublic static void main(String[] args) throws Exception {\nList<String> sent = new ArrayList<>();\nNotifier n = sent::add;\nnew Svc(n).place(\"A1\"); new Svc(n).place(\"A2\");\nSystem.out.println(sent);\n}\n}\n```\n",
    "options": [
      "[A1]",
      "Error",
      "[A1, A2]",
      "[]"
    ],
    "correctIndex": 2,
    "explanation": "Test doubles let you verify interactions. Verified by compiling and running it."
  },
  {
    "question": "What's the difference between a stub and a mock?",
    "options": [
      "Stubs return canned answers; mocks verify interactions",
      "Mocks are real objects",
      "Reverse",
      "Same"
    ],
    "correctIndex": 0,
    "explanation": "Mockito supports both."
  },
  {
    "question": "A test passes on retry but fails first time. What should you do?",
    "options": [
      "Delete it",
      "Ignore",
      "Treat it as flaky: find the root cause (timing, shared state), quarantine and fix",
      "Add retries forever"
    ],
    "correctIndex": 2,
    "explanation": "Retries mask problems."
  },
  {
    "question": "Where should test data come from for independent tests?",
    "options": [
      "Hard-coded in the UI",
      "A shared prod DB",
      "Order-dependent tests",
      "Created per test (builders/factories) and cleaned up, not shared mutable data"
    ],
    "correctIndex": 3,
    "explanation": "Isolation."
  },
  {
    "question": "What does REST Assured let you do in Java?",
    "options": [
      "Mock the DB",
      "Write readable given/when/then tests for HTTP APIs",
      "Load test",
      "Drive browsers"
    ],
    "correctIndex": 1,
    "explanation": "Verify status, headers and body."
  },
  {
    "question": "Why test at the API level more than the UI level?",
    "options": [
      "It's easier to see",
      "It's faster and more stable while covering the same logic",
      "Law",
      "UI can't be tested"
    ],
    "correctIndex": 1,
    "explanation": "The test pyramid."
  },
  {
    "question": "What does a Cucumber step definition do?",
    "options": [
      "Mocks",
      "Reports",
      "Binds a Gherkin step to executable code",
      "Locators"
    ],
    "correctIndex": 2,
    "explanation": "BDD."
  },
  {
    "question": "What does Selenium Grid add?",
    "options": [
      "Mocks",
      "Unit tests",
      "Parallel execution across browsers/nodes",
      "Reports"
    ],
    "correctIndex": 2,
    "explanation": "Drivers must be per-thread."
  },
  {
    "question": "Why give each parallel test thread its own WebDriver?",
    "options": [
      "Speed",
      "Law",
      "Style",
      "WebDriver isn't thread-safe"
    ],
    "correctIndex": 3,
    "explanation": "ThreadLocal<WebDriver>."
  },
  {
    "question": "What does contract testing (Pact) verify?",
    "options": [
      "Security",
      "That a consumer and provider agree on an API without a full integration environment",
      "UI",
      "Load"
    ],
    "correctIndex": 1,
    "explanation": "Consumer-driven."
  },
  {
    "question": "A service dependency is slow to start in CI. What helps?",
    "options": [
      "Skip",
      "Prod",
      "Sleep",
      "Containerized throwaway environments (Testcontainers/docker-compose)"
    ],
    "correctIndex": 3,
    "explanation": "Reproducible."
  },
  {
    "question": "Why is a 90% coverage gate not enough?",
    "options": [
      "Coverage doesn't measure assertion quality; use mutation testing and meaningful gates",
      "Coverage is fake",
      "It's perfect",
      "It's too low"
    ],
    "correctIndex": 0,
    "explanation": "Metrics can be gamed."
  },
  {
    "question": "An AI tool generates tests for you. What must you do?",
    "options": [
      "Review and run them; verify assertions are meaningful and no sensitive data was shared",
      "Trust them",
      "Delete old tests",
      "Commit blindly"
    ],
    "correctIndex": 0,
    "explanation": "AI is an assistant."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A tiny JUnit-style runner

Complete run(): discover @Test methods by reflection (sorted by name), create a fresh instance per test, run @BeforeEach methods, and report PASS/FAIL, honoring @Test(expected=...). Finish with a summary line.

**Starter code**

```python
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

**Expected output**

```text
PASS adds
FAIL fails: AssertionError expected 99
PASS freshStatePerTest
PASS throwsExpected
FAIL wrongException: IllegalArgumentException x
3 passed, 2 failed
```

<details><summary>Show solution and explanation</summary>

<pre><code>import java.lang.annotation.*;
import java.lang.reflect.*;
import java.util.*;

public class Main {
    @Retention(RetentionPolicy.RUNTIME) @interface Test { Class&lt;? extends Throwable&gt; expected() default None.class; }
    @Retention(RetentionPolicy.RUNTIME) @interface BeforeEach {}
    static class None extends Throwable {}

    static class CalcTests {
        int acc;
        @BeforeEach void reset() { acc = 10; }
        @Test void adds() { acc += 5; check(acc == 15, &quot;acc should be 15&quot;); }
        @Test void freshStatePerTest() { check(acc == 10, &quot;setup must run before each test&quot;); }
        @Test void fails() { check(acc == 99, &quot;expected 99&quot;); }
        @Test(expected = ArithmeticException.class) void throwsExpected() { int z = 0; int r = 1 / z; }
        @Test(expected = IllegalStateException.class) void wrongException() { throw new IllegalArgumentException(&quot;x&quot;); }
    }

    static void check(boolean ok, String msg) { if (!ok) throw new AssertionError(msg); }

    static void run(Class&lt;?&gt; cls) throws Exception {
        int pass = 0, fail = 0;
        List&lt;Method&gt; tests = new ArrayList&lt;&gt;();
        for (Method m : cls.getDeclaredMethods()) if (m.isAnnotationPresent(Test.class)) tests.add(m);
        tests.sort(Comparator.comparing(Method::getName));
        for (Method t : tests) {
            Object inst = cls.getDeclaredConstructor().newInstance();
            for (Method m : cls.getDeclaredMethods()) if (m.isAnnotationPresent(BeforeEach.class)) { m.setAccessible(true); m.invoke(inst); }
            Class&lt;? extends Throwable&gt; expected = t.getAnnotation(Test.class).expected();
            t.setAccessible(true);
            try {
                t.invoke(inst);
                if (expected != None.class) { fail++; System.out.println(&quot;FAIL &quot; + t.getName() + &quot;: expected &quot; + expected.getSimpleName()); }
                else { pass++; System.out.println(&quot;PASS &quot; + t.getName()); }
            } catch (InvocationTargetException e) {
                Throwable c = e.getCause();
                if (expected.isInstance(c)) { pass++; System.out.println(&quot;PASS &quot; + t.getName()); }
                else { fail++; System.out.println(&quot;FAIL &quot; + t.getName() + &quot;: &quot; + c.getClass().getSimpleName() + (c.getMessage() != null ? &quot; &quot; + c.getMessage() : &quot;&quot;)); }
            }
        }
        System.out.println(pass + &quot; passed, &quot; + fail + &quot; failed&quot;);
    }

    public static void main(String[] args) throws Exception { run(CalcTests.class); }
}
</code></pre>

<p>This is how JUnit works conceptually: annotations mark tests, a fresh instance per test gives isolation, and an exception either fails the test or satisfies `expected`. Verified by running the solution with java Main.java.</p>
</details>

### Test doubles for a checkout

Implement Checkout.pay: reject non-positive amounts before touching the gateway, charge, email a receipt on success and return PAID, otherwise DECLINED. The fake gateway and spy mailer record interactions so the assertions can verify them.

**Starter code**

```python
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

**Expected output**

```text
PAID
DECLINED
rejected: amount must be positive
gateway calls: [ada:500, bo:700]
emails: [ada|receipt 500]
no email for declined: true
gateway untouched for invalid amount: true
```

<details><summary>Show solution and explanation</summary>

<pre><code>import java.util.*;

public class Main {
    interface PaymentGateway { boolean charge(String customer, int cents); }
    interface Mailer { void send(String to, String body); }

    static class FakeGateway implements PaymentGateway {
        final List&lt;String&gt; calls = new ArrayList&lt;&gt;();
        final Set&lt;String&gt; declines = new HashSet&lt;&gt;();
        public boolean charge(String customer, int cents) { calls.add(customer + &quot;:&quot; + cents); return !declines.contains(customer); }
    }
    static class SpyMailer implements Mailer {
        final List&lt;String&gt; sent = new ArrayList&lt;&gt;();
        public void send(String to, String body) { sent.add(to + &quot;|&quot; + body); }
    }

    static class Checkout {
        final PaymentGateway gw; final Mailer mailer;
        Checkout(PaymentGateway gw, Mailer mailer) { this.gw = gw; this.mailer = mailer; }
        String pay(String customer, int cents) {
            if (cents &lt;= 0) throw new IllegalArgumentException(&quot;amount must be positive&quot;);
            if (gw.charge(customer, cents)) { mailer.send(customer, &quot;receipt &quot; + cents); return &quot;PAID&quot;; }
            return &quot;DECLINED&quot;;
        }
    }

    public static void main(String[] args) {
        FakeGateway gw = new FakeGateway(); SpyMailer mail = new SpyMailer();
        gw.declines.add(&quot;bo&quot;);
        Checkout c = new Checkout(gw, mail);
        System.out.println(c.pay(&quot;ada&quot;, 500));
        System.out.println(c.pay(&quot;bo&quot;, 700));
        try { c.pay(&quot;cy&quot;, 0); } catch (IllegalArgumentException e) { System.out.println(&quot;rejected: &quot; + e.getMessage()); }
        System.out.println(&quot;gateway calls: &quot; + gw.calls);
        System.out.println(&quot;emails: &quot; + mail.sent);
        System.out.println(&quot;no email for declined: &quot; + mail.sent.stream().noneMatch(s -&gt; s.startsWith(&quot;bo&quot;)));
        System.out.println(&quot;gateway untouched for invalid amount: &quot; + !gw.calls.contains(&quot;cy:0&quot;));
    }
}
</code></pre>

<p>A fake gateway controls outcomes and a spy mailer records calls, so behavior and interactions can be asserted without real services. Verified by running the solution with java Main.java.</p>
</details>
