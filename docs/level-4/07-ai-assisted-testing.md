# 07 · AI-Assisted Testing

AI tools (code-completion assistants, LLM-based test generators, AI-driven
visual diffing) are now a normal part of a testing workflow — the question
isn't whether to use them, but which testing tasks they genuinely help
with, and which ones they quietly make worse by looking helpful while
generating tests that assert nothing meaningful.

## 1. Where AI assistance is genuinely strong

- **Boilerplate generation** — a Page Object skeleton, a parameterized test
  shell, a Mockito setup for an interface with ten methods. High
  time-savings, low risk: a human still reviews the assertions.
- **Test data generation** — realistic-looking names, addresses, edge-case
  strings (unicode, very long, empty) for fuzz-style input testing.
- **Explaining a failure** — pasting a stack trace and getting a plausible
  first hypothesis for root cause, especially for unfamiliar frameworks.
- **Suggesting edge cases a human missed** — "what happens if this list is
  empty," "what about a negative number here" — a second pair of eyes on
  a spec, not a replacement for domain judgment.

## 2. Where it needs a skeptical human in the loop

```java
// Prompt given to an AI assistant: "write a test for this method"
public int applyCode(String code) {
    int sum = total();
    Integer percent = DISCOUNT_PERCENT.get(code);
    if (percent == null) return sum;
    return sum - (sum * percent / 100);
}

// A plausible AI-generated test -- looks reasonable, has a real bug
@Test
void testApplyCode() {
    Cart cart = new Cart();
    cart.addItem("item", 1000);
    int result = cart.applyCode("SAVE10");
    assertNotNull(result);   // int can never be null -- this assertion is dead weight
}
```

`assertNotNull` on a primitive `int` (autoboxed to `Integer` for the call,
always non-null) always passes and catches nothing — a pattern that shows
up often in AI-generated tests because it's syntactically valid and
"looks like" an assertion, without the generator having genuinely reasoned
about what value the method should actually produce.

```java
// The test that actually verifies behavior -- requires a human who knows
// the expected number, which an AI without full business context can't derive alone
@Test
void testApplyCode() {
    Cart cart = new Cart();
    cart.addItem("item", 1000);
    assertEquals(900, cart.applyCode("SAVE10"));   // the actual expected value
}
```

I ran both versions locally against Level 3 Module 10's `Cart`: the
`assertNotNull` version passes trivially and would keep passing even if
`applyCode` returned a completely wrong number; the corrected version with
`assertEquals(900, ...)` passes now and would correctly fail if the
discount logic broke.

```
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0   <- assertNotNull version, passes but proves nothing
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0   <- assertEquals version, passes AND proves the value
```

## 3. A checklist for reviewing AI-generated tests

1. **Does every assertion have a specific expected value**, not just
   "not null" / "no exception" / "is an instance of"?
2. **Would this test fail if the implementation had an off-by-one, wrong
   sign, or swapped-argument bug?** If you can't answer confidently, mutate
   the code slightly and check (this is literally mutation testing, Level 4
   Module 05, applied to one test manually).
3. **Does the test name describe the behavior, or just the method under
   test?** `testApplyCode` tells a reader nothing; `discountCodeAppliesPercentageOff`
   does.
4. **Is the test data realistic, or an unexamined placeholder** (`"foo"`,
   `"test123"`) that happens to avoid an edge case the real system will
   hit?
5. **Did a human actually run it and watch it fail once** (Level 4 Module
   02's TDD red step) before trusting the green?

## 4. AI-assisted test data generation — a legitimate strong use case

```java
// Using an AI-suggested list of adversarial strings for a fuzz-style test,
// combined with the allowlist validator from Level 4 Module 04
@ParameterizedTest
@ValueSource(strings = {
    "",                                   // empty
    " ",                                  // whitespace only
    "a",                                  // too short
    "ThisUsernameIsWayTooLongForTheField", // too long
    "user name",                          // embedded space
    "user\tname",                         // embedded tab
    "🙂🙂🙂",                              // emoji / multi-byte unicode
    "user@name",                          // embedded @, invalid character
    "-1",                                 // looks numeric
    "' OR 1=1 --"                         // injection-shaped (Level 4 Module 04)
})
void edgeCaseUsernamesAreAllRejected(String input) {
    assertFalse(InputValidator.isValidUsername(input));
}
```

This is where AI genuinely earns its keep: generating a *breadth* of
plausible-adversarial inputs faster than a human would brainstorm them, for
a human-written allowlist assertion the AI didn't need business context to
validate.

I ran this exact parameterized test locally against Level 4 Module 04's
`InputValidator` — with one instructive detour. The original list I wrote
included the literal string `"NULL"` as a case, expecting it to be
rejected. It wasn't: `"NULL"` is four alphanumeric characters, which
satisfies `isValidUsername`'s allowlist regex perfectly well, so the
assertion failed:

```
Tests run: 10, Failures: 1, Errors: 0, Skipped: 0
com.example.security.FuzzUsernameTest.edgeCaseUsernamesAreAllRejected(String)[8] FAILED
```

That's not a bug in the test framework — it's the AI-assisted-input-list
version of Trap 2 below: a plausible-looking adversarial string that isn't
actually adversarial *to this specific validator*, generated without
checking it against the real implementation first. `isValidUsername` was
never meant to reject every suspicious-looking string, only to enforce a
character/length shape — `"NULL"` is a bad *username choice*, not an
invalid one by this rule. I swapped it for `"user@name"` (genuinely
rejected, since `@` isn't in the allowlist) and reran:

```
Tests run: 10, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

## 5. AI-driven visual regression testing (concept)

```java
// Concept: an AI-based visual diff tool flags "meaningfully different"
// pixels rather than exact pixel-match, tolerating anti-aliasing/font
// rendering noise that breaks naive pixel-diff tools
VisualCheckResult result = visualTestClient.compare(
        "checkout-page", currentScreenshot, baselineScreenshot);

assertTrue(result.isMatch(),
        "Visual diff detected: " + result.diffPercentage() + "% changed, region: " + result.diffRegion());
```

Naive pixel-diffing (comparing raw bytes) is notoriously flaky across
different machines' font rendering and anti-aliasing; AI-based perceptual
diffing tools claim to reduce that noise while still catching real layout
regressions. This is conceptual here — no such tool is wired up or run in
this environment.

## 6. Testing traps

!!! warning "Trap 1 — trusting green without reading the assertion"
    A suite that grew rapidly using AI-generated tests can pass CI at
    100% while a meaningful fraction of its assertions are
    `assertNotNull`/`assertDoesNotThrow` equivalents that verify nothing
    specific. Coverage and pass-rate dashboards (Level 4 Module 05) don't
    distinguish this from real coverage — a human review pass or mutation
    testing does.

!!! warning "Trap 2 — AI-generated tests encoding a misunderstanding as fact"
    If the AI misunderstands the intended business rule (e.g. assumes a
    discount stacks when it shouldn't), it can generate a confident,
    well-structured test that locks in the *wrong* behavior — and that test
    will then fail the day someone correctly fixes the bug, creating
    exactly backwards pressure.

!!! warning "Trap 3 — over-reliance eroding the skill to write tests without it"
    A tester who has only ever accepted AI-suggested tests may struggle to
    recognize when a suggestion is subtly wrong, because the review skill
    itself atrophies without regular independent practice. Keep writing
    some tests unaided, deliberately, to keep the review instinct sharp.

!!! warning "Trap 4 — feeding sensitive data into an external AI tool"
    Pasting real production data, real credentials, or a proprietary
    algorithm into a third-party AI assistant to "help write a test" can
    violate data-handling policy even when the intent is entirely
    legitimate. Use synthetic data (section 4's fuzz list is a good model)
    for anything sent to an external service.
!!! warning "Trap 5 — AI-suggested flaky waits"
    An AI assistant unfamiliar with a specific app's timing characteristics
    will often suggest `Thread.sleep(2000)` as the path of least resistance
    for "wait for this to load" — exactly the anti-pattern Level 3 Module
    09 spent a full module fixing. Apply the same skepticism to an
    AI-suggested wait as to a human-written one.

## Cheat sheet

| Task | AI-assist confidence | Human responsibility |
|---|---|---|
| Test class/method boilerplate | High | Fill in real assertions |
| Fuzz/edge-case input lists | High | Confirm expected behavior for each |
| Explaining a stack trace | Medium-high | Verify the hypothesis against the actual code |
| Writing the actual expected value | Low (without full context) | Always confirm manually |
| Business-rule correctness | Low | Domain knowledge stays human |
| Visual diff tooling | Tool-dependent | Confirm flagged diffs are real regressions |

## Exercise

1. Take a method from any earlier module in this course, generate a test
   for it using an AI assistant (or simulate the exercise by writing one
   quickly without checking the expected value first), then apply the
   5-point checklist from section 3 to your own output.
2. Find (or deliberately write) one `assertNotNull`/`assertDoesNotThrow`-only
   test in your own suite, mutate the implementation to be subtly wrong,
   and confirm the test still passes — then fix the test with a real
   expected value.
3. Build the 10-case fuzz test from section 4 against
   `InputValidator.isValidUsername`, run it, and add three more adversarial
   strings of your own.
4. Write one sentence each for the five traps in section 6 describing a
   concrete way you'd catch that trap in a code review, specifically (not
   "be careful").
5. Pick one test you're confident is well-written from this course, and one
   you suspect might have a weak assertion. Apply mutation-testing logic
   (Level 4 Module 05) by hand to each: change the implementation slightly
   and see which test actually catches it.
