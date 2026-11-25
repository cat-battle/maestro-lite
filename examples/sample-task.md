# Example: Add Watchdog Timer Support to Sensor Module

This example shows what a complete Maestro-lite task flow looks like, end to end. All files are shown as they'd actually appear after each phase.

---

## Task Spec: `.maestro/tasks/001-watchdog-sensor.md`

```markdown
# Task 001: Add Watchdog Timer Support to Sensor Module

## Objective
Add watchdog timer kicking to the sensor acquisition loop so that a hung sensor read does not prevent the watchdog from firing and resetting the system.

## Context
The system uses an independent watchdog timer (IWDG) with a 2-second timeout. The sensor module's acquisition loop blocks on I2C reads, which can hang indefinitely if the sensor becomes unresponsive. Currently, the main loop kicks the watchdog, but if the sensor thread blocks, the watchdog is not kicked and the system does not reset as intended.

## Scope
- **Files in scope:**
  - `src/sensor/sensor.c`
  - `src/sensor/sensor.h`
  - `src/watchdog/watchdog.h` (read-only — do not modify)
- **Files out of scope:**
  - `src/watchdog/watchdog.c` (owned by platform team)
  - Any RTOS configuration files

## Acceptance Criteria
- [ ] Sensor acquisition loop kicks the watchdog on every successful iteration
- [ ] If I2C read exceeds 500ms timeout, acquisition returns an error (does not block indefinitely)
- [ ] Watchdog is NOT kicked on a failed/timed-out read (let it fire)
- [ ] No watchdog kicks occur inside ISR context

## Constraints
- Use the existing `watchdog_kick()` API from `watchdog.h` — do not access IWDG registers directly
- I2C timeout must use the existing HAL timeout mechanism, not a software timer
- No dynamic memory allocation

## Firmware Notes
- **Hardware dependencies:** STM32 IWDG, I2C1 peripheral
- **ISR interactions:** I2C completion uses DMA interrupt; sensor task runs at thread priority
- **Timing requirements:** Watchdog timeout is 2000ms; sensor loop target period is 100ms
- **Memory budget:** No new static buffers > 32 bytes

## Priority
High

## Assigned To
architect

## Status
In Progress

## Subtask Tracking
| ID | Description | Assigned | Status |
|----|-------------|----------|--------|
| 001a | Design | architect | Complete |
| 001b | Implementation | coder | Complete |
| 001c | Review | reviewer | Complete |
| 001d | Testing | tester | In Progress |
```

---

## Design Handoff: `.maestro/handoffs/001-design.md`

```markdown
# Handoff: Task 001 — Design

## From
architect

## To
coder

## Summary
The sensor module needs two changes: a timeout on its I2C blocking read, and a watchdog kick on successful reads. The existing HAL_I2C_Master_Receive_IT() pattern already supports timeout via HAL_GetTick() polling; we extend it. The watchdog_kick() API is already exposed; we call it in the right place.

## Overview
The sensor acquisition loop in sensor.c calls a blocking I2C read. If the sensor hangs, this call never returns and the watchdog fires (which is correct behavior). However, the current implementation blocks indefinitely — we want a 500ms timeout so the error can be logged before the reset.

We add:
1. A timeout wrapper around the I2C read
2. A watchdog kick on successful read completion
3. No kick on timeout/error (intentional — let watchdog fire)

## File / Module Layout

**sensor.c** — two changes:
- `sensor_read_raw()`: add 500ms timeout using HAL_GetTick() delta
- `sensor_acquire()`: add `watchdog_kick()` call after successful read

**sensor.h** — no interface changes required

## Interfaces and APIs

Existing (unchanged):
```c
// watchdog.h
void watchdog_kick(void);

// Existing sensor interface (unchanged)
sensor_status_t sensor_acquire(sensor_data_t *data);
```

New internal timeout constant in sensor.c:
```c
#define SENSOR_I2C_TIMEOUT_MS  (500U)
```

Modified internal function signature (no header change needed):
```c
// Before: blocks indefinitely
static sensor_status_t sensor_read_raw(uint8_t *buf, uint16_t len);

// After: returns SENSOR_TIMEOUT if read exceeds SENSOR_I2C_TIMEOUT_MS
static sensor_status_t sensor_read_raw(uint8_t *buf, uint16_t len);
```

## Data Flow

1. `sensor_acquire()` calls `sensor_read_raw()`
2. `sensor_read_raw()` initiates I2C transfer via HAL, polls for completion with timeout
3. On success → returns SENSOR_OK → `sensor_acquire()` calls `watchdog_kick()` → returns data
4. On timeout → returns SENSOR_TIMEOUT → `sensor_acquire()` returns error → NO watchdog kick

## Firmware-Specific Considerations
- **Memory:** `SENSOR_I2C_TIMEOUT_MS` is a compile-time constant. No new buffers.
- **Timing:** 500ms timeout < 2000ms watchdog. A timed-out read will NOT kick watchdog; system resets on next watchdog expiry.
- **ISR Safety:** `watchdog_kick()` and `HAL_GetTick()` are both safe to call from thread context. The I2C completion interrupt only sets a flag; we poll it in the loop.
- **Peripheral Ownership:** No change to I2C ownership.

## Risks and Open Questions
- Confirmed with platform team: `watchdog_kick()` is thread-safe.
- HAL_GetTick() rollover at ~49 days: use subtraction, not comparison, to handle rollover correctly.
```

---

## Implementation Handoff: `.maestro/handoffs/001-impl.md`

```markdown
# Handoff: Task 001 — Implementation

## From
coder

## To
reviewer

## Summary
Implemented per design. Added 500ms I2C timeout in sensor_read_raw() using HAL_GetTick() with rollover-safe arithmetic. Added watchdog_kick() in sensor_acquire() on success path only. No deviation from design.

## Artifacts
- `src/sensor/sensor.c` — Modified sensor_read_raw() and sensor_acquire()
- `src/sensor/sensor.h` — Unchanged (no interface changes needed)

## Key Implementation Notes
- Used `(uint32_t)(HAL_GetTick() - start_tick) >= SENSOR_I2C_TIMEOUT_MS` for rollover-safe timeout check
- watchdog_kick() placed immediately after NULL check on data pointer and before data copy, so it only fires on a good read
- Added SENSOR_TIMEOUT to sensor_status_t enum (was missing; needed for clean error propagation)

## No Deviations from Design

## Build
Compiles clean on STM32G4 target with `-Wall -Werror -O2`.
```

---

## Review Handoff: `.maestro/handoffs/001-review.md`

```markdown
# Review: Task 001 — Watchdog Sensor

**Verdict:** CHANGES_REQUESTED

## Summary
Implementation is correct in approach. One major finding: missing volatile on the DMA completion flag that the timeout loop polls. One minor: timeout constant should be in sensor.h (or a shared config) for testability.

## Findings

### FINDING-001 [Severity: major]
**File:** src/sensor/sensor.c:87
**Description:** `i2c_complete_flag` is set by the DMA interrupt handler but declared without `volatile`. The compiler may optimize the polling loop to never re-read it.
**Impact:** Polling loop may spin forever even after DMA completes, always timing out.
**Suggested Fix:** Declare as `static volatile uint8_t i2c_complete_flag = 0;`

### FINDING-002 [Severity: nit]
**File:** src/sensor/sensor.c:34
**Description:** `SENSOR_I2C_TIMEOUT_MS` defined as plain `#define 500U`. Not testable or tunable.
**Suggested Fix:** Move to sensor.h or a config header; annotate the acceptable range in a comment.
```

---

## Test Plan: `.maestro/handoffs/001-tests.md`

```markdown
# Test Plan: Task 001 — Watchdog Sensor

## Acceptance Criteria Coverage
| Criterion | Test Cases |
|-----------|-----------|
| Watchdog kicked on successful read | TC-001 |
| I2C timeout returns error after 500ms | TC-002, TC-003 |
| Watchdog NOT kicked on failed read | TC-004 |
| No watchdog kick in ISR context | TC-005 (static analysis + review) |

## Test Cases

### TC-001: Watchdog kick on success
**Type:** Unit (with mocks)
**Setup:** Mock HAL_I2C returning success, mock watchdog_kick() with call counter
**Steps:**
1. Call sensor_acquire() with valid data pointer
2. Check watchdog_kick() was called exactly once
**Pass Criteria:** watchdog_kick() call count == 1

### TC-002: I2C timeout triggers after ~500ms
**Type:** Integration (hardware)
**Setup:** Disconnect sensor I2C lines (pull SDA low to simulate hang)
**Steps:**
1. Record timestamp
2. Call sensor_acquire()
3. Record timestamp on return
**Pass Criteria:** Returns SENSOR_TIMEOUT, elapsed time 500ms–600ms

### TC-003: Timeout rollover (HAL_GetTick near UINT32_MAX)
**Type:** Unit
**Setup:** Mock HAL_GetTick() to return UINT32_MAX - 100 at start
**Steps:**
1. Call sensor_read_raw() with mocked tick starting near rollover
2. Have mock advance tick by 600 (rolling over)
**Pass Criteria:** Returns SENSOR_TIMEOUT correctly (not prematurely or never)

### TC-004: No watchdog kick on timeout
**Type:** Unit
**Setup:** Mock HAL_I2C to never complete, mock watchdog_kick() with call counter
**Steps:**
1. Call sensor_acquire(), wait for timeout
2. Check watchdog_kick() call count
**Pass Criteria:** watchdog_kick() call count == 0

### TC-005: ISR context review
**Type:** Static / Code Review
**Steps:** Confirm watchdog_kick() is not called from any interrupt handler in sensor.c
**Pass Criteria:** No call sites in ISR context (confirmed in review)
```

---

## Status Dashboard (Mid-Task)

This is what `.maestro/status.md` looks like partway through:

```markdown
# Maestro Status

## Active Tasks
| ID | Task | Status | Assigned | Phase |
|----|------|--------|----------|-------|
| 001 | Add watchdog to sensor module | In Progress | tester | Testing |

## Completed Tasks
_None yet_

## Notes
- 2026-11-20: Task 001 opened — watchdog kicked into sensor acquisition
- 2026-11-21: Design complete, approved at Design Review gate
- 2026-11-22: Implementation complete, one major finding in review (volatile missing)
- 2026-11-23: Review finding fixed, re-review passed. Testing in progress.
```
