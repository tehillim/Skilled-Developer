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

### Example: turning failure into useful feedback

When the result isn't what you expected, saying only "it doesn't work" forces Claude to guess at the cause all over again. Instead, share what you ran, what you expected, and what actually happened.

| Bad | Good |
|---|---|
| "You said you fixed it, it's still broken" | "I ran `npm test` and the 'account lockout' case in `login.test.ts` still fails. Here's the error: `Expected status LOCKED, received ACTIVE`" |
| "I don't like this approach" | "You implemented retries as an infinite loop, but our policy is to retry at most 3 times on external API failures and then alert. See the 'external integrations' rule in `CLAUDE.md`." |

Give Claude **verifiable facts** — the error message, the failing test name, the policy that was violated — and it can start fixing from exactly that point instead of guessing. And if the same failure keeps recurring, that feedback is a candidate for a rule in CLAUDE.md (see section 6).

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
