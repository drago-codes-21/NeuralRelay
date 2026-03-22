# Setting Up Your Project for NeuralRelay

NeuralRelay agents explore your codebase before doing their work. The better your project is set up for AI agents in general, the better NeuralRelay performs. This guide covers how to make NeuralRelay work well with your specific project.

## Your CLAUDE.md Matters

NeuralRelay agents read your project's `CLAUDE.md` as their first step. What you put there directly affects the quality of requirements, implementation, testing, and review.

**If you don't have a CLAUDE.md:** NeuralRelay still works — agents fall back to reading README, package.json, and exploring the directory structure. But they'll spend more tokens on exploration and may miss project conventions.

**What to include for best results:**

```markdown
# Project Name

## Architecture
- src/api/ — REST endpoints (Express)
- src/services/ — business logic (no direct DB access)
- src/models/ — Prisma models
- src/lib/ — shared utilities

## Conventions
- All API endpoints return { data, error } shape
- Services throw typed errors, controllers catch and format
- Tests live next to source files (foo.ts → foo.test.ts)
- Use zod for all input validation

## Key Patterns
- Authentication: JWT in httpOnly cookie, middleware in src/middleware/auth.ts
- Database: Prisma with PostgreSQL, migrations in prisma/migrations/
- Error handling: AppError class in src/lib/errors.ts

## Testing
- Jest with ts-jest
- Run: npm test
- Integration tests need DATABASE_URL pointing to test database
```

**Why this helps NeuralRelay specifically:**
- The BA agent uses module boundaries to identify `affected_modules` accurately
- The dev agent follows your conventions instead of guessing (less rework)
- The test agent knows your test framework and file patterns
- The review agent can check convention compliance

## Writing Good Task Descriptions

The task description is the input to the entire pipeline. A vague description produces vague requirements, generic code, and unfocused tests. A specific description produces targeted results.

### For `/neuralrelay start` (full pipeline)

The BA agent will analyze and expand your description, so you don't need to specify everything. But give enough context for the BA to ask the right questions.

**Bad — too vague:**
```
/neuralrelay start "fix authentication"
```
The BA doesn't know which auth issue, which endpoint, or what "fix" means.

**Bad — too implementation-specific:**
```
/neuralrelay start "add a try-catch in auth.ts line 45 and return 401"
```
This skips the analysis that makes NeuralRelay valuable. Just use Claude Code directly.

**Good — clear problem and scope:**
```
/neuralrelay start "Add OAuth2 login with Google and GitHub providers, including token refresh and account linking for existing email users"
```

**Good — bug with context:**
```
/neuralrelay start "Fix the race condition in token refresh — when two API calls trigger a refresh simultaneously, the second one fails with invalid_grant because the first call already rotated the refresh token"
```

**Good — feature with constraints:**
```
/neuralrelay start "Add rate limiting to all public API endpoints. Use a sliding window algorithm, 100 requests per minute per API key, return 429 with Retry-After header. Must not add Redis or external dependencies — use in-memory store"
```

### For `/neuralrelay quick` (2-stage pipeline)

No BA agent runs, so the dev agent extracts requirements directly from your description. Be more specific here — include what you'd want the BA to produce.

**Bad:**
```
/neuralrelay quick "fix the login bug"
```

**Good:**
```
/neuralrelay quick "Fix null pointer in UserService.getProfile (src/services/user.ts:45) — getProfile calls user.email.toLowerCase() without null check, throws when user signed up via phone number and has no email. Should return profile with email field as null instead of throwing."
```

**Good:**
```
/neuralrelay quick "Refactor the payment processing in src/services/payments.ts — extract the Stripe-specific logic into src/services/stripe-adapter.ts so we can add PayPal later. Keep the same public API, just move the implementation. All 12 existing payment tests should still pass."
```

### Task description patterns by type

| Task Type | What to Include | Example |
|---|---|---|
| New feature | What it does, who uses it, key constraints | "Add CSV export for admin dashboard reports, max 10K rows, streamed to avoid memory issues" |
| Bug fix | What's broken, where, reproduction steps | "User search returns 0 results when query contains a single quote — SQL injection protection is too aggressive in src/api/search.ts" |
| Refactor | What to change, what to preserve, success criteria | "Extract email sending from UserController into EmailService. All 8 controller tests must still pass. No new dependencies." |
| Security | What's vulnerable, threat model | "The password reset endpoint accepts any email without rate limiting. Add rate limiting: 3 requests per email per hour, 10 per IP per hour" |

## Pipeline Template Selection

Choose the right pipeline for your task:

| Your situation | Use this |
|---|---|
| Bug fix, you know exactly what's wrong | `/neuralrelay quick "description"` |
| Small feature, clear requirements | `/neuralrelay quick "description"` |
| New feature, needs analysis | `/neuralrelay start "description"` |
| Complex change, multiple modules | `/neuralrelay start "description"` |
| Touching auth, payments, PII | Enterprise-secure template |
| High-reliability, needs tests first | TDD-strict template |

## What NeuralRelay Agents Can and Cannot Access

NeuralRelay agents operate within Claude Code's standard tool access:

**They CAN:**
- Read any file in your project directory
- Write and edit files
- Run shell commands (build, test, lint)
- Search with glob and grep

**They CANNOT:**
- Access external services (unless you have CLI tools installed)
- Run your application and interact with it
- Access databases directly (only through your project's CLI/scripts)
- Make network requests to APIs
- Access browser-based UIs

Plan your task descriptions accordingly. If testing requires a running database, make sure your project has a `npm test` or equivalent that handles setup.

## Improving Results Over Time

After a few pipeline runs, use `/neuralrelay report` to see patterns:

- **Dev handoff completeness low?** The dev agent may need more context. Add module descriptions to CLAUDE.md.
- **High rework rate?** Requirements may be ambiguous. Write more specific task descriptions, or add domain context to CLAUDE.md.
- **Test agent missing bugs?** Check if coverage_gaps from the dev stage are specific enough. Consider using Opus for the dev stage.
- **Review agent too strict?** Check what categories of issues dominate. If they're all "style", your project may benefit from documenting style conventions in CLAUDE.md so agents follow them from the start.
