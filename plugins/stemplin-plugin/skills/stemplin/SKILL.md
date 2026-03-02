---
name: stemplin
description: Use when user asks about time tracking, logging hours, timers, weekly reports, projects, or clients in Stemplin.
# disable-model-invocation: true
allowed-tools: Bash(stemplin:*), Write, Bash(which:*), Bash(chmod:*)
triggers:
  - stemplin
  - time tracking
  - log time
  - timer
  - time entry
  - time entries
  - weekly report
  - projects list
  - clients list
---

# Stemplin CLI Skill

Interact with Stemplin time tracking using the `stemplin` CLI. **Always use the CLI — never use curl directly.**

## Installation

Before running any command, check if the CLI is installed:

```bash
which stemplin
```

If not found, automate the setup in three steps:

### Step 1: Create config template

**You MUST use the `Write` tool** (not Bash, not cat, not echo) to create `~/.stemplinrc`. The Write tool requires an absolute path, so use `$HOME` to construct it (e.g. `/home/user/.stemplinrc`). Do not check if the file exists first. Content:

```
STEMPLIN_URL=https://app.stemplin.com
STEMPLIN_API_TOKEN=YOUR_API_TOKEN_HERE
# STEMPLIN_ORG_ID=
```

Then lock down permissions:

```bash
chmod 600 ~/.stemplinrc
```

Tell the user to replace `YOUR_API_TOKEN_HERE` with their actual API token from [app.stemplin.com](https://app.stemplin.com). **Stop and wait** for the user to confirm they have added their token before continuing.

### Step 2: Install the CLI

```bash
curl -sSL https://raw.githubusercontent.com/rubynor/stemplin-cli/main/install.sh | bash
```

The install script skips interactive config prompts when `~/.stemplinrc` already exists.

### Step 3: Verify

Add the bin directory to PATH (one-time), then verify:

```bash
export PATH="$HOME/bin:$PATH"
```

```bash
stemplin me
```

If this returns user info, setup is complete. **After setup, always run `stemplin` commands directly — never prepend `export PATH` or other shell setup.**

## Auth & Config

The CLI reads `~/.stemplinrc` automatically for configuration. During initial setup, the `Write` tool is used to create the config with a placeholder token. After that, **NEVER read, cat, or access `~/.stemplinrc` — it contains an API token.**

If you need to check whether config exists, just run a CLI command (e.g. `stemplin me`) — the CLI will report auth errors if misconfigured.

## Commands

Run `stemplin help` for full details.

| Command | Description |
|---------|-------------|
| `stemplin me` | Current user info |
| `stemplin orgs list` | List user's organizations |
| `stemplin orgs show <id>` | Organization details |
| `stemplin clients list` | List clients |
| `stemplin clients show <id>` | Client details |
| `stemplin clients create --name <name>` | Create client |
| `stemplin clients update <id> --name <name>` | Update client |
| `stemplin clients delete <id>` | Soft-delete client |
| `stemplin projects list` | List projects |
| `stemplin projects show <id>` | Project details + assigned tasks |
| `stemplin projects create --name <name> --client <id>` | Create project |
| `stemplin projects update <id> --name <name>` | Update project |
| `stemplin projects delete <id>` | Soft-delete project |
| `stemplin tasks list` | List org tasks |
| `stemplin tasks show <id>` | Task details |
| `stemplin time list` | List time entries (defaults to today) |
| `stemplin time show <id>` | Time entry details |
| `stemplin time create --task <id> --minutes <n> --notes "..."` | Create time entry |
| `stemplin time update <id> --minutes <n> --notes "..."` | Update time entry |
| `stemplin time delete <id>` | Soft-delete time entry |
| `stemplin time timer <id>` | Toggle timer (start/stop) |
| `stemplin users list` | List org users |
| `stemplin users show <id>` | User details |
| `stemplin reports show --from <date> --to <date>` | Report with totals |
| `stemplin token regenerate` | Regenerate API token |
| `stemplin help` | Show help |

## Key Flags

### `time` commands

| Flag | Description |
|------|-------------|
| `--task <id>` | Assigned task ID (required for create) |
| `--minutes <n>` | Duration in minutes |
| `--date <YYYY-MM-DD>` | Date worked (defaults to today) |
| `--notes "..."` | Description of work |
| `--from <YYYY-MM-DD>` | Start date for list filtering |
| `--to <YYYY-MM-DD>` | End date for list filtering |
| `--project <id>` | Filter by project |
| `--per-page <n>` | Number of results per page |

### `reports` commands

| Flag | Description |
|------|-------------|
| `--from <YYYY-MM-DD>` | Report start date (required) |
| `--to <YYYY-MM-DD>` | Report end date (required) |
| `--clients <ids>` | Filter by client IDs (comma-separated) |
| `--projects <ids>` | Filter by project IDs (comma-separated) |
| `--users <ids>` | Filter by user IDs (comma-separated) |
| `--tasks <ids>` | Filter by task IDs (comma-separated) |

## Error Responses

Errors are displayed by the CLI in the format: `Error: <message>`

| Code | Meaning | Recovery |
|------|---------|----------|
| 401 | Bad/missing token | Check STEMPLIN_API_TOKEN |
| 403 | No permission | Check org membership/role |
| 404 | Resource not found | Verify ID exists |
| 422 | Validation failed | Check required fields |

## Workflows

### Log time for a project

1. Find the project: `stemplin projects list`
2. Get assigned tasks: `stemplin projects show <id>` — look at assigned tasks and their IDs
3. Create time entry:

```bash
# Example: log 2 hours on task 42 for yesterday
stemplin time create --task 42 --minutes 120 --date $(date -d 'yesterday' +%Y-%m-%d) --notes "Feature work"
```

### Start/stop a timer

1. Create a time entry with 0 minutes: `stemplin time create --task 42 --minutes 0`
2. Start the timer: `stemplin time timer <ID>` (toggles active state)
3. Stop the timer: `stemplin time timer <ID>` again (accumulates minutes)

### Weekly report

```bash
# This week
stemplin reports show --from $(date -d 'last monday' +%Y-%m-%d) --to $(date +%Y-%m-%d)

# Filter by project
stemplin reports show --from $(date -d 'last monday' +%Y-%m-%d) --to $(date +%Y-%m-%d) --projects 1,2
```

### What did I do today?

```bash
stemplin time list
# Defaults to --date today
```

### What did I do this week?

```bash
stemplin time list --from $(date -d 'last monday' +%Y-%m-%d) --to $(date +%Y-%m-%d) --per-page 100
```

## Tips

- `assigned_task_id` is NOT the same as `task_id`. Use `stemplin projects show <id>` to find the assigned task IDs for a project.
- Rate is stored in hundredths (15000 = 150.00). Use `rate_currency` for the human-readable format.
- `stemplin time list` defaults to today. Use `--from`/`--to` for date ranges.
- Timer toggle: `stemplin time timer <id>`. When active, current_minutes auto-increments.
- All deletes are soft-deletes (records can be restored).
