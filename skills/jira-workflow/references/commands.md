# jira-cli Command Reference

Full reference for all `jira-cli` commands. This file supplements the main SKILL.md.

## Table of Contents

1. [Issue Commands](#issue-commands)
2. [Epic Commands](#epic-commands)
3. [Sprint Commands](#sprint-commands)
4. [Release Commands](#release-commands)
5. [Other Commands](#other-commands)
6. [Scripting Patterns](#scripting-patterns)
7. [JQL Quick Reference](#jql-quick-reference)

---

## Issue Commands

### List Issues

```bash
# Basic list (ALWAYS use --plain in non-interactive contexts)
jira issue list --plain

# With specific columns
jira issue list --plain --columns key,summary,status,assignee,priority

# No table headers (for scripting)
jira issue list --plain --no-headers

# Raw JSON output
jira issue list --raw

# CSV output
jira issue list --csv

# Pagination: limit to N results
jira issue list --paginate 20 --plain

# Pagination: start from offset
jira issue list --paginate 10:50 --plain

# Custom delimiter
jira issue list --plain --delimiter "|"
```

#### Filtering Issues

```bash
# By assignee
jira issue list -a"user@email.com" --plain
jira issue list -a$(jira me) --plain          # Assigned to me
jira issue list -ax --plain                    # Unassigned

# By status
jira issue list -s"To Do" --plain
jira issue list -s"In Progress" --plain
jira issue list -s~Done --plain                # NOT Done (tilde = not)

# By priority
jira issue list -yHigh --plain
jira issue list -yCritical --plain

# By type
jira issue list -tBug --plain
jira issue list -tStory --plain
jira issue list -tTask --plain

# By label
jira issue list -lbackend --plain
jira issue list -lbackend -l"high-prio" --plain  # Multiple labels (AND)

# By reporter
jira issue list -r"user@email.com" --plain
jira issue list -r$(jira me) --plain           # Reported by me

# By component
jira issue list -CBackend --plain

# By creation date
jira issue list --created month --plain        # Created this month
jira issue list --created week --plain         # Created this week
jira issue list --created -7d --plain          # Created in last 7 days
jira issue list --created -1h --plain          # Created in last hour
jira issue list --created-before -24w --plain  # Created before 24 weeks ago

# By update date
jira issue list --updated -30m --plain         # Updated in last 30 minutes

# By resolution
jira issue list -R"Won't Do" --plain

# By project (override default)
jira issue list -pOTHER --plain

# Issues I'm watching
jira issue list -w --plain

# Recently viewed issues
jira issue list --history --plain

# Reverse order (oldest first)
jira issue list --reverse --plain

# Order by field
jira issue list --order-by rank --reverse --plain

# Raw JQL within project context
jira issue list -q "summary ~ cli" --plain
jira issue list -q "project IS NOT EMPTY" --plain  # All projects

# Combined filters
jira issue list -a$(jira me) -yHigh -s"In Progress" --created month -lbackend --plain
```

### View Issue

```bash
# Human-readable (use --plain for non-TTY)
jira issue view KEY-123 --plain

# Raw JSON (for parsing)
jira issue view KEY-123 --raw

# Show N recent comments
jira issue view KEY-123 --comments 5 --plain
```

### Create Issue

```bash
# Fully non-interactive
jira issue create -tBug -s"Bug summary" -yHigh -lbug -b"Bug description" --no-input

# With epic parent
jira issue create -tStory -s"Story summary" -PEPIC-42 --no-input

# With fix version
jira issue create -tTask -s"Task" --fix-version v2.0 -b"Description" --no-input

# With component
jira issue create -tBug -s"Bug" -CBackend -b"Description" --no-input

# Description from stdin
echo "Description body" | jira issue create -s"Summary" -tTask --no-input

# From template file
jira issue create --template /path/to/template.tmpl --no-input

# With custom fields (key=value format, see jira-cli discussions/346 for details)
jira issue create -tStory -s"Summary" --custom story-points=3 --no-input
```

### Edit Issue

```bash
# Edit summary
jira issue edit KEY-123 -s"New summary" --no-input

# Edit priority
jira issue edit KEY-123 -yHigh --no-input

# Edit description
jira issue edit KEY-123 -b"New description" --no-input

# Add and remove labels (minus prefix = remove)
jira issue edit KEY-123 --label -old-label --label new-label --no-input

# Add and remove components
jira issue edit KEY-123 --component -OldComponent --component NewComponent --no-input

# Add and remove fix versions
jira issue edit KEY-123 --fix-version -v1.0 --fix-version v2.0 --no-input
```

### Assign Issue

```bash
# Assign to specific user
jira issue assign KEY-123 "user@email.com"

# Assign to self
jira issue assign KEY-123 $(jira me)

# Assign to default assignee
jira issue assign KEY-123 default

# Unassign
jira issue assign KEY-123 x
```

### Transition / Move Issue

```bash
# Move to a specific status
jira issue move KEY-123 "In Progress"
jira issue move KEY-123 "Done"
jira issue move KEY-123 "In Review"

# Move with comment
jira issue move KEY-123 "In Progress" --comment "Starting work"

# Move with resolution and assignee
jira issue move KEY-123 Done -RFixed -a$(jira me)

# Open in browser after transition
jira issue move KEY-123 "In Progress" --web
```

Note: If the status name is wrong, the command will fail. Status names are
workflow-specific. Common names: "To Do", "In Progress", "In Review", "Done",
"Open", "Closed", "Backlog", "Selected for Development".

### Comment on Issue

```bash
# Add comment (non-interactive)
jira issue comment add KEY-123 "Comment body" --no-input

# Multiline comment
jira issue comment add KEY-123 $'Line 1\n\nLine 2\n\nLine 3' --no-input

# Internal comment (service desk)
jira issue comment add KEY-123 "Internal note" --internal --no-input

# Comment from stdin
echo "Comment from pipe" | jira issue comment add KEY-123 --no-input

# Comment from file
jira issue comment add KEY-123 --template /path/to/comment.md --no-input
```

### Link Issues

```bash
# Link two issues
jira issue link KEY-1 KEY-2 Blocks

# Common link types: Blocks, "is blocked by", Clones, "is cloned by",
# Duplicate, "is duplicated by", Relates
```

### Clone Issue

```bash
# Simple clone
jira issue clone KEY-123

# Clone with modifications
jira issue clone KEY-123 -s"Modified summary" -yHigh -a$(jira me)

# Clone with text replacement in summary and description
jira issue clone KEY-123 -H"find text:replace text"
```

### Delete Issue

```bash
# Delete (will prompt for confirmation)
jira issue delete KEY-123

# Delete with all subtasks
jira issue delete KEY-123 --cascade
```

### Worklog

```bash
# Add time log
jira issue worklog add KEY-123 "2d 3h 30m" --no-input

# Add worklog with comment
jira issue worklog add KEY-123 "1h" --comment "Code review" --no-input
```

---

## Epic Commands

```bash
# List epics
jira epic list --table --plain

# List issues in an epic
jira epic list EPIC-1 --plain

# Create epic
jira epic create -n"Epic Name" -s"Epic Summary" -yHigh -b"Description" --no-input

# Add issues to epic
jira epic add EPIC-1 KEY-2 KEY-3

# Remove issues from epic
jira epic remove KEY-2 KEY-3
```

---

## Sprint Commands

```bash
# List sprints
jira sprint list --table --plain

# Current sprint issues
jira sprint list --current --plain

# Current sprint, assigned to me
jira sprint list --current -a$(jira me) --plain

# Previous sprint
jira sprint list --prev --plain

# Next planned sprint
jira sprint list --next --plain

# Specific sprint by ID
jira sprint list SPRINT_ID --plain

# Add issues to sprint
jira sprint add SPRINT_ID KEY-1 KEY-2
```

---

## Release Commands

```bash
# List releases
jira release list --plain

# List releases for specific project
jira release list --project KEY --plain
```

---

## Other Commands

```bash
# Get current user
jira me

# Open project in browser
jira open

# Open specific issue in browser
jira open KEY-123

# List all accessible projects
jira project list --plain

# List boards
jira board list --plain

# Shell completion
jira completion --help
```

---

## Scripting Patterns

### Extract Issue Keys from List

```bash
# Get just the keys
jira issue list -a$(jira me) -s"To Do" --plain --columns key --no-headers
```

### Parse JSON Output

```bash
# Get status of an issue
jira issue view KEY-123 --raw | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f\"Status: {d['fields']['status']['name']}\")
print(f\"Summary: {d['fields']['summary']}\")
print(f\"Assignee: {d['fields']['assignee']['displayName'] if d['fields']['assignee'] else 'Unassigned'}\")
print(f\"Priority: {d['fields']['priority']['name']}\")
print(f\"Type: {d['fields']['issuetype']['name']}\")
"
```

### Batch Operations

```bash
# Move all my "To Do" issues to "In Progress"
jira issue list -a$(jira me) -s"To Do" --plain --columns key --no-headers | while read -r key; do
    jira issue move "$key" "In Progress" 2>&1
done

# Add label to multiple issues
for key in KEY-1 KEY-2 KEY-3; do
    jira issue edit "$key" --label new-label --no-input 2>&1
done
```

---

## JQL Quick Reference

Use with `jira issue list -q "..."`:

| JQL | Meaning |
|-----|---------|
| `summary ~ "search term"` | Summary contains text |
| `description ~ "text"` | Description contains text |
| `status = "In Progress"` | Exact status match |
| `status != Done` | Status is not Done |
| `assignee = currentUser()` | Assigned to me |
| `assignee is EMPTY` | Unassigned |
| `priority = High` | Priority filter |
| `created >= -7d` | Created in last 7 days |
| `updated >= -1h` | Updated in last hour |
| `labels in (backend, urgent)` | Has any of these labels |
| `component = Backend` | Component filter |
| `project = PROJ` | Specific project |
| `project IS NOT EMPTY` | All projects |
| `issuetype = Bug` | Type filter |
| `resolution = Unresolved` | Open issues |
| `sprint in openSprints()` | In active sprints |
| `ORDER BY created DESC` | Sort clause |

Combine with AND/OR: `status = "To Do" AND priority = High AND assignee = currentUser()`

---

## Config File Location

Default: `~/.jira/.config.yml`

Override with:
```bash
JIRA_CONFIG_FILE=./custom-config.yml jira issue list --plain
# or
jira issue list -c ./custom-config.yml --plain
```

Key config fields:
- `server` — Jira instance URL
- `login` — Username/email
- `project.key` — Default project key
- `project.type` — "classic" or "next-gen"
- `board.id` — Default board ID
- `board.type` — "scrum" or "kanban"
- `epic.name` — Epic name custom field
- `epic.link` — Epic link custom field
- `installation` — "cloud" or "local"
