# Playbook: Code Review

Use this checklist for reviewing firmware and embedded C/C++ code. Each section is a gate — don't mark a section clean unless you've actually checked it.

---

## Setup

Before reviewing, read:
1. The task spec (what was asked)
2. The design doc (what was planned)
3. The implementation handoff (what was built and any deviations)
4. The actual diff/changed files

---

## Checklist

### Correctness
- [ ] Does the code do what the spec requires?
- [ ] Does the implementation match the design? (Flag any deviations — they may be justified, but should be explicit)
- [ ] Are all acceptance criteria met?
- [ ] Are all edge cases handled?
- [ ] Is logic correct — no off-by-one, no inverted conditions?

### Memory
- [ ] No memory leaks (all allocations have a matching free on every code path)
- [ ] No use-after-free
- [ ] No unbounded allocation (stack or heap)
- [ ] Stack usage is reasonable for the target (large local arrays are a red flag)
- [ ] `malloc`/`new` usage is explicitly justified if present

### Concurrency and ISR Safety
- [ ] Variables shared between ISR and task context are `volatile`
- [ ] Multi-byte shared state reads/writes are atomic or protected
- [ ] No blocking calls in ISR context (no `printf`, no `malloc`, no mutex lock)
- [ ] No TOCTOU (check-then-act on shared state without holding a lock)
- [ ] Critical sections are minimal and don't block for long
- [ ] No priority inversion paths introduced

### Bounds and Overflow
- [ ] All array accesses are bounds-checked
- [ ] No unbounded string operations (`strcpy`, `sprintf` → use safe variants or manual bounds)
- [ ] Integer arithmetic won't overflow silently (especially mixing signed/unsigned, or small types)
- [ ] All pointer dereferences are validated before use
- [ ] Bitfield operations don't produce unintended results

### Error Handling
- [ ] All return values are checked (especially hardware API calls)
- [ ] Error paths clean up resources (open handles, acquired locks, allocated memory)
- [ ] Errors are propagated to callers or handled definitively — not silently swallowed
- [ ] Hardware error conditions (timeout, NAK, NACK, busy) are handled

### Firmware-Specific
- [ ] `volatile` applied correctly to hardware register pointers and ISR-shared variables
- [ ] Read-modify-write on hardware registers is safe (no race with ISR or hardware)
- [ ] Watchdog is handled for any long-running operations
- [ ] Power/reset sequences are not disrupted
- [ ] Timing assumptions are documented in comments
- [ ] No busy-wait loops that could starve other tasks or the watchdog

### Code Quality
- [ ] `const`-correct (non-modified params and pointers are `const`)
- [ ] Explicit integer types (`uint8_t`, `int32_t`, etc.)
- [ ] No magic numbers (all constants have names)
- [ ] Naming is clear and consistent with the rest of the codebase
- [ ] Comments explain *why*, not just *what*
- [ ] Hardware-specific behavior is commented (register values, timing, protocol quirks)
- [ ] No dead code, commented-out code, or TODO left behind without a tracking issue

### Build
- [ ] Compiles clean with `-Wall -Werror`
- [ ] No new warnings added
- [ ] No new external dependencies without justification
- [ ] Build system (Makefile/CMake/etc.) updated if new files were added

### Performance
- [ ] No unnecessary copies of large data structures
- [ ] Algorithmic complexity is appropriate (O(1) or O(n) is fine; O(n²) needs justification in embedded context)
- [ ] No unintended cache-hostile patterns (if cache is relevant on target)

---

## Writing Findings

For every issue found:
- **Severity:** critical / major / minor / nit
- **Location:** file.c:line
- **Description:** What's wrong
- **Impact:** What breaks if this isn't fixed
- **Suggestion:** What to do instead

## Verdict

| Verdict | When to use |
|---------|-------------|
| **PASS** | No required changes |
| **CHANGES_REQUESTED** | Major or minor issues that must be addressed |
| **BLOCKER** | Critical issues; work stops until resolved |
