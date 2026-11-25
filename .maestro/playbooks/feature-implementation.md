# Playbook: Feature Implementation

Use this playbook when implementing a new feature from scratch. Steps apply to all agents; each agent executes the steps in their phase.

---

## Phase 1: Intake (Maestro)

1. **Understand the feature request.** What exactly needs to exist that doesn't today? What problem does it solve?
2. **Identify affected areas.** Which modules, services, or components are involved?
3. **Check for dependencies.** Does this require changes to shared interfaces, data models, or infrastructure?
4. **Assess blast radius.** What existing behavior could this change affect?
5. **Define acceptance criteria.** What does "done" look like? What observable behavior will change?
6. **Decompose into subtasks.** At minimum: design → implement → review → test.

---

## Phase 2: Design (Architect)

1. **Read all existing code in scope.** Don't design against assumptions — read the actual code.
2. **Identify integration points.** Where does this feature connect to existing systems?
3. **Design the interface first.** Define the public API/interface before the implementation details.
4. **Consider backwards compatibility.** Does this change break any existing callers?
5. **Map data flow.** Trace the path from input to output, including error paths.
6. **Document dependencies.** What libraries, services, or infrastructure does this require?
7. **Write the design doc** — see architect instructions for required sections.

---

## Phase 3: Implementation (Coder)

1. **Read design doc completely** before writing any code.
2. **Implement the interface first.** Start with the contract, then fill in the body.
3. **Handle all error paths.** Every external call can fail. Handle it.
4. **No magic numbers.** All constants named.
5. **Write self-documenting code.** Clear names, small functions, single responsibility.
6. **Comment non-obvious decisions.** Why this algorithm? Why this limit? Say it.

---

## Phase 4: Code Review (Reviewer)

Review against the standard checklist plus:

- [ ] **Interface correctness** — Does the implementation match the designed API?
- [ ] **Error handling completeness** — All failure modes handled and propagated correctly
- [ ] **Input validation** — All trust boundaries validated
- [ ] **No scope creep** — Only the designed feature is present
- [ ] **Tests are meaningful** — Not just happy-path coverage

---

## Phase 5: Testing (Tester)

1. **Test every acceptance criterion.** No criterion without a test.
2. **Test integration points.** Does this feature work correctly with the systems it connects to?
3. **Test error scenarios.** What happens when dependencies fail?
4. **Test boundary conditions.** Min/max values, empty inputs, single elements.
5. **Regression test.** Does existing functionality still work?

---

## Sign-Off Checklist (Maestro)

Before closing the task:
- [ ] Design reviewed and approved by user
- [ ] Implementation matches design
- [ ] All acceptance criteria covered by tests
- [ ] Review passed (no BLOCKER or CHANGES_REQUESTED outstanding)
- [ ] Documentation updated (README, API docs, changelog as applicable)
