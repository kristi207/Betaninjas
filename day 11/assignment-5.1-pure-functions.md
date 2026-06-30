# Assignment 5.1 — Identify Pure Functions

## Part 1 — All Functions, File, and Line Number

| Function | File | Line |
|---|---|---|
| `getDaysRemaining` | `lib/utils/timeline.ts` | 4 |
| `getPhaseStatus` | `lib/utils/timeline.ts` | 10 |
| `isDone` (internal helper) | `lib/utils/progress.ts` | 7 |
| `calcPhaseProgress` | `lib/utils/progress.ts` | 12 |
| `calcOverallProgress` | `lib/utils/progress.ts` | 19 |
| `isPhaseUnlocked` | `lib/utils/progress.ts` | 27 |

---

## Part 2 — Pure or Not Pure

| Function | Pure? | Reason if Not Pure |
|---|---|---|
| `getDaysRemaining` | **Pure** | Takes two Date inputs, returns a number. No side effects. The `now` default can be overridden so tests never touch the real clock. |
| `getPhaseStatus` | **Pure** | Calls `getDaysRemaining` (also pure) and maps the result to a string. No external dependencies. |
| `isDone` | **Pure** | Reads a value from the map argument passed in — no external state. |
| `calcPhaseProgress` | **Pure** | Receives its data as arguments (tasks array + progressMap), performs math, returns a number. |
| `calcOverallProgress` | **Pure** | Delegates to `calcPhaseProgress` with the same arguments — no external reads or writes. |
| `isPhaseUnlocked` | **Pure (but broken)** | The real unlock logic is commented out; the function currently just returns `true`. Once the logic is restored it will still be pure — it only reads from `phases` and `progressMap`, both passed as arguments. |

No function in either file is impure. They all receive their data through parameters and return a value without touching a database, network, file system, or any shared mutable state.

---

## Part 3 — 3 AAA Tests for the Easiest Pure Function

I chose `getDaysRemaining` — it takes exactly two Date inputs and returns one number, which makes each test trivially small to arrange.

---

### Test 1 — "returns 5 when the due date is 5 days from now"

**ARRANGE**
dueDate = new Date("2026-07-04")
now     = new Date("2026-06-29")

**ACT**
result = getDaysRemaining(dueDate, now)

**ASSERT**
result === 5

---

### Test 2 — "returns 0 when the due date is today"

**ARRANGE**
dueDate = new Date("2026-06-29")
now     = new Date("2026-06-29")

**ACT**
result = getDaysRemaining(dueDate, now)

**ASSERT**
result === 0

---

### Test 3 — "returns a negative number when the due date has already passed"

**ARRANGE**
dueDate = new Date("2026-06-20")
now     = new Date("2026-06-29")

**ACT**
result = getDaysRemaining(dueDate, now)

**ASSERT**
result === -9