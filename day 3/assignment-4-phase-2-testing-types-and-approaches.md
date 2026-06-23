# Assignment 1 — Same Feature, Three Lenses
**Target:** Login feature — BetaninjasSEOLMS  
**Date:** June 17, 2026

---

## Step 1 — Black Box Testing (5 Test Cases)

Tested purely as a user from the browser — no code knowledge used.

| # | Test Case | Input | Expected | Actual | Result |
|---|-----------|-------|----------|--------|--------|
| 1 | Valid login with betaninjas email | Sign in with `@betaninjas.com` Google account | Login succeeds, redirected to dashboard | Logged in successfully | ✅ Pass |
| 2 | Non-betaninjas email can sign in | Sign in with personal `@gmail.com` Google account | Blocked — "Restricted to Betaninjas only" | Login succeeds, full access granted | ❌ Fail |
| 3 | Cancel Google sign-in popup | Click Sign In, then cancel the Google popup | Returns to login page cleanly | Crashes to server error page at `/api/auth/error?error=Configuration` | ❌ Fail |
| 4 | Sign in on mobile browser | Open app on mobile, tap Sign In with Google | Google OAuth works on mobile | Sign in does not work on mobile browser | ❌ Fail |
| 5 | Navigate to login while already signed in | Log in, then manually go back to `/login` via URL | Redirects to dashboard, login page not shown | Sign out option shown — login page not accessible while logged in | ✅ Pass |

---

## Step 2 — Grey Box Testing (2 Test Cases)

Read `auth.ts` and `middleware.ts` to understand auth internals before writing these.

**Grey Box Test 1 — Non-betaninjas email gets full access**

- **What I saw in the code:** In `auth.ts`, the `signIn` callback only checks `if (!user.email) return false` — there is no domain restriction. Any Google account returns `true` and gets saved to the database as a `"member"`.
- **Test:** Sign in with a personal `@gmail.com` Google account
- **Expected:** Blocked with a message — restricted to betaninjas emails only
- **Actual:** ❌ Fail — any Google account is accepted and created in the database as a member

**Grey Box Test 2 — API routes are completely unprotected**

- **What I saw in the code:** In `middleware.ts`, there is a check `if (isApiRoute)` that skips all authentication logic entirely — unauthenticated users can reach any `/api/` endpoint.
- **Test:** Access an API endpoint (e.g. `/api/users`) directly in the browser without being logged in
- **Expected:** Returns 401 Unauthorized
- **Actual:** API routes are reachable without authentication — no auth check enforced

---

## Step 3 — White Box Testing (getDaysRemaining)

**File:** `lib/utils/timeline.ts`

```ts
export function getDaysRemaining(dueDate: Date, now: Date = new Date()): number {
  const diff = dueDate.getTime() - now.getTime()
  return Math.ceil(diff / (1000 * 60 * 60 * 24))
}
```

**What it takes in:**
- `dueDate` — a `Date` object representing the deadline
- `now` — a `Date` object representing the current time (defaults to `new Date()` if not provided)

**What it returns:**
- A `number` — the days remaining until the due date. Can be negative if the due date has passed.

**3 Edge Cases:**

1. **Due date is today (a few minutes left)**
   - `diff` is a small positive number (e.g. 30 minutes in milliseconds)
   - `Math.ceil(0.02)` = `1` — function returns 1 day remaining
   - User could be misled into thinking they have a full day when the deadline is in 30 minutes

2. **Due date is in the past**
   - `diff` is negative — function returns a negative number (e.g. `-3`)
   - No guard exists — the function does not prevent returning negative values
   - The caller/UI must handle this, otherwise "-3 days remaining" could display

3. **Due date is exactly now (same millisecond)**
   - `diff = 0` → `Math.ceil(0) = 0` — returns `0`
   - This is the only case where `Math.ceil` does not round up
   - Could create confusion — is `0` the same as overdue, or is it still "today"?

---

## Step 4 — Testing Types That Apply to Login

| Testing Type | Why It Matters for Login |
|--------------|--------------------------|
| Functional | Login is a core feature — it must work correctly every time for every user |
| Regression | A future code change (e.g. updating NextAuth) could silently break the login flow |
| Security | Login is the #1 attack surface — vulnerabilities here expose the entire application and all user data |
| Accessibility | Users with disabilities must be able to log in too — screen readers and keyboard navigation must work |
| Usability | If error messages are confusing or the flow is unclear, users cannot recover from mistakes and lose trust |

---

## Step 5 — Reflection

Grey box testing found the most interesting issues. Reading `auth.ts` revealed that the domain restriction was never implemented — something that looked fine from the outside but was completely missing in the code. White box testing was the hardest because it required understanding exactly what the code does line by line, including how `Math.ceil` behaves on edge values like `0` or small decimals. Black box testing felt natural since it is just using the app as a normal user, but it only surfaces what is visibly broken. Grey box gave the best balance — enough internal knowledge to know where to look, without needing to understand every line.
