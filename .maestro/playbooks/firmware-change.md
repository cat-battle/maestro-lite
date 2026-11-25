# Playbook: Firmware Change

Use this playbook for any change to production firmware code. Steps apply to all agents; each agent executes the steps in their phase.

---

## Phase 1: Intake (Maestro)

1. **Understand the change request.** What exactly needs to change? What's the hardware target?
2. **Identify affected subsystems.** Which modules, peripherals, and drivers are involved?
3. **Check for dependencies.** Does this touch shared resources? ISRs? Power management sequences?
4. **Assess blast radius.** What breaks if this change is wrong? What else uses the affected code?
5. **Define acceptance criteria.** What does "done" look like? What observable behavior changes?
6. **Decompose into subtasks.** At minimum: design → implement → review → test.

---

## Phase 2: Design (Architect)

1. **Read all existing code in scope.** Don't design against assumptions — read the actual code.
2. **Check hardware documentation.** For any peripheral or register changes, review the datasheet.
3. **Identify ISR interactions.** Which code paths run in interrupt context? What data is shared?
4. **Map shared resource access.** Are there critical sections needed? Existing ones to extend?
5. **Design with rollback in mind.** If this change needs to be reverted, what does that look like?
6. **Document register and protocol changes.** Hardware team needs to know about these.
7. **Specify memory impact.** Stack depth changes, new static buffers, any heap usage.
8. **Write the design doc** — see architect instructions for required sections.

---

## Phase 3: Implementation (Coder)

1. **Read design doc completely** before writing any code.
2. **Apply const-correctness throughout.** If it won't be modified, it's `const`.
3. **Use explicit integer types.** `uint8_t`, `int32_t`, etc. — never bare `int` for hardware-facing code.
4. **Mark volatile correctly.** ISR-shared variables must be `volatile`. Register pointers must be `volatile`.
5. **Protect shared state.** Any variable accessed from both task and ISR context needs protection.
6. **No dynamic allocation** unless design explicitly authorizes it.
7. **Handle all error paths.** Every hardware operation can fail. Handle it.
8. **No magic numbers.** All constants named.
9. **Comment hardware-specific behavior.** Why a specific register value? Why a specific timeout? Say it.
10. **Build clean.** `-Wall -Werror`. No surprises.

---

## Phase 4: Code Review (Reviewer)

Review against this firmware-specific checklist:

- [ ] **const-correctness** — All non-modified pointers and params are const
- [ ] **No dynamic allocation** — Unless justified and approved
- [ ] **ISR safety** — Shared variables volatile, access protected
- [ ] **Race conditions** — No TOCTOU, multi-step shared-state ops are atomic
- [ ] **Buffer overflows** — All array accesses bounds-checked
- [ ] **Integer overflow** — Arithmetic on small types won't overflow silently
- [ ] **Register access** — volatile pointers, safe read-modify-write
- [ ] **Watchdog** — New code doesn't starve watchdog, kicks correctly if long-running
- [ ] **Power/reset sequences** — Change doesn't break startup, shutdown, or reset handling
- [ ] **Error handling** — All return values checked, resources cleaned up on error
- [ ] **Timing** — No new busy-wait loops, timing assumptions documented
- [ ] **Build** — Compiles with `-Wall -Werror`, no new warnings

---

## Phase 5: Testing (Tester)

1. **Unit tests for new logic.** Mock hardware dependencies where possible.
2. **Integration test with affected peripherals.** Real hardware if unit tests can't cover it.
3. **Power cycle test.** Does the system initialize correctly after cold reset?
4. **Error injection.** What if the hardware returns an error? Timeout? Bad data?
5. **Boundary conditions.** Min/max values, empty buffers, full buffers.
6. **Concurrent access.** What if the ISR fires mid-operation?
7. **Timing edge cases.** Exactly at the timeout boundary. Timer fires during initialization.
8. **Regression.** Does existing functionality still work?

---

## Sign-Off Checklist (Maestro)

Before closing the task:
- [ ] Design reviewed and approved by user
- [ ] Implementation matches design
- [ ] All acceptance criteria covered by tests
- [ ] Review passed (no BLOCKER or CHANGES_REQUESTED outstanding)
- [ ] Hardware team notified of register/protocol changes (if any)
- [ ] Documentation updated
