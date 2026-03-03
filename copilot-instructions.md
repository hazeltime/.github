# Hazeltime Organization — Copilot Instructions

## Git Workflow

- **Branch naming:** `ai/{task-slug}` for AI-generated work, `feature/{slug}` for human features, `fix/{slug}` for bugfixes, `chore/{slug}` for maintenance.
- **Never force-push** to shared branches (main, development, staging). Always merge via PR.
- **Commit messages:** imperative mood, max 72 chars subject. Body explains WHY, not WHAT. Include `Co-authored-by:` trailers when AI-assisted.
- **PRs:** one logical change per PR. Title matches the intent. Link to issue with `Closes #N` or `Fixes #N`.
- **Branch protection:** main/development/staging require PR + CI + code review. Never bypass.

## Code Quality

- **Minimal changes:** make the smallest possible edit to achieve the goal. Don't refactor unrelated code.
- **Don't break existing behavior:** run existing tests after changes. If tests fail, only fix failures caused by your changes.
- **Comments:** only where the WHY isn't obvious. No redundant comments restating what code does.
- **Error handling:** fail explicitly with clear messages. Never silently swallow errors.
- **Security:** never commit secrets, credentials, API keys, or tokens. Use environment variables or secret managers.

## Code Review

- Only flag issues that genuinely matter — bugs, security vulnerabilities, logic errors, performance regressions.
- Never comment on formatting, style preferences, or trivial naming.
- Every comment should include what's wrong AND how to fix it.

## Testing

- Run existing tests before and after changes to verify nothing breaks.
- Don't modify tests to make them pass unless the test expectation is genuinely wrong.