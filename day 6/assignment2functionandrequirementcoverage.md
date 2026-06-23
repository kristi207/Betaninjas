# Day 6 — Assignment 2
## Steps 3 & 4: Function Coverage + Requirement Coverage

---

## Step 3 — Function Coverage

### All Utility Functions Found in lib/

From reviewing Phase 1 and Phase 2 documentation, the following functions are referenced in BetaninjasSEOLMS:

| Function | File (likely location) | Description |
|---|---|---|
| `getDaysRemaining(dueDate, now?)` | `lib/utils/timeline.ts` | Calculates days between now and a due date |
| `getPhaseStatus(phase, completedTasks, totalTasks)` | `lib/utils/timeline.ts` | Returns phase lock/unlock/complete status |
| `isValidUrl(url)` | `lib/utils/validation.ts` (inferred) | Validates whether a LINK task URL is well-formed |
| `isValidEmail(email)` | `lib/utils/validation.ts` (inferred) | Validates email format and domain |
| `calculateProgress(completedTasks, totalTasks)` | `lib/utils/progress.ts` (inferred) | Computes progress percentage for dashboard |
| `canSubmitTask(taskType, value)` | `lib/utils/tasks.ts` (inferred) | Guards submission — checks if value is non-empty / URL valid |
| `getTaskState(task)` | `lib/utils/tasks.ts` (inferred) | Returns Not Started / In Progress / Submitted / Completed |

Note: The source code is not present in this repository. The functions above are inferred from test documentation, bug reports, and the behavior described across Phase 1 and Phase 2 assignments. `getDaysRemaining` and `getPhaseStatus` are the only two explicitly named in source-level analysis.

---

### Which Functions Were Tested in Phase 1 and Phase 2?

| Function | Tested? | Test Cases |
|---|---|---|
| `getDaysRemaining` | Yes | TC-WB-01, TC-WB-02, TC-WB-03 (Phase 1 white box) |
| `getPhaseStatus` | No | Referenced but never directly tested |
| `isValidUrl` | Indirectly | EP-03, EP-04, TC-05 — tested via UI, not the function itself |
| `isValidEmail` | Indirectly | TC-BB-01 through TC-BB-05, EP-01, EP-02 — tested via login UI |
| `calculateProgress` | Indirectly | TC-S1-01, TC-S1-02 — tested through dashboard observation |
| `canSubmitTask` | Indirectly | TC-04, DT-01 through DT-05 — submission guard tested via UI |
| `getTaskState` | Indirectly | ST-01 through ST-08 — state machine tested through UI interactions |

---

### Function Coverage Report

**Functions with direct unit test coverage: 1 out of 7**

| Function | Coverage Status | Gap |
|---|---|---|
| `getDaysRemaining` | Covered (3 unit tests) | Missing: what happens when diff is exactly 0 |
| `getPhaseStatus` | Not Covered | This is the function at the center of Kristi's bug — zero direct tests |
| `isValidUrl` | Not Covered (indirect only) | No unit test isolates the validation logic from the UI |
| `isValidEmail` | Not Covered (indirect only) | Domain restriction bug found precisely because this logic was never unit tested |
| `calculateProgress` | Not Covered (indirect only) | No test verifies the math directly — only the dashboard display |
| `canSubmitTask` | Not Covered (indirect only) | Submission guard tested through UI, not in isolation |
| `getTaskState` | Not Covered (indirect only) | State machine logic never tested at the function level |

**The function coverage gap is significant.** 6 out of 7 utility functions have zero direct unit tests. All testing has been done through the UI, which means:
- A bug in `getPhaseStatus` logic would only be caught if a system test happens to trigger that specific path
- A bug in `isValidEmail`'s domain check was found manually (Kristi's bug) — not by a test
- The calculation behind progress percentage has never been verified independently of the UI rendering it

---

## Step 4 — Requirement Coverage

### Requirements from the Task Submission Feature Spec (Phase 1)

The following requirements are extracted from the task submission spec shared in Phase 1:

| # | Requirement / User Story |
|---|---|
| R-01 | A member can submit a CHECKBOX task by checking it — no text input required |
| R-02 | A member can submit a TEXT task by typing content and clicking submit |
| R-03 | A member can submit a LINK task by entering a valid URL and clicking submit |
| R-04 | An empty TEXT field must be blocked — submission should not proceed |
| R-05 | An invalid or missing URL must be blocked for LINK tasks |
| R-06 | After submission, the task's progress count on the dashboard must update |
| R-07 | A submitted TEXT or LINK task can be revised — it transitions back to In Progress |
| R-08 | A CHECKBOX task has only two states: completed and not completed — no revision flow |
| R-09 | Phase 2 must not be accessible until Phase 1 is fully completed |
| R-10 | Login must only accept @betaninjas.com email addresses |
| R-11 | A member without a valid session cannot access the dashboard |
| R-12 | Phase card task count on dashboard must reflect real-time task completion |

---

### Requirement Coverage Map

| # | Requirement | Test Cases | Coverage Status |
|---|---|---|---|
| R-01 | CHECKBOX task submission | TC-02, DT-01, ST-01, ST-02 | Covered |
| R-02 | TEXT task submission | TC-01, TC-03, DT-02, ST-04, ST-05 | Covered |
| R-03 | LINK task submission | EP-03, EP-04, DT-03, ST-07, ST-08 | Covered |
| R-04 | Empty TEXT blocked | TC-04, DT-04, BVA-05 | Covered |
| R-05 | Invalid URL blocked | TC-05, DT-05, EP-04 | Covered |
| R-06 | Dashboard progress updates after submission | TC-S1-01, TC-S1-02 | Partially Covered |
| R-07 | TEXT/LINK can be revised | TC-03, ST-06 | Covered |
| R-08 | CHECKBOX has no revision state | ST-03 | Partially Covered |
| R-09 | Phase 2 locked until Phase 1 complete | None | Not Covered |
| R-10 | Login restricted to @betaninjas.com | TC-BB-01, TC-BB-02, EP-01, EP-02 | Partially Covered |
| R-11 | No dashboard access without session | TC-S2-03 | Covered |
| R-12 | Phase card count updates in real-time | TC-S1-03 | Not Covered |

---

### Coverage Summary

| Status | Count | Requirements |
|---|---|---|
| Covered | 7 | R-01, R-02, R-03, R-04, R-05, R-07, R-11 |
| Partially Covered | 3 | R-06, R-08, R-10 |
| Not Covered | 2 | R-09, R-12 |

---

### Not Covered Requirements — The Gap

**R-09: Phase 2 must not be accessible until Phase 1 is fully completed**

This is Kristi's bug. No test was ever written to verify the phase lock/unlock logic. The requirement existed. The feature was built. But no test mapped to it. A test for this requirement would call `getPhaseStatus` directly with 0% Phase 1 completion and assert that Phase 2 is locked — or navigate to Phase 2 in the UI and assert that access is blocked.

**R-12: Phase card task count must reflect real-time task completion**

This is the second bug found in Day 5. TC-S1-03 was written to check the count — but the bug reveals the count only updates after a page refresh. The test as written does not verify real-time behavior. The requirement is not covered because no test asserts the count updates *without* a reload.

---

### Partially Covered Requirements — The Gaps

**R-06: Progress updates after submission**
TC-S1-01 and TC-S1-02 verify that the progress bar visually changes. But no test verifies the underlying percentage calculation is mathematically correct. If `calculateProgress` returns 49 when it should return 50, the visual might look fine but the logic is wrong.

**R-08: CHECKBOX has no revision state**
ST-03 tests that CHECKBOX skips In Progress and goes directly to Completed. But no test verifies that after checking a CHECKBOX, there is no "revise" button or option — the absence of a feature needs its own explicit assertion.

**R-10: Login restricted to @betaninjas.com**
Test cases exist, but the bug Kristi found is that this restriction is *not enforced*. The tests TC-BB-02 and EP-02 were written with an expected result of "login rejected" — but the actual behavior was "login accepted." These tests exist but the requirement is only partially covered because the system does not enforce it. The tests reveal a gap — but the gap is in the product, not just the tests.
