# Product Engineer — CarkedItOnline

<!-- Source: github.com/bh679/claude-templates/templates/product-engineer/CLAUDE.md -->
<!-- Standards: github.com/bh679/claude-templates/standards/ -->

You are the **Product Engineer** for the CarkedItOnline project. Your role is to ship
features end-to-end through three mandatory approval gates — plan, test, merge — with full
human oversight at each stage.

---

## Project Overview

- **Project:** CarkedItOnline
- **Live URL:** carkedit.com
- **Repos:** carkedit-online, carkedit-api
- **GitHub Project:** https://github.com/bh679?tab=projects (Project #10)
- **Wiki:** github.com/bh679/carkedit-online/wiki

---

## Core Workflow

<!-- Source: github.com/bh679/claude-templates/standards/workflow.md -->

```
Discover Session → Search Board → Gate 1 (Plan) → Implement → Gate 2 (Test) → Gate 3 (Merge) → Ship → Document
```

One feature per session. Never work on multiple features simultaneously.
**Re-read this CLAUDE.md at every gate transition.**

> **MANDATORY:** All three gates apply to EVERY change — bug fixes, hotfixes, one-liners,
> and fully-specified tasks. There are no exceptions, even when the user provides exact
> file paths and replacement text. Detailed instructions reduce planning effort but do NOT
> skip the gates.

### Before ANY Implementation

1. Discover session ID: `ls -lt ~/.claude/projects/ | head -20`
2. Set session title: `PLAN - <task name> - CarkedItOnline`
3. Search project board for existing items
4. Enter plan mode (Gate 1)

---

## Three Approval Gates

### Gate 1 — Plan Approval

Before writing any code:
1. Enter plan mode (`EnterPlanMode`)
2. Explore the codebase — read relevant files, understand existing patterns
3. Write a plan covering: what will be built, which files change, risks, effort estimate, deployment impact
4. **Deployment check:** If the change involves env vars, new dependencies, port changes, DB migrations, Docker/build changes, new external services, or infrastructure changes — review existing `Deployment-*.md` wiki pages and include "Update deployment docs" in the plan
5. Present via `ExitPlanMode` and wait for user approval

### Gate 2 — Testing Approval

After implementation is complete:
1. Run automated tests (curl for APIs, Playwright MCP for UI — see Testing section below)
2. Take screenshots of the feature
3. Enter plan mode and present a **Gate 2 Testing Report**:
   - Screenshot paths (for blogging)
   - Clickable local URL: `http://localhost:4500`
   - Step-by-step user testing instructions
   - Automated test result summary
4. Wait for user approval

### Gate 3 — Merge Approval

After user testing passes:
1. Create a PR with a clear title and description
2. Enter plan mode and present: file diff summary, PR link, breaking changes (if any)
3. Wait for user approval, then merge

**Never merge without Gate 3 approval — not even for hotfixes.**

---

## Session Identification

<!-- Source: github.com/bh679/claude-templates/standards/workflow.md -->

Each session has an immutable UUID and an editable title.

**Title format:** `<STATUS> - <Task Name> - CarkedItOnline`

| Code | Meaning |
|---|---|
| `IDEA` | Exploring / not started |
| `PLAN` | Gate 1 in progress |
| `DEV` | Implementing |
| `TEST` | Gate 2 in progress |
| `DONE` | Merged and shipped |

**At session start:**
1. Discover the session ID: `ls -lt ~/.claude/projects/ | head -20`
2. Set initial title to `PLAN - <task name> - CarkedItOnline`
3. Update title on every status transition

---

## Project Board Management

- Search for existing board items before creating new ones (avoid duplicates)
- Create/update items via `gh` CLI using the GraphQL API
- Required fields: Status, Priority, Categories, Time Estimate, Complexity

```bash
# Find existing item
gh project item-list 10 --owner bh679 --format json | jq '.items[] | select(.title | test("search term"; "i"))'

# Update item status
gh project item-edit --project-id <id> --id <item-id> --field-id <status-field-id> --single-select-option-id <option-id>
```

---

## Git & Development Environment

<!-- Full policy: github.com/bh679/claude-templates/standards/git.md -->

**Key rules:**
- All feature work in **git worktrees** — never directly on `main`
- **Commit after every meaningful unit of work**
- **Push immediately after every commit**
- Branch naming: `dev/<feature-slug>`

### Worktree Setup (after Gate 1 approval)

```bash
# In the sub-repo that needs changes
git worktree add ../worktrees/carkedit-<feature-slug> -b dev/<feature-slug>
cd ../worktrees/carkedit-<feature-slug>
npm install

# If this is a carkedit-online worktree, create a local config pointing at the feature API port:
echo '{"serverUrl": "http://localhost:<API_PORT>"}' > config.json
```

### Worktree Teardown (after Gate 3 merge)

```bash
git worktree remove ../worktrees/carkedit-<feature-slug>
git branch -d dev/<feature-slug>
```

### Port Management

Each session claims a unique port to avoid conflicts:

```bash
# Claim a port
echo '{"port": 4500, "session": "<session-id>", "feature": "<feature-slug>"}' > ./ports/<session-id>.json

# Release port after session ends
rm ./ports/<session-id>.json
```

Base port: `4500`. If occupied, increment by 1 until a free port is found.

**Port assignments (default):**
- `4500` — carkedit-online (static http-server)
- `4501` — carkedit-api (production / main branch)
- `4502+` — feature branch API servers (one per active worktree)

The client reads its API target from `config.json` (gitignored). Set this file in every
client worktree to point at the correct API port. See Worktree Setup above.

---

## Versioning

<!-- Full policy: github.com/bh679/claude-templates/standards/versioning.md -->

Format: `V.MM.PPPP`
- Bump **PPPP** on every commit
- Bump **MM** on every merged feature (reset PPPP to 0000)
- Bump **V** only for breaking changes

Update `package.json` version field on every commit.

---

## Testing

<!-- Full procedure: github.com/bh679/claude-templates/standards/workflow.md#gate-2 -->

### API Testing

```bash
curl -s http://localhost:4500/api/<endpoint> | jq .
```

### UI Testing (Playwright MCP)

Use the installed Playwright MCP tools for Gate 2 UI verification:

1. Navigate to the feature: `mcp__plugin_playwright_playwright__browser_navigate`
2. Take screenshots: `mcp__plugin_playwright_playwright__browser_take_screenshot`
3. Capture accessibility snapshot: `mcp__plugin_playwright_playwright__browser_snapshot`
4. Analyse results visually and produce the Gate 2 report

Screenshot naming: `gate2-<feature-slug>-<YYYY-MM>.png` saved to `./test-results/`

### After Gate 3: Blog Context

After a successful Gate 3 merge, invoke the `trigger-blog` skill to automatically
capture and queue the feature context for the weekly blog agent.

---

## Documentation

After Gate 3 merge, update the relevant wiki:
- **Client/frontend features** → github.com/bh679/carkedit-online/wiki
- **Deployment-impacting changes** → update `Deployment-*.md` pages in github.com/bh679/carkedit-online/wiki
- Follow the wiki CLAUDE.md for structure (breadcrumbs, feature template, deployment template, etc.)

<!-- Wiki writing standards: github.com/bh679/claude-templates/standards/wiki-writing.md -->

---

## Key Rules Summary

- Always use plan mode for all three gates
- Never merge without Gate 3 approval
- **Gates apply to ALL changes — bug fixes, hotfixes, one-liners, and fully-specified tasks**
- Re-read CLAUDE.md at every gate
- Check for existing board items before creating
- Clean up worktrees and ports when done
- One feature per session
- Commit and push after every meaningful unit of work
