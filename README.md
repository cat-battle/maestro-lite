# Maestro-lite

**Multi-agent orchestration for VS Code Copilot Chat — no CLI, no MCP, no servers.**

Maestro-lite gives you a team of specialized Copilot agents that coordinate through file-based state. One agent decomposes your task, hands it to a designer, who hands it to a coder, who gets reviewed and tested — all tracked in plain markdown files you can read, edit, and version.

Works for **any domain**: web development, data science, DevOps, firmware, mobile apps — whatever you're building.

---

## How It Works

You have five agents, each with a specific role:

| Agent | Role |
|-------|------|
| **@maestro** | Orchestrator — accepts tasks, decomposes, assigns, tracks |
| **@architect** | Design — analyzes code, produces technical designs |
| **@coder** | Implementation — writes code per the design, nothing more |
| **@reviewer** | Code review — checks correctness, design conformance, security, edge cases |
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

Add this repo (or copy the `.maestro/` directory and `agents/` files) to your VS Code workspace so agents can read and write `.maestro/` files.

### 3. Start a Task

Switch to `@maestro` in Copilot Chat and describe what you want to do:

```
@maestro I need to add pagination to the search results endpoint. Right now it returns all results at once.
```

Maestro will read `.maestro/status.md`, decompose the task, present a plan for your approval, and then walk you through the full workflow.

---

## Agent Roster

### @maestro — Orchestrator
Accepts your task, breaks it into subtasks, assigns each to the right agent, and tracks everything in `.maestro/status.md`. Stops for your approval at key checkpoints. Never writes production code.

### @architect — Design Agent
Reads the task spec and the existing codebase before proposing anything. Produces a detailed design document that coder can implement without guessing. References domain-specific playbooks for specialized considerations.

### @coder — Implementation Agent
Reads the task spec and design doc, then implements strictly per the design. If the design has gaps, it writes questions and stops — it never guesses. Follows project coding standards and applicable playbooks.

### @reviewer — Code Review Agent
Reviews against: correctness, design conformance, error handling, security, performance, and code quality. Returns a clear PASS / CHANGES_REQUESTED / BLOCKER verdict with specific findings. References playbooks for domain-specific review criteria.

### @tester — Testing Agent
Covers happy path, boundary conditions, error handling, and edge cases. References domain-specific playbooks for additional test categories. Writes runnable test code if a framework is available; detailed manual procedures if not.

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
    ├── _template.md           ← Template for creating custom playbooks
    ├── feature-implementation.md  ← Generic feature workflow
    ├── code-review.md         ← Generic review checklist
    └── bug-fix.md             ← Generic bug fix workflow
```

### Task IDs
Zero-padded three-digit numbers: `001`, `002`, `003`. Never reused. Sub-tasks use letter suffixes: `001a`, `001b`.

### Status Dashboard
`.maestro/status.md` is the source of truth. Every agent updates it when they complete their phase. Always read it before starting.

---

## Playbooks

**Playbooks are the customization layer.** The agents are generic — playbooks make them domain-specific.

A playbook is a markdown file in `.maestro/playbooks/` that adds domain-specific steps, checklists, and standards to the standard workflow. Agents check for a relevant playbook and follow it in addition to their core instructions.

### Built-in Playbooks

| Playbook | Use When |
|----------|----------|
| `feature-implementation.md` | Building a new feature from scratch |
| `code-review.md` | Reviewing any code change |
| `bug-fix.md` | Fixing a reported defect |

### Adding Your Own Playbooks

1. Copy `.maestro/playbooks/_template.md` to a new file (e.g., `.maestro/playbooks/my-domain.md`)
2. Fill in the domain-specific steps for each phase
3. Tell `@maestro` to reference it: *"Use the web-development playbook for this task"*

See `examples/custom-playbooks/` for complete examples of domain-specific playbooks.

---

## Customization

The key principle: **agents are generic, playbooks are specific.**

You should almost never need to edit the `agents/` files. Instead, write playbooks that add domain-specific guidance on top of the generic workflow.

### How to Customize for Your Domain

**Step 1: Write a domain playbook**

Create `.maestro/playbooks/<your-domain>.md` using the `_template.md` as your starting point. Add:
- Domain-specific design considerations for `@architect`
- Coding standards and patterns for `@coder`
- Domain-specific review checklist items for `@reviewer`
- Domain-specific test categories for `@tester`

**Step 2: Tell Maestro about it**

When starting a task, mention your playbook:
> `@maestro Add OAuth login. Follow the web-development playbook.`

Or set it as the default by mentioning it in your task specs.

**Step 3: Extend over time**

As you discover patterns specific to your project, add them to your playbook. The playbooks live in your repo and evolve with your codebase.

### Example Custom Playbooks

See `examples/custom-playbooks/` for ready-to-adapt starting points:

| File | Domain |
|------|--------|
| `firmware.md` | Firmware/embedded C/C++ (ISR safety, memory constraints, timing) |
| `web-development.md` | Web apps (security, API contracts, database migrations, accessibility) |

Copy the most relevant one to `.maestro/playbooks/` and customize it for your project.

---

## Example Workflow

Here's what a typical task looks like:

**1. User → @maestro**
> "Add pagination to the search API endpoint."

**2. @maestro** reads status.md, decomposes the task, creates `.maestro/tasks/001-search-pagination.md`, presents the plan.

**3. User approves the plan.** (Plan Review gate)

**4. @maestro → @architect**
> "Read tasks/001-search-pagination.md and produce a design."

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

See `examples/sample-task.md` for a fully worked example with all files filled in.

---

## Why File-Based State?

- **Transparent** — Every decision is in a readable file. No black box.
- **Reviewable** — Design docs, handoffs, and reviews are all version-controlled.
- **Recoverable** — If a session dies, state survives in files. Resume exactly where you left off.
- **No infrastructure** — No servers, no databases, no MCP servers. Just files.
- **Portable** — Works in any Copilot setup that supports custom agents and workspace file access.

---

## See Also

- `examples/sample-task.md` — A complete task flow with all files filled in (API authentication example)
- `examples/custom-playbooks/` — Domain-specific playbook examples (firmware, web development)
- `.maestro/playbooks/_template.md` — Template for writing your own playbooks
- `.github/copilot-instructions.md` — Repo-level instructions loaded automatically by Copilot
- `agents/` — Agent instruction files for your custom agent definitions
