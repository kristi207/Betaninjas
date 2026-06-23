# Day 6 — Assignment 3
## Steps 5 & 6: Risk-Based Coverage + Coverage vs Confidence

---

## Step 5 — Risk-Based Coverage

### All Major Features in BetaninjasSEOLMS

| Feature | Risk Rating | Reasoning |
|---|---|---|
| Login / Authentication | High | If login breaks, no member can access anything. If domain restriction is not enforced, unauthorized users get in. Complete access blocker — everything downstream depends on it. |
| Phase Navigation / Lock Logic | High | This is where Kristi's bug lived. If phase 2 is accessible before phase 1 is complete, the entire learning sequence breaks. Members skip work, progress data becomes meaningless, the curriculum collapses. |
| Task Submission | High | This is the core action of the product. If a member cannot submit a task — or submits one silently without it saving — they lose their work and their progress is corrupted. No recovery path without admin intervention. |
| Progress Tracking | High | If progress calculates incorrectly, members think they are done when they are not, or the dashboard lies about their state. Corrupted progress data is hard to detect and harder to fix retroactively. |
| Dashboard | Medium | If the dashboard loads incorrectly or shows stale data, it is disorienting but not data-destroying. A member can still submit tasks. The bug found (phase card count not updating in real-time) is an inconvenience, not a blocker. |
| Admin View | Medium | If admin view breaks, an admin cannot see member progress. This is disruptive for the team but does not block a member from continuing their work. |
| Phase Content Display | Low | If the phase content renders incorrectly (wrong text, broken layout), members can still work around it. It is a UX problem, not a data problem. |

---

### Risk Matrix

```
HIGH RISK
  ├── Login / Authentication
  ├── Phase Navigation / Lock Logic
  ├── Task Submission
  └── Progress Tracking

MEDIUM RISK
  ├── Dashboard
  └── Admin View

LOW RISK
  └── Phase Content Display
```

---

### Are High Risk Features Getting the Most Coverage?

Looking at test cases from Phase 1 and Phase 2 honestly:

| Feature | Risk | Test Count (approx.) | Coverage Quality |
|---|---|---|---|
| Login / Auth | High | ~10 tests | Partially — domain restriction bug not caught |
| Phase Navigation / Lock | High | 0 direct tests | Not covered — this is where the bug lived |
| Task Submission | High | ~20 tests | Good — most tested area |
| Progress Tracking | High | ~4 tests | Weak — calculation never unit tested |
| Dashboard | Medium | ~6 tests | Reasonable |
| Admin View | Medium | 0 tests | Not covered |
| Phase Content Display | Low | 0 tests | Acceptable for Low risk |

**Honest answer:** The coverage does not match the risk profile. Task Submission is well-covered but it is one of four High risk features. Phase Navigation — the highest risk feature based on what we now know — has zero direct test coverage. Progress Tracking has a thin layer of system tests with no unit verification of the math. The coverage effort was front-loaded on what was easy to test (task submission forms), not on what was most dangerous to get wrong.

---

## Step 6 — Coverage vs Confidence

### Kristi's Bug

The bug: Phase 2 was accessible without completing Phase 1.

The function responsible: `getPhaseStatus` — which determines whether a phase is locked.

---

### Would a Test That Called getPhaseStatus and Passed Have Caught This Bug?

**No — not necessarily.**

Imagine this test existed:

```
Test: getPhaseStatus returns something when called
Input: phase = 2, completedTasks = 0, totalTasks = 5
Expected: function returns without throwing an error
Result: PASS
```

This test achieves function coverage. It achieves line coverage. The function ran. But the test did not verify what the function *returned*. If `getPhaseStatus` returned `"unlocked"` when it should have returned `"locked"`, this test would still pass — because the assertion only checks that the function ran, not that it returned the correct value.

This is the exact shape of Kristi's bug. The phase navigation probably worked — it navigated. The problem was that navigation worked when it should have been blocked. A test that just checks "navigation happens" would pass. A test that checks "navigation is blocked when Phase 1 is incomplete" would fail and catch the bug.

---

### Why Line or Function Coverage Alone Would Not Have Caught This Bug

**Line coverage** would show 100% if every line of `getPhaseStatus` was executed by any test. But if the test only calls it with Phase 1 already complete, the lock branch might never be exercised. Even if it was, the test would only verify that the line ran — not that the locked state prevented access.

**Function coverage** is even weaker. It only requires the function to be called once. If a system test calls `getPhaseStatus` as a side effect of loading the dashboard with a complete Phase 1, function coverage is satisfied — but the bug scenario (Phase 1 incomplete, Phase 2 accessible) was never triggered.

Neither metric answers the real question: *when Phase 1 is not complete, does the system actually prevent Phase 2 from loading?*

---

### What Specific Assertion Would Actually Catch This Bug

The assertion needs to verify the **outcome**, not just the execution.

**Unit test that would catch it:**

```
Test: getPhaseStatus locks Phase 2 when Phase 1 is incomplete
Input: phase = 2, phase1CompletedTasks = 0, phase1TotalTasks = 5
Assert: return value === "locked"
```

**System test that would catch it:**

```
Test: Phase 2 is inaccessible when Phase 1 is not complete
Precondition: User is logged in, Phase 1 has 0 of 5 tasks completed
Action: Navigate to Phase 2 URL directly
Assert: Page shows locked state OR redirects to Phase 1
Assert: Phase 2 task list is NOT visible
Assert: Phase 2 content is NOT accessible
```

The difference is that these assertions verify **what the system prevents**, not what it does. The bug is a missing prevention — so the test must assert the prevention exists.

---

### The Difference Between a Test That Runs Code and a Test That Proves Something

**A test that runs code:**
- Calls the function or navigates to the page
- Checks that no error was thrown
- Checks that something loaded
- Achieves coverage metrics
- Passes whether or not the feature works correctly

**A test that proves something:**
- Sets up a specific, meaningful precondition
- Takes the exact action that should trigger a specific outcome
- Asserts the precise outcome — not just that something happened, but that the *right* thing happened
- Would fail if the feature were broken in the specific way you care about

Kristi's bug would survive a test that proves "Phase 2 loads." It would be caught by a test that proves "Phase 2 does not load when Phase 1 is incomplete."

Coverage tells you what code ran. Confidence comes from assertions that would fail if the product broke in the ways that matter.

A test suite with 100% line coverage and weak assertions is a false safety net — it looks complete on a metrics dashboard but leaves the most important questions unanswered. Real confidence is built by writing tests that are designed to fail when something breaks — not tests that are designed to pass no matter what.
