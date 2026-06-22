# SOD Onboarding — First-Time Setup

Welcome! This guide walks you through setting up your Start-of-Day (SOD) morning digest. Work through each section, filling in your details. At the end, you'll have a working `sod-config.json` and everything installed.

**Time estimate:** 10–15 minutes.

> **How this works:** For each integration (Jira, Slack, Calendar), we'll first ask if you already have it set up. If yes, we grab the details. If not, we skip it and circle back at the end — no need to stop the whole onboarding to configure an API key.

---

## Checklist

- [ ] [1. Your Name](#1-your-name)
- [ ] [2. Weather Location](#2-weather-location)
- [ ] [3. Wiki / Knowledge Base](#3-wiki--knowledge-base)
- [ ] [4. Jira Setup](#4-jira-setup)
- [ ] [5. Slack Setup](#5-slack-setup)
- [ ] [6. Google Calendar Setup](#6-google-calendar-setup)
- [ ] [7. Timezone](#7-timezone)
- [ ] [8. EOD Integration](#8-eod-carry-forward-integration)
- [ ] [9. Generate Config](#9-generate-your-config)
- [ ] [10. Install Files](#10-install-files)
- [ ] [11. Test Run](#11-test-run)
- [ ] [12. Circle Back — Deferred Integrations](#12-circle-back--deferred-integrations)

---

## 1. Your Name

The SOD greeting says "Good morning, [Name]." What should it call you?

```
My name: _______________
```

**Example:** `Alex`, `Jordan`, `Dr. Smith`

This is used in the HTML greeting and nowhere else — pick whatever feels right for a morning briefing.

---

## 2. Weather Location

SOD fetches weather from [wttr.in](https://wttr.in). What location should it use?

```
My location: _______________
```

**Format:** `City,ST` (US), `City,Country` (international), zip/postal codes, or airport codes.

**Examples:**
- `Seattle,WA`
- `London,UK`
- `Tokyo,Japan`
- `90210` (zip codes work)
- `SFO` (airport codes work too)

**Test it now:** Open `https://wttr.in/YOUR_LOCATION?format=j1` in a browser. If you see JSON weather data, you're good.

---

## 3. Wiki / Knowledge Base

SOD writes HTML digests and log entries to a local directory. This is where your morning briefings accumulate.

**Do you already have a wiki/docs directory?**

- [ ] **Yes** — I have an existing directory:
  ```
  My wiki root: _______________
  ```
  (e.g. `~/Documents/my-wiki`, `~/notes`, `~/obsidian-vault`)

- [ ] **No** — I want to create one. Run:
  ```bash
  mkdir -p ~/Documents/sod-wiki/{sod,eod}
  touch ~/Documents/sod-wiki/log.md
  ```
  Your wiki root is: `~/Documents/sod-wiki`

SOD will create/use these subdirectories:
- `<wiki_root>/sod/` — daily HTML briefings (one per day)
- `<wiki_root>/eod/` — where it looks for EOD reports (optional)
- `<wiki_root>/log.md` — append-only run log

---

## 4. Jira Setup

SOD can pull your sprint tickets from Jira Cloud.

**Do you already have an existing Jira integration?**

- [ ] **Yes, I have Jira set up** — Great! SOD supports two connection methods:

  **Option A: Jira MCP** (if you have Atlassian/Jira MCP configured in your AI agent)
  - SOD uses `jira_search` or `atlassian_searchJiraIssuesUsingJql` — no extra auth needed.

  **Option B: Jira API Token** (most common for teams)
  - You'll need an [Atlassian API token](https://id.atlassian.com/manage-profile/security/api-tokens).
  - Store it securely — **never paste secrets into chat or config files that get committed to git**.
  - Recommended: store in an environment variable (`JIRA_API_TOKEN`) or a secrets file (`~/.secrets/env`) that's excluded from version control.
  - SOD will use `curl` with your token to query the Jira REST API.

  Either way, fill in your project details:
  ```
  Jira project key:  _______________   (e.g. ENG, IT, PLATFORM)
  Jira board ID:     _______________   (e.g. 42, 177)
  Connection method: _______________   (MCP or API Token)
  ```

  **How to find your board ID:**
  1. Open your Jira board in a browser
  2. Look at the URL: `https://your-org.atlassian.net/jira/software/projects/ENG/boards/42`
  3. The number after `/boards/` is your board ID

  **Multiple projects?** SOD's default JQL queries one project at a time. Pick your primary board. You can extend the JQL in `sod.md` later to cover multiple projects (e.g. `project in (IT, ICM)`).

- [ ] **No, I don't have Jira set up yet** — No problem. We'll skip this for now and circle back in [Step 12](#12-circle-back--deferred-integrations). SOD will simply omit the sprint section.

---

## 5. Slack Setup

SOD can summarize your overnight DMs, filtering out noisy bots so you only see what matters.

**Do you already have an existing Slack integration?**

- [ ] **Yes, I have Slack MCP configured** — Great! SOD uses `slack_slack_search_public_and_private` to pull DMs.

  SOD filters out noisy bots (Jira notifications, calendar reminders, etc.) so your digest shows humans first. To configure filters, you need each bot's **Slack Member ID**.

  **How to find a Slack bot's Member ID:**
  1. Open a DM conversation with the bot
  2. Click the bot's name at the top of the conversation
  3. Click **"View full profile"** from the dropdown menu
  4. On the profile panel, click the **⋮ (more)** button
  5. Select **"Copy member ID"**

  > **Can't find "Copy member ID"?** Some Slack workspaces or bot types don't expose it in the UI. Alternative: open the bot's profile in a browser — the URL often contains the ID (e.g. `https://app.slack.com/client/T.../U0BOTIDHERE`). Or ask your Slack admin.

  Fill in the bots you want filtered (leave blank to configure later):
  ```
  Jira bot ID:       _______________   (action: collapse to 1-line summary)
  GCal bot ID:       _______________   (action: filter entirely)
  Automation bot ID: _______________   (action: show if content, hide if empty)
  Other noisy bot:   _______________   (action: collapse)
  ```

  Don't have all of these? That's fine — unknown bots get collapsed to a count line automatically. You can always add more filters later.

- [ ] **No, I don't have Slack set up yet** — No problem. We'll skip this for now and circle back in [Step 12](#12-circle-back--deferred-integrations). SOD will omit the Slack DMs section. You can also pass `--no-slack` at any time.

---

## 6. Google Calendar Setup

SOD can show your meetings for the day with "needs action" badges and Google Meet join links.

**Do you already have an existing Google Calendar integration?**

- [ ] **Yes, I have Google Calendar MCP configured** — Great! SOD uses `google-calendar_list-events` to pull today's events. No extra config needed — just confirm it works:
  ```
  Google Calendar: ✅ Ready
  ```

- [ ] **No, I don't have Google Calendar set up yet** — No problem. We'll skip this for now and circle back in [Step 12](#12-circle-back--deferred-integrations). SOD will omit the calendar section. You can also pass `--no-cal` at any time.

---

## 7. Timezone

SOD detects your timezone automatically via `date`, but it's good to confirm:

```
My timezone: _______________
```

**Examples:** `America/New_York`, `America/Denver`, `America/Los_Angeles`, `Europe/London`, `Asia/Tokyo`

Run `date '+%Z'` in your terminal to see your current timezone abbreviation.

---

## 8. EOD Carry-Forward Integration

SOD can surface open questions and ADR candidates from your most recent End-of-Day (`/eod`) report.

**Do you plan to use the EOD command alongside SOD?**

- [ ] **Yes** — SOD will look for `.md` files in `<wiki_root>/eod/` with these sections:
  ```markdown
  ## Open questions / followups
  - Bullet items here...

  ## ADR candidates
  - Bullet items here...
  ```
  If your EOD uses different section names, update Step 5 in `sod.md`.

- [ ] **No** — The carry-forward section will be cleanly omitted. You can add EOD later — SOD automatically picks it up once EOD files appear in `<wiki_root>/eod/`.

---

## 9. Generate Your Config

Based on your answers above, create your `sod-config.json`:

```bash
cp sod-config.example.json sod-config.json
```

Then edit `sod-config.json` with your values:

```json
{
  "user_name": "<your name from step 1>",
  "weather_location": "<your location from step 2>",
  "wiki_root": "<your wiki path from step 3>",
  "config_root": "<depends on your agent — see table below>",
  "jira_project": "<your project key from step 4, or remove this line>",
  "jira_board_id": "<your board ID from step 4, or remove this line>",
  "jira_connection": "<MCP or API_TOKEN>",
  "slack_bot_filters": {
    "jira": {"id": "<from step 5>", "action": "collapse"},
    "calendar": {"id": "<from step 5>", "action": "filter"},
    "automation": {"id": "<from step 5>", "action": "collapse_empty"}
  }
}
```

**`config_root` depends on your AI coding agent:**

| Agent | `config_root` |
|-------|---------------|
| Claude Code | `~/.claude` |
| OpenCode | `~/.config/opencode` |
| Hermes | `~/.hermes` |
| Other | Check your agent's documentation for its config directory |

**If you skipped integrations**, remove those keys from the JSON. SOD handles missing config gracefully — omitted integrations just mean omitted sections.

---

## 10. Install Files

### 10a. Install the command and template

The install paths depend on your AI coding agent. Copy the files to the appropriate locations:

| File | Claude Code / OpenCode | Hermes | Other |
|------|----------------------|--------|-------|
| `sod.md` (command) | `~/.claude/commands/sod.md` | `~/.hermes/skills/sod/sod.md` | Consult your agent's docs for custom command locations |
| `sod-template.html` (template) | `~/.claude/templates/sod-template.html` | `~/.hermes/skills/sod/sod-template.html` | Same directory as the command, or a templates folder |

**Example — Claude Code / OpenCode:**
```bash
mkdir -p ~/.claude/commands ~/.claude/templates
cp sod.md ~/.claude/commands/sod.md
cp sod-template.html ~/.claude/templates/sod-template.html
```

**Example — Hermes:**
```bash
mkdir -p ~/.hermes/skills/sod
cp sod.md ~/.hermes/skills/sod/sod.md
cp sod-template.html ~/.hermes/skills/sod/sod-template.html
```

### 10b. Replace placeholders in BOTH files

Open the installed `sod.md` **and** `sod-template.html` and do a find-and-replace for each placeholder:

| Find | Replace with | In which file(s) |
|------|-------------|-------------------|
| `{{USER_NAME}}` | Your name (e.g. `Alex`) | **Both** sod.md and sod-template.html |
| `{{WEATHER_LOCATION}}` | Your location (e.g. `Seattle,WA`) | **Both** sod.md and sod-template.html |
| `{{WIKI_ROOT}}` | Your wiki path (e.g. `~/Documents/my-wiki`) | sod.md only |
| `{{CONFIG_ROOT}}` | Your config root (see table in Step 9) | sod.md only |
| `{{JIRA_PROJECT}}` | Your Jira project key (e.g. `ENG`) | sod.md only |
| `{{JIRA_BOARD_ID}}` | Your Jira board ID (e.g. `42`) | sod.md only |

> **Important:** The template also contains runtime tokens like `{{WEATHER_TEMP}}`, `{{DATE_LINE}}`, `{{CALENDAR_CONTENT}}`, etc. Do **NOT** replace those — the command fills them in dynamically each morning. Only replace `{{USER_NAME}}` and `{{WEATHER_LOCATION}}` in the template.

---

## 11. Test Run

Run SOD manually to verify everything works. The invocation depends on your AI coding agent:

```bash
# Claude Code:
claude '/sod'

# OpenCode:
opencode run '/sod'

# Hermes (via skill):
# Use the SOD skill from the Hermes chat interface

# Other agents:
# Use your agent's command/skill invocation method
```

**Expected outcome:**
- An HTML file appears at `<wiki_root>/sod/YYYY-MM-DD.html`
- It opens in your default browser
- A log entry is appended to `<wiki_root>/log.md`
- Sections with missing data sources are cleanly omitted (not broken)

**Troubleshooting:**
- **Weather shows "unavailable"?** Check your location string at `https://wttr.in/YOUR_LOCATION`
- **Jira section missing?** Verify your connection (MCP or API token) and that your project key/board ID are correct
- **Slack section missing?** Verify your Slack MCP works by testing the search tool manually
- **Calendar missing?** Check Google Calendar MCP auth (OAuth tokens can expire — re-authorize if needed)
- **Stray `{{...}}` in the HTML?** You missed a placeholder in step 10b. Check both the command file and the template.

---

## 12. Circle Back — Deferred Integrations

If you skipped any integrations during onboarding, set them up here. Each one is independent — do them in any order, whenever you're ready.

### Jira (if skipped in Step 4)

**Option A: Jira MCP**
- Consult your AI agent's MCP documentation to add the Atlassian/Jira MCP server:

  | Agent | MCP config location |
  |-------|-------------------|
  | Claude Code | `~/.claude/mcp.json` |
  | OpenCode | `~/.config/opencode/config.json` (under `mcpServers`) |
  | Hermes | Configure via the Hermes plugin system |
  | Other agents | Check your agent's MCP documentation |

- Once configured, go back to Step 4 and fill in your project key and board ID

**Option B: Jira REST API Token**
1. Go to [Atlassian API tokens](https://id.atlassian.com/manage-profile/security/api-tokens) and create a token
2. Store it securely — recommended approaches:
   - Environment variable: `export JIRA_API_TOKEN="your-token"` in `~/.secrets/env` or `~/.zshrc`
   - **Never** commit tokens to git or paste them into shared config files
3. Update `sod.md` Step 4 to use `curl` with your token instead of MCP:
   ```bash
   curl -s -u "your-email@company.com:$JIRA_API_TOKEN" \
     "https://your-org.atlassian.net/rest/api/3/search/jql" \
     -H "Content-Type: application/json" \
     -d '{"jql": "project = ENG AND sprint in openSprints() AND assignee = currentUser()"}'
   ```

### Slack (if skipped in Step 5)

- Set up the Slack MCP integration in your AI agent (consult your agent's MCP documentation — see the config table above)
- Common options: [Slack MCP](https://mcp.slack.com/mcp) (OAuth-based), or your org's custom Slack integration
- Once working, go back to Step 5 to configure bot filters

### Google Calendar (if skipped in Step 6)

- Set up the Google Calendar MCP integration (consult your agent's MCP documentation — see the config table above)
- Requires Google OAuth — follow your MCP provider's setup guide
- Once working, SOD will automatically include today's calendar events

---

## You're Done! 🎉

Your morning digest is ready. Tomorrow at 8 AM (or whenever you run `/sod`), you'll get a beautiful, glanceable summary of your day — weather, calendar, tickets, messages, and carry-forward items — all in one page.

**Tips:**
- Run `/sod` anytime — it's not limited to mornings
- Running twice on the same day overwrites the HTML (the log keeps both entries)
- Customize the template CSS to match your aesthetic preferences
- Add `--no-open` if you want to generate without opening the browser (useful for cron)

### Optional: Schedule It

Once your test run works, schedule SOD to run every weekday morning:

**With Hermes Agent:**
```
cronjob(action="create",
    name="morning-digest",
    schedule="0 8 * * 1-5",
    prompt="Run the SOD morning digest skill",
    skills=["sod"],
    enabled_toolsets=["terminal"],
    deliver="local")
```

> **Important:** Make sure your API keys are in `~/.hermes/.env` (not just your shell profile), or the cron job won't have access to them.

**With system cron (adjust the command for your agent):**
```bash
crontab -e
# Add (example using Claude Code — adjust binary path for your agent):
0 8 * * 1-5 /usr/local/bin/claude '/sod' >> /tmp/sod.log 2>&1
```
