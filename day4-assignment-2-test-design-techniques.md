# Day 4 — Assignment 2: Design Tests Like a Pro

**Target:** Task Submission Feature — BetaninjasSEOLMS

---

## Step 1 — Boundary Value Analysis

### Date Fields: Start Date and Due Date

**Rule:** Due Date must be after Start Date.

The boundary is the point where "after" flips to "not after" — one day apart is valid, same day is not.

| TC# | Start Date   | Due Date     | Expected Result                        |
|-----|--------------|--------------|----------------------------------------|
| BVA-01 | 2025-06-01 | 2025-06-01 | Invalid — due date must be after start date, not equal |
| BVA-02 | 2025-06-01 | 2025-06-02 | Valid — minimum valid gap (1 day apart) |
| BVA-03 | 2025-06-01 | 2025-05-31 | Invalid — due date is before start date |
| BVA-04 | (empty)    | 2025-06-02 | Invalid — start date is required       |
| BVA-05 | 2025-06-01 | (empty)    | Invalid — due date is required         |
| BVA-06 | (empty)    | (empty)    | Invalid — both fields required         |

---

### Character Limit on TEXT Task Submission

**Finding:** No character limit is defined or enforced.

- The textarea accepts input without any visible counter or cap.
- Testing with a 10,000-character paste: the form accepted it and submitted without error.
- The UI provides no feedback about length constraints.

**What the boundary is:** Undefined — there is no explicit character limit in the current implementation.

**What happens beyond it:** Unknown upper bound. The database column type determines the actual ceiling, but the application does not surface this to the user. A user could submit extremely long text without warning, which may cause unexpected truncation at the database layer or performance issues.

**Recommendation:** Define a maximum character count (e.g. 5,000), enforce it server-side, and display a live counter in the UI.

---

## Step 2 — Equivalence Partitioning

### Task Submission Form — Input Classes

| Class          | Description                                 | Representative Value         |
|----------------|---------------------------------------------|------------------------------|
| Valid text     | Non-empty text within any reasonable length | "This is my task response."  |
| Empty text     | Blank or whitespace-only input              | "" or "   "                  |
| Invalid type   | Non-string input (e.g. script injection)    | <script>alert(1)</script>    |
| Very long text | Text exceeding expected reasonable length   | 10,000-character string      |

**One test case per class:**

| TC# | Class          | Input                        | Expected Result                               |
|-----|----------------|------------------------------|-----------------------------------------------|
| EP-01 | Valid text   | "This is my task response."  | Submission succeeds, task moves to Submitted  |
| EP-02 | Empty text   | "" (blank textarea)          | Submit button disabled or error shown, no state change |
| EP-03 | Invalid type | <script>alert(1)</script>    | Input treated as plain text, no script executes |
| EP-04 | Very long text | 10,000-character string    | Accepted with no feedback — ideally blocked with a length error |

**Observation:** Four equivalence classes reduce the full input space to four representative tests. We do not need 100 valid strings — one covers the entire valid class.

---

## Step 3 — Decision Table

### Conditions for Task Submission

| Condition             | Rule 1 | Rule 2 | Rule 3 | Rule 4 | Rule 5 |
|-----------------------|--------|--------|--------|--------|--------|
| User authenticated    | N      | Y      | Y      | Y      | Y      |
| Task ID valid         | —      | N      | Y      | Y      | Y      |
| Status is valid enum  | —      | —      | N      | Y      | Y      |
| Text field non-empty  | —      | —      | —      | N      | Y      |
| Expected Outcome      | 401 Unauthorized | 404 Not Found | 400 Bad Request | 400 Bad Request | 200 Success |

> Authentication is checked first — an unauthenticated request returns 401 before the server inspects anything else.

**One test case per rule:**

| TC#   | Rule | Setup                                                       | Expected Outcome  |
|-------|------|-------------------------------------------------------------|-------------------|
| DT-01 | R1   | Send request with no auth token                             | 401 Unauthorized  |
| DT-02 | R2   | Authenticated; use a non-existent task ID                   | 404 Not Found     |
| DT-03 | R3   | Authenticated, valid task ID; send status "FLYING"          | 400 Bad Request   |
| DT-04 | R4   | Authenticated, valid task ID, valid status; empty text      | 400 Bad Request   |
| DT-05 | R5   | All conditions met; valid response text                     | 200 Success       |

---

## Step 4 — State Transition Testing

### CHECKBOX Tasks — State Diagram
    [NOT_STARTED]
         |
    (check box)
         |
         v
    [COMPLETED]
         |
    (uncheck box)
         |
         v
    [NOT_STARTED]
CHECKBOX tasks toggle between exactly two states. There are no intermediate states.
**Valid transitions:**
| Transition              | Trigger         |
|-------------------------|-----------------|
| NOT_STARTED -> COMPLETED | Check the box  |
| COMPLETED -> NOT_STARTED | Uncheck the box |
**Invalid transitions:**
| Invalid Transition          | Expected Behavior             |
|-----------------------------|-------------------------------|
| NOT_STARTED -> IN_PROGRESS  | Not possible — no such action |
| NOT_STARTED -> SUBMITTED    | Not possible — no such action |
| COMPLETED -> SUBMITTED      | Not possible — no such action |
**Test cases:**
| TC#    | Starting State | Action                                          | Expected End State |
|--------|----------------|-------------------------------------------------|--------------------|
| ST-C01 | NOT_STARTED    | Check the box                                   | COMPLETED          |
| ST-C02 | COMPLETED      | Uncheck the box                                 | NOT_STARTED        |
| ST-C03 | NOT_STARTED    | API call with status=SUBMITTED                  | Rejected / 400     |
---
### TEXT and LINK Tasks — State Diagram
    [NOT_STARTED]
         |
    (open/start task)
         |
         v
    [IN_PROGRESS]
         |
    (submit response)
         |
         v
    [SUBMITTED] <----+
         |           |
    (admin marks     | (member revises)
     complete)       |
         |      [IN_PROGRESS]
         v
    [COMPLETED]
**Valid transitions:**
| Transition                  | Trigger                  |
|-----------------------------|--------------------------|
| NOT_STARTED -> IN_PROGRESS  | Member opens/starts task |
| IN_PROGRESS -> SUBMITTED    | Member submits response  |
| SUBMITTED -> IN_PROGRESS    | Member revises           |
| SUBMITTED -> COMPLETED      | Admin approves           |
**Invalid transitions:**
| Invalid Transition         | Expected Behavior                             |
|----------------------------|-----------------------------------------------|
| NOT_STARTED -> SUBMITTED   | Should be rejected                            |
| NOT_STARTED -> COMPLETED   | Should be rejected                            |
| IN_PROGRESS -> COMPLETED   | Should be rejected — must go through SUBMITTED |
| COMPLETED -> NOT_STARTED   | Should be rejected                            |
**Test cases:**
| TC#    | Starting State | Action                           | Expected End State  | Notes                        |
|--------|----------------|----------------------------------|---------------------|------------------------------|
| ST-T01 | NOT_STARTED    | Open the task                    | IN_PROGRESS         | —                            |
| ST-T02 | IN_PROGRESS    | Submit a valid response          | SUBMITTED           | —                            |
| ST-T03 | SUBMITTED      | Edit and resubmit                | SUBMITTED (updated) | Stays in same state          |
| ST-T04 | SUBMITTED      | Admin marks complete             | COMPLETED           | Admin action                 |
| ST-T05 | NOT_STARTED    | API call with status=COMPLETED   | Rejected / 400      | Key test: skip middle states |
| ST-T06 | NOT_STARTED    | API call with status=SUBMITTED   | Rejected / 400      | Cannot jump to SUBMITTED     |
| ST-T07 | IN_PROGRESS    | API call with status=COMPLETED   | Rejected / 400      | Must pass through SUBMITTED  |
**Key finding for ST-T05:** Jumping directly from NOT_STARTED to COMPLETED on a TEXT task should NOT be allowed. The SUBMITTED state exists so an admin can review the work before marking it complete. The system must enforce the state machine server-side, not rely only on the UI.
---
## Step 5 — Testing Levels Mapped to Task Submission
| Level            | What to Test Here                                                                                                                                 |
|------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|
| Unit             | The function that validates task status transitions — confirm it rejects illegal state jumps in isolation.                                         |
| Integration      | The API endpoint receiving a submission: verify the controller, service, and database layer correctly persist state and return the right status codes. |
| System           | The full end-to-end flow — member logs in, submits a TEXT task, and progress updates on both the phase page and dashboard without a reload.        |
| Acceptance (UAT) | A real user walks through submitting a task and confirms the experience is clear, feedback makes sense, and the task appears correctly in history.  |
