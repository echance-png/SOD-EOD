# SOD — Start-of-Day Briefing

**SOD** is an AI-powered morning digest command that automatically compiles your weather forecast, calendar events, Jira sprint tickets, overnight Slack DMs, and carry-forward items from your last end-of-day review into a single, beautifully styled HTML briefing. Scheduled to run at 8 AM on weekdays via a cron job (or run on-demand), it opens in your browser so you can scan your day in 30 seconds. SOD eliminates the morning scramble of checking five different apps by pulling everything into one glanceable page.

---

## What's in This Package

| File | Purpose |
|------|---------|
| `sod.md` | The command file — drop into your AI coding agent's commands directory |
| `sod-template.html` | Self-contained HTML template with inline CSS (no external dependencies) |
| `sod-config.example.json` | Example config — copy to `sod-config.json` and fill in your values |
| `ONBOARDING.md` | Step-by-step setup questionnaire for first-time users |

## Prerequisites

- **AI coding agent** with MCP tool access (e.g., [Hermes Agent](https://hermes-agent.nousresearch.com), [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [OpenCode](https://opencode.ai), or similar)
- **MCP integrations** for your data sources (all optional — SOD degrades gracefully):
  - **Jira** — `jira_search` or Atlassian MCP for sprint tickets
  - **Google Calendar** — `google-calendar_list-events` MCP for today's meetings
  - **Slack** — `slack_slack_search_public_and_private` MCP for overnight DMs
- **curl** — for weather data from [wttr.in](https://wttr.in) (free, no API key)
- **macOS or Linux** — `open` (macOS) or `xdg-open` (Linux) for browser launch

## Quick Start

1. **Copy the config template:**
   ```bash
   cp sod-config.example.json sod-config.json
   ```

2. **Fill in your values** in `sod-config.json` (see `ONBOARDING.md` for a guided walkthrough).

3. **Install the command and template** into your agent's directory:

   | Agent | Command file location | Template file location |
   |-------|-----------------------|------------------------|
   | Claude Code / OpenCode | `~/.claude/commands/sod.md` | `~/.claude/templates/sod-template.html` |
   | Hermes Agent | `~/.hermes/skills/sod.md` | `~/.hermes/skills/sod-template.html` |
   | Other | Check your agent's docs for the custom command location | Same directory as the command file |

   ```bash
   # Example for Claude Code / OpenCode:
   cp sod.md ~/.claude/commands/sod.md
   cp sod-template.html ~/.claude/templates/sod-template.html

   # Example for Hermes Agent:
   cp sod.md ~/.hermes/skills/sod.md
   cp sod-template.html ~/.hermes/skills/sod-template.html
   ```

4. **Update paths in `sod.md`** to match your `sod-config.json` values (replace all `{{PLACEHOLDER}}` references with your actual paths/values).

5. **Test it:**
   ```bash
   # Claude Code:
   claude '/sod'

   # OpenCode:
   opencode run '/sod'

   # Hermes Agent — invoke the skill from chat:
   #   "Run the sod skill"
   ```

6. **(Optional) Schedule it** to run automatically on weekday mornings:

   **Hermes Agent cron:**
   ```
   cronjob(action="create",
       name="morning-digest",
       schedule="0 8 * * 1-5",
       prompt="Run the SOD (start-of-day) skill to generate my morning briefing.",
       skills=["sod"],
       enabled_toolsets=["terminal"],
       deliver="local")
   ```

   **System cron (works with any agent):**
   ```bash
   # crontab -e
   0 8 * * 1-5  /path/to/your-agent run '/sod' 2>&1 >> ~/sod-cron.log
   ```

## Customization

### Disable Sections

Pass flags to skip sections you don't use:
- `--no-jira` — skip sprint tickets
- `--no-cal` — skip calendar
- `--no-slack` — skip Slack DMs
- `--no-weather` — skip weather (shows "unavailable" placeholder)
- `--no-open` — generate HTML but don't open in browser

### Styling

Edit `sod-template.html` to change colors, fonts, or layout. All CSS is inline — no external dependencies. The template uses CSS custom properties (`:root` variables) for easy theming:

```css
:root {
  --bg: #f7f3ec;       /* page background */
  --ink: #202020;       /* body text */
  --muted: #686159;     /* secondary text */
  --line: #c9beb0;      /* dividers */
  --accent: #6b4f3f;    /* accent color 1 */
  --accent-2: #2f4a48;  /* accent color 2 */
}
```

### EOD Integration

SOD reads carry-forward items from an End-of-Day (`/eod`) report. If you don't have an EOD command, the carry-forward section is simply omitted. The expected EOD format has:
- `## Open questions / followups` — bullet list
- `## ADR candidates` — bullet list

### Weather

Weather comes from [wttr.in](https://wttr.in) — free, no API key, works worldwide. If you're in a location wttr.in doesn't recognize, try the nearest major city. Fallback alternatives:
- [api.weather.gov](https://api.weather.gov) — US-only, no key needed
- [OpenWeatherMap](https://openweathermap.org) — worldwide, free tier needs API key

## Architecture

```
/sod command
  ├── Step 1: Date/time detection
  ├── Step 2: Locate last EOD report
  ├── Step 3: Probe Jira sprint metadata
  ├── Step 4: Parallel data fetch (weather, calendar, slack, jira)
  ├── Step 5: Parse carry-forward from EOD
  ├── Step 6: Shape data for HTML rendering
  ├── Step 7: Render HTML from template (delete empty sections)
  ├── Step 8: Write file, append log, open browser
  └── Step 9: Report summary
```

## License

MIT — use, modify, and share freely.
