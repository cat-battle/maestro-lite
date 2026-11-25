# Copilot Instructions — Maestro-lite

This repository uses **Maestro-lite**, a file-based multi-agent orchestration system for structured software development with VS Code Copilot Chat.

---

## Before You Do Anything

Read `.maestro/status.md` to understand the current state of the project. Active tasks, current phase, and assigned agents are all tracked there. Never assume a clean slate.

---

## How the System Works

Work flows through specialized agents:
- **@maestro** — Orchestrator. Accepts tasks, decomposes them, assigns work, tracks status.
- **@architect** — Design. Analyzes code, produces technical designs before any implementation.
- **@coder** — Implementation. Writes code strictly per the architect's design.
- **@reviewer** — Code review. Checks correctness, firmware safety, and design conformance.
- **@tester** — Testing. Writes test plans and test code against acceptance criteria.

Agents communicate through files in `.maestro/`. They do not share context directly — only through documents.

---

## File Protocol

| Directory | Contents |
|-----------|----------|
| `.maestro/status.md` | Live dashboard — read this first |
| `.maestro/tasks/` | Task specs (NNN-task-name.md) |
| `.maestro/handoffs/` | Agent-to-agent handoff documents |
| `.maestro/playbooks/` | Standard workflows for common task types |

### Task IDs
Use zero-padded three-digit numbers: `001`, `002`, `003`. Never reuse an ID.

### Handoff Files
Each phase produces a handoff document:
- `NNN-design.md` — Architect's design
- `NNN-impl.md` — Coder's implementation summary
- `NNN-review.md` — Reviewer's findings
- `NNN-tests.md` — Tester's test plan
- `NNN-questions.md` — Open questions from any agent

---

## Scope Rules

- Only modify files listed in the task spec's **Files in scope** section.
- Never touch files in the **Files out of scope** section.
- State changes (status, handoffs, task files) always go in `.maestro/`.
- If you're unsure whether a file is in scope, ask Maestro before touching it.

---

## Playbooks

For standard workflows, follow the relevant playbook:
- `.maestro/playbooks/firmware-change.md` — Any firmware modification
- `.maestro/playbooks/code-review.md` — Code review checklist
- `.maestro/playbooks/bug-fix.md` — Bug fix workflow

If a playbook doesn't exist for your task type, ask Maestro to create one.

---

## Firmware Context

This codebase targets embedded/real-time systems. When in doubt:
- Prefer `const` and explicit integer types
- Assume ISR safety matters
- Assume memory is constrained
- Assume the hardware can lie to you
- Never silently swallow errors

---

## Task Templates

New tasks: `.maestro/tasks/_template.md`
New handoffs: `.maestro/handoffs/_template.md`

Always start from the template. Don't invent new document formats.
