# Day 6 — Assignment 1
## Steps 1 & 2: Testing Pyramid + Line, Branch, and Path Coverage

---

## Step 1 — The Testing Pyramid

### Categorizing All Test Cases from Phase 1 and Phase 2

**Phase 1 — Bug Trail**

| Test Case | Description | Level |
|---|---|---|
| TC-BB-01 | Valid login with @betaninjas.com email | System |
| TC-BB-02 | Login with non-@betaninjas.com email | System |
| TC-BB-03 | Login with empty email field | System |
| TC-BB-04 | Login with empty password field | System |
| TC-BB-05 | Login with both fields empty | System |
| TC-GB-01 | Auth token created after successful login | Integration |
| TC-GB-02 | Auth token absent after failed login | Integration |
| TC-WB-01 | getDaysRemaining with future date | Unit |
| TC-WB-02 | getDaysRemaining with past date (negative) | Unit |
| TC-WB-03 | getDaysRemaining with today as due date | Unit |

**Phase 2 — Test Plan (Assignment 2)**

| Test Case | Description | Level |
|---|---|---|
| TC-01 | Member submits TEXT task, progress updates | System |
| TC-02 | CHECKBOX task toggle on and off | System |
| TC-03 | TEXT/LINK task revision after submission | System |
| TC-04 | Empty TEXT submission blocked | System |
| TC-05 | Invalid URL for LINK task | System |

**Phase 2 — Design Techniques (Day 4)**

| Test Case | Description | Level |
|---|---|---|
| BVA-01 | Due date one day after start date (min valid boundary) | Integration |
| BVA-02 | Due date same as start date (invalid boundary) | Integration |
| BVA-03 | Due date one day before start date (below min) | Integration |
| BVA-04 | Due date far in future (no upper boundary enforced) | Integration |
| BVA-05 | Single character text submission | Unit |
| BVA-06 | Extremely long text submission (~1.2M chars) | Unit |
| EP-01 | Valid @betaninjas.com email | System |
| EP-02 | Invalid email format (no @) | System |
| EP-03 | Valid URL with https:// prefix | Integration |
| EP-04 | URL without https:// prefix | Integration |
| DT-01 | CHECKBOX task: completed = true, submitted = false | Unit |
| DT-02 | TEXT task: filled = true, submitted = false | Unit |
| DT-03 | LINK task: valid URL, submitted = false | Unit |
| DT-04 | TEXT task: empty field, submitted = false | Unit |
| DT-05 | LINK task: invalid URL, submitted = false | Unit |
| ST-01 | CHECKBOX: Not Started → Completed (check) | Unit |
| ST-02 | CHECKBOX: Completed → Not Started (uncheck) | Unit |
| ST-03 | CHECKBOX: no intermediate In Progress state | Unit |
| ST-04 | TEXT: Not Started → In Progress (type) | Unit |
| ST-05 | TEXT: In Progress → Submitted (submit) | Unit |
| ST-06 | TEXT: Submitted → In Progress (revise) | Unit |
| ST-07 | LINK: Not Started → In Progress (type URL) | Unit |
| ST-08 | LINK: In Progress → Submitted (submit) | Unit |

**Phase 2 — Dashboard Tests (Day 5)**

| Test Case | Description | Level |
|---|---|---|
| TC-S1-01 | Progress increases after CHECKBOX completion | System |
| TC-S1-02 | Progress decreases after undo | System |
| TC-S1-03 | Phase card shows correct task count | System |
| TC-S2-01 | Phase card navigation to correct phase | System |
| TC-S2-02 | Dashboard loads without errors | Acceptance |
| TC-S2-03 | Dashboard inaccessible without login | System |

---

### Count by Level

| Level | Count |
|---|---|
| Unit | 18 |
| Integration | 7 |
| System | 19 |
| Acceptance | 1 |
| **Total** | **45** |

---

### Shape Analysis

```
Acceptance  [1]       ▓
System     [19]       ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
Integration [7]       ▓▓▓▓▓▓▓
Unit       [18]       ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
```

The shape is roughly a **rectangle**, not a pyramid. Unit and System are nearly equal in count. Integration is under-represented and Acceptance is almost absent. This is a warning sign — the suite has too many high-level system tests relative to the unit foundation.

---

### Ideal Pyramid for BetaninjasSEOLMS

```
         /\
        /  \     Acceptance (~5%)
       /    \    End-to-end flows: full login → submit task → see progress
      /------\
     /        \  Integration (~20%)
    /          \ Auth token flow, API calls, DB state after submit, phase unlock logic
   /------------\
  /              \ Unit (~75%)
 /                \ getDaysRemaining, getPhaseStatus, validation logic,
/                  \ task state transitions, URL format checks
```

**What is currently missing:**
- **Unit level**: No tests for `getPhaseStatus`, no tests for the phase unlock guard (the bug Kristi found), no tests for URL validation logic in isolation
- **Integration level**: No tests that verify the database state after a task submission, no tests for the auth token lifecycle end-to-end
- **Acceptance level**: No complete user journey tests (login → navigate to phase → complete a task → see dashboard update)

---

## Step 2 — Line, Branch, and Path Coverage

### The Function

```typescript
export function getDaysRemaining(dueDate: Date, now: Date = new Date()): number {
  const diff = dueDate.getTime() - now.getTime()           // line 1
  return Math.ceil(diff / (1000 * 60 * 60 * 24))           // line 2
}
```

This function is 2 executable lines (the function signature is not executable logic).

---

### Line Coverage

If a single test calls `getDaysRemaining(someFutureDate)`:

| Line | Executed? |
|---|---|
| `const diff = dueDate.getTime() - now.getTime()` | Yes |
| `return Math.ceil(diff / ...)` | Yes |

**Result: 100% line coverage with one test.**

But this tells us almost nothing. Both lines always execute no matter what input is passed. Line coverage cannot distinguish between a future date, a past date, or today. A single call executes every line — the function has no branching.

---

### Branch Coverage

Looking carefully at `getDaysRemaining` — there are **zero explicit if/else branches** in the function body.

However, the function signature has a **default parameter**: `now: Date = new Date()`. This creates one implicit branch:
- Branch A: caller provides `now` explicitly
- Branch B: caller omits `now`, so the default `new Date()` is used

**Tests needed for full branch coverage: 2**
1. Call with only `dueDate` provided (uses default `now`)
2. Call with both `dueDate` and `now` provided explicitly

Branch coverage is more meaningful than line coverage here because it confirms the default parameter path is tested — but it still does not catch whether the return value is correct or meaningful.

---

### Path Coverage

A path is a unique route from function entry to function exit, including all combinations of branches.

For `getDaysRemaining`:
- There is only one route through the function body (no conditionals)
- The default parameter branch adds 2 paths: with `now` / without `now`

**Tests needed for full path coverage: 2** — same as branch coverage in this case.

However, **path coverage thinking forces you to consider the semantic paths**:
1. `dueDate` is in the future → positive result
2. `dueDate` is today → result is 0 or rounds up to 1
3. `dueDate` is in the past → negative result

For a more complex function with nested conditions, path coverage grows exponentially. A function with 3 independent if/else conditions has 2³ = 8 paths. Branch coverage only needs 6 tests (2 per branch). Path coverage needs all 8 combinations. That is why path coverage costs more — it scales with complexity.

---

### Difference in My Own Words (Using getDaysRemaining as the Example)

**Line coverage** just asks: did this line run? For `getDaysRemaining`, one test with any date achieves 100% line coverage. But it says nothing about whether the result is correct for a past date, a future date, or today.

**Branch coverage** asks: did every fork in the road get taken at least once? For `getDaysRemaining`, the only fork is the default parameter. Two tests cover it. Still does not verify correctness of the math.

**Path coverage** asks: did every complete route from start to finish get walked? For a simple function like this, path coverage equals branch coverage. But for a function with nested conditions — like `getPhaseStatus` which checks if a phase is locked, unlocked, or complete — path coverage requires you to test every combination of conditions, not just each condition in isolation. That is what makes it thorough and expensive.

The core lesson: `getDaysRemaining` achieves 100% line coverage trivially. It does not mean the function is well-tested. It means the code ran. Coverage measures execution, not correctness.
