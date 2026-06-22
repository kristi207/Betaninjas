# Assignment 5 — Document It Like It Matters

**Target:** Dashboard — betaninjas-seo-learning-tracker.vercel.app  
**Tested as:** Member account  
**Date:** 19 June 2026

---

## Step 1 — Test Scenarios and Test Cases

### Scenario 1 — Dashboard Progress Reflects Actual Task Completion

**What this tests:** When a member completes tasks, the dashboard must show updated progress numbers immediately and accurately. This is the core value of the dashboard — it should be a live, accurate picture of where the member stands.

---

**TC-S1-01 — Progress percentage increases after completing a CHECKBOX task**

Steps:
1. Log in as a member.
2. Go to the Dashboard and note the current overall progress percentage shown on the screen.
3. Click any phase card that has at least one CHECKBOX task still in NOT STARTED state.
4. Click the grey circle ("Mark as complete") on one CHECKBOX task.
5. Wait for the green tick and pink background to appear — confirming the task is marked complete.
6. Click the browser back button to return to the Dashboard.
7. Read the overall progress percentage now displayed.

Expected result: The progress percentage on the Dashboard is higher than it was in step 2. The increase reflects the one task just completed. The change appears without requiring a page reload.

---

**TC-S1-02 — Progress percentage decreases after undoing a CHECKBOX task**

Steps:
1. Log in as a member.
2. Go to the Dashboard and note the current overall progress percentage.
3. Click into a phase that has at least one CHECKBOX task already in COMPLETED state (green tick visible).
4. Click "Click to undo" on that task.
5. Wait for the task to return to NOT STARTED state (grey circle visible).
6. Navigate back to the Dashboard.
7. Read the overall progress percentage now displayed.

Expected result: The progress percentage is lower than it was in step 2. It has decreased by exactly one task's weight. The dashboard does not require a manual refresh to reflect the change.

---

**TC-S1-03 — Phase card on Dashboard shows correct task count per phase**

Steps:
1. Log in as a member.
2. On the Dashboard, read the task count or progress indicator shown on one specific phase card (e.g., "Phase 1 — 3 of 8 tasks complete").
3. Click that phase card to open it.
4. Manually count the total number of tasks visible in the phase.
5. Manually count how many tasks are in COMPLETED state (green tick or "Done" in dropdown).
6. Navigate back to the Dashboard.
7. Compare the count from step 5 to the number displayed on the phase card in step 2.

Expected result: The number of completed tasks displayed on the Dashboard phase card matches the count you manually verified inside the phase. No phantom completions, no missing ones.

---

### Scenario 2 — Dashboard Navigation Loads the Correct Content

**What this tests:** Every navigation element on the Dashboard must open the right page. If a phase card links to the wrong phase, or a link is broken, a member cannot access their work. This tests the Dashboard as a navigation hub.

---

**TC-S2-01 — Clicking a phase card opens the correct phase page**

Steps:
1. Log in as a member.
2. On the Dashboard, identify the phase cards listed (e.g., Phase 1, Phase 2, Phase 3).
3. Note the title of the first phase card shown.
4. Click that phase card.
5. Read the heading of the page that opens.

Expected result: The heading of the opened page matches the title of the phase card you clicked. The correct phase loads, not a generic page or a different phase.

---

**TC-S2-02 — Dashboard loads fully without errors after login**

Steps:
1. Open the browser and navigate to https://betaninjas-seo-learning-tracker.vercel.app.
2. Log in with valid member credentials.
3. After login, observe where the app redirects you.
4. Look at the full page — check for any blank sections, loading spinners that never resolve, or error messages.
5. Open browser DevTools (F12) → Console tab.
6. Check for any red error messages in the console.

Expected result: The Dashboard loads completely within 3 seconds. All sections are populated with content. No blank cards, no unresolved spinners, and no console errors are visible.

---

**TC-S2-03 — Dashboard is not accessible without login**

Steps:
1. Open a new private/incognito browser window.
2. Navigate directly to https://betaninjas-seo-learning-tracker.vercel.app/dashboard (without logging in).
3. Observe what the app shows.

Expected result: The app does not show the dashboard. It redirects to the login page or shows an "unauthorized" message. No private member data is visible to an unauthenticated user.

---

## Step 2 — Bug Report

**Title:** Login page states access is restricted to betaninjas.com emails but accepts any email domain

**Steps to reproduce:**
1. Open the browser and navigate to https://betaninjas-seo-learning-tracker.vercel.app.
2. On the login page, read the restriction message displayed (e.g., "This platform is restricted to betaninjas.com accounts" or similar wording).
3. Enter a valid email address from a non-betaninjas domain — for example, a Gmail address or any personal email — along with its password.
4. Click the Sign In button.
5. Observe whether access is granted or blocked.

**Expected result:** The login is rejected. The app displays an error such as "Access restricted to betaninjas.com email addresses only." The user cannot enter the platform with a non-betaninjas email.

**Actual result:** The login succeeds. The user is taken directly into the dashboard with full access to the platform, despite using an email address from outside the stated betaninjas.com domain. The domain restriction shown on the login page is not enforced anywhere.

**Environment:**
- Browser: Chrome 125
- OS: Windows 11

**Severity: High**  
Rationale: This is an access control failure — the stated security boundary of the app is not enforced. Any person with a non-betaninjas email can log in and view member data, progress, and curriculum content. The app communicates a restriction it does not actually implement, which is both a security gap and a misleading UI.

**Priority: High**  
Rationale: This sits at the front door of the app and affects every login attempt. Unlike a bug buried in an admin settings page, this is reachable by anyone who visits the site. It must be fixed immediately — either enforce the domain restriction server-side, or remove the misleading message if open access is intentional.

---

## Step 3 — Defect Lifecycle

**Bug tracked:** Login page states access is restricted to betaninjas.com emails but accepts any email domain

---

| Stage | Who is responsible | What happens |
|-------|--------------------|--------------|
| **New** | QA Engineer (the tester) | The bug is found while testing the login page. The tester signs in using a non-betaninjas email and successfully reaches the dashboard, despite the login page stating access is restricted to betaninjas.com. The full bug report is written and entered into the tracking system (e.g., Jira, Linear) with severity High and priority High. |
| **Assigned** | Team Lead / Project Manager | The lead reviews the report, confirms it is valid — the domain restriction is stated in the UI but not enforced — and assigns it to the backend or authentication developer responsible for the login and access control logic. The developer is notified. |
| **Open** | Developer (assigned) | The developer reproduces the bug by logging in with a non-betaninjas email and confirms full dashboard access is granted. They begin investigating the authentication flow. The status moves to Open. The likely cause: the domain validation check exists only as a UI label or was never implemented server-side — the auth provider (e.g., NextAuth, Firebase Auth) is accepting all valid credentials regardless of domain. |
| **Fixed** | Developer (assigned) | The developer adds server-side domain validation to the authentication handler. After sign-in, the server checks whether the authenticated email ends in @betaninjas.com — if not, the session is rejected and the user is redirected to the login page with an appropriate error message. The fix is committed, a PR is raised, and the bug status is set to Fixed. |
| **Retest** | QA Engineer (original tester) | The QA engineer tests on the fixed build. They attempt to log in with a non-betaninjas email and verify that access is now blocked with a clear error message. They also confirm that valid betaninjas.com accounts can still log in normally — no regression introduced at the login gate. |
| **Verified** | QA Engineer (original tester) | For this specific bug, Verified means: a non-betaninjas email is blocked at login with an appropriate error message, the dashboard is not accessible to that user, and betaninjas.com accounts continue to work normally. The fix is confirmed working. The QA engineer marks the bug as Verified. |
| **Closed** | QA Engineer / Team Lead | The bug is marked Closed once verified in the correct environment (staging or production). For this bug, Closed means: no non-betaninjas email can reach the dashboard in any tested browser, the fix is deployed to production, and no regression was found. The ticket is archived. |

---

## Step 4 — Entry and Exit Criteria

### Entry Criteria — Dashboard Testing

Testing of the Dashboard feature may begin only when ALL of the following are true:

1. The Dashboard feature is deployed to the staging environment (not just local development).
2. At least one valid member account with real task data is available — the dashboard must have something to display.
3. Login is working — if login is broken, the dashboard cannot be reached.
4. At least one phase exists in the system with at least one CHECKBOX task and one TEXT task, each in NOT STARTED state, so progress tracking can be tested from zero.
5. The progress percentage calculation logic has been reviewed and approved by the developer — we need to know what the formula is before we can verify it is correct.
6. The test environment's database is seeded with consistent, known data — testers must know what progress numbers to expect before they start verifying them.

### Exit Criteria — Dashboard Testing

Testing of the Dashboard may be signed off as complete only when ALL of the following are true:

1. All 6 test cases from Step 1 have been executed and results recorded.
2. All 6 test cases pass — no test case is in a FAILED state.
3. Zero Severity 1 (Critical) bugs are open.
4. Zero Severity 2 (High) bugs are open — or each open High bug has a documented risk acceptance sign-off from the project lead.
5. Progress percentages on the dashboard match manually verified task counts for at least two different members (cross-user verification).
6. The dashboard loads without console errors in Chrome, Firefox, and Safari.
7. The unauthenticated access test (TC-S2-03) passes — no data leaks to logged-out users.
8. All test results, including any skipped tests and their reasons, are documented in the test report below.

---

## Step 5 — Test Report

**Feature tested:** Dashboard  
**Tester:** Kristi Panta  
**Date tested:** 19 June 2026  
**Environment:** Chrome 125 / Windows 11, member account  

---

### Summary

6 test cases were designed and executed across 2 scenarios. 4 passed, 1 failed, and 1 was skipped.

---

### Results

| TC# | Test Case | Result | Notes |
|-----|-----------|--------|-------|
| TC-S1-01 | Progress percentage increases after completing a CHECKBOX task | PASS | Progress updated on dashboard without reload after marking task complete. |
| TC-S1-02 | Progress percentage decreases after undoing a CHECKBOX task | PASS | Progress decreased correctly when undo was applied. |
| TC-S1-03 | Phase card shows correct task count per phase | FAIL | The phase card on the dashboard showed 3 of 8 tasks complete. Manual count inside the phase showed 4 complete. The dashboard was one task behind. Refreshing the page corrected the count — the dashboard does not update the phase card count in real time after task changes made in the same session. |
| TC-S2-01 | Clicking a phase card opens the correct phase page | PASS | Each phase card opened the matching phase correctly. Titles matched. |
| TC-S2-02 | Dashboard loads fully without errors after login | PASS | Page loaded within 2 seconds, no console errors, all sections populated. |
| TC-S2-03 | Dashboard is not accessible without login | SKIP | Could not test in this session — accessing an incognito window was not possible in the remote testing environment. This test case must be run manually before sign-off. |

---

### What Passed

- Progress tracking works for CHECKBOX task completion and undo in both directions (TC-S1-01, TC-S1-02).
- Phase card navigation is correct — every card opens the right phase (TC-S2-01).
- Dashboard loads cleanly with no errors after login (TC-S2-02).

### What Failed

- **TC-S1-03 — Phase card task count is stale within a session.**  
  After completing or undoing tasks, the overall progress percentage updates correctly, but the individual phase card count does not update until a full page refresh. A member who completes tasks and returns to the dashboard will see an outdated count on the phase card. This is a UX defect — low severity, medium priority.

### What Was Skipped

- **TC-S2-03 — Unauthenticated access** was skipped because the test environment did not support opening an incognito session during this test run. This is a security-relevant test case and must not be permanently skipped — it must be completed before dashboard testing can be signed off.

### Quality Assessment

The dashboard is functional for its primary purpose: showing progress and navigating to phases. The core tracking logic works. One real defect was found (stale phase card counts), one test remains outstanding (unauthenticated access). The feature is not yet ready for sign-off — the skipped test must be run, and the stale count defect must be triaged before release.
