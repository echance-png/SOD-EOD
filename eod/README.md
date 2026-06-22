# EOD — End-of-Day Report Generator

The EOD command is an automated end-of-day report generator that works with **any AI coding agent** — or no agent at all. For agents that expose session history APIs (like OpenCode or Claude Code), it auto-extracts work from coding sessions, filters through a privacy-aware scope config, and merges manual journal entries. For any other setup, it builds the report entirely from a wiki journal file you maintain. Either way, the output is a structured markdown report saved to your personal wiki. It runs completely hands-off — no prompts, no interaction needed — making it ideal for developers who want automatic work journals without the overhead of manual logging.

## Two Modes

| Mode | Input source | What you need |
|---|---|---|
| **Session mode** | Auto-reads coding sessions via `session_list`/`session_read` APIs + journal | An agent with session history APIs (e.g., OpenCode, Claude Code) + scope config |
| **Journal-only mode** | Wiki journal file only | Any agent, a text editor, or nothing at all |

**Journal-only mode is first-class**, not a fallback. Many users prefer writing a quick journal file during the day and letting the cron job format it into a proper EOD report. Session mode simply adds automatic extraction on top for agents that support it.

The mode is auto-detected: if session APIs and a scope config are present, session mode activates. Otherwise journal-only mode runs.

## Prerequisites

- **[Hermes Agent](https://github.com/NousResearch/hermes-agent)** or a compatible agent runtime that supports slash commands and cron scheduling
- **Any AI coding agent** (for session mode: one with session history APIs like OpenCode or Claude Code) **OR just a text editor** (for journal-only mode)
- A markdown wiki directory structure (see [Wiki Setup](#wiki-setup) below)

## Quick Start

1. **Copy the command file** into your agent's commands directory:
   ```bash
   # Hermes Agent
   cp eod.md ~/.hermes/skills/eod.md

   # Claude Code
   cp eod.md ~/.claude/commands/eod.md

   # Or wherever your agent loads slash commands from
   ```

2. **Create your scope config** *(session mode only — skip for journal-only)*:
   ```bash
   cp eod-scope.example.json ~/.config/myagent/eod-scope.json
   # Edit to include YOUR project paths
   ```

3. **Set up your wiki directory structure:**
   ```bash
   mkdir -p ~/Documents/my-wiki/{inbox,raw/journal,wiki/eod}
   ```

4. **Configure placeholders** in `eod.md` (see the Configuration section at the top of the file):
   - `{{WIKI_ROOT}}` — path to your wiki root (e.g., `~/Documents/my-wiki`)
   - `{{CONFIG_ROOT}}` — path to your config directory (e.g., `~/.config/myagent`)

5. **(Optional) Schedule as a cron job** to run every evening:
   ```
   # Example: run at 6pm every weekday
   0 18 * * 1-5  hermes run /eod
   ```

## Wiki Setup

The command expects this directory layout under your `{{WIKI_ROOT}}`:

```
{{WIKI_ROOT}}/
├── inbox/                    # Drop journal-YYYY-MM-DD.md files here
├── raw/journal/              # Processed journals are moved here
├── wiki/
│   ├── eod/                  # Generated EOD reports land here
│   ├── index.md              # Wiki index (EOD links inserted here)
│   └── ...                   # Your wiki pages (activity logs updated)
├── README.md
├── getting-started.md
└── overview.md
```

## Scope Config (Session Mode Only)

The scope config (`eod-scope.json`) is an **allowlist** — only sessions whose `project_path` matches an `include_prefix` are summarized. Everything else is silently skipped. This prevents personal projects from leaking into work reports.

This file is **only needed for session mode**. In journal-only mode, the scope config is not used.

See `eod-scope.example.json` for the format.

## Customization

- **Journal merge**: Drop a `journal-YYYY-MM-DD.md` file into `{{WIKI_ROOT}}/inbox/` before the cron runs to include manually-written notes in that day's report. In journal-only mode, this is the sole source of content.
- **Late-night handling**: If run between midnight and 4am, the command defaults to reporting on *yesterday* (configurable in the workflow logic).
- **Idempotent**: Running twice on the same date overwrites the previous report — safe to re-run.
- **Report structure**: The generated report uses four sections: "What I touched", "What I decided / learned", "Open questions / followups", and "ADR candidates". Customize the template in `eod.md` if you prefer different sections.

## License

MIT
