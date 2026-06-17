# Assignment 2 — Test Plan: Task Submission Feature

## Scope

**Testing:**
- CHECKBOX task completion and toggle
- TEXT task submission and revision
- LINK task submission with URL validation
- Progress percentage updating on phase page and dashboard
- Task state transitions (Not Started → In Progress → Submitted/Completed)
- Error states (empty text, invalid URL, network error)

**Not Testing:**
- Admin approving or rejecting submissions
- File uploads
- Notifications
- Admin submitting on behalf of a member
- Mobile responsiveness

---

## Entry Criteria

- The feature is deployed to staging
- At least one member account and one admin account are available for testing
- The curriculum has at least one task of each type (CHECKBOX, TEXT, LINK)
- Progress percentage is visible on both the phase page and dashboard

---

## Exit Criteria

- All 5 test cases pass
- Progress percentage updates correctly after each submission without a page reload
- No failed submission shows a success state
- Submitted data persists after page refresh

---

## Test Cases

---

**TC-01 — Happy Path: Member submits a TEXT task and progress updates**
**Type:** Validation

Steps:
1. Log in as a member
2. Go to Dashboard — note the current progress percentage
3. Click a phase card
4. Click a TEXT task
5. Verify task state changes to In Progress
6. Type a written response in the textarea
7. Click Submit
8. Return to phase page — check progress percentage
9. Return to dashboard — check overall progress percentage
10. Refresh the page — verify the response is still saved

Expected result: Task moves to Submitted, progress updates on both pages immediately, response persists after refresh.

Verification or Validation: **Validation** — checks that the full flow makes sense for a real user completing a task, not just that buttons work.

---

**TC-02 — Edge Case: CHECKBOX task can be toggled on and off**
**Type:** Verification

Steps:
1. Log in as a member
2. Navigate to a CHECKBOX task
3. Click to complete it — verify state changes to Completed and progress updates
4. Click again to undo — verify state reverts and progress decreases

Expected result: Toggle works in both directions, progress recalculates correctly each time.

Verification or Validation: **Verification** — directly checks the business rule stated in the spec.

---

**TC-03 — Edge Case: TEXT or LINK task can be revised after submission**
**Type:** Verification

Steps:
1. Log in as a member
2. Submit a TEXT task with response "First answer"
3. Return to the task
4. Edit the response to "Revised answer" and resubmit
5. Refresh the page

Expected result: The revised response replaces the original. Task remains in Submitted state. Progress percentage does not change.

Verification or Validation: **Verification** — checks the revision rule from the spec is implemented correctly.

---

**TC-04 — Error Scenario: Member tries to submit empty TEXT field**
**Type:** Verification

Steps:
1. Log in as a member
2. Navigate to a TEXT task
3. Leave the textarea empty
4. Attempt to click Submit

Expected result: Submit button is disabled — submission is blocked, no error message needed, no state change occurs.

Verification or Validation: **Verification** — checks the business rule: empty TEXT submission must be blocked.

---

**TC-05 — Error Scenario: Member submits an invalid URL for a LINK task**
**Type:** Validation

Steps:
1. Log in as a member
2. Navigate to a LINK task
3. Type "notavalidurl" in the URL field
4. Click Submit

Expected result: Inline error appears next to the field, submission is blocked, task state does not change to Submitted.

Verification or Validation: **Validation** — also check whether the error message is clear enough for a real user to understand what went wrong. If the error just says "invalid" with no guidance, it passes verification but fails validation.
