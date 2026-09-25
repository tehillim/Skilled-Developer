# What It Takes to Make Claude a Skilled Developer

> 한국어 버전: [Claude를 유능한 개발자로 만들기 위해 필요한 것](../claude-를-유능한-개발자로.md)

An AI coding agent (Claude) writes good code on its own, but **what makes it work like a truly skilled developer is the environment and the way you collaborate with it**. Just as a capable new hire needs onboarding, Claude needs the same.

It comes down to six things.

1. [Context — documentation that explains the project](#1-context)
2. [A verifiable development environment](#2-a-verifiable-development-environment)
3. [A fast feedback loop](#3-a-fast-feedback-loop)
4. [The right permissions and tools](#4-the-right-permissions-and-tools)
5. [Good collaboration practices](#5-good-collaboration-practices)
6. [Continuous improvement — accumulating what you learn](#6-continuous-improvement)

---

## 1. Context

Claude sees your project for the first time in every session. The tacit knowledge a human developer builds up over months must be made explicit in documentation.

### Write a CLAUDE.md (most important)

`CLAUDE.md` at the repository root is an onboarding document Claude reads automatically at the start of each session. Include:

- **Project overview**: a paragraph or two on what the project does
- **Common commands**: how to build, test, lint, and run (e.g. `npm test`, `make build`)
- **Directory structure**: what lives where
- **Coding conventions**: naming rules, error handling, patterns to use or avoid
- **Caveats**: files not to touch, known pitfalls, legacy areas

The fastest way is to generate a draft with the `/init` command and refine it.

### Other documents that help

- **Architecture docs**: module dependencies, data flow, why the structure was chosen (ADRs)
- **Glossary**: how domain terms map to names in the code
- **API/schema docs**: when external contracts are hard to infer from code alone

> Principle: "Could a newly hired senior developer open their first PR from the docs alone?" If yes, Claude can work well too.

### Example: a minimal CLAUDE.md

Aiming for a complete document delays getting started. Something this size is enough for a first version — fill in the gaps as later sessions reveal them (see section 6).

```markdown
# MyApp

Commerce backend handling orders/payments. Node.js 20 + TypeScript + PostgreSQL.

## Commands

- Dev server: `npm run dev`
- Tests: `npm test` / type check: `npm run typecheck`
- Environment setup: `./scripts/setup.sh` does it all

## Structure

- `src/api/` — HTTP handlers (keep thin, no business logic)
- `src/services/` — business logic
- `src/db/` — schema and migrations

## Conventions

- Return a `Result` type instead of throwing errors
- Amounts are always integer KRW — use only the `Money` type

## Caveats

- Do not modify `src/legacy/` (slated for replacement, no tests)
```

Even at one line per item, spelling out what to run, where to look, and what to avoid dramatically cuts the exploration time and wrong guesses at the start of a session.

### Example: vague vs. specific descriptions

The same item can be actionable or useless depending on how specifically it is written.

| Vague | Specific |
|---|---|
| "The payments module is complex, be careful" | "`src/payments/` integrates directly with an external payment gateway API. Amounts are always integer KRW; a rounding bug once caused a real billing incident, so use only the `Money` type." |
| "Please write thorough tests" | "For every new API endpoint, write 1 success case + at least 2 failure cases in `tests/api/`, and confirm they pass with `npm run test:api`." |

Words like "complex" and "thorough" are vague even for humans. Concrete file paths, reasons, and verification steps are what let Claude act on them.

### Example: glossary

When domain terms and code names differ, Claude searches the wrong places or invents new names. Even a short mapping table keeps code navigation and naming consistent.

```markdown
## Glossary

| Domain term | Code name | Notes |
|---|---|---|
| Partner store | `Merchant` | The UI says "store", but code uses only Merchant |
| Settlement | `Settlement` | `Payout` is the legacy name — do not use in new code |
| Order confirmation | `OrderConfirmation` | Distinct from payment completion (`PaymentCompleted`) |
```

Given a request like "fix the settlement logic", a glossary sends Claude to `Settlement` rather than the legacy `Payout`, and new code keeps using the names the team actually uses.

### Example: ADR (architecture decision record)

Code records the "what" but not the "why". When the reasoning behind a structure isn't written down, Claude may re-propose an approach the team already considered and rejected, calling it an "improvement". One short file per decision is enough.

```markdown
# ADR-007: Publish order events via an outbox table, not directly to the queue

## Status
Accepted (2025-03)

## Context
Order creation and event publishing span different systems;
when the queue was down, orders were created but events were lost.

## Decision
Write events to an outbox table inside the same DB transaction,
and let a separate relay forward them to the queue.

## Consequences
- Orders and events are atomic; publishing may lag by a few seconds
- Calling the queue client directly from `src/orders/` is forbidden
```

With this file in `docs/adr/`, Claude follows the outbox pattern instead of suggesting "just publish straight to the queue — it's simpler", and can flag code that violates the decision.

### Example: per-directory CLAUDE.md files in a monorepo

In a monorepo with many packages, cramming every rule into one root CLAUDE.md makes the document long while the rules needed for any given package get harder to find. CLAUDE.md files can also live in subdirectories, and Claude consults them when working on files in that directory.

```text
repo/
├── CLAUDE.md              ← shared: overall structure, common commands, commit rules
├── apps/web/CLAUDE.md     ← web only: Next.js conventions, component rules
├── apps/api/CLAUDE.md     ← API only: error response format, DB access rules
└── packages/shared/CLAUDE.md ← caveats like "changes require testing both apps"
```

Keep only what applies everywhere at the root, and push package-specific rules down into that package's CLAUDE.md. A session that only touches `apps/api/` no longer has to read frontend conventions, and because rules sit close to the code they govern, the docs are more likely to be updated in the same PR when that code changes (see section 6).

## 2. A Verifiable Development Environment

Claude is far more capable when it can **run the code and check the result**. Code written by guessing and code verified by execution differ greatly in quality.

- **Build and tests must run with one command.** Provide standard entry points like `npm test`, `pytest`, or `make check`.
- **Dependency installation should be automated.** Commit lockfiles and provide a setup script (e.g. `scripts/setup.sh`).
- **Environment variables and secrets**: use `.env.example` to show which values are needed. Ideally, tests run without real secrets.
- **For remote/web sessions**, a SessionStart hook can automate dependency installation.

### Example: setup script

Bundling dependency install, `.env` preparation, and a health check into one script means Claude never has to guess "what do I run first?" In `CLAUDE.md`, one line is enough: "Environment setup is just `./scripts/setup.sh`."

```bash
#!/usr/bin/env bash
# scripts/setup.sh
set -euo pipefail

echo "Installing dependencies..."
npm ci

if [ ! -f .env ]; then
  echo "Creating .env from .env.example"
  cp .env.example .env
fi

echo "Running healthcheck..."
npm run typecheck
```

### Example: .env.example

Don't just list variable names — use comments to say what each value is, where to get it, and what happens if it is left empty. That lets Claude understand what configuration is needed without any real secrets, and tell environment problems apart from code problems.

```bash
# .env.example — never commit real values; put them only in .env

# Local development DB. scripts/setup.sh starts it via Docker
DATABASE_URL=postgres://localhost:5432/myapp_dev

# Payment gateway sandbox key — issued on the team wiki's "Secrets" page
PAYMENT_API_KEY=

# If empty, notifications fall back to console output (tests pass without it)
SLACK_WEBHOOK_URL=
```

When the file even says "which tests fail without this key", Claude running in an environment without secrets can report the cause accurately instead of misattributing the failure to the code.

### Example: run external dependencies with Docker

If tests need a DB or Redis that isn't present in the environment, Claude ends up skipping them or substituting mocks and moving on. Define external dependencies in a `docker-compose.yml` and the real tests run with the same command in any environment.

```yaml
# docker-compose.yml — external dependencies needed by tests
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: myapp_dev
    ports:
      - "5432:5432"
  redis:
    image: redis:7
    ports:
      - "6379:6379"
```

In CLAUDE.md, one line is enough: "Run `docker compose up -d` before testing" — and if you fold it into the setup script (example above), even that line becomes unnecessary. Since dependency versions are pinned in the file, "works on my machine" environment differences disappear too.

### Example: seed data that makes verification predictable

If the DB is up (example above) but empty — or holds different data every time — Claude can run the code and still have no way to judge whether the result is correct. Provide a seed command that always produces the same state, and document what that data contains.

```markdown
## Test data

`npm run db:seed` — resets the local DB to a fixed, known state.

- 3 users: alice (admin), bob, carol
- 5 orders: 3 PAID, 2 CANCELLED (all owned by bob)
- 2 products: 1 in stock, 1 sold out
```

Now, given the task "add an API that lists only cancelled orders", Claude can call the API and confirm it returns **exactly 2 rows** before declaring the work done. Without the seed contents in the docs, an execution result proves nothing — the last piece of a "runnable environment" is predictable data.

### Example: pin the runtime version in a file too

A lockfile pins library versions, but not the version of Node or Python they run on. If local is on Node 20 while CI runs Node 18, Claude ends up suspecting the code for a failure it cannot reproduce. Commit the runtime version to the repository as a file as well.

```bash
# .nvmrc — read by nvm and by setup-node in CI
20.11.1
```

```json
// package.json — fail fast at install time when versions don't match
{
  "engines": { "node": ">=20 <21" }
}
```

Once the version exists as a file, three places line up: a human's local machine (`nvm use`), CI (`node-version-file` in `actions/setup-node`), and Claude — when the environment misbehaves, it can compare this file against the actual version (`node --version`) and diagnose "not a code problem, a runtime version mismatch". For Python, use `.python-version`; with multiple tools, a single `.tool-versions` from mise does the same job. Where the Docker example above pins the versions of external dependencies, this one pins the version of the runtime itself.

## 3. A Fast Feedback Loop

A skilled developer notices on their own when their code is wrong. Automated verification is what gives Claude that sense.

| Tool | Role |
|---|---|
| **Test suite** | Guarantees behavior. Higher coverage lets Claude refactor boldly |
| **Type system** | TypeScript, mypy, etc. Catches bad code before it runs |
| **Linter/formatter** | ESLint, ruff, prettier, etc. Delegates style debates to a machine |
| **CI** | Automatic verification on every PR. Claude can see CI failures and fix them itself |

The point is that **Claude must be able to run these tools itself**. Tests are useless if the docs never say how to run them.

### Example: CI workflow

When tests and lint run automatically on every PR, CI catches what Claude missed locally, and Claude can see the result and fix it on its own.

```yaml
# .github/workflows/ci.yml
name: CI
on: [pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run lint
      - run: npm test
```

The key is to **run the exact same commands in CI that you use locally** (`npm run lint`, `npm test`). If they differ, you get endless "works locally, fails only in CI" situations.

### Example: partial test runs when the full suite is slow

If the full suite takes 10 minutes, Claude either waits 10 minutes per edit-verify cycle or skips verification and proceeds on guesswork. Document in CLAUDE.md how to quickly run just one file or one case.

```markdown
## Running tests

- Full suite: `npm test` (~10 min — right before committing only)
- One file: `npx jest tests/api/orders.test.ts`
- One case: `npx jest -t "order cancellation"`
- Type check only: `npm run typecheck` (seconds — first check right after an edit)
```

When the feedback loop shrinks to seconds, Claude verifies as it goes, after every change. The shorter the verification cycle, the sooner it can turn back before going far in the wrong direction.

### Example: make one-shot commands the default, not watch mode

Humans prefer watch mode, which reruns automatically on every save — but to Claude, a watch command is a **command that never ends**. The process doesn't exit, so Claude either waits without ever getting a result or has to kill it. Make the default commands in your docs and scripts run once and finish with an exit code.

```json
// package.json — the default runs once; watch mode is split out for humans
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest"
  }
}
```

If the default is a long-lived command like `vitest`, `jest --watch`, or `tsc --watch`, Claude trips over it every time. For the same reason, when an interactive command waits for a y/n answer mid-run, document its non-interactive flag (`--yes` and the like) alongside it. A good command for Claude completes the cycle "run → exit → exit code and output" in one shot — which is exactly what the CI example above wants too. Agent-friendly commands and CI-friendly commands turn out to be the same thing.

### Example: enforcing conventions as lint rules

A rule written in CLAUDE.md is a promise Claude reads and follows; a rule you can express as a lint rule becomes one a machine verifies. For example, "no imports from `src/legacy/` in new code" can be enforced with ESLint.

```json
// .eslintrc.json
{
  "rules": {
    "no-restricted-imports": ["error", {
      "patterns": [{
        "group": ["**/legacy/*"],
        "message": "src/legacy/ is being replaced. Use the new implementation in src/services/."
      }]
    }]
  }
}
```

A documented rule can be forgotten, but a lint rule surfaces as an error message the moment it is violated — and Claude sees that message and fixes the code itself. Put the alternative in the `message`, and "what is banned" and "what to use instead" arrive together. Not every convention can move into lint, but each one that can makes CLAUDE.md that much shorter and the rule that much more certain.

### Example: test names that explain their own failures

When a test fails, the test name and the assertion message are essentially all the information Claude gets. If the name describes the behavior, the failure list itself becomes a spec of "which behavior broke"; if not, Claude has to go back to the test code and reverse-engineer the intent.

```ts
// Bad — a failure tells you nothing about what broke
test('cancel test 3', () => {
  expect(result.ok).toBe(true);
});

// Good — the failure output is a bug report
test('cancelling an order restores stock to its pre-order quantity', () => {
  expect(stock.quantity).toBe(10);
});
```

When the former fails, all you get is `Expected: true, Received: false`; when the latter fails, the run log alone says "stock restoration broke, and a value that should be 10 is something else." If chapter 5's "turning failure into useful feedback" is feedback a human gives, well-named tests are feedback the test suite gives on its own. As a bonus, Claude follows the style of existing names when it adds new tests, so good names propagate themselves once established.

## 4. The Right Permissions and Tools

- **Permission settings** (`.claude/settings.json`): pre-allow safe, frequently used commands (tests, lint, read-only operations) so work flows without a confirmation prompt every time.
- **MCP servers**: connect external systems like GitHub, Slack, or databases via MCP. Claude can then read issues, open PRs, and check CI logs.
- **Skills** (`.claude/skills/`): turn recurring procedures (deploys, release notes, review checklists) into skills so they are performed consistently, your team's way.
- **Subagents/hooks**: enable parallelizing large tasks, auto-formatting, and similar automation.

### Example: permission settings

Pre-allowing safe, frequent commands in `.claude/settings.json` removes the confirmation step from every invocation.

```json
{
  "permissions": {
    "allow": [
      "Bash(npm test:*)",
      "Bash(npm run lint:*)",
      "Bash(git status)",
      "Bash(git diff:*)"
    ]
  }
}
```

### Example: blocking dangerous commands with deny

If `allow` is the list of commands that don't need a prompt every time, `deny` is the list of commands that won't be permitted even when asked. Block reads of secret files and hard-to-undo commands up front.

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./secrets/**)",
      "Bash(git push --force:*)",
      "Bash(rm -rf:*)"
    ]
  }
}
```

`deny` takes precedence over `allow`, so the block holds even when a command accidentally matches a broad allow pattern. If review (section 5) is the safety net where a person checks the output, `deny` is the safety net that keeps dangerous things from happening in the first place — putting a hard-to-undo command on the list up front is cheaper than writing a rule after the incident.

### Example: connecting an MCP server

Committing a `.mcp.json` at the repository root lets the whole team share the same external-system integrations. With the GitHub MCP server connected, for example, a single request like "read issue #123, fix it, and open a PR" can cover everything from reading the issue to creating the PR.

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/"
    }
  }
}
```

For servers that require authentication, pass tokens via environment variables instead of writing them into the config file. Record the required variable names in `.env.example` (see section 2) so teammates can connect the same way.

### Example: SessionStart hook

To automatically install dependencies or run a health check when a new session starts, register a `SessionStart` hook. Especially useful for remote/web sessions.

```json
{
  "hooks": {
    "SessionStart": [
      { "hooks": [{ "type": "command", "command": "npm install" }] }
    ]
  }
}
```

### Example: auto-formatting hook

To run your formatter automatically every time a file is modified, register a `PostToolUse` hook. No more asking for formatting each time, and no more CI failures over style differences alone.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write"
          }
        ]
      }
    ]
  }
}
```

The key point is that hooks run **always**, independent of Claude's judgment. Writing "please run the formatter" in CLAUDE.md is a request; wiring it as a hook makes it a rule. If the linter/formatter in chapter 3 "delegates style debates to a machine", a hook automates even running that machine.

### Example: skill

For procedures you find yourself explaining in the same order every time — deploys, release notes — create a markdown file under `.claude/skills/`, and Claude will load it on its own when needed and follow your team's process.

```markdown
---
name: release-notes
description: Drafts release notes from recently merged PRs. Use for requests like "write the release notes".
---

1. List the PRs merged since the last tag.
2. Categorize them as fix / feature / breaking change.
3. Write a draft following the `CHANGELOG.md` format.
4. Ask the user for review, and commit after approval.
```

### Example: subagent

Roles that can be delegated independently — review, exploration — can be defined as markdown files under `.claude/agents/`, and Claude will hand that work to a subagent with its own separate context. The benefits: the main task's context stays clean, and you can restrict the subagent to only the tools its role needs.

```markdown
---
name: code-reviewer
description: Reviews code changes. Use before creating a commit or PR.
tools: Read, Grep, Glob
---

Review the changed code and check the following.

1. Potential bugs — boundary conditions, missing error handling
2. Convention violations — against the rules in CLAUDE.md
3. Missing tests — is new behavior covered by tests?

Report the issues you find, ordered by severity. Do not modify the code yourself.
```

Giving a review-only subagent nothing but read tools structurally prevents it from "fixing" code mid-review.

## 5. Good Collaboration Practices

Developers of equal skill produce far better results when requirements are clear.

- **Share the purpose, not just the task.** "When a user does X, Y happens, but it should be Z" is far better than "fix this function".
- **Plan first for large tasks.** For work that needs design, reviewing a plan in plan mode before implementing reduces the cost of course corrections.
- **Scope work appropriately.** One purpose per PR. Easier to review, easier to roll back.
- **Review is still necessary.** Review Claude's output like a teammate's code. Claude can address review comments itself.
- **Report failures.** Feedback like "that didn't work, and here's why" changes the next result.

### Example: a request with purpose

The same bug fix can land very differently depending on the request. Instead of saying only "what to fix", convey "in what situation, what goes wrong, and how it should behave".

| Bad | Good |
|---|---|
| "Fix the login function" | "After 3 wrong password attempts the account should lock, but right now it never locks. Check `verifyPassword` in `auth/login.ts`." |
| "This API seems slow" | "`/api/orders` takes over 500ms. It seems to start after the commit that added a join on the `orders` table (`a1b2c3d`). Check whether an index is missing." |

Passing along the symptom, expected behavior, and suspected cause or relevant files makes it much more likely that Claude finds and fixes the verified cause instead of guessing.

### Example: reviewing a plan first in plan mode

For design-heavy work that touches multiple files, don't jump straight into implementation — use plan mode (`Shift+Tab`) to get a plan first. In this mode Claude only investigates, without modifying code, and then presents an implementation plan.

```text
> (plan mode) Add an order-cancellation feature, including payment refund integration.

Claude's plan:
1. Add cancelOrder() to src/services/orders.ts — with status validation
2. Reuse refund() from src/services/payments.ts (no new implementation)
3. Cancellable-status rule: allow only PAID and PREPARING
4. Tests: 1 success case + 2 non-cancellable-status cases

> On #3: PREPARING involves partial refunds and needs a policy decision.
> Proceed with PAID only for now.
```

Correcting direction before implementation starts costs almost nothing — a very different story from saying "actually, that's not what I meant" after dozens of files have changed.

### Example: scoping work into units

Asking for "add filtering, sorting, and CSV export to the order list" all at once produces a PR touching dozens of files — hard to review, and hard to pinpoint when something breaks. Split it by purpose and request the pieces in order.

1. **PR 1** — add filter/sort parameters to the query API (+ tests)
2. **PR 2** — wire up the filter/sort UI on the frontend
3. **PR 3** — add CSV export

Each PR can be reviewed and shipped independently, and if something goes wrong, only that PR needs to be reverted. It also lets you give Claude a clear boundary — "only do step 1 for now" — which prevents unintended changes from sneaking in alongside the intended ones.

### Example: writing issues Claude can pick up as-is

For "handle issue #123" to work as a request, the issue itself has to be a good request. An issue that carries reproduction steps, expected behavior, and completion criteria becomes a work order that can be executed as-is, without a conversation to fill in the gaps.

```markdown
## Bug: stock is not restored after an order is canceled

**Steps to reproduce**
1. Order product A (stock 10 → 9)
2. Cancel the order
3. Stock is still 9 (expected: 10)

**Expected behavior**
On cancellation, stock should be restored to the pre-order quantity

**Completion criteria**
- [ ] Fix the cancel → restore-stock logic + add a test
- [ ] Backfilling previously canceled orders is out of scope (separate issue #124)
```

An issue that is only a title ("stock bug") means re-supplying the context in conversation anyway — but an issue written like this can be handed to an MCP-connected Claude (see chapter 4) by number alone. It bakes the "request with purpose" principle above into your issue template, and spelling out "out of scope" in the completion criteria prevents unintended expansion before it starts.

### Example: letting Claude address review comments itself

Review Claude's PRs by the same standard as a teammate's code — but the follow-up work can go back to Claude. Review comments follow the same principle as requests above: the more specific the location and the reason, the more accurate the fix.

```text
Review comment:
> src/services/orders.ts line 87 — cancelOrder() returns 200 even on
> failure. Per our API rules, domain errors must return 409 with an
> error code. (See docs/api-conventions.md)

To Claude:
> Address the review comments on PR #142.
```

When a comment carries the file location, the violated rule, and a reference doc, Claude finds the exact spot, fixes it to match the rule, and commits. Conversely, a comment like "the error handling seems off" needs a follow-up question — from Claude just as it would from a human developer. When the reviewer keeps the approval decision but delegates the mechanical follow-up, review round-trips get cheaper without lowering the review bar.

### Example: turning failure into useful feedback

When the result isn't what you expected, saying only "it doesn't work" forces Claude to guess at the cause all over again. Instead, share what you ran, what you expected, and what actually happened.

| Bad | Good |
|---|---|
| "You said you fixed it, it's still broken" | "I ran `npm test` and the 'account lockout' case in `login.test.ts` still fails. Here's the error: `Expected status LOCKED, received ACTIVE`" |
| "I don't like this approach" | "You implemented retries as an infinite loop, but our policy is to retry at most 3 times on external API failures and then alert. See the 'external integrations' rule in `CLAUDE.md`." |

Give Claude **verifiable facts** — the error message, the failing test name, the policy that was violated — and it can start fixing from exactly that point instead of guessing. And if the same failure keeps recurring, that feedback is a candidate for a rule in CLAUDE.md (see section 6).

### Example: cleaning up context when the conversation runs long

All Claude remembers at any moment is the current conversation's context, and that context has a size limit. Carrying unrelated tasks through a single session keeps the earlier task's file contents and dead ends occupying that space, and accuracy degrades the further you go.

- **Start fresh with `/clear` when the task changes.** If you've finished a bug fix and are starting a new feature, wiping the slate beats continuing the thread. The context that matters is already in CLAUDE.md and the code (see section 1).
- **For long tasks, have Claude write intermediate state to a file.** A summary file, rather than the conversation history, becomes the next session's context — so the work continues even when the session doesn't.

```text
> Write what we've decided so far and what remains to TODO.md,
  so the next session can pick up from that file alone.

(in a new session)
> Read TODO.md and continue from there.
```

"One task per session" is good for the same reason as "one purpose per PR" above — the clearer the boundary, the more accurate the result. And note that TODO.md and CLAUDE.md hold different things: progress that only applies to the current task goes in TODO.md; rules that should apply to every session go in CLAUDE.md (see section 6).

## 6. Continuous Improvement

Onboarding is not a one-time event. Competence comes from accumulation.

- **When the same mistake repeats, write it as a rule in CLAUDE.md.** One line like "never commit without tests" or "this module follows pattern X" applies to every future session.
- **Turn repeatedly explained procedures into skills.** The cost of explaining drops to zero.
- **Manage docs together with code.** When the structure changes, update CLAUDE.md in the same PR. Stale docs are worse than no docs.

### Example: turning mistakes into rules

When Claude repeats a mistake, add one line to CLAUDE.md right then. Recording the "why" behind the rule helps not just the next session's Claude but human teammates understand the context.

```markdown
## Caveats

- After modifying a DB migration file, always verify locally with `npm run migrate:test`
  (a merge without verification once broke the staging DB)
```

The cost of adding one line is small, but the benefit of never repeating the same correction compounds across every future session.

### Example: capturing rules mid-conversation with the `#` shortcut

The reason "add one line right then" rarely happens in practice is simple — it means breaking your flow to open and edit a file, so it becomes "I'll write it down later," and later never comes. In Claude Code, starting your input with `#` adds that content straight into a memory file (CLAUDE.md).

```text
> # After modifying a DB migration, always verify with npm run migrate:test

→ Pick which memory file to save to (project CLAUDE.md / personal
  settings, etc.) and it is appended as one line. Your work continues
  uninterrupted.
```

When the moment you point out a mistake and the moment you record the rule become the same moment, the friction of accumulation drops to nearly zero. Asking at the end of a session "is there anything from this session worth adding to CLAUDE.md?" is a habit with the same goal. If the heart of chapter 6 is accumulation, the biggest enemy of accumulation is "later."

### Example: updating docs in the same PR as the code

When a PR that changes structure also carries the doc update, the docs never get a chance to go stale. For example, a PR that moves a REST handler to a GraphQL resolver should have a file list like this:

```text
PR: Migrate order queries to GraphQL

  src/api/orders.ts          (deleted)
  src/graphql/orders.ts      (added)
  tests/graphql/orders.ts    (added)
  CLAUDE.md                  (modified) ← now says "query APIs are written as resolvers in src/graphql/"
```

Reviewers only need one checklist item: "did the docs change too?" Deferring doc updates to a separate task means they are usually forgotten — and the next session's Claude reads the stale doc and tries to add code to the deleted `src/api/`.

### Example: putting CLAUDE.md on a diet — pruning stale rules

As rules accumulate, CLAUDE.md only ever grows. But the longer the document, the more the rules that really matter get buried under the ones that matter less — and a single rule that is no longer true erodes trust in the whole document. About once a quarter, ask three questions of each rule.

```text
For each rule:

1. Is it still true?
   → "never modify src/legacy/" — if legacy is already deleted,
     delete the rule too

2. Can a machine enforce it?
   → "import order: stdlib → external → internal" — move it into
     a lint rule (see chapter 3) and remove it from the doc

3. Has this rule actually helped in the last 3 months?
   → If not, it is either too obvious or its moment has passed —
     a candidate for deletion
```

You can even delegate the cleanup itself to Claude — ask "verify that each rule in CLAUDE.md still matches the current codebase," and it will find rules pointing at deleted directories or rules already enforced by lint, and report them as pruning candidates. When the accumulation from chapter 6 (adding) and the pruning in this example (removing) run together, CLAUDE.md is maintained by density, not length.

---

## Checklist

Check whether your project has the following in place.

- [ ] `CLAUDE.md` — project overview, commands, conventions, caveats
- [ ] Tests that run with one command (`npm test`, etc.)
- [ ] Linter/formatter configuration and how to run it
- [ ] Automated dependency installation (lockfile + setup script)
- [ ] Environment setup guidance such as `.env.example`
- [ ] A CI pipeline
- [ ] Permission settings in `.claude/settings.json`
- [ ] Recurring procedures turned into skills (optional)
- [ ] A habit of accumulating mistakes/rules in CLAUDE.md

Even half of this list makes a visible difference in the quality of Claude's output. And every item here is **just as good for human developers** — in the end, it is no different from building a good engineering environment.
