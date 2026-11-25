# Custom Playbook Example: Firmware / Embedded C/C++

> This file lives in `examples/custom-playbooks/` as a **starting point**.
> Copy it to `.maestro/playbooks/firmware-change.md` in your project and customize it.

---

# Playbook: Firmware Change

> **Purpose:** Use this playbook for any change to production firmware code on embedded/real-time systems.
>
> **Domain:** Firmware / Embedded C/C++

---

## Overview

Firmware changes carry risks not present in typical software: hardware can lie, timing constraints are hard, and bugs can cause physical damage or safety failures. This playbook adds firmware-specific checkpoints to the standard workflow.

---

## Phase 1: Intake (Maestro)

Additional intake steps for firmware:

1. **Identify the hardware target.** Which MCU, board, and peripheral set?
2. **Check for dependencies.** Does this touch shared resources? ISRs? Power management sequences?
3. **Assess blast radius.** What breaks if this change is wrong? What else uses the affected code?
4. **Capture in task spec:** hardware dependencies, ISR interactions, timing requirements, memory budget.

---

## Phase 2: Design (Architect)

Firmware-specific design steps:

1. **Read hardware documentation.** For any peripheral or register changes, review the datasheet.
2. **Identify ISR interactions.** Which code paths run in interrupt context? What data is shared?
3. **Map shared resource access.** Are there critical sections needed? Existing ones to extend?
4. **Specify memory impact.** Stack depth changes, new static buffers, any heap usage.
5. **Design with rollback in mind.** If this change needs to be reverted, what's the blast radius?

Additional design doc sections for firmware:
- **Memory:** Stack depth, static allocations, buffer sizes
- **Timing:** Any hard deadlines, polling intervals, timeout values
- **ISR Safety:** Which code runs in interrupt context? What data is shared between ISR and task context?
- **Peripheral Ownership:** Who initializes what? Any shared peripherals?

---

## Phase 3: Implementation (Coder)

Firmware coding standards:

1. **const-correct** — If a pointer or parameter won't be modified, it's `const`.
2. **Static where appropriate** — File-local functions and variables are `static`.
3. **Explicit types** — Use `uint8_t`, `int32_t`, etc. from `<stdint.h>`. No `int` for hardware registers.
4. **Mark volatile correctly** — ISR-shared variables must be `volatile`. Register pointers must be `volatile`.
5. **Protect shared state** — Any variable accessed from both task and ISR context needs protection.
6. **No dynamic allocation** unless design explicitly authorizes it.
7. **Handle all error paths** — Every hardware operation can fail.
8. **Build clean** — `-Wall -Werror`. No surprises.

---

## Phase 4: Code Review (Reviewer)

Firmware-specific review checklist:

- [ ] **const-correctness** — All non-modified pointers and params are const
- [ ] **No dynamic allocation** — Unless justified and approved
- [ ] **ISR safety** — Shared variables volatile, access protected
- [ ] **Race conditions** — No TOCTOU, multi-step shared-state ops are atomic
- [ ] **Buffer overflows** — All array accesses bounds-checked
- [ ] **Integer overflow** — Arithmetic on small types won't overflow silently
- [ ] **Register access** — volatile pointers, safe read-modify-write
- [ ] **Watchdog** — New code doesn't starve watchdog, kicks correctly if long-running
- [ ] **Power/reset sequences** — Change doesn't break startup, shutdown, or reset handling
- [ ] **Timing** — No new busy-wait loops, timing assumptions documented
- [ ] **Build** — Compiles with `-Wall -Werror`, no new warnings

---

## Phase 5: Testing (Tester)

Firmware-specific test categories:

1. **Unit tests for new logic** — Mock hardware dependencies where possible (Unity, CMock, GoogleTest)
2. **Power cycle test** — Does the system initialize correctly after cold reset?
3. **Error injection** — What if the hardware returns an error? Timeout? Bad data?
4. **Concurrent access** — What if the ISR fires mid-operation?
5. **Timing edge cases** — Exactly at the timeout boundary. Timer fires during initialization.
6. **Peripheral quirks** — Peripheral not ready at startup. Peripheral returns unexpected status codes.
7. **Watchdog interaction** — Does code run long enough to need a watchdog kick?

If no test framework is available:
- Write detailed manual test procedures for hardware execution
- Include exact setup instructions, expected observable behavior, and pass/fail criteria
- If the test requires a logic analyzer or oscilloscope, say so explicitly

---

## Sign-Off Checklist (Maestro)

Additional sign-off for firmware:

- [ ] Hardware team notified of register/protocol changes (if any)
- [ ] Memory footprint impact assessed and acceptable
- [ ] Real-hardware validation completed (or hardware-in-loop test)
