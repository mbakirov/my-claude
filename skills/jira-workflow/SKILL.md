---
name: jira-workflow
description: >
  Automate Jira-driven development workflows using the `jira-cli` tool (https://github.com/ankitpokhrel/jira-cli).
  Use this skill whenever the user says things like "fix issue BE-1234", "work on PROJ-567", "pick up ticket XYZ-99",
  "start working on <issue-key>", "grab issue <key>", "implement <key>", "resolve <key>", or any variation where a
  Jira issue key is mentioned alongside an intent to code against it. Also trigger when the user asks to "check my
  Jira tickets", "list my issues", "transition issue to Done", "move ticket to In Progress", "assign issue to me",
  "comment on ticket", or any other Jira CLI operation. This skill covers the full lifecycle: loading issue context,
  branch management (create/checkout with issue key), solving the issue, transitioning status, and posting comments.
  If the user mentions `jira` CLI commands, issue keys matching the pattern [A-Z]+-\d+, or development workflows
  that involve Jira — use this skill.
---

# Jira-Driven Development Workflow

This skill uses the [`jira-cli`](https://github.com/ankitpokhrel/jira-cli) tool to drive a
complete development workflow: load a Jira issue into context, manage git branches, solve the
issue, and update Jira status.

## Prerequisites

Before using any commands, verify the environment:

```bash
# Check jira-cli is installed and authenticated
jira me 2>/dev/null || echo "JIRA_CLI_NOT_CONFIGURED"

# Check we're in a git repo
git rev-parse --is-inside-work-tree 2>/dev/null || echo "NOT_A_GIT_REPO"
```

If `jira me` fails, tell the user they need to:
1. Install jira-cli: `brew install ankitpokhrel/jira-cli/jira-cli` (or see repo for other methods)
2. Run `jira init` to configure their project
3. Set `JIRA_API_TOKEN` environment variable

If not in a git repo, ask the user which repo to work in.

---

## Core Workflow: "Fix issue KEY-123"

When the user says something like **"Fix issue BE-1234"**, **"work on PROJ-567"**, or
**"pick up KEY-99"**, execute the following steps **in order**. Do NOT skip the issue-loading
step — you need the context to solve the issue properly.

### Step 1: Extract the Issue Key

Parse the issue key from the user's message. Jira issue keys match the pattern `[A-Z][A-Z0-9]+-\d+`.
If the user provides only a number (e.g., "fix 1234"), check if the jira-cli has a default project
configured and prepend it:

```bash
# Get the configured default project key
jira me  # verifies auth works
```

If ambiguous, ask the user for the full key.

### Step 2: Load Issue into Context

Fetch the full issue details so you understand **what needs to be done**:

```bash
# Get the human-readable view (best for understanding the issue)
jira issue view KEY-123 --plain 2>&1

# If you need structured data (JSON), use --raw
jira issue view KEY-123 --raw 2>&1
```

**What to extract from the issue:**
- **Summary**: The title / what needs to be done
- **Description**: Detailed requirements, acceptance criteria, reproduction steps
- **Issue Type**: Bug, Story, Task, Sub-task — affects how you approach the fix
- **Priority**: Helps decide thoroughness of solution
- **Status**: Current workflow state (if already "In Progress", skip transition)
- **Labels / Components**: Hints about which part of the codebase is affected
- **Subtasks**: If present, understand the breakdown
- **Linked Issues**: Related context, blockers, duplicates
- **Comments**: Often contain crucial context, clarifications, or prior attempts

**Critical**: Read the ENTIRE issue output carefully. Comments often contain the most important
context — stack traces, reproduction steps, architectural decisions, or "actually, ignore the
description, the real problem is X".

### Step 3: Branch Management

Create or switch to a branch named after the issue key. The branch naming convention uses
the **lowercase issue key** as the branch name (or prefix). Check all three locations:
local branches, remote branches, and worktrees.

```bash
ISSUE_KEY="KEY-123"
# Normalize to lowercase for branch name
BRANCH_NAME=$(echo "$ISSUE_KEY" | tr '[:upper:]' '[:lower:]')

# --- Option A: Check if branch already exists locally ---
if git show-ref --verify --quiet "refs/heads/$BRANCH_NAME"; then
    echo "LOCAL_BRANCH_EXISTS"
    git checkout "$BRANCH_NAME"

# --- Option B: Check if branch exists on remote ---
elif git ls-remote --heads origin "$BRANCH_NAME" 2>/dev/null | grep -q "$BRANCH_NAME"; then
    echo "REMOTE_BRANCH_EXISTS"
    git fetch origin "$BRANCH_NAME"
    git checkout -b "$BRANCH_NAME" "origin/$BRANCH_NAME"

# --- Option C: Create new branch from default branch ---
else
    echo "CREATING_NEW_BRANCH"
    # Determine the default branch (main or master)
    DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
    if [ -z "$DEFAULT_BRANCH" ]; then
        DEFAULT_BRANCH=$(git branch -r | grep -E 'origin/(main|master)$' | head -1 | sed 's@.*origin/@@')
    fi
    if [ -z "$DEFAULT_BRANCH" ]; then
        DEFAULT_BRANCH="main"
    fi

    # Make sure we have the latest
    git fetch origin "$DEFAULT_BRANCH" 2>/dev/null
    git checkout -b "$BRANCH_NAME" "origin/$DEFAULT_BRANCH"
fi

echo "Now on branch: $(git branch --show-current)"
```

**Branch name variations**: Some teams use conventions like `feature/KEY-123-short-desc`,
`bugfix/KEY-123`, or `KEY-123/short-description`. Before creating a new branch, check if
any existing branch contains the issue key:

```bash
# Check for any branch containing the issue key (case-insensitive)
git branch -a | grep -i "$ISSUE_KEY" || echo "NO_MATCHING_BRANCHES"
```

If a matching branch exists with a different naming convention, use that branch instead
of creating a new one. If multiple matches exist, ask the user which one to use.

### Step 4: Transition Issue to "In Progress"

Move the issue to "In Progress" (or equivalent) if it's not already there:

```bash
# Check current status first (already loaded in Step 2, but verify)
CURRENT_STATUS=$(jira issue view KEY-123 --raw 2>/dev/null | python3 -c "
import sys, json
data = json.load(sys.stdin)
print(data['fields']['status']['name'])
" 2>/dev/null)

echo "Current status: $CURRENT_STATUS"

# Only transition if not already in progress
if [ "$CURRENT_STATUS" != "In Progress" ]; then
    jira issue move KEY-123 "In Progress" 2>&1
fi
```

**Important**: The transition name is workflow-dependent. "In Progress" is common but your
project might use "Start Progress", "In Development", "Doing", etc. If the move fails,
list available transitions:

```bash
# The move command will show available transitions interactively if the state is wrong.
# Alternatively, use --raw on the issue to see available transitions.
# When move fails, try without quotes or with different state names.
```

### Step 5: Assign to Self (if unassigned)

```bash
# Assign the issue to yourself
jira issue assign KEY-123 $(jira me) 2>&1
```

### Step 6: Solve the Issue

Now that you have:
- The full issue context (summary, description, comments, subtasks)
- A working branch
- The issue in "In Progress" status

**Proceed to actually solve the issue.** Use the issue description, type, and comments to
guide your approach:

- **Bug**: Look for reproduction steps, stack traces, affected components. Find the root cause,
  fix it, and add tests.
- **Story/Feature**: Implement per acceptance criteria in the description. Check subtasks
  for breakdown.
- **Task**: Follow the instructions in the description.
- **Sub-task**: Implement the specific piece described; check the parent issue for broader context.

### Step 7: Post-Fix Actions (after the code changes are complete)

After solving the issue and the user is satisfied:

```bash
# Add a comment summarizing what was done
jira issue comment add KEY-123 "Fixed via branch $BRANCH_NAME. Changes: <brief summary of changes>" --no-input 2>&1

# Transition to review/done if the user asks
jira issue move KEY-123 "In Review" 2>&1
# or
jira issue move KEY-123 "Done" 2>&1
```

Do NOT auto-transition to Done unless the user explicitly asks. The typical flow is:
In Progress → In Review → Done.

---

## Other Common Operations

Read `references/commands.md` for the full command reference when you need to perform
operations beyond the core workflow, such as:

- Listing and searching issues
- Creating new issues
- Epic and sprint management
- Worklog tracking
- Issue linking and cloning
- Bulk operations via scripting

---

## Important Flags for Non-Interactive Use

Since Claude Code runs non-interactively, always use these flags where applicable:

| Flag | Purpose |
|------|---------|
| `--no-input` | Skip interactive prompts (for create, edit, comment) |
| `--plain` | Plain text output instead of TUI (for list commands) |
| `--no-headers` | Skip table headers in plain mode |
| `--raw` | Raw JSON output (for programmatic parsing) |
| `--columns key,summary,status,assignee` | Select specific columns in list output |

**Never** run `jira issue list` without `--plain` — the interactive TUI will hang in a
non-interactive terminal.

---

## Error Handling

| Error | Likely Cause | Fix |
|-------|-------------|-----|
| `jira: command not found` | Not installed | Install via brew/go/binary |
| `401 Unauthorized` | Bad or expired token | Re-export `JIRA_API_TOKEN` |
| `404 Not Found` | Wrong issue key or project | Verify key format and project access |
| `transition not found` | Wrong status name | List transitions with `jira issue move KEY-123` (it shows options) |
| Hangs / no output | Interactive mode in non-TTY | Add `--plain` or `--no-input` flag |

---

## Configuration Notes

- Config lives at `~/.jira/.config.yml`
- Per-project config: use `JIRA_CONFIG_FILE` env var or `-c` flag
- The `project.key` in config sets the default project prefix
- `jira me` returns the configured login (email/username)
