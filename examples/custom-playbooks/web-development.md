# Custom Playbook Example: Web Development

> This file lives in `examples/custom-playbooks/` as a **starting point**.
> Copy it to `.maestro/playbooks/web-development.md` in your project and customize it.

---

# Playbook: Web Development

> **Purpose:** Use this playbook for frontend and backend web application changes.
>
> **Domain:** Web Development (JavaScript/TypeScript, React, Node.js, REST APIs, etc.)

---

## Overview

Web development spans frontend UI, backend APIs, databases, and infrastructure. This playbook adds web-specific checkpoints covering security, accessibility, performance, and cross-browser compatibility.

---

## Phase 1: Intake (Maestro)

Additional intake steps for web changes:

1. **Identify the layer.** Frontend only? Backend API? Both? Database schema change?
2. **Identify affected routes/endpoints.** Are there URL or API contract changes that affect consumers?
3. **Check for breaking changes.** Does this change any existing API contracts or data formats?
4. **Capture in task spec:** browser targets, authentication requirements, accessibility requirements, mobile considerations.

---

## Phase 2: Design (Architect)

Web-specific design steps:

1. **API contract first.** For backend changes, define request/response shapes before implementation.
2. **Consider backwards compatibility.** Are old clients affected by this API change?
3. **Map authentication and authorization.** Which endpoints require auth? What roles are needed?
4. **Document state management.** For frontend: where does state live (local, global store, server)?
5. **Consider caching.** What can be cached? What must be invalidated?

Additional design doc sections for web:
- **API Contract:** Request/response shapes, status codes, error formats
- **Auth/Authz:** Required authentication level and permission checks
- **Data model changes:** Any schema migrations needed?
- **Breaking changes:** Impact on existing clients or consumers

---

## Phase 3: Implementation (Coder)

Web coding standards:

1. **Input validation on the server** — Never trust client input.
2. **Parameterized queries** — Never concatenate user input into SQL or NoSQL queries.
3. **Environment variables for secrets** — No hardcoded API keys, passwords, or JWT secrets.
4. **CORS configured correctly** — Explicit allowed origins, not wildcard in production.
5. **Meaningful HTTP status codes** — 400 for bad input, 401 for unauthed, 403 for forbidden, 404 for not found, 500 for server error.
6. **Consistent error format** — Match the project's established error response shape.
7. **TypeScript types** — If the project uses TypeScript, all new code is typed; no `any` without justification.

---

## Phase 4: Code Review (Reviewer)

Web-specific review checklist:

- [ ] **SQL/NoSQL injection** — All queries parameterized or use ORM safely
- [ ] **XSS prevention** — User content is escaped before rendering in HTML
- [ ] **CSRF protection** — State-changing requests protected (token or SameSite cookie)
- [ ] **Auth checks** — Every protected endpoint verifies authentication and authorization
- [ ] **Input validation** — All user-supplied data validated before use
- [ ] **Sensitive data** — No passwords/secrets in logs, error messages, or response bodies
- [ ] **API contract** — Response shape matches what was designed
- [ ] **Status codes** — Correct HTTP status codes used throughout
- [ ] **N+1 queries** — No ORM patterns that generate one query per list item
- [ ] **Migration safety** — DB migrations are backwards-compatible (additive, not destructive)

---

## Phase 5: Testing (Tester)

Web-specific test categories:

1. **API contract tests** — Verify response shape and status codes match the spec
2. **Authentication tests** — Unauthenticated requests return 401, unauthorized return 403
3. **Input validation tests** — Malformed, empty, XL, and special-character inputs
4. **Integration tests** — Full request → handler → database → response cycle
5. **Browser compatibility** — Test on target browsers if frontend changes are involved
6. **Accessibility** — Run an axe/lighthouse audit if UI changes are involved
7. **Performance baseline** — Check response times haven't regressed on critical paths

Test infrastructure expectations:
- API tests: use the project's existing HTTP test client (supertest, jest, etc.)
- Frontend: React Testing Library or equivalent; Playwright/Cypress for E2E
- Database: test against a real (test) database, not mocks, for integration tests

---

## Sign-Off Checklist (Maestro)

Additional sign-off for web changes:

- [ ] API documentation updated (OpenAPI spec, README, etc.)
- [ ] Database migrations tested in staging environment
- [ ] Security checklist completed (OWASP Top 10 considered)
- [ ] Cross-browser testing completed (if frontend changed)
