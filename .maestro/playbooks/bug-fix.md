# Playbook: Bug Fix

Use this playbook when fixing a reported defect. The goal is to fix the root cause, not the symptom — and not to introduce new problems in the process.

---

## Phase 1: Intake and Reproduction (Maestro + User)

1. **Document the bug completely.**
   - What is the observed behavior?
   - What is the expected behavior?
   - How is it triggered? (Steps to reproduce)
   - How often does it occur? (Always, intermittent, only under specific conditions?)
   - Environment/version where it was observed

2. **Reproduce the bug** (or document the reproduction conditions precisely).
   - If you can't reproduce it, say so explicitly — a fix for an unreproducible bug is a guess.

3. **Assess severity.**
   - Does this affect critical user flows or data integrity?
   - What's the workaround, if any?

---

## Phase 2: Root Cause Analysis (Architect)

1. **Don't guess.** Read the code. Trace the execution path.
2. **Find the root cause.** The proximate cause (where the failure surfaces) is not the root cause. Find where the wrong decision was made.
3. **Check for related instances.** Is this same bug pattern present elsewhere in the codebase? A systematic search is worth doing.
4. **Document the root cause** in the design handoff before proposing a fix.
5. **Design a minimal fix.** The fix should be as small as possible while fully addressing the root cause.
6. **Consider side effects.** What does the fix change? What other behavior might it affect?
7. **Design a regression test.** The fix is incomplete without a test that would have caught the original bug.

---

## Phase 3: Implementation (Coder)

1. **Implement the fix as designed.** No opportunistic refactoring. This is not the time.
2. **Minimal blast radius.** Change only what's needed to fix the bug.
3. **Comment the fix.** Add a comment at the fix site explaining what was wrong and why this fixes it.

   Example:
   ```
   // BUG FIX [Task 007]: Comparison was using signed arithmetic, causing negative
   // values to pass the validation check. Use absolute comparison instead.
   ```

4. **Write the regression test** at this stage — don't leave it for the tester to invent.

---

## Phase 4: Review (Reviewer)

In addition to the standard review checklist, check:

- [ ] **Root cause addressed** — Does the fix address the root cause, not just the symptom?
- [ ] **No new issues** — Does the fix introduce new edge cases or failure modes?
- [ ] **Minimal change** — Is anything changed that didn't need to be?
- [ ] **Regression test exists** — Is there a test that would have caught this bug?
- [ ] **Same pattern elsewhere** — Has the team checked for the same bug pattern in other places?

---

## Phase 5: Testing (Tester)

1. **Reproduce the original bug** with the fix applied — it should not trigger.
2. **Run the regression test.**
3. **Test the specific conditions** that triggered the bug (edge cases, timing, specific inputs).
4. **Test adjacent behavior** — make sure the fix didn't break anything nearby.
5. **If the bug was intermittent** — run the fix under stress conditions to increase confidence.

---

## Documentation

After the fix is merged, document:
- **What was wrong** — Root cause, not just symptoms
- **Why it was wrong** — Coding error, design flaw, incorrect assumption?
- **How it was fixed** — What changed
- **How to verify** — Regression test reference

This goes in the handoff doc and should be preserved in the commit message.

---

## Commit Message Format

```
fix(module): short description of what was fixed

Root cause: [one sentence]
Fix: [one sentence]
Regression test: [TC-NNN or test file reference]

Task: #NNN
```
