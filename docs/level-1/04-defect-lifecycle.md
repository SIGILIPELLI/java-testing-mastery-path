# 04 · Defect Lifecycle & Bug Reporting

Finding a bug is half the job. The other half is communicating it so clearly
that a developer who has never seen your screen can reproduce it in under a
minute. A brilliantly-found defect described as "login broken pls fix" wastes
everyone's time and often gets closed as *Cannot Reproduce*.

This module covers the defect lifecycle, severity vs priority, the anatomy of
a bug report that works, and the Jira mechanics you will use daily.

## 1. The defect lifecycle

Every defect moves through a state machine. Names vary slightly between
tools, but the shape is universal.

```
            ┌──────────┐
            │   New    │  ← tester logs it
            └────┬─────┘
                 │ triage
        ┌────────┼────────┬───────────┬──────────┐
        ▼        ▼        ▼           ▼          ▼
   ┌────────┐ ┌──────┐ ┌──────────┐ ┌────────┐ ┌──────────┐
   │Assigned│ │Rejected│ Duplicate │ Deferred │ Not a Bug │
   └───┬────┘ └──────┘ └──────────┘ └────────┘ └──────────┘
       │ developer works on it
       ▼
   ┌────────┐
   │  Open  │ (In Progress)
   └───┬────┘
       ▼
   ┌────────┐
   │ Fixed  │ ← developer commits fix, moves to QA build
   └───┬────┘
       │ tester re-tests
   ┌───┴────┐
   ▼        ▼
┌────────┐ ┌──────────┐
│Verified│ │ Reopened │ → back to Assigned
└───┬────┘ └──────────┘
    ▼
┌────────┐
│ Closed │
└────────┘
```

| Status | Meaning | Who sets it |
|---|---|---|
| **New** | Just logged, not yet reviewed | Tester |
| **Assigned** | Triaged and given to a developer | Lead / triage meeting |
| **Open / In Progress** | Developer is actively working on it | Developer |
| **Fixed / Resolved** | Fix committed and available in a build | Developer |
| **Pending Retest** | Waiting for QA to verify in the new build | Developer / lead |
| **Verified** | QA confirmed the fix works | Tester |
| **Reopened** | Fix did not work, or broke something else | Tester |
| **Closed** | Done — verified and no longer tracked | Tester / lead |
| **Rejected / Not a Bug** | Behaviour is actually correct | Developer / BA |
| **Duplicate** | Already reported as another ID | Triage |
| **Deferred / Postponed** | Real bug, but will be fixed in a later release | Product owner |
| **Cannot Reproduce** | Developer could not make it happen | Developer |

!!! warning "Only the tester who raised a defect should close it"
    A developer marking their own bug *Closed* skips verification. The
    developer's job ends at *Fixed*; QA moves it to *Verified* and *Closed*.
    This single rule prevents a whole category of "fixed" bugs shipping
    broken.

### What to do when a defect comes back as *Cannot Reproduce*

This is nearly always a gap in your report, not a lie from the developer.
Check: did you state the environment (browser, version, OS)? The exact test
data? Whether the account was in a special state? Whether it happens every
time or intermittently? Add a screen recording, add the browser console log,
and reopen with the extra detail.

## 2. Severity vs priority

The single most-asked interview question in QA, and the one most people get
subtly wrong.

| | **Severity** | **Priority** |
|---|---|---|
| Measures | **Technical impact** on the system | **Business urgency** of fixing it |
| Set by | Tester | Product owner / business |
| Question | How badly does this break the product? | How soon must this be fixed? |
| Changes over time? | Rarely | Often — a release date shifts priority |

**Severity scale:**

| Level | Definition | Example |
|---|---|---|
| **Critical / S1** | System unusable; no workaround; data loss | Application crashes on launch; payments deduct money without creating an order |
| **Major / S2** | Key feature broken; workaround exists but painful | Cannot checkout with a saved card, but a new card works |
| **Minor / S3** | Feature works with a small defect | Sort by price ascending puts out-of-stock items last regardless |
| **Trivial / S4** | Cosmetic, no functional impact | Footer copyright reads "2019" |

**Priority scale:**

| Level | Definition |
|---|---|
| **P1 / Urgent** | Fix immediately, block the release |
| **P2 / High** | Fix in this release |
| **P3 / Medium** | Fix in an upcoming release |
| **P4 / Low** | Fix when convenient / backlog |

### The four combinations — worth memorizing

| Combination | Example | Reasoning |
|---|---|---|
| **High severity, High priority** | Checkout crashes for every user | Broken and business-critical — drop everything |
| **High severity, Low priority** | The app crashes on a browser version used by 0.02% of users on a page reached only by an admin | Technically severe, commercially irrelevant right now |
| **Low severity, High priority** | The company name is misspelled on the homepage banner, or the price shows "$" instead of "€" | Trivial code change, enormous brand/legal embarrassment |
| **Low severity, Low priority** | A tooltip's text is misaligned by 2px on an internal settings page | Fix it when there's time |

The "low severity, high priority" example is the one interviewers look for.
If you can produce it unprompted, you understand the distinction.

## 3. Anatomy of a bug report

| Field | What goes in it |
|---|---|
| **Bug ID** | Auto-generated by the tool — `BUG-114`, `PROJ-2381` |
| **Summary / Title** | One line: *what* fails, *where*, and *when* |
| **Description** | Short context — the requirement or expected behaviour |
| **Environment** | Build/version, browser + version, OS, device, test environment (QA/Staging), URL |
| **Preconditions** | Account state, data setup, feature flags |
| **Steps to Reproduce** | Numbered, atomic, exact — with the real data used |
| **Expected Result** | What *should* happen, with the requirement ID if possible |
| **Actual Result** | What *did* happen, verbatim (quote error text exactly) |
| **Severity** | S1–S4 |
| **Priority** | P1–P4 (may be adjusted at triage) |
| **Reproducibility** | Always / Intermittent (e.g. 3 out of 10) / Once |
| **Attachments** | Screenshot with the problem circled, screen recording, browser console log, server log excerpt, HAR file |
| **Assigned To** | Developer or team |
| **Reported By / Date** | You, and when |
| **Linked Test Case** | The TC ID that failed — feeds the RTM |

### Writing the title

The title is what appears in every list, dashboard and standup. Make it carry
the information.

| ❌ Weak | ✅ Strong |
|---|---|
| Login not working | Login fails with HTTP 500 when the password contains an ampersand (&) |
| Bug in cart | Cart total does not update after removing the last item; still shows the removed item's price |
| App slow | Search results page takes 14s to load when the query returns 500+ products |
| Error message | Password reset shows raw SQL exception text instead of a user-friendly error |

The formula: **[What happens] + [where] + [under what condition]**.

## 4. A complete worked bug report

!!! info "BUG-114"
    **Summary:** Login with an incorrect password displays the generic
    message "Error" instead of "Invalid username or password."

    **Description:** Per REQ-AUTH-03, a failed login must display the
    message "Invalid username or password." above the login form. The
    application currently renders only the word "Error" in default black
    text, giving the user no actionable information.

    **Environment:**

    - Build: 4.2.0 (QA environment)
    - URL: `https://qa.example-shop.com/login`
    - Browser: Chrome 126.0.6478.127 (64-bit)
    - OS: Windows 11 Pro 23H2

    **Preconditions:** An active, non-locked account exists with the email
    `qa.user@example.com`.

    **Steps to Reproduce:**

    1. Navigate to `https://qa.example-shop.com/login`.
    2. Enter `qa.user@example.com` in the **Username** field.
    3. Enter `WrongPass123!` in the **Password** field.
    4. Click the **Login** button.

    **Expected Result:** The user remains on `/login` and the message
    *"Invalid username or password."* is displayed above the form in the
    error style (red, 14px), per REQ-AUTH-03.

    **Actual Result:** The user remains on `/login`, and the text
    *"Error"* is displayed above the form in default black body text. The
    browser console logs: `Uncaught TypeError: Cannot read properties of
    undefined (reading 'message') at auth.bundle.js:2201`.

    **Severity:** S3 – Minor (login correctly rejects the attempt; only the
    message is wrong)
    **Priority:** P2 – High (user-facing text on the most-visited page;
    also a UX/support-cost issue)
    **Reproducibility:** Always (10/10 attempts)
    **Attachments:** `bug114-screenshot.png`, `bug114-console.log`
    **Linked Test Case:** TC_LOGIN_002
    **Reported By:** QA Engineer · 2026-07-30

Note what makes this work: the developer gets a URL, a browser version, exact
data, a console stack trace pointing at `auth.bundle.js:2201`, and a
requirement ID justifying the expectation. That bug can be fixed without a
single follow-up question.

## 5. Rules for reproduction steps

1. **Start from a known state** — "Navigate to X", not "from where you were".
2. **Number every step, one action each.**
3. **Use the real data you used**, not "enter a valid email".
4. **Minimize.** Before filing, try to remove steps. If the bug reproduces in
   4 steps instead of 11, delete the other 7. A minimal reproduction is the
   single most valuable thing you can give a developer.
5. **State reproducibility honestly.** Intermittent bugs are real bugs;
   saying "3 out of 10 attempts" tells the developer to look for a race
   condition rather than a logic error.
6. **Quote error text verbatim**, including punctuation and casing. Paraphrasing
   makes the string unsearchable in the codebase.

!!! warning "Never write conclusions you cannot prove"
    "The database is corrupted" or "the API is broken" is diagnosis, not
    observation — and when it is wrong, it sends the developer to the wrong
    team for a day. Report the *symptom*. If you have supporting evidence
    (a 500 response, a stack trace), attach it as evidence rather than
    stating it as cause.

## 6. Jira basics for testers

Most teams track defects in Jira. The mechanics you actually need:

| Concept | What it is |
|---|---|
| **Project** | The container, with a key like `SHOP` — every issue is `SHOP-123` |
| **Issue type** | Bug, Story, Task, Epic, Sub-task. Testers mostly create **Bug** |
| **Workflow** | The configured state machine (the lifecycle in section 1) |
| **Transition** | Moving an issue between statuses via a button |
| **Sprint / Board** | The Scrum or Kanban view of current work |
| **Component** | Which part of the product — Authentication, Cart, Payments |
| **Affects Version / Fix Version** | The build where the bug appears / where it will be fixed |
| **Label** | Free-form tags — `regression`, `security`, `flaky` |
| **Link** | Relationships: *blocks*, *is blocked by*, *duplicates*, *relates to* |
| **Attachment** | Screenshots, recordings, logs |
| **Comment** | The conversation trail — where re-test results go |

### JQL — the queries you will use constantly

Jira Query Language is a filter syntax. A handful of queries covers most
daily needs:

```jql
-- All open bugs assigned to me
project = SHOP AND issuetype = Bug AND assignee = currentUser() AND status != Closed

-- Everything I need to re-test right now
project = SHOP AND issuetype = Bug AND status = "Pending Retest" AND reporter = currentUser()

-- Release blockers
project = SHOP AND priority in (P1, Urgent) AND status not in (Closed, Verified)

-- Bugs I raised in the last week, newest first
project = SHOP AND reporter = currentUser() AND created >= -7d ORDER BY created DESC

-- Open bugs in one component, sorted by severity
project = SHOP AND component = Authentication AND status = Open ORDER BY priority DESC
```

### Zephyr / Xray

Test-management plugins that add Test, Test Execution and Test Cycle issue
types to Jira, so test cases live alongside defects and the RTM builds
itself. If your team uses one, your test cases from Module 02 go here rather
than in a spreadsheet.

## 7. Defect metrics you will be asked about

| Metric | Formula | Tells you |
|---|---|---|
| **Defect density** | Defects ÷ size (KLOC or function points) | Which modules are the most defect-prone |
| **Defect leakage / escape rate** | Defects found in production ÷ total defects × 100 | How effective your testing was |
| **Defect removal efficiency (DRE)** | Defects found in testing ÷ (found in testing + found in production) × 100 | Overall QA effectiveness; aim above 90% |
| **Defect rejection ratio** | Rejected defects ÷ total raised × 100 | Report quality — high means you are raising non-bugs |
| **Reopen rate** | Reopened ÷ total fixed × 100 | Fix quality |
| **Mean time to resolve** | Average (closed date − created date) | Team responsiveness |

## Cheat sheet

| Concept | One-liner |
|---|---|
| Severity | How badly it breaks the product (tester decides) |
| Priority | How soon it must be fixed (business decides) |
| Reproducibility | Always / intermittent — say which |
| Minimal reproduction | Fewest steps that still show the bug — the highest-value thing you can provide |
| Verified vs Closed | Verified = fix confirmed; Closed = tracking finished |
| Reopened | Fix failed re-testing |
| Deferred | Real bug, postponed by the business |
| DRE | % of defects caught before production |

## Exercise

You are testing an online bookstore, build 3.1.0, on Chrome 126 / macOS 14.

You observe: adding a book to the wishlist while logged out shows a "Saved!"
toast, but after logging in the wishlist is empty. It happens every time. The
browser console shows `POST /api/wishlist 401 (Unauthorized)`.

1. Write the **complete bug report** using the template in section 3 — a
   strong title, environment, numbered reproduction steps, expected vs actual
   result, severity, priority and reproducibility. Justify your severity and
   priority ratings in one line each.
2. The developer closes it as **Cannot Reproduce**. Write the comment you
   would add when reopening it — what extra information would you gather
   first?
3. For each of these, assign a severity **and** a priority, with a one-line
   justification:
   - The "Buy Now" button is greyed out for all users on the product page.
   - The company's name is spelled "Amazom" in the browser tab title.
   - A book's page count shows `-1` for one out-of-print title.
   - Checkout fails only on Internet Explorer 11, which 0.01% of users use.
4. Write the **JQL query** that would show you every P1 or P2 bug in the
   `BOOKS` project that is currently waiting for you to re-test.
