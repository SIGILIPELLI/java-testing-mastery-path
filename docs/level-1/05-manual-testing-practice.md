# 05 · Manual Testing in Practice

Exhaustive testing is impossible. A single field accepting 10 alphanumeric
characters has more than 3 × 10^15 possible inputs — you could not test them
all if you started at the Big Bang. So the entire craft of manual testing is
**choosing a small set of inputs with the highest probability of exposing a
defect.**

This module gives you four formal techniques for making that choice —
equivalence partitioning, boundary value analysis, decision tables, and state
transition testing — plus exploratory testing, the disciplined form of "just
poking at it."

## 1. Equivalence Partitioning (EP)

**The idea:** divide the input domain into groups (*partitions*) where every
value in a group should be handled identically by the software. Test one
representative value from each group. If it works for one, it works for all
of them; if it fails for one, it fails for all of them.

### Worked example — an age field

Requirement: *The Age field accepts whole numbers from 18 to 60. Values
outside that range are rejected with "Age must be between 18 and 60."*

| Partition | Range | Type | Representative value |
|---|---|---|---|
| P1 — Below range | 0 to 17 | Invalid | `10` |
| P2 — Valid range | 18 to 60 | **Valid** | `35` |
| P3 — Above range | 61 and above | Invalid | `75` |
| P4 — Negative | −∞ to −1 | Invalid | `-5` |
| P5 — Non-numeric | letters, symbols | Invalid | `abc` |
| P6 — Empty | "" | Invalid | *(blank)* |
| P7 — Decimal | non-integers | Invalid | `25.5` |

**7 test cases instead of infinity.** That is the entire value of the
technique. Note that partitions P4–P7 are the ones beginners forget; the
requirement only mentioned a numeric range, but a real text input can receive
anything a keyboard can produce.

!!! info "One valid partition per field, or one per rule?"
    If different valid inputs trigger *different behaviour*, they are
    different partitions. A shipping-cost field where orders under ₹500 pay
    ₹50 and orders of ₹500+ ship free has **two** valid partitions, not one —
    because the software treats them differently.

### Worked example — a discount tier

Requirement: *Orders under ₹1,000 get no discount; ₹1,000–₹4,999 get 5%;
₹5,000–₹9,999 get 10%; ₹10,000 and above get 15%.*

| Partition | Range | Expected | Representative |
|---|---|---|---|
| P1 | ₹0 – ₹999.99 | 0% discount | ₹500 |
| P2 | ₹1,000 – ₹4,999.99 | 5% discount | ₹2,500 |
| P3 | ₹5,000 – ₹9,999.99 | 10% discount | ₹7,500 |
| P4 | ₹10,000+ | 15% discount | ₹15,000 |
| P5 | negative | Rejected | −₹100 |
| P6 | zero | Rejected or 0% (clarify with BA!) | ₹0 |

P6 is a genuine ambiguity — the requirement does not say. Raising it is
static testing (Module 03) and prevents a defect before code is written.

## 2. Boundary Value Analysis (BVA)

**The idea:** defects cluster at the *edges* of partitions, because that is
where developers write `<` when they meant `<=`. So test the values right at
and around each boundary.

For a boundary at value *B*, test **B−1, B, B+1** (the three-value method) or
just **B and B+1** on each side (the two-value method). The three-value form
is the safer default.

### Worked example — the same age field (18–60)

| Value | Partition side | Expected | Why it's tested |
|---|---|---|---|
| `17` | Just below lower bound | ❌ Rejected | Catches `age <= 18` written as `age < 18` errors |
| `18` | **Lower bound** | ✅ Accepted | The minimum valid value |
| `19` | Just above lower bound | ✅ Accepted | Confirms the range opens correctly |
| `59` | Just below upper bound | ✅ Accepted | Confirms the range is still open |
| `60` | **Upper bound** | ✅ Accepted | The maximum valid value — the classic off-by-one |
| `61` | Just above upper bound | ❌ Rejected | Catches `age <= 60` written as `age < 60` |

Six boundary tests. Combine them with the EP representatives above and you
have a thorough, defensible 13-case set for one field.

!!! warning "The off-by-one defect"
    If a developer writes `if (age > 18)` instead of `if (age >= 18)`, the
    application rejects an 18-year-old. A tester who only tried `35` would
    never find it. **The value `60` is far more likely to expose a bug than
    the value `35`.** This is why BVA is the highest-yield technique per test
    case in all of manual testing.

### Boundaries hide in more than numbers

| Field type | Boundaries to test |
|---|---|
| Text length (3–20 chars) | 2, 3, 4, 19, 20, 21 characters |
| Date range | First and last day of the range; also 31 Jan, 28/29 Feb (leap year), 31 Dec |
| File upload (max 5 MB) | 4.9 MB, exactly 5 MB, 5.1 MB, 0-byte file |
| Pagination (20 per page) | 0, 1, 19, 20, 21, 40, 41 records |
| Array/list | Empty list, one item, maximum items, maximum + 1 |
| Currency | 0.00, 0.01, the smallest and largest allowed amounts |
| Time | 23:59, 00:00, DST changeover, timezone edges |

### Combining EP and BVA — a password field

Requirement: *Password must be 8–20 characters, contain at least one
uppercase letter, one digit and one special character.*

| # | Test data | Technique | Expected |
|---|---|---|---|
| 1 | `Pass1@ab` (8 chars) | BVA — lower bound | ✅ Accepted |
| 2 | `Pas1@ab` (7 chars) | BVA — below lower | ❌ "Minimum 8 characters" |
| 3 | `Pass1@abc` (9 chars) | BVA — above lower | ✅ Accepted |
| 4 | 20-char valid password | BVA — upper bound | ✅ Accepted |
| 5 | 21-char valid password | BVA — above upper | ❌ "Maximum 20 characters" |
| 6 | `password1@` | EP — no uppercase | ❌ "Must contain an uppercase letter" |
| 7 | `Password@x` | EP — no digit | ❌ "Must contain a digit" |
| 8 | `Password12` | EP — no special char | ❌ "Must contain a special character" |
| 9 | `PASSWORD1@` | EP — no lowercase | ✅ Accepted (not required by the spec!) |
| 10 | *(empty)* | EP — empty | ❌ "Password is required" |
| 11 | `Pass 1@ab` (with a space) | EP — whitespace | ❓ Undefined — ask the BA |

Rows 9 and 11 are the interesting ones. Row 9 passes because the requirement
never demanded a lowercase letter — testers must test the requirement as
written, then *question* it separately. Row 11 exposes an undocumented case.

## 3. Decision tables

When output depends on a **combination** of conditions, EP and BVA are not
enough — you need to cover the combinations systematically. A decision table
lays out every combination of conditions and the action for each.

### Worked example — an e-commerce discount

Rules: *Premium members get 10% off. Orders over ₹5,000 get free shipping.
Anyone using a valid coupon gets an extra 5%. Premium members always get free
shipping regardless of order value.*

Three conditions → 2³ = 8 rules.

| Condition | R1 | R2 | R3 | R4 | R5 | R6 | R7 | R8 |
|---|---|---|---|---|---|---|---|---|
| Premium member? | Y | Y | Y | Y | N | N | N | N |
| Order > ₹5,000? | Y | Y | N | N | Y | Y | N | N |
| Valid coupon? | Y | N | Y | N | Y | N | Y | N |
| **Action** | | | | | | | | |
| Member discount 10% | ✔ | ✔ | ✔ | ✔ | — | — | — | — |
| Coupon discount 5% | ✔ | — | ✔ | — | ✔ | — | ✔ | — |
| Free shipping | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | — | — |
| **Total discount** | 15% | 10% | 15% | 10% | 5% | 0% | 5% | 0% |
| **Shipping** | Free | Free | Free | Free | Free | Free | Paid | Paid |

Each column is one test case — eight cases that cover every combination. Read
across R3: a premium member with a small order and a coupon still gets free
shipping. That is the rule most likely to be implemented wrong, and the table
made it visible.

!!! info "Collapsing a decision table"
    With many conditions the table explodes (5 conditions = 32 rules). You
    can collapse columns where a condition is *irrelevant* to the outcome,
    marking it `-` (don't care). In the table above, for premium members the
    "order > ₹5,000" condition is irrelevant to shipping, so R1/R3 and R2/R4
    can be merged for the shipping action alone. Collapse carefully —
    over-collapsing loses real coverage.

## 4. State transition testing

Use this when the system behaves differently depending on what happened
before — logins, order status, subscriptions, media players.

### Worked example — login with account lockout

States: `Logged Out` → `Logged In`, with a failed-attempt counter and a
`Locked` state after 3 failures.

| Current state | Event | Next state | Notes |
|---|---|---|---|
| Logged Out (0 fails) | Correct credentials | Logged In | Counter resets to 0 |
| Logged Out (0 fails) | Wrong credentials | Logged Out (1 fail) | |
| Logged Out (1 fail) | Wrong credentials | Logged Out (2 fails) | |
| Logged Out (2 fails) | Wrong credentials | **Locked** | Lockout email sent |
| Logged Out (2 fails) | **Correct** credentials | Logged In | Counter must reset — commonly buggy |
| Locked | Correct credentials | **Locked** | Must still refuse |
| Locked | 15 minutes elapse | Logged Out (0 fails) | Auto-unlock |
| Logged In | Click Logout | Logged Out (0 fails) | |
| Logged In | 30 min inactivity | Logged Out | Session timeout |

The two rows to notice: *correct credentials after 2 failures* (does the
counter reset?) and *correct credentials while locked* (does lockout actually
hold?). Both are classic production bugs that only a state-based approach
surfaces.

**Invalid transitions matter too.** Test that you *cannot* reach the checkout
confirmation page by pasting its URL without going through payment. Skipping
states is a whole class of security defect.

## 5. Exploratory testing

Everything above is **scripted** — you decide the checks in advance.
Exploratory testing is *simultaneous* learning, test design and execution:
you use what you just discovered to decide what to try next. It is not
random clicking; it is structured investigation.

### Session-based test management

The professional form gives exploratory work a container:

1. **Charter** — a one-line mission. *"Explore the checkout flow with
   multiple payment methods to discover defects in order total calculation."*
2. **Time box** — typically 60 or 90 minutes, uninterrupted.
3. **Notes** — record every action, observation, question and bug as you go.
4. **Debrief** — report what you covered, what you found, what you could not
   reach, and what should become a scripted test case.

### Heuristics for what to try

| Heuristic | Ask |
|---|---|
| **Goldilocks** | Too big, too small, just right — for every input |
| **CRUD** | Can I create, read, update and delete this? What if I delete something in use? |
| **Interruption** | What if I close the tab, hit Back, refresh, or lose network mid-flow? |
| **Double-click / double-submit** | Does clicking Pay twice charge twice? |
| **Boundaries of time** | Midnight, month end, leap day, DST switch, expired session |
| **Zero, one, many** | Empty cart, one item, 500 items |
| **Injection-flavoured input** | `<script>alert(1)</script>`, `' OR 1=1 --`, `%00`, emoji, right-to-left text |
| **Copy-paste** | Paste formatted text, paste 50,000 characters, paste with trailing whitespace |
| **Permissions** | Log in as a low-privilege user and hit an admin URL directly |
| **Concurrency** | Two tabs, same account, conflicting actions |

### A quick note on "test data with teeth"

Keep a personal file of nasty-but-legal inputs and reuse it on every text
field you meet:

```
O'Brien                      (apostrophe — breaks naive SQL and string escaping)
Zoë Müller-Ærø               (accents, umlauts, ligatures — encoding issues)
你好世界                        (non-Latin — font and byte-length issues)
🙂🎉                          (emoji — 4-byte UTF-8, breaks utf8 vs utf8mb4 columns)
   leading and trailing      (whitespace — is it trimmed?)
aaaa…(5,000 chars)           (length — is there a server-side limit?)
<script>alert(1)</script>    (XSS — is output escaped?)
' OR '1'='1                  (SQL injection shape)
../../etc/passwd             (path traversal shape)
-1, 0, 0.1, 1e10, NaN        (numeric edges)
```

!!! warning "Ethics and scope"
    Only run injection-flavoured inputs against systems you are authorized to
    test, on non-production environments. Testing your employer's QA
    environment as part of your job is fine; probing a third party's live
    site is not. Level 4, Module 04 covers security testing properly.

## 6. Choosing a technique

| Situation | Technique |
|---|---|
| A single input field with a range or format | Equivalence partitioning + BVA |
| Any numeric, length or date limit | Boundary value analysis |
| Output depends on several conditions combined | Decision table |
| Behaviour depends on previous events / status | State transition testing |
| New feature, thin requirements, need to learn fast | Exploratory (session-based) |
| Rules stated as complex business logic | Decision table, then EP on each input |
| Long form with many fields | EP + BVA per field, then a decision table for cross-field rules |

## Cheat sheet

| Technique | Core idea | Test count |
|---|---|---|
| **EP** | One value per behaviour group | 1 per partition |
| **BVA** | Test at the edges: B−1, B, B+1 | 3 per boundary |
| **Decision table** | Every combination of conditions | 2ⁿ (collapsible) |
| **State transition** | Every valid and invalid state change | 1 per transition |
| **Exploratory** | Learn and test simultaneously, time-boxed with a charter | Session-based |

## Exercise

A hotel booking form has these rules:

- **Check-in date** must be today or later; **check-out** must be after
  check-in; a stay may be 1 to 28 nights.
- **Number of guests** is 1 to 6 per room.
- **Promo code** is optional; a valid code gives 12% off.
- Guests who are **loyalty members** get free breakfast. Bookings of **7+
  nights** also get free breakfast. Bookings of 7+ nights by loyalty members
  additionally get a free room upgrade.

Produce:

1. An **equivalence partition table** for *Number of guests*, including all
   invalid partitions a text input could receive.
2. A **BVA table** for *Number of nights* (1–28), listing every value you
   would test and the expected outcome.
3. A **decision table** for the breakfast/upgrade rules, with columns for
   every combination of `loyalty member` and `7+ nights`, plus a `valid promo
   code` condition. State the expected actions for each rule.
4. A **state transition table** for a booking's lifecycle: `Draft →
   Confirmed → Checked In → Checked Out`, plus `Cancelled`. Include at least
   two **invalid** transitions you would test (e.g. cancelling a booking
   that is already checked out).
5. A **60-minute exploratory charter** for the date-selection part of this
   form, plus five specific things you would try, drawn from the heuristics
   table in section 5.
