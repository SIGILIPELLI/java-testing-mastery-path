# 02 · Test Case Design & Documentation

A test case is the unit of work in manual testing. It is a written,
repeatable instruction set that says: *given this setup, do these steps, and
this exact thing should happen*. Written well, anyone on the team — including
someone who joined yesterday — can execute it and get the same verdict you
would. Written badly, it is a vague note only its author understands.

This module covers the three artefacts you will produce constantly as a
tester: **test scenarios**, **test cases**, and the **Requirement
Traceability Matrix (RTM)**.

## 1. Scenario vs case — the difference

| | Test scenario | Test case |
|---|---|---|
| Level | High-level, one line | Detailed, step-by-step |
| Answers | *What* to test | *How* to test it |
| Example | "Verify login functionality" | "TC-001: Login with valid credentials" |
| Count | One scenario spawns many cases | Many per scenario |
| Written during | Early — after requirement analysis | Later — test case design phase |

Scenarios are written first, as a coverage checklist. Then each scenario is
broken into positive and negative test cases.

**Scenario:** Verify the login functionality of the application.

Breaking it down into cases:

- TC-001 · Login with valid username and valid password *(positive)*
- TC-002 · Login with valid username and invalid password *(negative)*
- TC-003 · Login with invalid username and valid password *(negative)*
- TC-004 · Login with both fields empty *(negative)*
- TC-005 · Login with username containing leading/trailing spaces *(edge)*
- TC-006 · Password field masks the entered characters *(UI)*
- TC-007 · Account locks after 5 consecutive failed attempts *(security)*
- TC-008 · "Remember me" keeps the session after browser restart *(functional)*

Notice the ratio: one scenario, eight cases, and most of them are **negative**.
That is normal and healthy. Beginners write only the happy path; real defects
live in the unhappy ones.

## 2. The test case template

This is the standard column set. Every company tweaks it slightly, but if you
can fill this in, you can adapt to any tool (Jira/Xray, TestRail, Zephyr, or
a plain spreadsheet).

| Field | Purpose |
|---|---|
| **Test Case ID** | Unique, stable identifier — `TC_LOGIN_001` |
| **Module / Feature** | Which area of the app |
| **Test Case Title** | One line describing the objective |
| **Requirement ID** | Links back to the requirement (feeds the RTM) |
| **Priority** | P1/P2/P3 — execution order when time is short |
| **Type** | Positive / Negative / Boundary / UI / Security |
| **Preconditions** | What must be true before step 1 |
| **Test Data** | The exact values to use |
| **Steps** | Numbered, atomic, unambiguous actions |
| **Expected Result** | Precisely what should happen |
| **Actual Result** | Filled in at execution time |
| **Status** | Pass / Fail / Blocked / Not Executed / Skipped |
| **Defect ID** | Bug reference if it failed |
| **Executed By / Date** | Audit trail |

### A fully worked example

| Field | Value |
|---|---|
| **Test Case ID** | TC_LOGIN_002 |
| **Module** | Authentication |
| **Title** | Verify login is rejected with a valid username and an incorrect password |
| **Requirement ID** | REQ-AUTH-03 |
| **Priority** | P1 |
| **Type** | Negative |
| **Preconditions** | 1. Application is deployed and reachable. 2. A user account `qa.user@example.com` exists and is active (not locked). 3. Browser is on the `/login` page. |
| **Test Data** | Username: `qa.user@example.com` · Password: `WrongPass123!` |
| **Steps** | 1. Enter `qa.user@example.com` in the Username field.<br>2. Enter `WrongPass123!` in the Password field.<br>3. Click the **Login** button. |
| **Expected Result** | 1. User remains on `/login`.<br>2. Error message *"Invalid username or password."* is displayed above the form in red.<br>3. The Password field is cleared; the Username field retains its value.<br>4. The failed-attempt counter for the account increments by 1. |
| **Actual Result** | *(filled at execution)* |
| **Status** | Not Executed |
| **Defect ID** | — |

!!! info "Why the expected result has four numbered parts"
    A weak expected result says "login fails." That passes even if the app
    shows a raw stack trace, or silently clears both fields, or fails to
    increment the lockout counter. **Specific expected results catch specific
    defects.** If you cannot say precisely what should happen, the
    requirement is ambiguous — go ask the business analyst before writing the
    case.

## 3. Rules for writing steps that actually work

1. **One action per step.** "Enter credentials and click Login" is two steps.
   If it fails, which half failed?
2. **Use exact UI labels.** "Click the **Submit** button", not "submit the
   form". Quote the label as it appears on screen.
3. **Be data-explicit.** "Enter a valid email" is not reproducible. "Enter
   `qa.user@example.com`" is.
4. **No assumed context.** If the case needs the user logged in, that belongs
   in Preconditions, not implied.
5. **Independent cases.** TC-005 must not require TC-004 to have run first. A
   dependent case becomes *Blocked* the moment its predecessor fails, and
   your suite loses coverage in a cascade.
6. **Atomic scope.** One test case verifies one thing. A case titled "Verify
   login, dashboard and logout" reports a single Pass/Fail for three
   features — useless data.
7. **Write in the imperative present.** "Click", "Enter", "Verify" — not "The
   tester should click".

### Before and after

!!! warning "Poorly written case"
    **Title:** Test login
    **Steps:** Login to the app with correct details and see if it works.
    **Expected:** Should work fine.

!!! info "The same case, rewritten"
    **Title:** TC_LOGIN_001 · Verify successful login with valid credentials
    **Preconditions:** Active account `qa.user@example.com` exists; browser on `/login`.
    **Test Data:** Username `qa.user@example.com`, Password `Valid@Pass1`
    **Steps:**
    1. Enter `qa.user@example.com` in the **Username** field.
    2. Enter `Valid@Pass1` in the **Password** field.
    3. Click the **Login** button.
    **Expected Result:**
    1. Browser navigates to `/dashboard` within 3 seconds.
    2. The header displays "Welcome, QA User".
    3. The **Logout** link is visible in the top-right corner.

## 4. Status values and what they mean

| Status | Use when |
|---|---|
| **Pass** | Every expected result was observed |
| **Fail** | At least one expected result was not observed — raise a defect |
| **Blocked** | Could not execute due to an external problem (environment down, prerequisite defect) |
| **Not Executed** | Not yet run in this cycle |
| **Skipped** | Deliberately excluded — out of scope for this cycle |
| **Deferred** | Feature not ready; will be tested in a later cycle |

The distinction between **Fail** and **Blocked** matters for reporting. A
50-case suite reported as "10 pass, 40 fail" panics management. "10 pass, 2
fail, 38 blocked because the test database was offline" tells the truth.

## 5. The Requirement Traceability Matrix (RTM)

The RTM is a table that maps every requirement to the test cases covering it,
and onward to any defects found. It answers three questions management always
asks:

- **Coverage:** is every requirement tested? (Any requirement row with zero
  test cases is a coverage gap.)
- **Impact:** requirement REQ-AUTH-03 just changed — which test cases must I
  update? (Read across the row.)
- **Quality:** which requirements are producing the most defects?

### Forward, backward, bidirectional

| Type | Direction | Answers |
|---|---|---|
| **Forward** | Requirements → test cases | Is every requirement covered by a test? |
| **Backward** | Test cases → requirements | Is every test justified, or are we testing things nobody asked for? |
| **Bidirectional** | Both | The complete picture — what most teams maintain |

### A sample RTM

| Req ID | Requirement description | Test Case IDs | Cases | Executed | Passed | Failed | Defects | Status |
|---|---|---|---|---|---|---|---|---|
| REQ-AUTH-01 | User can log in with a registered email and password | TC_LOGIN_001 | 1 | 1 | 1 | 0 | — | ✅ Covered |
| REQ-AUTH-02 | Password field must mask input | TC_LOGIN_006 | 1 | 1 | 1 | 0 | — | ✅ Covered |
| REQ-AUTH-03 | Invalid credentials show a generic error message | TC_LOGIN_002, TC_LOGIN_003 | 2 | 2 | 1 | 1 | BUG-114 | ⚠️ Defect open |
| REQ-AUTH-04 | Account locks for 15 min after 5 failed attempts | TC_LOGIN_007 | 1 | 0 | 0 | 0 | — | ⏳ Not executed |
| REQ-AUTH-05 | Session expires after 30 minutes of inactivity | — | 0 | 0 | 0 | 0 | — | ❌ **Coverage gap** |

REQ-AUTH-05 is exactly what an RTM exists to surface: a requirement nobody
wrote a test for. Without this table, it ships untested and nobody notices
until a user complains.

!!! info "Generic error messages are deliberate"
    Note that REQ-AUTH-03 says *"generic"*. If the app says "Password
    incorrect" for a real account and "User not found" for a fake one, an
    attacker can enumerate valid accounts. This is why TC_LOGIN_002 and
    TC_LOGIN_003 must both expect the *identical* message — a security check
    hiding inside a functional test case.

## 6. What goes in a test plan (vs a test case)

Beginners often confuse these. A **test plan** is one document for the whole
project; **test cases** are hundreds of small items.

A minimal test plan contains:

1. **Scope** — what is being tested, and explicitly what is *not*.
2. **Test approach** — manual vs automated, levels, types.
3. **Test environment** — browsers, OS, devices, test data sources.
4. **Entry & exit criteria** — when testing starts, when it is done.
5. **Schedule & effort estimate** — cycles and dates.
6. **Roles & responsibilities** — who executes, who triages, who signs off.
7. **Risks & mitigations** — e.g. "third-party payment sandbox may be
   unstable → mock the gateway for regression runs."
8. **Deliverables** — test cases, RTM, execution report, summary report.

You will write a real one in this level's project (Module 10).

## Cheat sheet

| Artefact | One-line purpose |
|---|---|
| Test scenario | High-level *what* to test |
| Test case | Detailed, repeatable *how* to test it, with an exact expected result |
| Test suite | A grouped set of cases run together |
| Test data | The specific input values a case needs |
| Preconditions | State that must exist before step 1 |
| Test plan | The project-level strategy, scope and schedule document |
| RTM | Requirement ↔ test case ↔ defect mapping; proves coverage |
| Test summary report | End-of-cycle results and recommendation to release |

## Exercise

Use this requirement set for a **user registration form**:

- **REQ-REG-01** — The form has fields: Full Name, Email, Password, Confirm
  Password, and a "I accept the Terms" checkbox.
- **REQ-REG-02** — Email must be unique across all accounts.
- **REQ-REG-03** — Password must be 8–20 characters and contain at least one
  uppercase letter, one digit and one special character.
- **REQ-REG-04** — Confirm Password must match Password.
- **REQ-REG-05** — The **Register** button stays disabled until the Terms
  checkbox is ticked.
- **REQ-REG-06** — On success, the user sees a "Verify your email"
  confirmation screen.

Then:

1. Write **three test scenarios** covering this form.
2. Write **eight full test cases** using the template from section 2 —
   complete with ID, preconditions, test data, numbered steps and specific
   expected results. At least five must be negative or boundary cases.
3. Build an **RTM** with one row per requirement, listing which of your test
   case IDs cover it. Deliberately check whether any requirement ends up with
   zero cases — if so, write the missing case.
4. Take your weakest test case and rewrite it applying all seven rules from
   section 3. Note in one sentence what specifically improved.
