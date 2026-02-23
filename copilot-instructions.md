# Hazeltime Organization — Copilot Instructions
# Applies to ALL repos, ALL Copilot surfaces (coding agent, Chat, CLI, code review)
# Stack-agnostic: no language, framework, or tooling specifics here.

## Git Workflow

- **Branch naming:** `ai/{task-slug}` for AI-generated work, `feature/{slug}` for human features, `fix/{slug}` for bugfixes, `chore/{slug}` for maintenance.
- **Never force-push** to shared branches (main, development, staging). Always merge via PR.
- **Commit messages:** imperative mood, max 72 chars subject. Body explains WHY, not WHAT. Include `Co-authored-by:` trailers when AI-assisted.
- **PRs:** one logical change per PR. Title matches the intent. Link to issue with `Closes #N` or `Fixes #N`.
- **Branch protection:** main/development/staging require PR + CI + code review. Never bypass.

## ⚡ Budget, Quota & Resource Efficiency (CRITICAL — READ FIRST)

This section governs ALL AI interactions across the org. Every token, API call, and compute cycle has cost. Be relentlessly frugal.

### Token & Context Management
- **Never read entire files** when you only need a few lines. Use line ranges, `head`, `tail`, or targeted grep.
- **Never re-read files already in context** this session. Work from memory. If you read a file once, remember its content.
- **Summarize, don't dump.** When reporting results, extract key findings. Never paste raw command output, logs, or file contents unless specifically asked.
- **Minimize response length.** Say what's needed, nothing more. No filler, no preamble, no restating what the user said.
- **Truncate early.** If a search returns 500 results, show the first 5-10 relevant ones. Use `head_limit`, `Select-Object -First`, or `| head`.
- **Avoid exploratory reading.** Know what you're looking for before opening a file. Don't browse — search.
- **Keep conversation history lean.** Don't repeat prior decisions, plans, or context that's already established. Reference it, don't restate it.
- **Use structured data (SQL, JSON) over prose** for tracking state, todos, batch items. Structured data is cheaper to query than re-reading paragraphs.

### API Call & Round-Trip Reduction
- **Batch ALL independent tool calls into a single response.** Reading 5 files? Make 5 parallel calls, NOT 5 sequential turns. This is the single biggest cost saver.
- **Chain shell commands** with `&&` or `;` to do multiple things in one invocation. `git status && git diff --stat && git log --oneline -5` is ONE call, not three.
- **Suppress verbose output** on every command. Use `--quiet`, `--no-pager`, `--short`, `-q`, `2>$null`. Never let a tool dump pages of output you won't use.
- **Use specific queries** over broad ones. `grep -n "functionName" src/file.ts` beats `grep -r "functionName" .` when you know the file.
- **Pre-filter before processing.** `glob` to find files, THEN `grep` only those files — don't search the entire tree.
- **One tool call per purpose.** Don't call the same tool twice for the same information. Cache results mentally.
- **Avoid polling loops.** When waiting for a command, use appropriate `initial_wait` (30-120s) and exponential backoff on reads (15s, 30s, 60s). Never tight-loop with 1-5s delays.

### Model Routing & Cost Tiers
- **Use the cheapest model that won't compromise the task:**
  - **Free:** in-memory knowledge, local tools (grep, glob, view, LSP), SQL session store
  - **Cheap:** explore agents (haiku), simple task agents (gpt-4.1), Context7 docs lookup, memory MCP
  - **Medium:** implementation agents (sonnet/codex), code review
  - **Expensive:** architecture/security review (opus), web search, Playwright browser
- **Never spawn a premium model for:** file reading, grep/search, simple edits, status checks, or mechanical refactoring.
- **Never spawn multiple agents when one suffices.** Batch related questions into ONE explore agent instead of 3 separate ones.
- **Limit parallel agents to 3-5** to avoid rate limiting and quota burn.
- **Don't use background mode then immediately block on read.** Only background a task if you'll do other work while it runs.

### Automation & Script Abstraction (MANDATORY)
- **If you run a command sequence more than twice, write a script.** Extract repeated shell pipelines, multi-step operations, and common workflows into reusable `.ps1`, `.sh`, or equivalent scripts.
- **Check for existing scripts before writing new ones.** Look in repo `scripts/`, `.github/`, `tools/`, or user `~/.config/ai/scripts/` first. Don't duplicate what exists.
- **Parameterize scripts.** Don't hardcode values. Use `param()` blocks, arguments, or environment variables so the same script works across repos and contexts.
- **Scripts over inline commands** for anything over 3 chained operations. A script is reusable; an inline chain is single-use token waste.
- **Cache expensive computation to disk.** If a build, analysis, or API call produces results you'll need again, write them to a temp file instead of re-running. Use `$env:TEMP/{purpose}-{identifier}` or equivalent.
- **Pre-compute in background.** If you know you'll need build output, test results, or analysis — start it as a background task while doing other work. Don't serialize everything.
- **Reuse CI/CD artifacts.** If CI already built/tested something, don't rebuild locally. Download artifacts or check run results instead.

### Memory & State Persistence
- **Use persistent memory (MCP) for facts that survive sessions:** architecture decisions, project state, active branches, deployment status. Don't re-discover what was already established.
- **Use SQL session database for operational tracking:** todos, batch progress, test results, file inventories. SQL is queryable and cheap; re-reading conversation history is expensive.
- **Use plan.md for multi-step work** so plans survive context compaction. After compaction, re-read plan.md — don't reconstruct from memory.
- **Write decisions to persistent storage immediately.** Don't defer writing to memory/plan "for later" — context may compact and you lose it.
- **Before starting work in any repo, query memory and session store** for prior context. Don't repeat discovery that was already done in a previous session.

### Resource-Aware Behavior
- **Check resource availability before consuming it.** Disk space before large operations. Port availability before servers. Process list before spawning duplicates.
- **Clean up after yourself.** Delete temp files, stop background processes, close browser tabs, remove debug artifacts when done.
- **Don't install packages globally** unless necessary. Prefer project-local installs. Don't leave system-wide side effects.
- **Fail fast.** If prerequisites are missing (wrong directory, missing tool, auth expired), detect and report immediately. Don't burn 10 tool calls discovering what one check would have caught.
- **Time-box exploration.** If you haven't found what you're looking for after 3-5 search attempts, stop and ask the user for guidance. Don't spiral into diminishing-return searches.

## AI Agent Orchestration

- **Delegation:** break complex work into independent sub-tasks. Parallelize where possible. But ONLY parallelize when tasks are truly independent — don't create 5 agents for work that requires sequential reasoning.
- **Background workers:** use async/background execution for builds, tests, lints. Don't block on long-running operations — check results when ready.
- **Model routing:** see cost tiers above. Match model to task complexity. Discovery/search → cheap. Implementation → medium. Architecture/security → premium.
- **Retry strategy:** if a cheaper model fails, escalate one tier. Never retry more than once at the same tier. Two failures at the same level = escalate or ask user.
- **Isolation:** each agent or task should work on its own branch. Never have two agents editing the same file simultaneously. Merge via orchestrator or PR.
- **Concurrency:** check for port conflicts before starting servers. Use unique temp paths per agent/session. Verify file isn't being edited by another process before writing.
- **Agent lifecycle:** spawn → do work → report result → terminate. Don't leave idle agents running. Don't spawn agents "just in case."
- **Prefer fewer, smarter agents** over many dumb ones. One well-prompted sonnet agent beats five poorly-prompted haiku agents.

## Code Quality (General)

- **Minimal changes:** make the smallest possible edit to achieve the goal. Don't refactor unrelated code.
- **Don't break existing behavior:** run existing tests after changes. If tests fail, only fix failures caused by your changes.
- **Comments:** only where the WHY isn't obvious. No redundant comments restating what code does.
- **Error handling:** fail explicitly with clear messages. Never silently swallow errors.
- **Security:** never commit secrets, credentials, API keys, or tokens. Use environment variables or secret managers.

## Code Review Preferences

- **Signal over noise:** only flag issues that genuinely matter — bugs, security vulnerabilities, logic errors, performance regressions.
- **Never comment on:** formatting, style preferences, trivial naming, or opinions that don't affect correctness.
- **Actionable feedback:** every comment should include what's wrong AND how to fix it.
- **Approve if correct:** don't block PRs for nitpicks. Approve with suggestions if minor.

## MCP & Tool Usage

- **Cost hierarchy (always follow this order):**
  1. **Free:** in-memory knowledge, grep, glob, view, LSP, SQL session store — use these FIRST
  2. **Free:** shell commands (git, build tools, file operations) — combine with `&&`
  3. **Low cost:** Context7 (library docs), memory MCP (persistent facts) — use when local tools can't answer
  4. **Medium cost:** sub-agents (explore/task) — batch related work into fewer agents
  5. **High cost:** web search, web fetch, Playwright browser — absolute last resort, only for information that doesn't exist locally
- **Context7:** use for library/framework documentation lookups. Don't use web_search or Playwright to read documentation that Context7 can provide.
- **Memory MCP:** read before starting work (free). Write significant decisions and state (low cost). Don't write trivial or transient data to memory.
- **Playwright:** web UI testing ONLY. Never use to browse documentation, read web pages, or gather information. Use web_fetch or Context7 instead.
- **Tool efficiency:** batch independent operations into parallel calls. Chain shell commands with `&&`. Suppress verbose output. Use `--quiet`, `--no-pager`, pipe to `head`/`grep` when appropriate.
- **MCP server health:** if an MCP server fails or times out, skip it and use fallback tools. Don't retry MCP calls more than once — the cost of retries exceeds the value.

## Testing

- **Run existing tests before and after changes** to verify nothing breaks.
- **Test what you change:** add or update tests for new/modified behavior.
- **Don't modify tests to make them pass** unless the test expectation is genuinely wrong.
- **Use existing test infrastructure** — don't introduce new test frameworks or tools unless explicitly requested.

## Documentation

- **Update docs when behavior changes.** If you change an API, update its docs. If you change a config, update examples.
- **Don't create planning/tracking docs in repos.** Use issues, PRs, and project boards instead.
- **README accuracy:** if a README references behavior you're changing, update it.

## CI/CD

- **Don't skip CI.** Every PR must pass checks before merge.
- **Fix your failures:** if CI fails on your changes, fix the root cause. Don't retry hoping for a different result.
- **Reusable workflows:** prefer existing shared workflows over duplicating pipeline logic.

## Agent Behavior

- **Ask before assuming** when scope is ambiguous, multiple valid approaches exist, or the change has significant architectural impact.
- **Work incrementally:** commit working states. Don't accumulate a massive diff before verifying.
- **Clean up:** remove temporary files, debug logging, and scratch work before finalizing.
- **Respect existing patterns:** match the conventions already in the codebase. Don't impose external style preferences.
