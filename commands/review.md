---
description: 'Full multi-agent review of a diff, PR, or path. Usage: /review [path | PR number | "staged"]'
argument-hint: "[path | PR# | staged]"
---
Run a full MorviLabs review-crew review on: $ARGUMENTS (default: the current branch against the default branch).

You are the dispatcher. The specialists run as subagents, and a subagent cannot start other subagents, so routing and fan-out happen here in the main conversation. Do not review the code yourself.

## 1. Change set
| `$ARGUMENTS` | Diff command | File list |
|---|---|---|
| a PR number | `gh pr diff <n>` | `gh pr diff <n> --name-only` |
| `staged` | `git diff --cached` | `git diff --cached --name-only` |
| a path | review the path as a whole | `git ls-files <path>` |
| empty | `git diff <base>`: everything on this branch, committed or not | `git diff <base> --name-only`, plus untracked files from `git ls-files --others --exclude-standard` |

`<base>` is `git merge-base origin/HEAD HEAD`; fall back to `origin/main`, then `main`. A branch with no commits yet still gets reviewed this way.

Keep the exact command you used. Specialists rerun it themselves; never paste the diff into their prompts. Untracked files are in no diff: list them under Files marked `(new, untracked)` so specialists read them whole.
Name the target so the report says what was reviewed, e.g. `fix/x (committed + uncommitted vs origin/main)`.
Empty change set → say so in one line and stop.

## 2. Setup
- Load the `claude-review-crew:review-rubric` skill for severity levels, finding format, and verdict rules.
- Read `review.config.json` at the repo root if it exists. Defaults: `stack: auto`, `bilingual: false`, `openapi_spec: null`, `max_findings: 25`.
- Resolve the stack. If `stack` is a name or a list of names from `react`, `next`, `vue`, `node`, use it (`next` includes `react`). If it is `auto` or missing, detect it: read the root `package.json` and the nearest `package.json` above each touched file, then map their dependencies and take the union.

| Dependency | Stack |
|---|---|
| `next` | `next`, `react` |
| `react` | `react` |
| `vue`, `nuxt` | `vue` |
| `express`, `fastify`, `@nestjs/core`, `hono`, `koa` | `node` |

  Nothing matched → `none`; specialists then run their general checks only.

## 3. Route
Pick specialists from the file list:

| Specialist | Run when the change set touches |
|---|---|
| `review-correctness` | Any source file |
| `review-code-quality` | Any source file |
| `review-security` | Auth, session, middleware, API routes, server handlers or controllers, server actions, env or config files, anything that renders user input as HTML |
| `review-performance` | Components, hooks, composables, data fetching, server handlers, DB queries, ORM models |
| `review-api-contract` | API clients or services, route handlers or controllers, validation schemas or DTOs, generated API types, the OpenAPI spec, shared schema packages, MSW handlers |
| `review-tests` | Any source change (to check coverage) or any test file |
| `review-ux-a11y` | `.tsx` `.jsx` `.vue` `.svelte` `.html`, CSS or Tailwind classes, i18n or locale files |
| `review-architecture` | New, moved, or renamed files; imports that cross a feature, app, or package boundary; barrel `index` files; workspace or path-alias config; any review of a whole path |
| `review-dependencies` | Any `package.json`, a lockfile, `pnpm-workspace.yaml`, `.npmrc` |
| `review-production` | Routes and app or server entry points, error boundaries and error pages, env files or env schema, build config (`vite`, `next`, `nuxt`, `tsconfig`, `turbo`, Dockerfile), CI workflows, DB migrations |

MSW mock handlers and test fixtures are test code, not server handlers: on their own they route correctness, code-quality, tests, and api-contract, never security or performance.

Docs-only change (`*.md`, comments only) → route nothing. Skip step 4 and give the orchestrator an empty crew.

## 4. Dispatch
Launch every selected specialist in ONE message as parallel subagent calls. Plugin agents carry the plugin prefix: `claude-review-crew:review-correctness`, `claude-review-crew:review-security`, and so on. Give each exactly this brief:

```
Target: <branch, PR #, or path>
Diff command: <the exact command from step 1>
Files: <the file list>
Stack: <the resolved stack list, or none>
Config: bilingual=<…> openapi_spec=<…>
Return findings in your output format, or your exact no-findings line.
```

Wait for all of them. A specialist that errors → retry once, then list it as `<name> (failed)` under Skipped.

## 5. Merge and report
Launch `claude-review-crew:review-orchestrator` with:
- Target
- Stack: the resolved stack list
- Crew: the specialists that ran
- Skipped: specialists not routed, plus any that failed
- Config: `bilingual`, `max_findings`
- Every specialist's reply verbatim, each under a `### <specialist>` heading

Print the report it returns exactly as returned. Nothing before it, nothing after the footer.
