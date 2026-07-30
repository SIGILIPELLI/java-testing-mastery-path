# 01 · What Is Software Testing?

Software testing is the disciplined process of evaluating a product to find
the gap between **what it does** and **what it is supposed to do**. That gap
is a defect. Testing does not "prove software works" — you can never run
every possible input — it *reduces the risk* of shipping something broken, by
deliberately choosing the checks most likely to expose problems.

This module gives you the vocabulary and mental model the rest of the course
builds on: manual vs automation, QA vs QC, the tester mindset, and the two
lifecycles (SDLC and STLC) that describe where testing fits.

## 1. Why testing exists

Every defect has a cost, and that cost multiplies the later it is found.

| Found during | Relative cost to fix | Why |
|---|---|---|
| Requirements | 1× | Edit a sentence in a document |
| Design | 3–5× | Redraw a diagram, no code written yet |
| Coding | 10× | Developer changes code, re-tests locally |
| System testing | 15–40× | Bug report, triage, fix, retest, regression |
| Production | 30–100×+ | Hotfix, rollback, support tickets, lost users, reputational damage |

This table is the single strongest argument for a tester's existence, and it
also explains **shift-left testing** (Level 4): the earlier you look, the
cheaper the find. A tester who reviews a requirements document and asks "what
happens if the user enters a negative amount?" has just prevented a defect
for the cost of one question.

!!! info "Testing vs debugging"
    **Testing** finds *that* something is wrong and describes the symptom.
    **Debugging** finds *why* it is wrong and fixes it. Testing is normally a
    QA activity; debugging is normally a developer activity. Keeping the two
    separate is deliberate — a tester who guesses at root causes in a bug
    report often sends developers down the wrong path.

## 2. Manual testing vs automation testing

These are not competing choices; they are tools for different jobs.

| Aspect | Manual testing | Automation testing |
|---|---|---|
| Who performs it | A human executes steps and observes | A script executes steps and asserts |
| Best for | Exploratory, usability, ad-hoc, one-off, UX judgement | Regression, repetitive checks, data-driven, load |
| Cost profile | Low setup, high per-run cost | High setup, near-zero per-run cost |
| Speed | Slow, human-paced | Fast, runs overnight or on every commit |
| Reliability | Humans miss things when bored | Runs identically every time (unless flaky) |
| Finds new bugs? | Yes — a human notices "that looks odd" | Rarely — it only checks what you told it to |
| Handles UI changes | Adapts instantly | Breaks until the script is updated |

The practical rule most teams settle on:

- **Automate** anything you will run more than about five times, that has a
  clear pass/fail answer — login, checkout, API contracts, calculations.
- **Test manually** anything requiring judgement — is this error message
  actually helpful? Does this layout look broken on a phone? What happens if
  I do something weird?

!!! warning "The most common beginner misconception"
    "Automation will replace manual testing." It does not. Automation replaces
    *repeated execution*, not *thinking*. A suite of 2,000 automated tests
    still cannot tell you the checkout button is now invisible against the
    background. Every strong automation engineer started by learning to test
    manually — which is exactly why Modules 01–05 of this course come before
    a single line of Java.

## 3. QA vs QC vs testing

These three terms get used interchangeably in job ads and it causes real
confusion. They are distinct.

| Term | Full form | Focus | Nature | Example activity |
|---|---|---|---|---|
| **QA** | Quality Assurance | The **process** that builds the product | Preventive, proactive | Defining a code review standard, setting the definition of "done" |
| **QC** | Quality Control | The **product** that was built | Corrective, reactive | Executing a regression suite before release |
| **Testing** | — | A specific activity | A subset of QC | Running test case TC-014 and logging the result |

A one-line memory hook: **QA prevents defects; QC detects defects; testing is
how QC detects them.**

## 4. The tester mindset

Skills are teachable in weeks. The mindset is what separates a good tester
from someone who just clicks through a checklist.

1. **Assume nothing works until shown otherwise.** The developer's "it works
   on my machine" is a hypothesis, not evidence.
2. **Ask "what if?" relentlessly.** What if the field is empty? What if it
   holds 10,000 characters? What if I click Submit twice? What if the network
   drops mid-request? What if the date is 29 February?
3. **Think like the user who did not read the manual.** Real users paste
   formatted text, use the back button mid-flow, and leave a tab open for
   three days.
4. **Report facts, not opinions.** "Login is broken" is an opinion. "Clicking
   Login with a valid email and correct password returns a 500 error page;
   expected the dashboard" is a fact someone can act on.
5. **Be an advocate for quality, not an adversary.** You are not trying to
   make developers look bad; you are both trying to keep users happy. Tone in
   bug reports is a career skill.
6. **Accept that you cannot test everything.** Exhaustive testing is
   impossible — a single field accepting 10 characters has more combinations
   than seconds in the universe's lifetime. Your job is *risk-based
   prioritization*, which Module 05 teaches formally.

!!! info "The seven principles of testing"
    A widely-taught ISTQB framing worth memorizing:
    (1) testing shows the presence of defects, never their absence;
    (2) exhaustive testing is impossible;
    (3) early testing saves time and money;
    (4) defects cluster — a small number of modules hold most bugs;
    (5) the pesticide paradox — the same tests stop finding new bugs, so tests
    must be reviewed and refreshed;
    (6) testing is context-dependent — a banking app is tested differently
    from a game;
    (7) absence-of-errors is a fallacy — a bug-free product that solves the
    wrong problem is still a failure.

## 5. SDLC — where testing sits

The **Software Development Life Cycle** is the end-to-end process of building
software. The classic phases:

1. **Requirement gathering** — what should it do? *(Testers review for
   testability and ambiguity.)*
2. **Analysis & planning** — feasibility, scope, effort. *(Testers estimate
   test effort.)*
3. **Design** — architecture, database, UI mockups. *(Testers begin writing
   test scenarios.)*
4. **Implementation / coding** — developers build it. *(Testers write test
   cases and set up environments.)*
5. **Testing** — defects found, reported, fixed, retested.
6. **Deployment** — released to production.
7. **Maintenance** — patches, enhancements, regression testing forever after.

Common SDLC models you will hear named:

| Model | How it works | Testing implication |
|---|---|---|
| **Waterfall** | Strictly sequential phases | Testing is a late, large phase — defects found expensively |
| **V-Model** | Each dev phase paired with a matching test phase | Test planning starts alongside requirements |
| **Agile / Scrum** | Short iterations (sprints), continuous delivery | Testers embedded in the team, testing every sprint |
| **DevOps** | Continuous integration and deployment | Automated tests gate every commit (Level 3) |

Most teams you join today will be Agile or DevOps-flavoured, which is why
automation and CI (Level 3, Module 03) matter so much in modern QA roles.

## 6. STLC — the testing life cycle

The **Software Testing Life Cycle** zooms into the testing work itself. Each
phase has an entry criterion, an activity, and a deliverable — knowing the
deliverables is a common interview question.

| # | Phase | Activity | Deliverable |
|---|---|---|---|
| 1 | **Requirement analysis** | Study requirements, identify what is testable, raise ambiguities | Requirement clarification list, RTM draft |
| 2 | **Test planning** | Define scope, approach, resources, schedule, risks | **Test plan**, effort estimate |
| 3 | **Test case design** | Write test scenarios and detailed test cases, prepare test data | Test cases, test data, updated RTM |
| 4 | **Test environment setup** | Prepare hardware/software/test servers, smoke-test the build | Ready environment, smoke test results |
| 5 | **Test execution** | Run test cases, log results, raise defects, retest fixes | Execution report, defect reports |
| 6 | **Test closure** | Analyse results, capture lessons learned, archive artefacts | **Test summary report**, closure report |

!!! info "Entry and exit criteria"
    Each STLC phase has an **entry criterion** (what must be true to start)
    and an **exit criterion** (what must be true to finish). For test
    execution, entry might be "build deployed and smoke test passed"; exit
    might be "100% of planned test cases executed, zero open Critical or High
    defects, all Medium defects triaged." Defining exit criteria *before*
    execution is what stops the endless "are we done testing yet?" argument.

## 7. Common roles you will meet

| Role | Primary responsibility |
|---|---|
| Manual / Functional Tester | Designs and executes test cases, raises defects |
| Automation Test Engineer / SDET | Builds and maintains automated suites and frameworks |
| Performance Test Engineer | Load, stress and endurance testing (Level 3, Module 05) |
| QA Lead / Test Manager | Test strategy, estimation, resourcing, reporting to stakeholders |
| Business Analyst | Owns requirements — your first stop when a spec is ambiguous |
| Developer | Writes code and unit tests; fixes your defects |

## Glossary

| Term | Meaning |
|---|---|
| **Error / Mistake** | A human action producing an incorrect result (a developer's typo) |
| **Defect / Bug / Fault** | The flaw in the code caused by that error |
| **Failure** | What the user experiences when the defect is executed |
| **Verification** | "Are we building the product right?" — reviews, static analysis |
| **Validation** | "Are we building the right product?" — actual execution |
| **Test scenario** | A high-level "what to test" statement |
| **Test case** | A detailed, step-by-step, executable check with an expected result |
| **Test suite** | A collection of related test cases |
| **Build** | A compiled, deployable version of the application given to testing |
| **RTM** | Requirement Traceability Matrix — maps requirements to test cases |
| **Regression** | Re-testing existing features to confirm a change broke nothing |

## Exercise

Pick any app you use daily — a food-delivery app, your banking app, or a
website's login page.

1. Write down **five "what if?" questions** a tester would ask about one
   single screen of it. At least two must be about invalid or unusual input.
2. For that same screen, classify each of your five questions as something
   you would **automate** or **test manually**, and write one sentence
   justifying each choice.
3. Write a two-line explanation, in your own words and without looking back
   at the table, of the difference between **QA** and **QC**.
4. Take one thing you noticed that is genuinely wrong or confusing in that
   app. Describe it in the *fact-based* style from section 4: what you did,
   what happened, what you expected instead. You will formalize this into a
   real bug report in Module 04.
