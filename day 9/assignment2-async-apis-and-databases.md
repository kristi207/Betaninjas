# Day 9 — Assignment 2
## Trace One Request End to End

---

## Step 1 — The Async Chain

The POST handler in `app/api/progress/route.ts` has 3 await calls, in this order:

**1. `const session = await auth()`**
- What is being awaited: the NextAuth `auth()` function, which reads the session cookie and validates it
- What it returns: a session object containing the user's id, name, and email — or `null` if no valid session exists
- What happens if it fails: the code checks `if (!session?.user?.id)` immediately after and returns a 401 Unauthorized response. This is handled correctly.

**2. `const body = await request.json()`**
- What is being awaited: parsing the incoming HTTP request body as JSON
- What it returns: a plain JavaScript object containing whatever the browser sent in the request body
- What happens if it fails: if the body is not valid JSON, `request.json()` throws an error. There is no try/catch around this call — the error propagates up and Next.js returns an unstructured 500 response. Not handled.

**3. `const progress = await prisma.progress.upsert(...)`**
- What is being awaited: a Prisma database operation that either creates a new Progress row or updates the existing one for this user and task
- What it returns: the full Progress record that was created or updated, matching the shape of the Progress model in the schema
- What happens if it fails: if the database is unavailable or the query fails, Prisma throws an error. There is no try/catch around this call — the error propagates up and crashes the handler with a 500. Not handled.

---

## Step 2 — The Request

### What the browser sends (POST request)

**URL:** `/api/progress`

**Method:** `POST`

**Body (JSON):**
| Field | Type | Description |
|---|---|---|
| `taskId` | number | The id of the task being submitted |
| `status` | string | The new status — `"SUBMITTED"` for TEXT and LINK tasks, `"COMPLETED"` for CHECKBOX tasks |
| `response` | string (optional) | The member's typed response — only present for TEXT tasks |
| `link` | string (optional) | The submitted URL — only present for LINK tasks |

**Auth:** The member's session is proved by a cookie that NextAuth sets at login time. The browser sends this cookie automatically with every request to the same domain. The server reads it by calling `auth()` — the cookie is not in the request body, it travels in the HTTP headers.

### What the server sends back on success

**Status code:** `200 OK`

**Response body (JSON):** The full Progress record that was written to the database — including `id`, `userId`, `taskId`, `status`, `response`, `link`, `completedAt`, and `updatedAt`.

---

## Step 3 — The Database

### Progress model — every field

| Field | Type | Required? | Notes |
|---|---|---|---|
| `id` | Int | Required | Auto-incremented primary key |
| `userId` | String | Required | Foreign key to User.id |
| `user` | User (relation) | Required | The member this progress belongs to |
| `taskId` | Int | Required | Foreign key to Task.id |
| `task` | Task (relation) | Required | The task this progress belongs to |
| `status` | TaskStatus enum | Required | Defaults to `NOT_STARTED`. Values: `NOT_STARTED`, `IN_PROGRESS`, `SUBMITTED`, `COMPLETED` |
| `response` | String | Optional | The member's typed text — only set for TEXT tasks |
| `link` | String | Optional | The submitted URL — only set for LINK tasks |
| `completedAt` | DateTime | Optional | Timestamp of completion — see below |
| `updatedAt` | DateTime | Required | Auto-updated by Prisma every time the row changes |

### The unique constraint — @@unique([userId, taskId])

In plain English: there can only ever be one Progress row for a given member and a given task. If a member has already started Task 3, the database will not allow a second row to be inserted for that same member and task. The `upsert` in the API uses this constraint deliberately — instead of inserting a new row every time a member submits, it finds the existing row and updates it in place. This is what makes revision possible: submitting again overwrites the previous submission rather than creating a duplicate.

### completedAt — what it stores and what it means

`completedAt` is a nullable timestamp. It is set to the current time (`new Date()`) only when the status being written is `"COMPLETED"` — which happens for CHECKBOX tasks when a member checks them. It is set to `null` when the status is `"NOT_STARTED"` — which happens when a member unchecks a CHECKBOX. For TEXT and LINK tasks, which use the `"SUBMITTED"` status, `completedAt` is never set by the current upsert logic.

For progress calculation, `completedAt` signals that a CHECKBOX task is definitively done. TEXT and LINK tasks are counted as done by checking whether their status is `SUBMITTED` or `COMPLETED`.

### Relation diagram

```
Progress ──── Task ──── Phase
   │            │
   │            └── type (CHECKBOX / TEXT / LINK)
   │            └── phaseId
   │
   └── userId (→ User)
   └── taskId (→ Task)
```

A Progress record belongs to one User and one Task. A Task belongs to one Phase. To find all progress for a member in a specific phase, you go: User → Progress records → filter by Task.phaseId.

---

## Step 4 — What the UI Shows vs What the Database Stores

**1. The UI shows "3 of 8 tasks". Is this stored or calculated?**

Calculated every time the page loads. The database stores individual Progress rows with a status per task. When the page renders, the code counts how many of those rows have a status of `SUBMITTED` or `COMPLETED` for that user in that phase. The number "3 of 8" is never written to any column — it is computed fresh from the raw rows on every request.

**2. A member submits a TEXT task. The UI shows "Submitted." What is stored?**

The database stores the string `"SUBMITTED"` in the `status` column of the Progress row. The word "Submitted" shown in the UI is a display label — the stored value is the enum value `SUBMITTED`.

**3. A member checks a CHECKBOX task. The UI shows a green tick. What is stored?**

The database stores `"COMPLETED"` in the `status` column. Additionally, the `completedAt` field is written with the current timestamp (`new Date()`). Two fields change: `status` and `completedAt`. The green tick in the UI is derived by checking whether status equals `COMPLETED` — neither the tick colour nor the visual state is stored anywhere.

**4. The UI shows a phase card with status "ON TRACK". Is this stored?**

No. `"ON TRACK"` is never written to the database. It is a calculated value produced by the `getPhaseStatus()` function in `lib/utils/timeline.ts`. That function takes the phase's `startDate`, `dueDate`, and the member's current completion percentage, and returns a status string at render time. Every time the page loads, it recalculates from those inputs. The database stores the raw dates (in `PhaseTimeline`) and the individual task statuses (in `Progress`) — the timeline status label is always derived, never persisted.

**5. Is there any difference between what the member typed and what gets stored?**

Yes — for TEXT tasks, `TextSubmission.tsx` transforms the response before sending it. If `existingNote` is present, the component appends it to the member's text before the fetch:

```
const fullResponse = existingNote
  ? `${response.trim()}\n[NOTE]: ${existingNote}`
  : response.trim()
```

What gets stored in the `response` column includes the appended `[NOTE]: ...` suffix that the member never typed. The stored value can differ from what the member wrote. For members without a note, the only transformation is `.trim()` — leading and trailing whitespace is removed before storage.
