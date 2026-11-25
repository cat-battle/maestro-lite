# Example: Add User Authentication to the API

This example shows what a complete Maestro-lite task flow looks like, end to end. All files are shown as they'd actually appear after each phase.

---

## Task Spec: `.maestro/tasks/001-user-auth.md`

```markdown
# Task 001: Add User Authentication to the API

## Objective
Add JWT-based authentication to the REST API so that protected endpoints require a valid token. Unauthenticated requests should receive a 401 response.

## Context
The API currently has no authentication. All endpoints are publicly accessible. We need to protect user-specific endpoints (`/users/:id`, `/orders`, `/profile`) while keeping public endpoints (`/products`, `/health`) open.

## Scope
- **Files in scope:**
  - `src/middleware/auth.js`  (create new)
  - `src/routes/users.js`
  - `src/routes/orders.js`
  - `src/routes/profile.js`
  - `src/config/index.js`  (add JWT_SECRET config key)
  - `tests/middleware/auth.test.js`  (create new)
- **Files out of scope:**
  - `src/routes/products.js`  (stays public)
  - `src/routes/health.js`  (stays public)
  - Database schema files  (no schema changes in this task)

## Acceptance Criteria
- [ ] Protected endpoints return 401 when no token is provided
- [ ] Protected endpoints return 401 when an invalid or expired token is provided
- [ ] Protected endpoints return 200 with correct data when a valid token is provided
- [ ] Public endpoints continue to work without a token
- [ ] Tokens expire after 24 hours

## Constraints
- Use the existing `jsonwebtoken` library (already in package.json)
- JWT secret must be loaded from environment variable `JWT_SECRET`, never hardcoded
- Middleware must be applied per-route, not globally (to preserve public endpoints)

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
Add a JWT authentication middleware and apply it to the three protected route files. The middleware validates the Authorization header, verifies the token, and either attaches the decoded user to the request or returns 401. No database calls in the middleware — token validation is stateless.

## Overview
We need a reusable Express middleware function that validates JWT tokens. It reads the `Authorization: Bearer <token>` header, verifies the token using `jsonwebtoken`, and either attaches the decoded payload to `req.user` or sends a 401. Protected routes require this middleware; public routes do not.

## File / Module Layout

**`src/middleware/auth.js`** (new file)
- Export a single middleware function: `authenticateToken(req, res, next)`
- Reads `Authorization` header, extracts Bearer token
- Verifies with `jwt.verify(token, process.env.JWT_SECRET)`
- On success: attaches decoded payload to `req.user`, calls `next()`
- On failure: responds 401 with `{ error: "Unauthorized" }`

**`src/routes/users.js`**, **`orders.js`**, **`profile.js`** (modify)
- Import `authenticateToken` from `../middleware/auth`
- Apply as route-level middleware on protected endpoints

**`src/config/index.js`** (modify)
- Add `JWT_SECRET: process.env.JWT_SECRET` to config exports
- Add validation: throw on startup if `JWT_SECRET` is not set

## Interfaces and APIs

```javascript
// src/middleware/auth.js
function authenticateToken(req, res, next)
// Attaches req.user = { id, email, role } on success
// Sends 401 on missing/invalid/expired token
```

## Data Flow

1. Request arrives at protected route
2. `authenticateToken` middleware runs first
3. Extracts token from `Authorization: Bearer <token>` header
4. Calls `jwt.verify(token, JWT_SECRET)`
5. Success → `req.user = decoded payload` → `next()` → route handler runs
6. Failure (missing, malformed, expired) → `res.status(401).json({ error: "Unauthorized" })`

## Dependencies
- `jsonwebtoken` (already installed)
- `process.env.JWT_SECRET` (must be set in environment)

## Domain-Specific Considerations
- **Secret management:** JWT_SECRET must never be hardcoded. Validate it exists at startup, not at first request.
- **Token expiry:** Use `jwt.verify` built-in expiry check — do not manually check `exp` claim.
- **Error messages:** Return generic "Unauthorized" — do not reveal whether the token was missing vs. expired vs. invalid (avoid information leakage).

## Risks and Open Questions
- Token refresh is out of scope for this task (tracked as future task).
- If JWT_SECRET is rotated, all existing tokens are invalidated — operations team is aware.
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
Implemented per design. Created auth middleware, applied to three route files, added config validation. No deviations from design.

## Artifacts
- `src/middleware/auth.js` — New file, implements `authenticateToken`
- `src/routes/users.js` — Added `authenticateToken` middleware to 3 routes
- `src/routes/orders.js` — Added `authenticateToken` middleware to 2 routes
- `src/routes/profile.js` — Added `authenticateToken` middleware to all routes
- `src/config/index.js` — Added JWT_SECRET validation on startup

## Key Implementation Notes
- Used `try/catch` around `jwt.verify` to handle all error cases uniformly
- Startup validation throws `Error("JWT_SECRET environment variable is required")` if not set
- Error response is always `{ error: "Unauthorized" }` regardless of failure reason (no info leak)
- Applied middleware at route level, not router level, to allow mixed auth on same router

## No Deviations from Design
```

---

## Review Handoff: `.maestro/handoffs/001-review.md`

```markdown
# Review: Task 001 — User Authentication

**Verdict:** CHANGES_REQUESTED

## Summary
Implementation is correct and well-structured. One major finding: the `Authorization` header parsing doesn't handle the case where the header exists but has the wrong format (e.g., "Basic xyz" instead of "Bearer xyz"). One minor: the startup validation error message should include the config key name.

## Findings

### FINDING-001 [Severity: major]
**File:** src/middleware/auth.js:12
**Description:** Code does `const token = authHeader.split(' ')[1]` without checking that `authHeader` starts with "Bearer ". If the header is "Basic xyz", this silently passes a wrong token to `jwt.verify` instead of returning 401 immediately.
**Impact:** Unexpected error from jwt.verify rather than a clean 401. Could leak internal error details depending on error handling.
**Suggested Fix:** Check `authHeader.startsWith('Bearer ')` before splitting.

### FINDING-002 [Severity: minor]
**File:** src/config/index.js:8
**Description:** Error message says "Required environment variable is not set" without naming the variable.
**Suggested Fix:** `throw new Error("Required environment variable JWT_SECRET is not set")`
```

---

## Test Plan: `.maestro/handoffs/001-tests.md`

```markdown
# Test Plan: Task 001 — User Authentication

## Acceptance Criteria Coverage
| Criterion | Test Cases |
|-----------|-----------|
| Protected → 401 with no token | TC-001 |
| Protected → 401 with invalid/expired token | TC-002, TC-003 |
| Protected → 200 with valid token | TC-004 |
| Public endpoints still work | TC-005 |
| Tokens expire after 24 hours | TC-006 |

## Test Cases

### TC-001: No token returns 401
**Type:** Integration
**Setup:** Test server running, no Authorization header
**Steps:**
1. GET /users/123 with no Authorization header
**Expected Result:** 401 with body `{ error: "Unauthorized" }`

### TC-002: Invalid token returns 401
**Type:** Integration
**Setup:** Test server running
**Steps:**
1. GET /users/123 with `Authorization: Bearer notarealtoken`
**Expected Result:** 401 with body `{ error: "Unauthorized" }`

### TC-003: Expired token returns 401
**Type:** Unit
**Setup:** Generate a token with `expiresIn: -1` (already expired)
**Steps:**
1. Call `authenticateToken` middleware with expired token
**Expected Result:** Calls `res.status(401)`, does NOT call `next()`

### TC-004: Valid token passes through
**Type:** Unit
**Setup:** Generate a valid token signed with test JWT_SECRET
**Steps:**
1. Call `authenticateToken` middleware with valid token
2. Check that `req.user` is populated and `next()` is called
**Expected Result:** `req.user.id` matches token payload, `next()` called once

### TC-005: Public endpoints unaffected
**Type:** Integration
**Steps:**
1. GET /products with no Authorization header
**Expected Result:** 200 with product data (not 401)

### TC-006: 24-hour expiry
**Type:** Unit
**Setup:** Inspect token generated by login route
**Steps:**
1. Decode token (without verifying) and inspect `exp` claim
2. Check that `exp - iat == 86400` (24 hours in seconds)
**Expected Result:** Expiry is exactly 24 hours from issue time
```

---

## Status Dashboard (Mid-Task)

This is what `.maestro/status.md` looks like partway through:

```markdown
# Maestro Status

## Active Tasks
| ID | Task | Status | Assigned | Phase |
|----|------|--------|----------|-------|
| 001 | Add user authentication to API | In Progress | tester | Testing |

## Completed Tasks
_None yet_

## Notes
- 2026-11-20: Task 001 opened — JWT auth middleware
- 2026-11-21: Design complete, approved at Design Review gate
- 2026-11-22: Implementation complete, one major finding in review (Bearer prefix check)
- 2026-11-23: Review finding fixed, re-review passed. Testing in progress.
```

---

## Memory Entries After This Task

Here's what maestro would write to the memory files at the end of this task. These entries carry the project's knowledge forward into future sessions.

### `.maestro/memory/decisions.md` — New Entry

```markdown
## [2026-11-23] JWT Over Session Tokens for API Authentication
**Context:** Needed to add authentication to the REST API. Evaluated stateful session tokens (server-side session store) vs. stateless JWT.
**Decision:** Use JWT with 24-hour expiry, loaded via `jsonwebtoken` library.
**Rationale:** The API is stateless by design. JWTs allow horizontal scaling without a shared session store. The existing `jsonwebtoken` library was already a dependency.
**Alternatives Considered:** Server-side sessions (rejected: requires shared session store, adds infrastructure); opaque API keys (rejected: no built-in expiry, harder to rotate).
**Status:** Active
```

### `.maestro/memory/patterns.md` — New Entry

```markdown
## Authentication Middleware Pattern
**When to use:** Any endpoint that requires a valid authenticated user.
**Implementation:** Import `authenticateToken` from `src/middleware/auth.js` and apply it as route-level middleware — not router-level — so public and protected routes can coexist on the same router.
**Example:**
```javascript
const { authenticateToken } = require('../middleware/auth');
router.get('/profile', authenticateToken, profileHandler);
router.get('/public-info', publicHandler); // no middleware
```
**Rationale:** Route-level application keeps public endpoints accessible without splitting routers by auth status.
```

### `.maestro/memory/lessons.md` — New Entry

```markdown
## [2026-11-22] Authorization Header Format Not Validated
**What happened:** Reviewer caught that the initial implementation split the Authorization header without first checking it started with "Bearer ". A "Basic xyz" header would pass malformed input to jwt.verify instead of returning a clean 401.
**Root cause:** Assumed the header format would always be correct; didn't consider non-Bearer schemes.
**Lesson:** When parsing structured header values, always validate the scheme prefix before splitting. Use `header.startsWith('Bearer ')` before extracting the token.
**Tags:** #security #http #input-validation
```

### `.maestro/memory/log/2026-11-23.md` — Session Log

```markdown
# Session Log — 2026-11-23

## Tasks Worked On
- Task 001: Add JWT authentication middleware

## Key Decisions
- Chose JWT over session tokens (see decisions.md)

## Discoveries
- Must validate Authorization header scheme prefix before splitting (see lessons.md)
- Added authentication middleware pattern to patterns.md

## Open Items
- Token refresh not yet implemented — tracked as future task
- Operations team needs to be notified before rotating JWT_SECRET (invalidates all active tokens)
```
