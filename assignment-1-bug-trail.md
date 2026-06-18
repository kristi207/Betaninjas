cat > assignment-1-bug-trail.md << 'EOF'
# Assignment 1 — The Bug Trail

## The Bug

**What:** Phase 2 can be accessed without completing Phase 1. The app is supposed to gate phases sequentially, but the restriction is not enforced — a member can jump to any phase regardless of their progress.

**How to reproduce:**
1. Log in as a member
2. Go to the Dashboard
3. Click on Phase 2 without completing any tasks in Phase 1
4. Phase 2 opens successfully — no restriction shown

---

## Point 1 — Which Principle

This is the **absence-of-errors fallacy**. The team tested that a member can open a phase and navigate through tasks — and it worked. But nobody tested whether the gating between phases was actually enforced. Confirming that phases open is not the same as confirming that they open only when they should. The bug existed the whole time, hidden behind tests that never challenged the rule.

---

## Point 2 — Which SDLC Stage Could Have Caught It

**Design stage.** When the phase progression was being designed, QA should have asked: "what prevents a member from skipping ahead — is this enforced in the UI, the backend, or both?" That question would have surfaced the gap before development started. At minimum it should have been caught in the testing phase with a specific test case: attempt to open Phase 2 with zero Phase 1 tasks completed and verify it is blocked.

---

## Point 3 — Static or Dynamic

**Dynamic only.** Reading the requirements might have told you that phases are sequential, but you would never know the gating wasn't enforced until you actually clicked Phase 2 without finishing Phase 1. No document review catches a backend that simply doesn't match the expected behavior. You had to run the app and try it yourself.
EOF