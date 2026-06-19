# Day 4 — Assignment 2: Design Tests Like a Pro

**Target:** Task Submission Feature — BetaninjasSEOLMS  
**URL:** https://betaninjas-seo-learning-tracker.vercel.app  
**Tested as:** Admin (Prem)

---

## Step 1 — Boundary Value Analysis

### Date Fields: Start Date and Due Date (/admin/timeline)

**Rule observed on site:** Due Date must be strictly after Start Date.  
**Validation type:** Client-side — error appears inline next to the Save button in red text.  
**Error message:** "Due date must be after start date"

| TC# | Start Date | Due Date | Action | Actual Result |
|-----|------------|----------|--------|---------------|
| BVA-01 | 18/04/2026 | 18/04/2026 | Click Save | ❌ Red error — "Due date must be after start date" |
| BVA-02 | 18/04/2026 | 19/04/2026 | Click Save | ✅ Green "✓ Saved" — minimum valid gap accepted |
| BVA-03 | 18/04/2026 | 17/04/2026 | Click Save | ❌ Red error — "Due date must be after start date" |
| BVA-04 | (empty) | 19/04/2026 | Click Save | Not tested — date picker does not allow clearing easily |
| BVA-05 | 18/04/2026 | (empty) | Click Save | Not tested — date picker does not allow clearing easily |
| BVA-06 | (empty) | (empty) | Click Save | ❌ App crashes — "Application error: a client-side exception has occurred" |

**Key finding — BVA-01:** Same date is rejected. The rule is strictly "after", not "on or after".  
**Key finding — BVA-06 (Bug):** Clearing both date fields and saving causes a full client-side crash. The app shows a blank error page. No graceful error message. This is a bug — empty dates should show a validation error, not crash the app.

---

### Character Limit on TEXT Task Submission

**Observed on:** Task "Write 10 SEO terms in your own words"  
**UI element:** Textarea with a character counter in the bottom-right corner

| TC# | Input | Counter shown | Submit button | Actual Result |
|-----|-------|---------------|---------------|---------------|
| CL-01 | Empty (0 chars) | 0 chars | Faded/disabled | ❌ Cannot submit — button is disabled |
| CL-02 | 31 chars | 31 chars | Active (blue) | ✅ Submittable |
| CL-03 | ~325,000 chars | 325,220 chars | Active (blue) | ✅ Submitted successfully — "✓ Submitted!" |
| CL-04 | ~1.2 million chars | 1,231,190 chars | Active (blue) | ✅ Submitted successfully — no block |

**Finding:** There is a character counter in the UI but no character limit is enforced. The textarea accepted and submitted 1,231,190 characters without any error or warning. The only enforcement is blocking empty submissions. There is no upper boundary defined or communicated to the user.

**Recommendation:** Define a maximum (e.g. 5,000 chars), enforce it server-side, turn the counter red when exceeded, and disable Submit at the limit.

---

## Step 2 — Equivalence Partitioning

### Task Submission Form — Input Classes

The submission form has one textarea field ("YOUR SUBMISSION") and a "Submit Response" button.

| Class | Description | Representative Input |
|-------|-------------|----------------------|
| Valid text | Non-empty text, any reasonable length | "SEO stands for Search Engine Optimisation" |
| Empty text | 0 characters in textarea | "" (cleared textarea) |
| Very long text | No defined upper limit — counter keeps going | 1,231,190 character string |
| Invalid/unexpected type | Script injection attempt | `<script>alert(1)</script>` |

**One test case per class:**

| TC# | Class | Input | Expected Result | Actual Behavior |
|-----|-------|-------|-----------------|-----------------|
| EP-01 | Valid text | "SEO stands for Search Engine Optimisation" | Submission succeeds, "✓ Submitted!" shown | ✅ Works as expected |
| EP-02 | Empty text | "" (blank textarea, 0 chars) | Submit blocked | ✅ Button is faded and disabled — cannot click |
| EP-03 | Very long text | 1,231,190 char string | Should be blocked or warned | ❌ Accepted and submitted with no warning |
| EP-04 | Invalid type | `<script>alert(1)</script>` | Treated as plain text, no script runs | Not tested — requires browser console verification |

**Observation:** Three of the four classes are covered by two UI states — the button is either disabled (empty) or active (anything else). The app makes no distinction between a normal response and a 1.2 million character dump. Fewer tests still expose the same gaps — EP-03 finds the missing upper boundary in one test case.

---

## Step 3 — Decision Table

### Conditions for Task Submission

| Condition | Rule 1 | Rule 2 | Rule 3 | Rule 4 | Rule 5 |
|-----------|--------|--------|--------|--------|--------|
| User authenticated | N | Y | Y | Y | Y |
| Task ID valid | — | N | Y | Y | Y |
| Status is valid enum | — | — | N | Y | Y |
| Text field non-empty | — | — | — | N | Y |
| **Expected Outcome** | **401 Unauthorized** | **404 Not Found** | **400 Bad Request** | **400 Bad Request** | **200 Success** |

> Authentication is checked first — an unauthenticated request returns 401 before the server checks anything else. Rules 2–5 only apply to authenticated users.

**One test case per rule:**

| TC# | Rule | Setup | Expected Outcome |
|-----|------|-------|------------------|
| DT-01 | R1 | Send API request with no auth token | 401 Unauthorized |
| DT-02 | R2 | Authenticated; use a non-existent task ID | 404 Not Found |
| DT-03 | R3 | Authenticated, valid task ID; send status "FLYING" (not a valid enum) | 400 Bad Request |
| DT-04 | R4 | Authenticated, valid task ID, valid status; send empty text field | 400 Bad Request |
| DT-05 | R5 | All conditions met; valid non-empty response text | 200 Success |

> UI observation: The Submit button is disabled when text is empty (DT-04 is blocked at the UI level). DT-01 through DT-03 require direct API testing to verify server-side enforcement.

---

## Step 4 — State Transition Testing

### CHECKBOX Tasks — State Diagram

Observed on: "Watch HubSpot SEO Level 1" and "Watch HubSpot SEO Level 2"
    [NOT STARTED]
    Grey circle
    "Mark as complete"
    "Click to confirm you have completed this task"
         |
     (click)
         |
         v
    [COMPLETED]
    Green tick, pink background
    "Marked as complete"
    "Click to undo"
         |
     (click to undo)
         |
         v
    [NOT STARTED]
CHECKBOX tasks have exactly two states. There are no IN_PROGRESS or SUBMITTED states.
**Valid transitions:**
| Transition | Trigger | UI Change |
|------------|---------|-----------|
| NOT STARTED → COMPLETED | Click the checkbox | Grey → green tick, pink background, "Click to undo" appears |
| COMPLETED → NOT STARTED | Click "Click to undo" | Green → grey, returns to "Mark as complete" |
**Invalid transitions:**
| Invalid Transition | Expected Behavior |
|--------------------|-------------------|
| NOT STARTED → SUBMITTED | Not possible — no such action in UI |
| NOT STARTED → IN_PROGRESS | Not possible — no such action in UI |
| COMPLETED → SUBMITTED | Not possible — no such action in UI |
**Test cases:**
| TC# | Starting State | Action | Expected End State | Notes |
|-----|---------------|--------|--------------------|-------|
| ST-C01 | NOT STARTED | Click "Mark as complete" | COMPLETED — green tick, pink background | Valid |
| ST-C02 | COMPLETED | Click "Click to undo" | NOT STARTED — grey circle returns | Valid |
| ST-C03 | NOT STARTED | API call with status=SUBMITTED | Rejected / 400 | Invalid — must be tested via API |
---
### TEXT and LINK Tasks — State Diagram
Observed states in UI dropdown: "Not Started", "Done"  
Note: IN_PROGRESS and SUBMITTED may exist at the API level — "Done" appears to map to COMPLETED in the UI.
    [NOT STARTED]
    Empty textarea, 0 chars
    Submit button disabled
         |
    (type response)
         |
         v
    [IN PROGRESS]
    Textarea has content
    Submit button active (blue)
         |
    (click Submit Response)
         |
         v
    [SUBMITTED / DONE]
    "✓ Submitted!" shown in green
    Dropdown shows "Done"
         |
    (admin reviews and marks complete)
         |
         v
    [COMPLETED]
**Valid transitions:**
| Transition | Trigger | UI Feedback |
|------------|---------|-------------|
| NOT STARTED → IN PROGRESS | Type in textarea | Submit button becomes active |
| IN PROGRESS → SUBMITTED | Click "Submit Response" | "✓ Submitted!" shown in green |
| SUBMITTED → IN PROGRESS | Edit textarea and resubmit | Overwrites previous submission |
| SUBMITTED → COMPLETED | Admin marks complete | Status dropdown changes |
**Invalid transitions:**
| Invalid Transition | Expected Behavior |
|--------------------|-------------------|
| NOT STARTED → SUBMITTED | Blocked — Submit button disabled at 0 chars |
| NOT STARTED → COMPLETED | Should be rejected server-side |
| IN PROGRESS → COMPLETED | Should be rejected — must pass through SUBMITTED |
| COMPLETED → NOT STARTED | Should be rejected |
**Test cases:**
| TC# | Starting State | Action | Expected End State | Notes |
|-----|---------------|--------|--------------------|-------|
| ST-T01 | NOT STARTED | Leave textarea empty, observe Submit | Submit disabled | ✅ Confirmed in testing |
| ST-T02 | NOT STARTED | Type a response | IN PROGRESS — Submit becomes active | ✅ Confirmed |
| ST-T03 | IN PROGRESS | Click Submit Response | SUBMITTED — "✓ Submitted!" shown | ✅ Confirmed |
| ST-T04 | SUBMITTED | Edit and resubmit | SUBMITTED (updated) | Overwrites previous |
| ST-T05 | NOT STARTED | API call with status=COMPLETED | Rejected / 400 | Key test — skip middle states |
| ST-T06 | IN PROGRESS | API call with status=COMPLETED | Rejected / 400 | Must pass through SUBMITTED |
**Key finding for ST-T05:** Jumping directly from NOT STARTED to COMPLETED on a TEXT task must be blocked server-side. The UI prevents it (disabled button), but the state machine must also be enforced at the API level — a direct API call should not be able to skip the SUBMITTED review step.
---
## Step 5 — Testing Levels Mapped to Task Submission
| Level | What to Test Here |
|-------|-------------------|
| **Unit** | The function that validates state transitions — confirm it rejects illegal jumps like NOT_STARTED → COMPLETED in complete isolation from the rest of the app. |
| **Integration** | The API endpoint that receives a submission — verify the controller, validation layer, and database work together correctly to persist state, enforce auth, and return the right status codes. |
| **System** | The full end-to-end flow — member opens a TEXT task, types a response, submits it, and the task status updates to Done on the phase page and dashboard without a reload. |
| **Acceptance (UAT)** | A real user walks through submitting a task and confirms the flow feels clear — the disabled button makes sense, the "✓ Submitted!" confirmation is visible, and the task shows as Done in their progress view. |
