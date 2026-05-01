# update-translations

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) that updates a project's i18n language file when new user-facing strings are added to the codebase.

The skill auto-detects:

- The **schema** of the language file (flat vs. nested, string vs. object entries, field names like `text`/`message`/`defaultMessage`, description field names).
- The **naming convention** of existing keys (`snake_case`, `camelCase`, `kebab-case`, `PascalCase`, `dot.case`, mixed).
- The **i18n library** wired up in the project (`next-intl`, `react-intl`, `i18next`, `react-i18next`, `@lingui`, `vue-i18n`, …) and the import/call patterns it uses in this codebase.

so it works regardless of the format the project chose.

All reads from and writes to the language file are delegated to subagents, so the parent agent never loads the file's contents into its own context — large translation files won't blow out the context window.

## Installation

Place this directory under `~/.claude/skills/` (user-level) or `.claude/skills/` (project-level):

```
~/.claude/skills/update-translations/
├── SKILL.md
└── README.md
```

Claude Code auto-discovers skills in those locations.

## Usage

Invoke explicitly with `/update-translations`, or let Claude trigger it automatically when you ask something like:

- "Add the new strings I just wrote to the language file."
- "Update translations."
- "I added a new message — update `en.json`."

The skill will:

1. Locate `**/en.json` in the repo.
2. Probe its schema and key naming convention via a subagent.
3. Identify newly-added user-facing strings (from the in-session edits, or from `git diff` / untracked files).
4. Detect the project's i18n library and how it's invoked.
5. Add new keys to the language file (atomic write, conflict-checked).
6. Replace the hardcoded strings in source files with the appropriate translator call.
7. Backfill empty/missing description fields if the schema has one.

## When you don't need it

If you've already updated the language file yourself with description/metadata fields filled in (or the schema has no such field), there's nothing for the skill to do — skip it.

## Details

See [SKILL.md](./SKILL.md) for the full step-by-step procedure Claude follows, including the verbatim subagent prompts used at each stage.
