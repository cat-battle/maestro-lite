# Playbook: Code Review

Use this checklist when reviewing any code change. Each section is a gate — don't mark a section clean unless you've actually checked it.

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

### Error Handling
- [ ] All return values and errors are checked
- [ ] All error paths clean up resources (open handles, acquired locks, allocated memory, transactions)
- [ ] Errors are propagated to callers or handled definitively — not silently swallowed
- [ ] Failure modes from dependencies are handled

### Security
- [ ] Input validated at trust boundaries
- [ ] No injection vulnerabilities (SQL, command, path traversal, template injection) where applicable
- [ ] No sensitive data logged, returned in error messages, or stored inappropriately
- [ ] Authentication and authorization enforced where required
- [ ] Secrets/credentials not hardcoded

### Performance
- [ ] No unnecessary copies of large data structures
- [ ] Algorithmic complexity is appropriate for expected input size
- [ ] No unintended blocking in latency-sensitive paths
- [ ] No N+1 query patterns or unnecessary repeated work

### Code Quality
- [ ] No magic numbers — all constants have names
- [ ] Naming is clear and consistent with the rest of the codebase
- [ ] Comments explain *why*, not just *what*
- [ ] No dead code, commented-out code, or TODO left behind without a tracking issue
- [ ] Functions and classes have clear, single responsibilities
- [ ] Code is readable without deep context

### Design Conformance
- [ ] Implementation matches design doc
- [ ] No scope creep — only the agreed changes are present
- [ ] New abstractions are justified and consistent with existing patterns

### Tests
- [ ] New code has test coverage
- [ ] Tests cover edge cases and error paths, not just the happy path
- [ ] Tests are deterministic and isolated
- [ ] No test logic that always passes or never actually asserts

### Dependencies
- [ ] No new dependencies without justification
- [ ] New dependencies are well-maintained and license-compatible
- [ ] Build/package files updated correctly if new files or dependencies were added

---

## Writing Findings

For every issue found:
- **Severity:** critical / major / minor / nit
- **Location:** file.ext:line
- **Description:** What's wrong
- **Impact:** What breaks if this isn't fixed
- **Suggestion:** What to do instead

## Verdict

| Verdict | When to use |
|---------|-------------|
| **PASS** | No required changes |
| **CHANGES_REQUESTED** | Major or minor issues that must be addressed |
| **BLOCKER** | Critical issues; work stops until resolved |
