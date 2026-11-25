# Maestro-lite

**Multi-agent orchestration for VS Code Copilot Chat — no CLI, no MCP, no servers.**

Maestro-lite gives you a team of specialized Copilot agents that coordinate through file-based state. One agent decomposes your task, hands it to a designer, who hands it to a coder, who gets reviewed and tested — all tracked in plain markdown files you can read, edit, and version.

Built for firmware engineers who want structured AI-assisted development without giving up control.

---

## How It Works

You have five agents, each with a specific role:

| Agent | Role |
|-------|------|
| **@maestro** | Orchestrator — accepts tasks, decomposes, assigns, tracks |
| **@architect** | Design — analyzes code, produces technical designs |
| **@coder** | Implementation — writes code per the design, nothing more |
| **@reviewer** | Code review — checks correctness, firmware safety, design conformance |
| **@tester** | Testing — writes test plans and test code against acceptance criteria |

Agents communicate exclusively through files in `.maestro/`. They share no context directly — each one reads the previous agent's output document and produces the next one.

**The user stays in control.** Maestro stops for human approval at four gates: Plan Review, Design Review, Implementation Review, and Completion. Nothing advances without your sign-off.

---

## Quick Start

### 1. Set Up Your Custom Agents in VS Code Copilot

In VS Code, open the Copilot Chat panel. For each agent, create a custom agent using the contents of the `agents/` directory:

1. Open **Copilot Chat** → **`...` menu** → **Configure agents** (or edit `.vscode/copilot-agents.json`)
2. Create an agent named `maestro` with the contents of `agents/maestro.md` as its instructions
3. Repeat for `architect`, `coder`, `reviewer`, `tester`

> The `agents/` directory contains the instruction files you load into your VS Code Copilot custom agent definitions.

### 2. Add the Repo to Your Workspace

Add this repo to your VS Code workspace so agents can read and write `.maestro/` files.

### 3. Start a Task

Switch to `@maestro` in Copilot Chat and describe what you want to do:

```
@maestro I need to add I2C error handling to the sensor module. Right now if the sensor hangs, the entire acquisition loop blocks.
```

Maestro will read `.maestro/status.md`, decompose the task, present a plan for your approval, and then walk you through the full workflow.

---

## Agent Roster

### @maestro — Orchestrator
Accepts your task, breaks it into subtasks, assigns each to the right agent, and tracks everything in `.maestro/status.md`. Stops for your approval at key checkpoints. Never writes production code.

### @architect — Design Agent
Reads the task spec and the existing codebase before proposing anything. Produces a detailed design document that coder can implement without guessing. For firmware: explicitly handles ISR safety, memory constraints, timing, and peripheral dependencies.

### @coder — Implementation Agent
Reads the task spec and design doc, then implements strictly per the design. If the design has gaps, it writes questions and stops — it never guesses. Follows firmware coding standards: const-correct, no dynamic alloc, ISR-safe, defensive error handling.

### @reviewer — Code Review Agent
Reviews against: correctness, design conformance, firmware safety (ISR safety, race conditions, buffer overflows, integer overflow, resource cleanup). Returns a clear PASS / CHANGES_REQUESTED / BLOCKER verdict with specific findings.

### @tester — Testing Agent
Covers happy path, boundary conditions, error handling, edge cases, and firmware-specific scenarios (power failure, concurrent access, timing edge cases). Writes runnable test code if a framework is available; detailed manual procedures if not.

---

## File Protocol

```
.maestro/
├── status.md              ← Live dashboard; read this first
├── tasks/
│   ├── _template.md       ← Copy this for new tasks
│   └── 001-task-name.md   ← Task specs (NNN format)
├── handoffs/
│   ├── _template.md       ← Copy this for new handoffs
│   ├── 001-design.md      ← Architect → Coder
│   ├── 001-impl.md        ← Coder → Reviewer
│   ├── 001-review.md      ← Reviewer findings
│   └── 001-tests.md       ← Tester's test plan
└── playbooks/
    ├── firmware-change.md ← Full firmware change workflow
    ├── code-review.md     ← Review checklist
    └── bug-fix.md         ← Bug fix workflow
```

### Task IDs
Zero-padded three-digit numbers: `001`, `002`, `003`. Never reused. Sub-tasks use letter suffixes: `001a`, `001b`.

### Status Dashboard
`.maestro/status.md` is the source of truth. Every agent updates it when they complete their phase. Always read it before starting.

---

## Playbooks

Playbooks are step-by-step workflows for common task types. Agents follow the relevant playbook automatically — you can also reference them directly.

| Playbook | Use When |
|----------|----------|
| `firmware-change.md` | Any change to production firmware |
| `code-review.md` | Reviewing any code change |
| `bug-fix.md` | Fixing a reported defect |

**Adding your own:** Create a new `.md` file in `.maestro/playbooks/`. Maestro will reference it when it's relevant to the task type.

---

## Example Workflow

Here's what a typical task looks like:

**1. User → @maestro**
> "Add watchdog timer kicking to the sensor module."

**2. @maestro** reads status.md, decomposes the task, creates `.maestro/tasks/001-watchdog-sensor.md`, presents the plan.

**3. User approves the plan.** (Plan Review gate)

**4. @maestro → @architect**
> "Read tasks/001-watchdog-sensor.md and produce a design."

**5. @architect** reads the codebase, writes `.maestro/handoffs/001-design.md`, updates status.

**6. @maestro** presents design summary. **User approves.** (Design Review gate)

**7. @maestro → @coder**
> "Read tasks/001 and handoffs/001-design.md and implement."

**8. @coder** implements per design, writes `.maestro/handoffs/001-impl.md`, updates status.

**9. @maestro → @reviewer**
> "Read tasks/001, handoffs/001-design.md, and handoffs/001-impl.md and review."

**10. @reviewer** writes `.maestro/handoffs/001-review.md` with verdict and findings.

**11. If CHANGES_REQUESTED:** @coder addresses findings, @reviewer re-reviews.

**12. @maestro → @tester**
> "Read tasks/001 and write a test plan."

**13. @tester** writes `.maestro/handoffs/001-tests.md`, updates status.

**14. @maestro** presents final summary. **User approves.** (Completion gate) Task closed.

---

## Why File-Based State?

- **Transparent** — Every decision is in a readable file. No black box.
- **Reviewable** — Design docs, handoffs, and reviews are all version-controlled.
- **Recoverable** — If a session dies, state survives in files. Resume exactly where you left off.
- **No infrastructure** — No servers, no databases, no MCP servers. Just files.
- **Portable** — Works in any Copilot setup that supports custom agents and workspace file access.

---

## Firmware-Specific Design

Maestro-lite was built with embedded/firmware development in mind. Every agent instruction is written for C/C++ on real-time systems:

- Architect explicitly considers ISR safety, memory constraints, timing budgets, peripheral dependencies
- Coder enforces const-correctness, explicit integer types, no dynamic allocation, volatile where needed
- Reviewer checks for ISR races, buffer overflows, integer overflow, watchdog starvation
- Tester covers power failure, concurrent access, timing edge cases, memory boundaries

This isn't a web-dev workflow with "firmware" sprinkled in. It's firmware-first.

---

## See Also

- `examples/sample-task.md` — A complete task flow with all files filled in (watchdog timer example)
- `.github/copilot-instructions.md` — Repo-level instructions loaded automatically by Copilot
- `agents/` — Agent instruction files for your custom agent definitions
