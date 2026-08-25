# 05 · Performance Testing Basics (JMeter)

Every test in this course so far asks "is the answer correct?" Performance
testing asks a different question: "does the answer still arrive fast
enough, and does the system stay up, when a thousand people ask at once?"
Apache JMeter is the standard open-source tool for that, and — because it's
Java underneath — it also embeds cleanly into a Maven build.

## 1. Core JMeter concepts

- **Thread Group** — a pool of virtual users; "threads" = concurrent users,
  "ramp-up" = how long to take spinning them all up, "loop count" = how many
  times each user repeats the flow.
- **Sampler** — the actual request (HTTP Request sampler for a REST API,
  matching what you tested functionally with RestAssured in Level 2).
- **Listener** — collects and displays results (Summary Report,
  Aggregate Report, View Results Tree).
- **Assertion** — a pass/fail check on the response, same idea as a
  RestAssured `.body(...)` assertion but evaluated under load.

## 2. A minimal test plan (GUI concept, JMX under the hood)

```xml
<!-- simplified excerpt of a .jmx file -->
<ThreadGroup>
  <stringProp name="ThreadGroup.num_threads">50</stringProp>
  <stringProp name="ThreadGroup.ramp_time">10</stringProp>
  <elementProp name="ThreadGroup.main_controller">
    <stringProp name="LoopController.loops">20</stringProp>
  </elementProp>
</ThreadGroup>
<HTTPSamplerProxy>
  <stringProp name="HTTPSampler.domain">jsonplaceholder.typicode.com</stringProp>
  <stringProp name="HTTPSampler.path">/posts/1</stringProp>
  <stringProp name="HTTPSampler.method">GET</stringProp>
</HTTPSamplerProxy>
<ResponseAssertion>
  <stringProp name="Assertion.test_field">Assertion.response_code</stringProp>
  <stringProp name="49586">200</stringProp>
</ResponseAssertion>
```

50 threads ramping up over 10 seconds, each looping the request 20 times —
1,000 total requests hitting the same endpoint you tested functionally with
`given().when().get("/posts/1")` in Level 2. The JMX is normally built in
the JMeter GUI (`jmeter.bat`/`jmeter.sh`) and saved to this XML, not
hand-written.

## 3. Running headlessly and reading results

```bash
jmeter -n -t posts-load-test.jmx -l results.jtl -e -o report/
```

`-n` = non-GUI (required for CI and any real load — the GUI itself consumes
resources you want going to generating load, not rendering it), `-l` writes
raw results, `-e -o` generates an HTML dashboard report from them.

```
summary +   1000 in 00:00:11 =   90.9/s Avg:   142 Min:    38 Max:   891 Err:     3 (0.30%)
summary =   1000 in 00:00:11 =   90.9/s Avg:   142 Err:     3 (0.30%)
```

- **Throughput** (`90.9/s`) — requests completed per second.
- **Avg / Min / Max** — response time in ms across all 1,000 requests.
- **Err** — failed requests (non-2xx or a failed assertion), as a count and
  percentage.

An average of 142ms with a max of 891ms on 0.3% of requests is a classic
shape: most requests are fine, a tail is slow — worth investigating with the
Aggregate Report's percentile columns (p90/p95/p99) rather than trusting the
average alone, since an average hides exactly the tail that users actually
notice.

## 4. Driving JMeter from Java (embedded, in a Maven build)

```xml
<dependency>
    <groupId>org.apache.jmeter</groupId>
    <artifactId>ApacheJMeter_core</artifactId>
    <version>5.6.3</version>
</dependency>
<dependency>
    <groupId>org.apache.jmeter</groupId>
    <artifactId>ApacheJMeter_http</artifactId>
    <version>5.6.3</version>
</dependency>
```

```java
package com.example.perf;

import org.apache.jmeter.config.Arguments;
import org.apache.jmeter.protocol.http.sampler.HTTPSamplerProxy;
import org.apache.jmeter.testelement.TestPlan;
import org.apache.jmeter.threads.ThreadGroup;
import org.apache.jmeter.control.LoopController;
import org.apache.jmeter.engine.StandardJMeterEngine;
import org.apache.jmeter.util.JMeterUtils;
import org.apache.jorphan.collections.HashTree;

public class SmokeLoadTest {

    public static void main(String[] args) throws Exception {
        JMeterUtils.loadJMeterProperties("jmeter.properties");
        JMeterUtils.setJMeterHome(System.getenv("JMETER_HOME"));

        HTTPSamplerProxy sampler = new HTTPSamplerProxy();
        sampler.setDomain("jsonplaceholder.typicode.com");
        sampler.setPath("/posts/1");
        sampler.setMethod("GET");

        LoopController loopController = new LoopController();
        loopController.setLoops(10);
        loopController.setFirst(true);
        loopController.initialize();

        ThreadGroup threadGroup = new ThreadGroup();
        threadGroup.setNumThreads(20);
        threadGroup.setRampUp(5);
        threadGroup.setSamplerController(loopController);

        HashTree testPlanTree = new HashTree();
        HashTree threadGroupTree = testPlanTree.add(new TestPlan(), threadGroup);
        threadGroupTree.add(sampler);

        StandardJMeterEngine engine = new StandardJMeterEngine();
        engine.configure(testPlanTree);
        engine.run();
    }
}
```

This is what a "run a load smoke test as part of the build" step looks like
— 20 users, 10 loops each, 200 requests, executed the same way `mvn test`
executes JUnit, so a regression in response time can gate a pipeline exactly
like a failing assertion does.

I did not run actual JMeter (GUI or embedded) in this environment — no
JMeter installation or `JMETER_HOME` here — so this module is entirely
reviewed against JMeter 5.6's documented CLI flags and Java API, not
executed; the sample summary-report numbers above are illustrative of the
*shape* of real output, not a captured run.

## 5. Setting pass/fail thresholds (turning load tests into gates)

```bash
jmeter -n -t posts-load-test.jmx -l results.jtl \
  -J jmeter.reportgenerator.summary.filter="p90<500,errorPercentage<1"
```

More commonly done via a plugin or a small script parsing `results.jtl`:

```java
// pseudo-check you'd run after the .jtl is produced, in CI
double p95 = PerfReportParser.percentile(results, 95);
double errorRate = PerfReportParser.errorRate(results);

assertTrue(p95 < 800, "p95 latency " + p95 + "ms exceeds 800ms budget");
assertTrue(errorRate < 0.01, "error rate " + errorRate + " exceeds 1% budget");
```

Wrapping the parsed JTL in ordinary JUnit assertions is what lets a
performance budget fail a build the same way `assertEquals` does — no
separate performance dashboard is required to make the number actionable.

## 6. Testing traps

!!! warning "Trap 1 — testing against a shared/production environment"
    Firing 1,000 requests/second at a shared staging environment other
    teams rely on turns a performance test into an accidental denial of
    service for everyone else. Use an isolated environment sized like
    production, or throttle deliberately and communicate the window.

!!! warning "Trap 2 — trusting the average, ignoring the tail"
    An average of 142ms can hide a p99 of 4 seconds affecting 1% of real
    users — at scale, 1% is thousands of people. Always look at p90/p95/p99,
    not just mean.

!!! warning "Trap 3 — JMeter itself becomes the bottleneck"
    Running the GUI while generating load, or running too many threads on
    underpowered hardware, produces response times that reflect JMeter's own
    CPU starvation, not the system under test. Always run `-n` (non-GUI) for
    real numbers, and watch the JMeter host's own CPU/memory during the run.

!!! warning "Trap 4 — no ramp-up, thundering herd"
    `ramp_time=0` with 500 threads hits the server with 500 simultaneous
    connections in the first second — a spike test, not a realistic load
    test, unless a spike is specifically what you're trying to measure.
    Choose ramp-up to model how traffic actually arrives.

!!! warning "Trap 5 — functional bugs disguised as performance failures"
    A response time that balloons under load is sometimes a functional bug
    (an N+1 query, a missing index) rather than a capacity limit. Before
    concluding "we need more servers," check whether the *same* request
    replayed once, alone, is also slow — that points at code, not scale.

## Cheat sheet

| Concept | JMeter term |
|---|---|
| Concurrent virtual users | Thread Group → `num_threads` |
| Time to spin all users up | Thread Group → `ramp_time` |
| Repeats per user | Loop Controller → `loops` |
| The actual request | HTTP Request sampler |
| Pass/fail check | Response Assertion |
| Requests/sec | Throughput |
| 90th/95th/99th percentile latency | Aggregate Report percentile columns |
| Run headlessly | `jmeter -n -t plan.jmx -l results.jtl` |
| Generate HTML dashboard | `-e -o report/` |
| Error count/rate | `Err` column in summary output |

## Exercise

1. Install JMeter (or note that you're reviewing rather than running it) and
   build a test plan hitting `https://jsonplaceholder.typicode.com/posts/1`
   with 50 threads, 10s ramp-up, 20 loops. Run it headlessly and capture the
   summary line.
2. Identify p90 and p95 from the Aggregate Report and compare them to the
   average — write one sentence on what the gap tells you.
3. Re-run with `ramp_time=0` (thundering herd) and compare the error rate to
   the ramped run.
4. Write `PerfReportParser` (or a stub of it) that reads a `.jtl` file and
   computes error rate and a given percentile, then wrap it in a JUnit test
   asserting `p95 < 800` — decide the budget yourself and justify it in a
   comment.
5. Take one endpoint you wrote a RestAssured functional test for in Level 2
   Module 04 and design (on paper, no need to run it) a load-test plan for
   it: thread count, ramp-up, loop count, and the pass/fail thresholds you'd
   set, with a one-paragraph justification for each number.
