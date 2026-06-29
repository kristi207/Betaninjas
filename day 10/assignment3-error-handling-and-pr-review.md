# Day 10 — Assignment 3
## Find What Was Never Handled

---

## Step 1 — The Silent Error (TextSubmission.tsx)

### What is inside the try block

The try block does two things: it builds the full response string (appending the existing note if one exists), then sends a POST request to `/api/progress` with the taskId, status, and response. If the server responds with `res.ok`, it sets `saved` to true and calls `router.refresh()` to reload the page data.

### What happens in the catch block

There is no catch block. The function has a try and a finally — no catch.

### What the finally block does

The finally block sets `setLoading(false)`. This runs no matter what happens — whether the request succeeded, failed, or threw an error. It stops the loading spinner.

### The gap

When a submission fails — network error, server returning 500, fetch throwing entirely — the finally block runs and the spinner disappears. `saved` is never set to true, so the "Submitted!" confirmation never appears. But no error state is set and no error message exists in the component. The member sees the button return to its normal state. There is no indication that their submission failed. Their work was not saved and they do not know it.

### What good error handling would look like

The component needs an `error` state variable. The try/catch should be:

```
try {
  ...fetch...
  if (res.ok) {
    setSaved(true)
    router.refresh()
  } else {
    setError("Submission failed. Please try again.")
  }
} catch {
  setError("Network error. Check your connection and try again.")
} finally {
  setLoading(false)
}
```

The UI should render the error message below the submit button so the member knows their work was not saved and can try again without losing what they typed.

### LinkSubmission.tsx comparison

The error handling in `LinkSubmission.tsx` is identical — no catch block, only a finally that clears loading — so a failed link submission is equally silent.

---

## Step 2 — The API Gap (app/api/progress/route.ts)

### Is there a try/catch around the prisma.progress.upsert() call?

No. The `prisma.progress.upsert()` call on line 33 has no try/catch around it. There is no error handling anywhere in the POST handler after validation passes.

### What happens if the database is unavailable and this call throws

The error propagates up uncaught. Next.js catches it at the framework level and returns a generic `500 Internal Server Error` response. The response body is not a structured JSON error — it is whatever Next.js produces for an unhandled exception.

### What the member experiences

The fetch in the component receives a non-ok response (`res.ok` is false). Since the component has no catch or error state, the member sees the loading spinner disappear and nothing else. They have no confirmation and no error message. Their submission was not saved.

### What correct error handling should look like

The POST handler should wrap the upsert in try/catch:

```typescript
try {
  const progress = await prisma.progress.upsert({ ... })
  return NextResponse.json(progress)
} catch (error) {
  console.error("Database error saving progress:", error)
  return NextResponse.json(
    { error: "Failed to save progress. Please try again." },
    { status: 500 }
  )
}
```

The server should return a consistent JSON error with status 500. The component should read that error and display it to the member. Together they close the gap — the server fails gracefully, the UI tells the member what happened.

---

## Step 3 — Read PR #28

PR #28: "Refactor member task detail UI with timeline status and progress tracking"

### What MemberTaskDetail.tsx displays

`MemberTaskDetail.tsx` shows an admin the full task-by-task progress of a specific member, organized by phase. Each phase is displayed as a collapsible accordion showing:

- The phase order number, title, and completion percentage
- A timeline status badge — Completed, On track, Behind, Overdue, or Not started — calculated from the phase's start and due dates against current progress
- Each task inside the phase with a done/not-done icon, task title, and a status badge (Done / Not submitted / Not done)
- For completed tasks: the exact timestamp when it was submitted or completed
- For LINK tasks: a clickable link to the submitted URL
- For TEXT tasks: the full submitted response text, shown quoted

### What SendReminderButton.tsx does

`SendReminderButton.tsx` adds a button that lets an admin send a reminder to a specific member. When clicked, it sends a POST to `/api/reminders` with the member's userId, shows "Sending…" while the request is in flight, then switches to "Sent!" and disables itself so the reminder cannot be sent twice in one session.

### 3 Targeted Test Cases for PR #28

**TC-PR28-01: Timeline status badge reflects correct state**
- Precondition: A member has completed 100% of Phase 1 tasks. Phase 1's due date has not passed.
- Action: Admin opens the member detail page.
- Expected: Phase 1 accordion shows the "Completed" badge in green.
- Why this test exists: `getPhaseStatus` is newly wired into this component. This PR is the first place that status badge appears in an admin view.

**TC-PR28-02: Phase accordion auto-expands when member has partial progress**
- Precondition: A member has completed at least one task in Phase 2 but not all.
- Action: Admin opens the member detail page.
- Expected: Phase 2 accordion is open by default, showing individual task cards.
- Why this test exists: The accordion's default open state is controlled by `pct > 0` — new logic introduced in this PR that did not exist before.

**TC-PR28-03: Send reminder button disables after being clicked once**
- Precondition: Admin is on a member detail page. The member has not received a reminder this session.
- Action: Admin clicks "Send reminder." Request completes.
- Expected: Button shows "Sent!" and is disabled. Clicking it again does nothing — no second request is sent.
- Why this test exists: `SendReminderButton` is a new component added entirely in this PR. The disable-after-send behavior is its core contract and needs explicit verification.

---

## Step 4 — The Question

### Which gap is worse for the member and why

The unhandled API crash in `route.ts` is worse. When the database is unavailable, every member trying to submit at that moment is affected — it is not one member's network problem, it is a server-wide failure. The upsert throws, Next.js returns a 500, and every submission during that window is silently lost. The member sees nothing, their work is gone, and they have no way to know whether to retry or wait. The UI gap in the submission components is a consequence of this — but even if the UI were fixed to show an error message, the root cause is the API returning an unstructured crash response instead of a handled error.

### Which gap is easier to fix

The UI gap in `TextSubmission.tsx` and `LinkSubmission.tsx` is easier to fix. Both components need the same change: add a catch block, add an `error` state variable, and render an error message. It is a self-contained change inside two components with no database or infrastructure dependency.

### One test that catches both gaps at the same time

**TC-DUAL-01: Member sees an error message when a submission fails at the server**
- Precondition: Mock the POST `/api/progress` endpoint to return a 500 response (simulating a database crash).
- Action: Member fills in the text field and clicks "Submit Response."
- Expected: An error message appears — something like "Submission failed. Please try again." The "Submitted!" confirmation does not appear. The text the member typed is still in the field.
- Why this catches both: The 500 response exposes the API gap — the server failed without a handled error. If the UI shows nothing, the test fails, exposing the UI gap. For this test to pass, the API must return a structured error response AND the UI must read it and display it. Both gaps must be closed.
