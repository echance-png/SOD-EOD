---
description: End-of-day report from coding sessions (if session APIs available) + merge wiki journal + update wiki Activity Logs + save markdown report
---

# /eod — End-of-Day Report (Session-Driven + Journal Merge)

## Modes

This command supports two modes of operation:

| Mode | How it works | Best for |
|---|---|---|
| **Session mode** | Auto-extracts work from coding sessions via `session_list`/`session_read` APIs, then merges the wiki journal | Agents with session history APIs (e.g., OpenCode, Claude Code) |
| **Journal-only mode** | Relies entirely on the wiki journal file at `{{WIKI_ROOT}}/inbox/journal-YYYY-MM-DD.md` | Any agent, any editor, or no agent at all |

**The journal file is the universal input.** It works regardless of which agent you use or whether you use an agent at all. Session mode adds automatic extraction on top — it does not replace the journal.

**Mode is auto-detected:** if `session_list`/`session_read` APIs are available and `{{CONFIG_ROOT}}/eod-scope.json` exists, session mode activates. Otherwise the command runs in journal-only mode.

## Configuration

Before using this command, configure the following placeholders (replace them in this file or set them in your agent's variable system):

| Placeholder | Description | Example |
|---|---|---|
| `{{WIKI_ROOT}}` | Root directory of your markdown wiki | `~/Documents/my-wiki` |
| `{{CONFIG_ROOT}}` | Directory containing config files and templates | `~/.config/myagent` |
| `{{USER_NAME}}` | Your name (used in first-person report voice) | `Jane` |

### Required files

- **`{{WIKI_ROOT}}/`** — Wiki directory with the structure described in README.md.

### Session mode only (optional)

- **`{{CONFIG_ROOT}}/eod-scope.json`** — Allowlist of project paths to include in the report. See `eod-scope.example.json` for format. Only needed when using session mode.

## Purpose

Produce a single end-of-day report combining:

1. **All in-scope coding sessions** for the target date *(session mode only)* — scope = allowlist in `{{CONFIG_ROOT}}/eod-scope.json`; mechanical, exhaustive within scope (catches forgotten work without leaking personal sessions)
2. **Any wiki journal file** at `{{WIKI_ROOT}}/inbox/journal-YYYY-MM-DD.md` — curated, subjective (catches work done outside coding sessions, or all work in journal-only mode)

Save to `{{WIKI_ROOT}}/wiki/eod/YYYY-MM-DD.md`. Update wiki project Activity Logs.

**Designed to run hands-off** — typically as a cron job. The command never prompts and resolves all ambiguity via deterministic defaults.

## Hard Constraints

- **In session mode: read sessions from the target date that match the configured scope** in `{{CONFIG_ROOT}}/eod-scope.json`. Scope is **allowlist-based (default-deny)** — out-of-scope sessions are silently skipped, never summarized, never leaked into the work EOD report.
- **First-person from user's perspective.** "I" = the user ({{USER_NAME}}). "I" is NEVER the LLM or any agent.
- **Do not invent facts.** Preserve uncertainty with "seems," "likely," "unclear."
- **No raw transcripts.** Summarize.
- **No internal agent/tool mechanics** in the report unless they materially changed the outcome.
- **Concrete names**: hosts, projects, tickets, vendors, repos, services, docs, people, incidents.
- **No echoing of credentials, tokens, API keys, or PII** found in sessions.
- **Merge duplicate discoveries** across sessions into one bullet.
- **Empty section → "None captured."**
- **Never prompt.** This command runs unattended.

## Workflow

### Step 1: Determine target date

1. Run `date '+%Y-%m-%d %Z %H:%M:%S %:z'` for local date, timezone, time, and offset.
2. If the invocation includes an explicit date, use that.
3. Otherwise, if local time is **00:00–04:00**, default to **yesterday**. Otherwise default to **today**.
4. **Never prompt.**
5. Compute ISO 8601 bounds with local offset.

### Step 2: List sessions for the date and apply scope filter (session mode only)

> **If `session_list`/`session_read` APIs are not available, or `{{CONFIG_ROOT}}/eod-scope.json` does not exist, skip Steps 2–4 and proceed directly to Step 6 (journal merge).**

#### 2a. Pull all sessions

```
session_list(from_date=<from>, to_date=<to>, project_path="")
```

Pass `project_path=""` explicitly.

#### 2b. Load scope config

Read `{{CONFIG_ROOT}}/eod-scope.json`:

```json
{
  "include_prefixes": ["/path/to/project-a", "/path/to/project-b"],
  "exclude_prefixes": ["/path/to/project-a/personal-subdir"]
}
```

If the file is missing or malformed: **treat as default-deny (zero in-scope sessions) and log a warning**.

#### 2c. Apply filter

A session's `project_path` is **in scope** if BOTH:
1. It matches at least one `include_prefix` (segment-boundary match)
2. It matches **no** `exclude_prefix`

Segment-boundary matching: `path == prefix OR path.startswith(prefix + "/")`.

### Step 3: Read each in-scope session (session mode only)

For each session in `in_scope_sessions`:

```
session_read(session_id=<id>, include_todos=true, include_transcript=false)
```

### Step 4: Mechanical extraction (session mode only)

Per session, capture: Project name, Completed todos, Incomplete todos, User intents.

### Step 5: Section routing

Build the report in this structure:

```markdown
# EOD — YYYY-MM-DD

## What I touched
- <bullet>

## What I decided / learned
- <bullet>

## Open questions / followups
- <bullet>

## ADR candidates
- <bullet>
```

Routing logic preserved for each section. Bullet rules: one line per item, concrete names, preserve voice, merge duplicates, empty section → "None captured."

### Step 6: Merge wiki journal if present

Check for `{{WIKI_ROOT}}/inbox/journal-<date>.md`. If present, merge its content into the appropriate report sections. Move the processed journal file to `{{WIKI_ROOT}}/raw/journal/`.

**In journal-only mode, this is the sole source of report content.** The journal file should contain bullets organized under the same H2 headings used in the report (or freeform text that the agent routes to the appropriate sections).

### Step 7: Save the report

Write to `{{WIKI_ROOT}}/wiki/eod/<date>.md` with frontmatter. Write primary artifact FIRST before metadata updates.

### Step 7b: Refresh page counts

Refresh page-count claims in README.md, getting-started.md, index.md, overview.md.

### Step 7c: Update wiki project Activity Logs

For each wiki project/entity/concept mentioned: append dated entry to Activity Log section. Never auto-create pages. Append to log.md.

### Step 7d: Link the new EOD in wiki/index.md

Insert entry at top of EOD Reports section (most recent first).

### Step 8: Report to user

Print summary: path, sessions processed/skipped (if session mode), journal merged, wiki pages updated, ADR candidates.

## Hands-Off Operation

This command never prompts. All ambiguity resolved by deterministic defaults.

## Idempotency

Running twice on same date produces same report modulo overwrite handling.
