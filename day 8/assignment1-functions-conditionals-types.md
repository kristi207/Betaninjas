# Assignment 1 - Read Three Functions

## Function 1 - getPhaseStatus in lib/utils/timeline.ts

### Inputs

The function takes 4 inputs:

1. progressPercent - type number. This is the percentage of how much of the phase is done.
2. startDate - type Date. When the phase officially begins.
3. dueDate - type Date. The deadline for the phase.
4. now - type Date. This one has a default value of new Date(), which means if you do not pass anything in for it, it automatically uses the current date and time.

### Outputs

All 5 possible return values are strings. They come from the TimelineStatus type which is a union of string literals:

1. "COMPLETED"
2. "NOT_STARTED"
3. "OVERDUE"
4. "ON_TRACK"
5. "BEHIND"

### Branches

Reading through the function line by line, there are 5 separate paths:

- If progressPercent equals 100, the function returns COMPLETED right away and nothing else runs.
- If now is before startDate, meaning the phase has not started yet, the function returns NOT_STARTED.
- If now is past the dueDate and progressPercent is still less than 100, the function returns OVERDUE.
- If progressPercent is greater than or equal to the percentage of time that has already passed, the function returns ON_TRACK.
- If none of the above are true, meaning progress is falling behind the time elapsed, the function returns BEHIND.

### Minimum Tests

You need 5 test cases. One for each return value. Every path in the function leads to a different return value, and there is no way to cover all 5 outcomes with fewer than 5 tests. Each condition in the code creates a unique path that a test needs to walk through separately.

### Data Type Edge Case

If you pass progressPercent as the string "100" instead of the number 100, the first branch does not fire. The check is `progressPercent === 100` which uses strict equality. In JavaScript, "100" === 100 is false because one is a string and one is a number. The function would keep going and check the other conditions instead of returning COMPLETED. TypeScript would catch this mistake during compilation, but if the value is coming from something like a URL parameter or a form input without proper parsing, it could slip through at runtime and give you a wrong status.

---

## Function 2 - calcPhaseProgress in lib/utils/progress.ts

### Inputs

tasks expects an array of objects where each object has an id property that is a number. Something like `[{ id: 1 }, { id: 2 }]`. It does not need anything else on each object, just the id.

progressMap expects an object where the keys are task IDs (numbers) and the values are objects that have a status property which is a string. The type is `Record<number, { status: string }>`. So it would look like `{ 1: { status: "COMPLETED" }, 2: { status: "IN_PROGRESS" } }`.

### Output

It returns a number. The range of possible values is 0 to 100, representing a percentage. 0 means no tasks are done, 100 means all tasks are done. The value is always a whole number because of Math.round().

### The Branch

The one if condition is `if (tasks.length === 0) return 0`. It checks whether the tasks array is empty. If it is, the function returns 0 right away. This branch exists to prevent a divide-by-zero situation. If tasks is empty and you skip this check, you end up doing `0 / 0` which gives you NaN in JavaScript. Math.round(NaN) is still NaN, not 0. The function would return a bad value and anything downstream that relies on a number would break. The guard makes the function safe to call even with an empty list.

### isDone

isDone checks if `status === "SUBMITTED"` or `status === "COMPLETED"`. Both count as done for the purpose of calculating progress. So yes, a task can be counted as "done" without being COMPLETED. If a student has submitted their work but the instructor has not marked it complete yet, that task still counts toward the phase progress. This is a deliberate design choice, not a bug, because the progress reflects student effort rather than instructor review status.

### Edge Case - All Tasks SUBMITTED, None COMPLETED

If every task in the array has a status of SUBMITTED, isDone returns true for each of them because SUBMITTED is one of the two statuses it accepts. So all tasks pass the filter. The function calculates `tasks.length / tasks.length * 100` which is 100. The function returns 100.

### Isolation Question

If I were writing a unit test for calcPhaseProgress, I would not use the real database. The whole point of a unit test is to test one function in complete isolation from everything else. If you connect to the real database, your test now depends on whatever data happens to be in there, and if that data changes your test breaks for a reason that has nothing to do with the function itself. Instead of querying the database, I would pass in a plain JavaScript object as the progressMap, for example `{ 1: { status: "COMPLETED" }, 2: { status: "IN_PROGRESS" } }`. This is called a fake or a stub. It gives the function exactly what it needs to run, the shape matches the ProgressMap type, and the test results are predictable every single time. The function does not care where progressMap came from, it just reads from it, so passing a hardcoded object works perfectly and keeps the test fast and reliable.

---

## Function 3 - isPhaseUnlocked in lib/utils/progress.ts

### What It Currently Does

Right now, the function always returns true. It does not look at phaseOrder, it does not look at phases, it does not look at progressMap. No matter what you pass in, you get true back every time. The only line that runs is `return true`.

### What It Is Supposed To Do

The commented-out code shows what the real logic was meant to be. If the phase being checked is phase 1, it should return true automatically because the first phase is always available. For any other phase, it should find the phase that comes just before it (the one with order equal to phaseOrder minus 1). If that previous phase exists, it checks whether every single task in it is done using isDone. The phase is only unlocked if all tasks in the previous phase are complete. If no previous phase is found, it returns true as a fallback.

### Why This Is a Bug

This is exactly the bug Kristi found in Phase 1. The phase gate does not work. Because the function always returns true, users can jump to any phase even if they have not finished the one before it. The locking mechanism exists in the code but it is turned off. Any user can skip phase 1, go straight to phase 3, and the system lets them through. That is what Kristi reported, and this is the line that causes it.

### What a Unit Test Would Need to Prove

If the real logic were uncommented, a test that would catch this bug would look like this:

- Input: phaseOrder is 2, phases is `[{ order: 1, tasks: [{ id: 1 }] }]`, progressMap is `{ 1: { status: "NOT_STARTED" } }`
- Expected output: false

Phase 2 should be locked because the task in phase 1 is NOT_STARTED, so isDone returns false, and every() returns false. If the function still returns true with this input, the bug is still there. This test would fail against the current broken code and pass only once the real logic is restored.

---

## Data Types - One Finding from progressSchema in lib/validations/progress.ts

### What happens if taskId is sent as the string "5"

Zod does not coerce by default. `z.number()` strictly expects a number type. If the API receives `"5"` as a string, Zod rejects it immediately and returns a validation error. The request does not go through. You would get something like "Expected number, received string" in the error output.

### What happens if taskId is -1

`z.number().int().positive()` chains three checks. positive() means the number must be greater than 0. -1 fails that check. Zod rejects it with a validation error. Zero would also fail because positive() does not include 0.

### What happens if status is sent as "DONE"

`z.enum(["NOT_STARTED", "IN_PROGRESS", "SUBMITTED", "COMPLETED"])` only allows those four exact strings. "DONE" is not in the list. Zod rejects it. The API returns a validation error.

### New Decision Table Entry for Phase 2

| Input Field | Input Value | Type | Expected Result |
|---|---|---|---|
| taskId | "5" (string) | Wrong type | Zod rejects, API returns 400 validation error |
