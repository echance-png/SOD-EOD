# AI Agent Setup Instructions

> **You are the intended reader of this file.** You're an AI coding agent and your user wants to install the SOD/EOD toolkit. Follow the steps below.

### Step 0 — Confirm before proceeding

Before executing any setup steps, ask the user: **"I've read the SOD/EOD toolkit setup guide. Want me to run through the setup now?"** Do not proceed until they confirm.

### Step 1 — Identify which agent you are

Determine your agent type to know where files should be installed:

| Agent | Commands dir | Templates dir | Config dir |
|-------|-------------|---------------|------------|
| Claude Code | `~/.claude/commands/` | `~/.claude/templates/` | `~/.claude/` |
| OpenCode | `~/.claude/commands/` | `~/.claude/templates/` | `~/.config/opencode/` |
| Hermes Agent | `~/.hermes/skills/` | `~/.hermes/skills/` | `~/.hermes/` |
| Cursor | Ask the user | Ask the user | Ask the user |
| Other | Ask the user | Ask the user | Ask the user |

### Step 2 — Run the SOD onboarding questionnaire

Read [`sod/ONBOARDING.md`](sod/ONBOARDING.md) and walk the user through it **interactively, one question at a time**. This will collect:

- Their name and location (for weather/timezone)
- Which integrations to enable (calendar, GitHub, news, etc.)
- API keys and preferences
- Display and delivery preferences

At the end, generate their personalized config file and install the SOD command + template to the correct directories from Step 1.

### Step 3 — Install the EOD command

Read [`eod/README.md`](eod/README.md) for full setup details. Ask the user which mode they want:

- **Session mode** — automatically captures work from your coding session (requires OpenCode or Claude Code)
- **Journal-only mode** — the user describes their day and you format it (works with any agent)

Install the EOD command and config to the correct directories from Step 1.

### Step 4 — Verify

Run a test invocation of `/sod` (or the equivalent command for your agent). If it produces an HTML file and opens it in the user's browser, setup is complete. If it fails, check the config file for missing API keys or paths.

### Step 5 — Schedule (optional)

Offer to set up automated cron jobs:

- **SOD** at `0 8 * * 1-5` (8:00 AM weekdays)
- **EOD** at `0 18 * * 1-5` (6:00 PM weekdays)

Confirm the user's preferred times and timezone before installing.
