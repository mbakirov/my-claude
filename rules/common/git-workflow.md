# Git Workflow

## Branch Naming

Branch names must match the Jira issue code: `BCOR-123`, `DEOVRWEB-555`.
This ensures dev-server deployment works without domain name resolution or SSL issues.

## Commit Message Format
```
type (issue-code) Subject

body (optional)

footer (optional)
```

**Types:**
- `feat` — feature, common product issue
- `fix` — bugfix
- `sup` — optimisation, tech debt, artefacts
- `chore` — housekeeping, cleaning
- `ci` — continuous integration
- `test` — test coverage
- `docs` — documenting
- `style` — code style improvement

**Issue code:** Use the Jira issue code from the branch name, e.g. `feat (DEOVRWEB-555) New user login window`. For work not tied to a Jira issue, use `NO_ISSUE`.

**Subject rules:**
- Max 50 characters
- Imperative mood: "Fix", not "Fixed" or "Fixing"
- First letter capitalised
- No trailing period
- First commit in branch should use Jira issue summary; subsequent commits use short description of changes

**Body (optional):** Concisely explain *what* and *why*, not *how*. Developers can diff the code for implementation details.

**Footer (optional):** Links to related commits or pertinent references.

### Examples
```
# Minimal
feat (DEOVRWEB-555) New user login window
```
```
# Detailed
feat (DEOVRWEB-555) New user login window

- Added new model LoginModel
- Added new API endpoints /v2/login, /v2/register
- Changed login.tpl view

Login, API, DEOVR, Dao
```

Note: Attribution disabled globally via ~/.claude/settings.json.

## Pull Request Workflow

One pull request must resolve only one task/issue.

For features split into multiple Jira tasks (e.g. a user story with subtasks or an epic with linked issues):
1. Create a *feature master branch* from `master` for the user story/epic
2. Create sub-branches for each subtask from the *feature master branch*
3. All subtask PRs target the *feature master branch*, not `master`

When creating PRs:
1. Analyze full commit history (not just latest commit)
2. Use `git diff [base-branch]...HEAD` to see all changes
3. Draft comprehensive PR summary
4. Include test plan with TODOs
5. Push with `-u` flag if new branch

> For the full development process (planning, TDD, code review) before git operations,
> see [development-workflow.md](./development-workflow.md).
