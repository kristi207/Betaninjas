# Day 7 — Assignment 2
## Coverage Strategy and Mobile Thinking — BetaninjasSEOLMS

---

## Step 1 — Coverage Strategy

The risk matrix from Day 1 placed features into three tiers. This strategy defines what coverage looks like for each tier — not as a percentage, but as a deliberate set of decisions about what gets tested, how deep, and what would never be skipped.

---

### High Risk Features — Deep Coverage

#### Login / Authentication

**What deep coverage looks like:**

The domain restriction logic (`isValidEmail`) must have direct unit tests — not just UI-level system tests. The bugs found so far (unauthorized access allowed) happened precisely because the validation function was never tested in isolation. Deep coverage here means testing the function itself, not just the login form.

**Scenarios that must always be tested:**
- Valid `@betaninjas.com` email + correct password → login succeeds, session created, auth token present
- Valid `@betaninjas.com` email + wrong password → login fails, no session, no token
- Email with valid format but wrong domain (e.g. `@gmail.com`) → login rejected at the domain check
- Email with no `@` symbol → login rejected at format check
- Empty email field → submission blocked before any network call
- Empty password field → submission blocked before any network call
- Both fields empty → submission blocked
- Session cookie absent → any protected page redirects to login
- Session cookie tampered or expired → treated as unauthenticated

**What would never be skipped:**
- The domain restriction test (`@gmail.com` must be rejected). This is the bug that was already found once. Skipping it means the regression lives indefinitely.
- The session guard test (no dashboard access without login). This is a security control, not a convenience — skipping it is never acceptable.

---

#### Phase Navigation / Lock Logic

**What deep coverage looks like:**

`getPhaseStatus` must have direct unit tests for every combination of completion state that affects locking. This is the function where Kristi's bug lived. The system tests never caught it because the test was written after the fact with Phase 1 already complete. Deep coverage means testing the guard conditions at the unit level before anyone relies on them.

**Scenarios that must always be tested:**
- Phase 1 at 0% complete → Phase 2 locked (unit: `getPhaseStatus` returns `"locked"`)
- Phase 1 at 99% complete (one task remaining) → Phase 2 locked
- Phase 1 at 100% complete → Phase 2 unlocked
- Phase 2 at 100% complete → Phase 3 locked (same logic applied to the next boundary)
- Direct URL navigation to Phase 2 when Phase 1 is incomplete → page blocks access, does not render Phase 2 content
- Direct URL navigation to Phase 2 when Phase 1 is complete → access granted, Phase 2 renders correctly

**What would never be skipped:**
- The locked-at-one-task-remaining scenario. Off-by-one errors in completion checks are the most common way a phase lock breaks. If the check is `>= totalTasks` instead of `=== totalTasks`, the lock breaks one task early. This boundary test catches it.
- The direct URL navigation test. If the lock only exists in the navigation menu but not at the page level, a member can bypass it by typing the URL directly.

---

#### Task Submission

**What deep coverage looks like:**

Task submission is the most-tested feature in Phase 1 and Phase 2 — but most tests went through the UI. Deep coverage adds unit-level tests for the submission guard (`canSubmitTask`) and the state machine (`getTaskState`) to verify the logic independently of the interface.

**Scenarios that must always be tested:**

*CHECKBOX:*
- Check → task transitions to Completed, progress count increments
- Uncheck → task transitions back to Not Started, progress count decrements
- No intermediate In Progress state exists

*TEXT:*
- Type content → task moves to In Progress
- Submit non-empty TEXT → task transitions to Submitted, progress increments
- Submit empty TEXT → submission blocked, no state change, no progress change
- Revise a Submitted TEXT task → returns to In Progress (regression risk — revise must not duplicate the submission)

*LINK:*
- Enter valid `https://` URL → task moves to In Progress
- Submit valid URL → task transitions to Submitted, progress increments
- Submit invalid URL (no scheme, bare domain) → submission blocked
- Revise a Submitted LINK task → returns to In Progress

*Submission guard (`canSubmitTask`) unit tests:*
- `canSubmitTask("TEXT", "")` → false
- `canSubmitTask("TEXT", "any content")` → true
- `canSubmitTask("LINK", "https://valid.com")` → true
- `canSubmitTask("LINK", "not-a-url")` → false
- `canSubmitTask("CHECKBOX", null)` → true (checkbox needs no value)

**What would never be skipped:**
- Empty TEXT submission blocked. This is R-04 — a hard requirement and a guard against data corruption.
- Invalid URL blocked. A LINK task submitted with no scheme (`example.com`) is not a valid URL and should never reach the server.
- The revision regression test. If revision creates a second submission record instead of updating the existing one, progress counts become corrupted — and it is invisible from the UI.

---

#### Progress Tracking

**What deep coverage looks like:**

The dashboard shows progress visually, but `calculateProgress` must be verified at the unit level. The math has never been directly tested. Four system tests observe the visual output — but none assert the number itself.

**Scenarios that must always be tested:**
- `calculateProgress(0, 5)` → 0%
- `calculateProgress(1, 5)` → 20%
- `calculateProgress(5, 5)` → 100%
- `calculateProgress(3, 5)` → 60% (not 59% from rounding, not 61%)
- Progress percentage updates on the dashboard immediately after task submission (no reload required)
- Progress percentage decrements immediately after un-checking a CHECKBOX (no reload required)
- Dashboard shows correct progress independently for each phase card

**What would never be skipped:**
- The real-time update test (no page reload). This is exactly the bug found in Day 5 — the count only updated after refresh. Skipping this test allows that regression to silently return.
- The rounding test (`calculateProgress(3, 5)` = 60%). Rounding errors compound when multiple phase progress values are displayed side by side.

---

### Medium Risk Features — Reasonable Coverage

#### Dashboard

**What reasonable coverage looks like:**

The dashboard aggregates data from multiple sources (task completion, phase state, progress). Coverage should verify that the dashboard correctly reads and displays that data — but does not need to re-test the underlying data logic (which belongs at the unit and integration level for the High risk features).

**Minimum that gives confidence:**
- Dashboard loads without errors when authenticated
- Dashboard is inaccessible without a valid session (redirect to login)
- Each phase card displays the correct task completion count for that phase
- Phase cards link to the correct phase (clicking Phase 2 card opens Phase 2, not Phase 1)
- Progress bar reflects the percentage from `calculateProgress` accurately
- Dashboard updates phase card counts after task submission without a page reload

**What is the minimum:**
A single smoke test confirming the dashboard loads and a test confirming it is blocked for unauthenticated users. Everything else can be inferred from the underlying unit and integration tests — but real-time update verification must stay because it has already failed once.

---

#### Admin View

**What reasonable coverage looks like:**

Admin view shows member progress. It does not create or mutate task data — it reads it. Coverage should verify that the view renders correctly and shows accurate data, but deep scenario testing is not necessary.

**Minimum that gives confidence:**
- Admin can log in and reach the admin view
- Admin view lists all members with their current phase and progress
- Member progress shown in admin view matches the member's actual completion state
- A non-admin user cannot access admin view (access control)

**What is the minimum:**
The access control test (non-admin cannot reach admin view) must exist — it is a security control. The data accuracy test (one member's progress matches reality) should exist. The rest is lower priority than any High risk gap.

---

### Low Risk Features — Light Coverage

#### Phase Content Display

**What light coverage looks like:**

Phase content is static or near-static — the text, descriptions, and media that appear when a member opens a phase. If it renders incorrectly, it is a UX problem. It does not corrupt data or block workflows.

**Absolute minimum:**
- Each phase loads without a JavaScript error
- At least one task within each phase is visible and interactive (smoke test that the content loaded)

No deeper scenario coverage is warranted. If a heading is misaligned or a paragraph has wrong formatting, that is caught by visual review, not automated tests.

---

## Step 2 — Mobile Coverage Thinking

BetaninjasSEOLMS is a web app. The following matrix imagines it as a mobile app used by the Betaninjas team on their phones.

---

### Device Coverage Matrix

| Priority | Device | OS | Reasoning |
|---|---|---|---|
| Must Test | iPhone 15 / 16 | iOS 17–18 | Most likely primary device for a tech team in 2025. Safari on iOS is the highest-risk browser — it has historically diverged from Chrome in form handling, input behavior, and URL scheme handling. |
| Must Test | Samsung Galaxy S23/S24 | Android 14 | Most common Android flagship. Chrome on Android is the reference implementation — if it breaks here, it breaks everywhere on Android. |
| Must Test | Google Pixel 8 | Android 14 | Pure Android reference device. No manufacturer skin to interfere. Good baseline for detecting Chrome-specific issues on stock Android. |
| Should Test | iPhone 13 | iOS 16 | Some team members may be on older hardware. iOS 16 had notable differences in keyboard behavior and fixed-position element handling. |
| Should Test | Samsung Galaxy A54 | Android 13 | Mid-range Android. Screens smaller and less powerful than flagships. Exposes layout issues that do not appear on high-resolution flagship screens. |
| Skip | Older devices (iOS < 15, Android < 12) | — | Betaninjas is a tech team. Pre-2021 devices are not representative of the actual user base. Testing cost exceeds the risk of those users existing. |

**Why not test every device:**
Device coverage is about representative samples, not exhaustive enumeration. Testing Safari on iOS covers all iOS devices — Apple forces all browsers on iOS to use WebKit. Testing Chrome on Android covers the dominant Android rendering engine. One flagship and one mid-range per major OS captures the real variation.

---

### Network Coverage Matrix

| Condition | Speed | Latency | Priority |
|---|---|---|---|
| Strong WiFi | 50+ Mbps | <10ms | Must Test — baseline, should always work |
| 4G LTE | 10–25 Mbps | 30–50ms | Must Test — likely the most common real-world condition |
| 3G (slow) | 1–3 Mbps | 100–300ms | Must Test — exposes timeout issues, loading states, and race conditions |
| 2G / Edge | <100 Kbps | 500ms+ | Should Test — reveals whether the app degrades gracefully or breaks |
| Offline | 0 Mbps | N/A | Should Test — what happens when connection drops mid-submission |
| Intermittent | Drops every 5–10s | Variable | Nice to Have — simulates real subway/elevator conditions |

---

### Network Condition × Feature Risk Matrix

| Feature | Strong WiFi | 4G LTE | 3G Slow | 2G / Edge | Offline |
|---|---|---|---|---|---|
| Login / Auth | Low risk | Low risk | **Medium** — login request may timeout silently | **High** — token request may fail with no feedback to user | **High** — app must show clear "no connection" message, not a silent hang |
| Task Submission | Low risk | Low risk | **High** — submit request may reach the server but response times out; member thinks it failed but it saved. Duplicate submission on retry. | **High** — same race condition at worse scale | **High** — most dangerous. Member submits, loses connection, thinks it saved. Progress shows wrong state. |
| Progress Tracking | Low risk | Low risk | **Medium** — progress update may lag; stale data displayed | **High** — progress may not update; member confused about actual state | **High** — cached vs actual progress state conflict |
| Phase Navigation | Low risk | Low risk | **Medium** — phase lock check may use cached state; stale lock state | **Medium** — same risk amplified | **Medium** — if phase state is cached client-side, offline access might bypass the lock |
| Dashboard | Low risk | Low risk | **Medium** — dashboard may load with partial data | **High** — dashboard may fail to load at all, or show empty state with no explanation | **High** — must show meaningful error or last-known state, not a blank screen |
| Phase Content | Low risk | Low risk | **Low** — content loads slowly but eventually | **Medium** — content may time out; member sees blank phase | **High** — static content should ideally be available offline (cache); if not, clear error needed |

---

### Key Risk Findings from Network Analysis

**Task Submission on 3G and offline is the highest combined risk.**

The specific scenario: a member fills out a TEXT task, hits submit on a 3G connection, the HTTP request leaves the client but the response times out. The app shows an error. The member submits again. Now there are two submission records — or worse, the second submission overwrites the first with identical content and the member never knows the first one went through. This is a data integrity risk that only surfaces under slow network conditions.

**Login on offline is the second highest risk.**

If the auth token is not cached client-side, an offline member cannot log in even to view their existing tasks. Whether this is acceptable depends on product requirements — but it must be a deliberate decision, not an accident.

**Phase lock on offline requires explicit handling.**

If the phase lock check makes a server call to determine lock state, going offline could make all phases appear unlocked (request fails open) or all phases appear locked (request fails closed). Failing open is the riskier outcome — it defeats the curriculum sequencing entirely.

---

## Step 3 — The Gap Between Coverage and Confidence

### Not Covered Requirements — What Could Go Wrong

| Requirement | What could go wrong if never tested |
|---|---|
| R-09: Phase 2 locked until Phase 1 complete | A member completes Phase 1 tasks out of order, or a bug in `getPhaseStatus` silently allows access to Phase 2 before they are ready — the entire curriculum sequencing breaks with no visible error, and the team never knows unless someone manually checks. |
| R-12: Phase card count updates in real-time | A member completes a task, looks at their dashboard, sees the old count, and does not know whether their submission was saved — they may resubmit, causing duplicate data, or lose trust in the platform and manually track progress elsewhere. |

---

### Covered Requirement — Honest Evaluation

**Chosen requirement: R-01 — A member can submit a CHECKBOX task by checking it**

**Test cases mapped to it:** TC-02, DT-01, ST-01, ST-02

**Honest question: is this proving the requirement works, or just running through the steps?**

TC-02 steps through the CHECKBOX toggle interaction — check it, observe that the state changes on screen, uncheck it, observe it returns. It achieves line coverage of the state transition. But what does it actually assert?

If TC-02 asserts: `checkbox is checked → task state shows Completed on the UI` — that is coverage. The step ran.

If TC-02 asserts: `checkbox is checked → progress count increments by 1 → task record in DB shows status: "completed"` — that is confidence. The outcome is verified at the data level, not just the display level.

Looking at how TC-02 was written, it verifies visual state change on the UI. It does not verify that the backend recorded the submission, that the progress count changed, or that the CHECKBOX task appears as complete on the dashboard after a page reload.

This means TC-02 is **coverage, not confidence**. It tells us the checkbox interacts as expected on-screen. It does not prove the submission persisted.

---

### The Test Case That Gives the Most Confidence

**TC-BB-02 — Login with a non-@betaninjas.com email is rejected.**

This test gives the most confidence because:
1. It tests a specific security control with a precise assertion — not "something happens" but "access is denied."
2. The precondition is meaningful: it uses a syntactically valid email that passes format checks, but the domain is wrong. This distinguishes domain rejection from format rejection.
3. It would fail if the restriction were removed or broken — there is no way for this test to pass if the bug Kristi found is reintroduced.
4. The assertion is the outcome the requirement demands: the user does not get in. A test that just checks the error message text is weaker — the message could change. A test that asserts the session was not created is stronger.

This is what a confident test looks like: it is designed to fail in exactly the way the requirement would break.

---

## Step 4 — What Good Coverage Looks Like for BetaninjasSEOLMS

### Current Coverage State

From Phase 1 and Phase 2 work:

**Well covered:**
- Task submission flows (CHECKBOX, TEXT, LINK) — tested at multiple levels (unit state machine, integration submission guard, system UI)
- Login form behavior — domain and format validation tested via system tests
- Session guard — dashboard inaccessible without login is verified
- Dashboard loading — smoke test exists

**Partially covered:**
- Progress tracking — visual changes tested but underlying math never verified at unit level
- CHECKBOX-specific revision guard — ST-03 tests the absence of In Progress state but does not assert the absence of a revise option on a completed CHECKBOX
- Login domain restriction — tests exist but the bug was active, meaning the tests were passing incorrectly or the expectation was wrong

**Not covered:**
- Phase lock logic — zero direct tests for `getPhaseStatus`; Kristi's bug lived here and would survive any current test run
- Real-time dashboard updates — R-12 is not covered; the bug was found manually, not by a test
- Admin view — no tests exist
- `calculateProgress` unit verification — the math behind the progress bar has never been asserted directly

---

### Biggest Gap

**The phase lock logic has no test coverage and it has already caused a real bug.**

R-09 is unverified. `getPhaseStatus` is untested. A member can — and did — access Phase 2 without completing Phase 1. The fact that this is fixed now does not mean it is tested. Without a test asserting the lock behavior, the next change to phase logic could reintroduce it invisibly.

This is the one missing piece that would give the most confidence if fixed. One unit test for `getPhaseStatus` with Phase 1 incomplete would have caught Kristi's original bug before it reached any member.

---

### 5 Test Cases to Write Next — In Priority Order

| # | Test Case | Why This One |
|---|---|---|
| 1 | `getPhaseStatus(phase=2, phase1Completed=0, phase1Total=5)` returns `"locked"` | This is the exact bug that was found. One unit test. It runs in milliseconds. It would have caught the regression before it happened. No other test comes close in cost-to-confidence ratio. |
| 2 | Direct URL navigation to Phase 2 when Phase 1 is 0% complete → access is blocked, Phase 2 content is not visible | The unit test covers the function. This covers the system. A clever member could bypass the menu by typing the URL directly. This test catches that bypass. |
| 3 | `calculateProgress(completed, total)` is mathematically correct for 0, partial, and full completion | Progress has never been unit tested. The visual changes — but if the math is off by one due to rounding, every member's dashboard lies. This test is cheap and closes a gap in a High risk feature. |
| 4 | Phase 2 task count on dashboard updates immediately after submitting a task — no page reload required | R-12 was a confirmed bug found manually. No regression test exists. Without this test, the bug can silently return with any future change to the dashboard update logic. |
| 5 | `canSubmitTask("TEXT", "")` returns false AND empty submission attempt leaves task in its current state with no progress change | The empty TEXT submission is blocked at the UI level — but the guard function has no unit test. This test covers the logic independently of the interface and would catch a regression if the guard were accidentally removed. |

---

*These five tests address the four High risk features, close the two Not Covered requirements, and eliminate the most dangerous regression paths. They are not the most obvious tests to write — the obvious tests for CHECKBOX and TEXT submission already exist. These are the tests that the suite is missing because they test what the product must prevent, not just what it must do.*
