---
description: Start-of-day briefing — render a styled HTML morning digest from carry-forward (last EOD), today's calendar, sprint tickets, overnight Slack DMs, and local weather. Writes to `{{WIKI_ROOT}}/sod/YYYY-MM-DD.html`, appends `{{WIKI_ROOT}}/log.md`, opens in browser. Never asks questions.
---

# /sod — Start-of-Day Briefing

## Configuration

Before using this command, replace the following placeholders with your values (or reference your `sod-config.json`):

| Placeholder | Description | Example |
|---|---|---|
| `{{USER_NAME}}` | Your first name (used in greeting) | `Alex` |
| `{{WEATHER_LOCATION}}` | City and state/country for wttr.in | `Seattle,WA` or `London,UK` |
| `{{WIKI_ROOT}}` | Root path to your wiki/knowledge-base | `~/Documents/my-wiki` |
| `{{CONFIG_ROOT}}` | Root path where templates and config live | `~/.hermes/skills`, `~/.claude`, or your agent's config dir |
| `{{JIRA_PROJECT}}` | Your Jira project key | `ENG` |
| `{{JIRA_BOARD_ID}}` | Your Jira board number | `42` |
| `{{SLACK_BOT_FILTERS}}` | Bot user IDs to filter/collapse (see below) | see table |

**Slack bot filter list** (customize these IDs for your workspace):

| Bot | Action | Your User ID |
|---|---|---|
| Jira | collapse to 1-line summary | `YOUR_JIRA_BOT_ID` |
| Google Calendar | filter entirely | `YOUR_GCAL_BOT_ID` |
| Automation bot | filter if empty, show if non-empty | `YOUR_AUTOMATION_BOT_ID` |
| Other bots | collapse to count | (automatic) |

To find Slack bot user IDs: open a DM from the bot → click the bot name → "Copy member ID".

## Purpose

Render a single self-contained HTML morning digest combining:

1. **Greeting + date** — `Good morning, {{USER_NAME}}. Today is <day>, MM/DD/YYYY.`
2. **{{WEATHER_LOCATION}} weather** — current temp, condition, high/low/wind/precip summary.
3. **Carry-forward from the most recent `/eod`** — Open questions and ADR candidates still on the books. Presentation only; triage is parked for a future `/triage` command.
4. **Today's Google Calendar** — meetings today, with explicit `needsAction` flags.
5. **Sprint tickets** — open issues on the {{JIRA_PROJECT}} board (board {{JIRA_BOARD_ID}}), grouped by status.
6. **Overnight Slack DMs** — direct messages received since the last `/eod` ran, humans first, bots collapsed.
7. **Suggested focus** — synthesized 2–4 item priority list for the day.

Persist three things:

- HTML file at `{{WIKI_ROOT}}/sod/YYYY-MM-DD.html`.
- One-line entry in `{{WIKI_ROOT}}/log.md`.
- Browser tab opened on the rendered HTML.

## Hard Constraints

- **NEVER ask the user questions.** This is a presentation, not a conversation. No `question` tool calls. No clarifying prompts. Pick a sensible default and move on.
- **NEVER modify any file under `{{WIKI_ROOT}}/eod/`.** SOD only reads them.
- **NEVER update `{{WIKI_ROOT}}/projects/*` Activity Logs.** That's `/eod`'s job.
- **NEVER update `{{WIKI_ROOT}}/index.md`.** SOD is daily ephemeral; the EOD report is the curated artifact.
- **First-person from the user's perspective.** "I" / "your" = {{USER_NAME}}, never the LLM.
- **No raw transcripts.** Summarize Slack DMs and meeting details.
- **No credential/token/PII echoing** from DMs or events.
- **Concrete names**: ticket keys, attendee names, project page slugs, vendor names.
- **Omit empty sections entirely.** When data is empty for a section, delete the matching `<section data-section="…">…</section>` block from the rendered HTML (do not leave empty stubs visible).
- **Graceful degradation, not failure.** A service outage skips that section (or weather) with the rest of the digest intact.
- **Self-contained HTML.** All CSS lives inline in the template. No external fonts, no CDN, no JS. Must render correctly opened directly from disk with no network access.

## Workflow

### Step 1: Determine target date & time

1. Run `date '+%Y-%m-%d %A %m/%d/%Y %Z %H:%M:%S %:z'` to get YYYY-MM-DD, day-of-week, MM/DD/YYYY, timezone, time, offset.
2. If invocation includes `--date YYYY-MM-DD`, use that. Otherwise default to today's local date.
3. Compute:
   - `<day>` — full day name capitalized (e.g. `Tuesday`).
   - `<date_us>` — MM/DD/YYYY format (e.g. `05/19/2026`).
   - `<date_iso>` — YYYY-MM-DD format (e.g. `2026-05-19`).
   - `<timestamp_local>` — for the footer (e.g. `2026-05-19 08:32 MDT`).
4. **Late-fire soft notice.** If current local time is more than 2 hours past 08:00, prepare a banner string: `Running at HH:MM <TZ> — later than typical morning. Treating as midday catch-up.` Do NOT bail out.

### Step 2: Locate the most recent EOD

1. `ls -t {{WIKI_ROOT}}/eod/*.md | head -1` — newest by mtime.
2. Parse `<last_eod_date>` from filename (`YYYY-MM-DD.md`). Compute `days_since_last_eod = today - last_eod_date`. If > 1, prepare banner: `Last EOD was N days ago (<last_eod_date>).`
3. `stat` the file for mtime; record as Unix timestamp `<eod_ts>` — needed for Slack boundary.
4. If no EOD file exists, set `<last_eod_date> = none`, mark carry-forward as empty.

### Step 3: Probe sprint metadata

Call `jira_search` (or `atlassian_searchJiraIssuesUsingJql`) with JQL:

```
project = {{JIRA_PROJECT}} AND sprint in openSprints() AND assignee = currentUser()
```

Include `customfield_10020`. From any returned issue's sprint object, extract `name`, `startDate`, `endDate`.

Compute:

- `<sprint_name>` — strip the parenthetical date range for the status strip (e.g. `ENG S10 2026`).
- `<sprint_end_local>` — convert UTC `endDate` to your local timezone.
- `<workdays_remaining>` — count Mon–Fri dates from today (inclusive if a weekday) up to and including the last weekday before `sprint_end_local`.

If zero issues / no sprint metadata: set sprint name to `—`, workdays to `—`, and skip the sprint section.

### Step 4: Fetch sources (parallel)

Fire in the same response:

- **Weather**: `curl -s 'https://wttr.in/{{WEATHER_LOCATION}}?format=j1'`. Parse `current_condition[0]` for `temp_F`, `weatherDesc[0].value`, `windspeedMiles`, `precipMM`, `FeelsLikeF`. Parse `weather[0]` for `maxtempF`, `mintempF`. On failure or `--no-weather` flag: mark weather as unavailable (the weather block stays visible but shows `—` for temp and `Weather unavailable` for condition).
- **GCal**: `google-calendar_get-current-time` then `google-calendar_list-events` with `calendarId: "primary"`, `timeMin: <today 00:00 local ISO>`, `timeMax: <tomorrow 00:00 local ISO>`, fields including `attendees`, `organizer`, `hangoutLink`, `eventType`, `location`, `transparency`. Per the gcal skill: do NOT call `list-calendars`.
- **Slack DMs**: `slack_slack_search_public_and_private` with `query: "to:me"`, `channel_types: "im"`, `content_types: "messages"`, `after: <eod_ts>`, `sort: "timestamp"`, `sort_dir: "desc"`, `limit: 20`. Use the Unix `after` parameter, NOT `after:YYYY-MM-DD`.
- **Jira sprint tickets**: same JQL as Step 3, pulling `status`, `priority`, `summary`, `issuetype`. If Step 3 cached, reuse.

Honor `--no-jira`, `--no-cal`, `--no-slack`, `--no-weather` flags. On per-service runtime failure: mark section empty (which deletes it from the HTML in Step 7) and proceed.

### Step 5: Parse carry-forward from EOD

Read the most recent EOD file as plain text. Extract bullets from these **exact** H2 sections:

- `## Open questions / followups` → all bullets → `<open_questions[]>`
- `## ADR candidates` → all bullets → `<adr_candidates[]>`

Bullet format: `- text`. Multi-line bullets (bold lead-ins like `- **Brainstorm SOD skill** — ...`) count as one item — preserve content up to the next bullet or blank line.

**Presentation only.** Do NOT triage. Triage is parked for a future `/triage` command.

### Step 6: Shape data for HTML rendering

**Weather → digest values:**

- `<weather_temp>` — `<temp_F>°` (e.g. `64°`).
- `<weather_condition>` — `weatherDesc[0].value` (e.g. `Mostly clear`). Title-case it if wttr.in returns lower.
- `<weather_detail>` — built from: `High <maxtempF>° / Low <mintempF>° · <wind_phrase> · <precip_phrase>`.
  - Wind phrase: `calm` (0–4 mph) / `light breeze` (5–11) / `breezy` (12–19) / `windy` (20+).
  - Precip phrase: `dry morning` (precip 0 / chance <20%) / `mostly dry` (20–40%) / `chance of rain` (41–60%) / `rain likely` (61+%).

**Status strip values:**

- `<run_status>` — `Generated <HH:MM> <TZ>` (e.g. `Generated 08:32 MDT`). If late-fire applies, suffix `· catch-up`.
- `<sprint_name>` — from Step 3, or `—`.
- `<workdays_left>` — number, suffixed with `workday` (singular) or `workdays` (plural). E.g. `3 workdays` or `1 workday`. `—` if no sprint.

**Banner block** (`<banners_html>`) — for the header below the date line:

- Build a `<p class="banner">…</p>` for the late-fire notice if applicable.
- Build a `<p class="banner">…</p>` for the "Last EOD was N days ago" notice if applicable.
- Concatenate with `\n`. If no banners apply, this slot is the empty string.

**GCal events → calendar HTML:**

- Filter `eventType: "workingLocation"` out of the main list. Use the most prominent one to derive a leading `<p class="lead">…</p>` line (e.g. `🏠 Working from home today` or `🏢 In-office today`). If none, omit the lead line.
- Sort remaining events chronologically.
- Per event, render as `<li>`:
  ```html
  <li>
    <span class="meeting-time">HH:MM–HH:MM</span> — <summary>
    <span class="badge needs-action">needs action</span>            <!-- only if my responseStatus is needsAction -->
    <a class="meet-link" href="<hangoutLink>">join</a>               <!-- only if hangoutLink present -->
    <span class="dm-meta">📍 <location-token></span>                 <!-- only if non-Google-Meet location -->
  </li>
  ```
- Strip `[Google Meet]` resource-room noise from `location`; show only the first meaningful location token if onsite.
- Wrap items in `<ul>…</ul>`. The full `{{CALENDAR_CONTENT}}` is `[<lead-line>]<ul>…</ul>`.
- If zero non-workingLocation events: mark the section empty (Step 7 will delete it).

**Jira tickets → sprint HTML:**

- Group by status: `In Progress` first, then `To Do`, then any other open statuses.
- Exclude `Done` and `On-Hold`.
- Per group, emit a `<p class="group-label">In Progress (N)</p>` then a `<ul>` of `<li>`s.
- Per ticket: `<li><span class="ticket-key">{{JIRA_PROJECT}}-1234</span> — <summary> <span class="dm-meta">(<priority-shortname>)</span></li>`.
- Cap at 15 visible total; append `<li><em>+N more not shown</em></li>` if truncated.
- If zero open tickets: mark the section empty.

**Slack DMs → DMs HTML:**

- Bot taxonomy (apply in this order, using your configured bot IDs from `{{SLACK_BOT_FILTERS}}`):
  - **Jira bot**: collapse to one line — `<li>Jira: N notifications, most recent: <subject></li>`.
  - **Google Calendar bot**: filter entirely.
  - **Automation bot** with empty text: filter. With non-empty text: `<li><span class="dm-meta">HH:MM · Automation</span> — <preview></li>`.
  - **Other bots**: collapse to one line — `<li><span class="dm-meta">Other bots: N notifications</span></li>`.
- Human DMs: `<li><span class="dm-meta">HH:MM · <sender_display_name></span> — <preview></li>`. If the search returned `Context before:` thread context, append `<span class="dm-meta">(thread: <context-preview>)</span>`.
- Order: humans first, then Jira collapsed, then other-bots collapsed.
- Wrap in `<ul>`.
- If zero items (after filtering): mark the section empty.

**Carry-forward → counts + HTML:**

- `<open_questions_count>` — integer count of `<open_questions[]>` from Step 5.
- `<adr_count>` — integer count of `<adr_candidates[]>`.
- `<open_questions_list>` — `<ul>` of `<li>` per bullet. Preserve the bullet's bold lead-in if present (e.g. `<li><strong>Brainstorm SOD skill</strong> — Start-of-Day counterpart to /eod…</li>`). Use `<strong>…</strong>` for the lead-in, plain text after the em-dash.
- `<adr_list>` — same shape.
- If both counts are zero AND no prior EOD: mark the section empty.
- If counts are zero but a prior EOD exists: emit the section with the heading `Carrying forward from <last_eod_date>` and a single `<p class="lead">Backlog clear.</p>` body (do NOT emit empty count markers).

**Suggested focus → focus HTML:**

- Synthesize 2–4 priority items from: `needsAction` calendar items today, `In Progress` Jira tickets, high-signal human DMs (especially ones referencing actions like "please" / "can you" / "@me"), the EOD's clearly time-sensitive open questions.
- Render as `<ol>…</ol>` with one `<li>` per priority. Each item is one sentence, concrete (mention the ticket key / person / event by name).
- Don't pad. If fewer than 2 priorities surface honestly, render 1 and stop. If zero: mark section empty.

**Footer values:**

- `<timestamp>` — `<timestamp_local>` from Step 1.
- `<sources_summary>` — comma-joined list like `jira (N tickets), gcal (M events), slack (H humans, B bot lines), weather`. Skip any disabled or unavailable source.

### Step 7: Render HTML

1. Read the template: `{{CONFIG_ROOT}}/templates/sod-template.html`.
2. **Substitute all `{{TOKEN}}` placeholders** with the values computed in Step 6. Tokens in the template:

   | Token | Value |
   |---|---|
   | `{{TITLE}}` | `Morning Digest — <day> <date_us>` |
   | `{{DATE_LINE}}` | `<day>, <date_us>` |
   | `{{BANNERS}}` | `<banners_html>` or empty string |
   | `{{WEATHER_TEMP}}` | `<weather_temp>` or `—` |
   | `{{WEATHER_CONDITION}}` | `<weather_condition>` or `Weather unavailable` |
   | `{{WEATHER_DETAIL}}` | `<weather_detail>` or empty string |
   | `{{RUN_STATUS}}` | `<run_status>` |
   | `{{SPRINT_NAME}}` | `<sprint_name>` |
   | `{{WORKDAYS_LEFT}}` | `<workdays_left>` |
   | `{{LAST_EOD_DATE}}` | `<last_eod_date>` (appears twice in template — header section title and footer) |
   | `{{CALENDAR_CONTENT}}` | calendar HTML or empty |
   | `{{SPRINT_CONTENT}}` | sprint HTML or empty |
   | `{{DMS_CONTENT}}` | DMs HTML or empty |
   | `{{OPEN_QUESTIONS_COUNT}}` | `<open_questions_count>` |
   | `{{OPEN_QUESTIONS_LIST}}` | `<open_questions_list>` or empty |
   | `{{ADR_COUNT}}` | `<adr_count>` |
   | `{{ADR_LIST}}` | `<adr_list>` or empty |
   | `{{FOCUS_CONTENT}}` | focus HTML or empty |
   | `{{TIMESTAMP}}` | `<timestamp>` |
   | `{{DAYS_SINCE_EOD}}` | `days_since_last_eod` |
   | `{{SOURCES_SUMMARY}}` | `<sources_summary>` |

3. **Delete empty sections.** For each `<section data-section="X">…</section>` block where the content slot is empty (or marked empty in Step 6), remove the entire `<section>` element including its opening and closing tags. The available `data-section` values are: `calendar`, `sprint`, `dms`, `carryforward`. The `<aside>` `<section>` (Suggested focus) has no `data-section` attribute — remove it manually if focus content is empty.

4. **Sanity-check the output**: ensure no `{{...}}` placeholders remain in the final HTML. If any do, this is a render bug — fix before writing.

### Step 8: Write file, append log, open browser

1. `mkdir -p {{WIKI_ROOT}}/sod/`
2. Write the rendered HTML to `{{WIKI_ROOT}}/sod/<date_iso>.html`.
3. **Verify the HTML write**: `ls -la` the file (confirm non-zero size) and `grep -c "{{" <file>` (confirm zero — no stray placeholders).
4. Append to `{{WIKI_ROOT}}/log.md`:

   ```markdown
   ## [<date_iso>] sod | Day briefing generated

   File: {{WIKI_ROOT}}/sod/<date_iso>.html
   Sources: jira (<N> open tickets), gcal (<M> events), slack (<H> human DMs, <B> bot lines), weather (<weather_temp> <weather_condition>)
   Last EOD: <last_eod_date> (<days_since> days ago)
   Sprint: <sprint_name>, <workdays_left> left
   Carry-forward surfaced: <N> open questions, <M> ADR candidates (presented only; triage parked for future /triage command)
   ```

5. **Verify the log append**: `tail -n 20 {{WIKI_ROOT}}/log.md` and confirm the new heading appears.
6. **Open in browser**: `open {{WIKI_ROOT}}/sod/<date_iso>.html` (macOS) or `xdg-open` (Linux). Skip if `--no-open` was passed.

### Step 9: Report to the user

One-line summary:

```
SOD rendered: {{WIKI_ROOT}}/sod/<date_iso>.html
Logged: {{WIKI_ROOT}}/log.md
```

If the HTML file did not open (or `--no-open`), include the absolute path so the user can open it manually.

## Clarifying Questions

**Do not ask any.** This command is a presentation, not a conversation. Pick sensible defaults silently:

- Multiple active sprints returned? Use the one on board {{JIRA_BOARD_ID}} ({{JIRA_PROJECT}}).
- Ambiguous source data? Surface what's available, drop what's broken with a section deletion.
- `--date` references a date with no prior EOD? Render today's sources only, omit the carry-forward section.
- Render produces a placeholder that didn't substitute? That's a bug — fix it before writing, don't ship broken HTML.

## Flags

- `--date YYYY-MM-DD` — explicit target date (default: today's local date).
- `--no-jira` — skip the sprint section.
- `--no-cal` — skip the calendar section.
- `--no-slack` — skip the DM section.
- `--no-weather` — skip weather fetch; weather block falls back to `—` / `Weather unavailable`.
- `--no-open` — write the HTML file but don't auto-open in browser.

## Idempotency

Running `/sod` twice on the same date overwrites the same `{{WIKI_ROOT}}/sod/<date>.html` file and appends a second `log.md` entry. Both behaviors are intentional — the rendered HTML reflects the latest run, the log preserves run history. To preserve an earlier render, copy the HTML file aside before re-running.

## Notes for Maintaining This Command

- **Template lives at `{{CONFIG_ROOT}}/templates/sod-template.html`.** All visual design — colors, fonts, layout, responsive breakpoints, print styles — is in there. To restyle the digest, edit the template, not the command logic.
- **Section deletion uses `data-section` attributes** (not HTML comments). The matchable values are `calendar`, `sprint`, `dms`, `carryforward`. The aside's "Suggested focus" `<section>` has no `data-section` attribute and is removed by manual string match if focus is empty.
- **Self-contained HTML is non-negotiable.** No external font loads, no CDN, no JS. The digest must render fully when opened from disk offline (e.g. on a plane, on a corp network with strict egress).
- **Carry-forward parser is exact-string sensitive** on EOD H2 headers. If `/eod` ever renames `## Open questions / followups` or `## ADR candidates`, update Step 5.
- **Slack boundary precision** uses EOD file mtime as a Unix timestamp via the `after` parameter. The `after:YYYY-MM-DD` operator is day-granular and misses messages sent on the EOD day after EOD ran.
- **Bot filter list** is configured via `{{SLACK_BOT_FILTERS}}`. Extend if new noisy bots show up in DMs. To find a bot's user ID: open a DM from the bot → click its name → "Copy member ID".
- **Sprint workdays-remaining** excludes weekends but does NOT exclude holidays. Integrate a holiday calendar if it starts mattering.
- **Weather location** is configured as `{{WEATHER_LOCATION}}`. Supports any location wttr.in recognizes (city names, airport codes, coordinates).
- **Weather source** is `wttr.in`. Free, no API key, decent uptime. If it flakes, NWS (`api.weather.gov`) is a US-government alternative with no key either; OpenWeatherMap is a fallback (needs free API key).
- **No questions, ever.** If a future change to `/sod` is tempted to ask the user for input, that change belongs in a separate command instead.
- **Late-fire threshold (2 hours past 08:00)** is a soft notice only. If midday runs become routine, drop the banner or move the threshold.
- **No `{{WIKI_ROOT}}/index.md` update.** SOD reports are daily ephemeral; the EOD report is the curated artifact that gets indexed.
- **Tools used**:
  - Weather: `curl https://wttr.in/{{WEATHER_LOCATION}}?format=j1` (no MCP)
  - Jira: `jira_search` (preferred) or `atlassian_searchJiraIssuesUsingJql`
  - GCal: `google-calendar_get-current-time`, `google-calendar_list-events`
  - Slack: `slack_slack_search_public_and_private`
  - File: `mkdir -p`, write, `tail`, `grep`, `open` (macOS browser launcher)
- **Parked feature: `/triage`.** The original `/sod` design included interactive triage (Still Relevant / Done / Drop) of carry-forward items. That feature was pulled out to keep SOD purely presentational. A future `/triage` command should:
  - Read the most recent EOD's `## Open questions / followups` + `## ADR candidates` bullets.
  - Present them via the `question` tool with 3-option triage + custom answers.
  - Log outcomes to `{{WIKI_ROOT}}/log.md` as `## [<date>] triage | Carry-forward triaged`.
  - Provide a structured log format `/eod` can read to auto-resolve items in tomorrow's report.
  - Be runnable independently from `/sod` — invoke whenever the user wants to clear backlog, not tied to morning.
  - Build this when carry-forward starts accumulating beyond what's scannable in a SOD brief (current rough threshold: ~15 combined items).
