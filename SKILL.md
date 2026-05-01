---
name: update-translations
description: Update language files when a new message was added to the codebase. Auto-detects the i18n schema, the key naming convention used in the existing file, and the i18n library in use, so the skill works regardless of the project's chosen format. If you have already updated the language file with metadata/description fields filled in (or the schema has no such field), you do not need this skill.
---

# update-translations

Create a language file if it doesn't exist, and update it with the new message key and value.

All reads from and writes to the language file are delegated to subagents (via the Task tool), so the parent agent never loads the file's contents into its own context. Three things are **discovered**, not assumed:

- The **schema** of the language file (flat vs. nested, string vs. object entries, field names).
- The **naming convention** used for existing keys (snake_case, camelCase, kebab-case, PascalCase, dot.path, mixed, …).
- The **i18n library** wired up in the project (next-intl, react-intl, i18next, react-i18next, @lingui, vue-i18n, …) and the import patterns it uses **in this codebase**.

Every subsequent step is parameterized by these discovered values.

## Instructions

1. Find a file matching the following pattern `**/en.json` in the entire codebase. If this file doesn't exist, notify the user and ask how to set up i18n. **Do not read the content of `en.json` directly** — it can contain a massive amount of text and strain the context window rapidly. All inspection and modification must go through a subagent as described below.

2. **Discover the schema and naming convention.** Dispatch a subagent via the Task tool with these instructions verbatim:

   > Read the JSON file at `<path>`. Inspect its structure and the names of its keys, sampling several entries (not just the first) to confirm consistency. Output only a single JSON object — no prose, no code fences. The object must have these fields:
   >
   > - `keyShape`: `"flat"` if the top-level keys map directly to translation entries, or `"nested"` if entries are organized under namespace objects that must be addressed by dot-path (e.g. i18next style).
   > - `entryShape`: `"string"` if each translation entry is a bare string, or `"object"` if each entry is an object with named fields.
   > - `textField`: the name of the field inside an entry object that holds the user-facing text (e.g. `"text"`, `"message"`, `"defaultMessage"`, `"value"`). Use `null` if `entryShape` is `"string"`.
   > - `descriptionField`: the name of the field inside an entry object that holds developer-facing metadata describing where/how the text is used (e.g. `"description"`, `"comment"`, `"note"`, `"context"`). Use `null` if `entryShape` is `"string"` or if no such field exists.
   > - `entryShapeExample`: a representative example of one entry, with placeholder values, e.g. `{"text": "<TEXT>", "description": "<DESCRIPTION>"}` or `"<TEXT>"`. Do not include real data from the file.
   > - `namingConvention`: a label describing the convention observed in the leaf keys of the file. Use one of `"snake_case"`, `"camelCase"`, `"kebab-case"`, `"PascalCase"`, `"SCREAMING_SNAKE_CASE"`, `"dot.case"` if it cleanly matches; otherwise `"mixed"` with a short freeform description appended after a colon (e.g. `"mixed: mostly camelCase, some snake_case"`); or `null` if there are too few keys to tell.
   > - `namespaceConvention`: only relevant when `keyShape` is `"nested"`. The convention used for namespace names, in the same vocabulary as `namingConvention`. Otherwise `null`.
   > - `inconsistencies`: a string describing any structural or naming inconsistencies you noticed across entries, or `null` if entries are uniform.
   >
   > If the file is empty or `{}`, return `keyShape: "flat"`, `entryShape: "object"`, both field names as `null`, both convention fields as `null`, and note this in `inconsistencies`.

   Call the result `<schema>`. If `inconsistencies` is non-null, mention it to the user before proceeding. If `namingConvention` is `null` (empty file or too few keys), ask the user which convention to use, then treat their answer as `<schema>.namingConvention`.

3. Identify which user-facing strings need new translation entries.

   **If you yourself introduced the new strings earlier in this same conversation, skip the git inspection entirely** — you already know directly which strings you added and in which files, and re-discovering them via `git diff` only risks picking up unrelated noise (e.g., other in-flight edits in the working tree). Take your in-session list of newly-added user-facing strings as the result and proceed straight to [add new translation keys](#adding-new-translation-keys-to-the-language-file).

   Otherwise, discover the strings from git: run `git diff HEAD -- '*.tsx' '*.jsx'` and `git ls-files --others --exclude-standard '*.tsx' '*.jsx'` to check both uncommitted changes and untracked files. If there are new messages within JSX, [add new translation keys](#adding-new-translation-keys-to-the-language-file).

4. **If `<schema>.descriptionField` is `null`, skip this step entirely** — there is no description concept in this schema. Otherwise, dispatch a subagent via the Task tool with these instructions verbatim, substituting `<descField>` with `<schema>.descriptionField` and `<keyShape>` with `<schema>.keyShape`:

   > Read the JSON file at `<path>`. Find every translation entry whose `<descField>` field is the empty string `""`, `null`, or missing. Output only a single JSON array of identifiers — no prose, no code fences. If `<keyShape>` is `"flat"`, identifiers are top-level keys (e.g. `["key_a", "key_b"]`). If `<keyShape>` is `"nested"`, identifiers are dot-paths to the entry (e.g. `["common.save", "errors.network"]`). If there are no such entries, output `[]`.

   If the returned array is non-empty, [add description fields](#fix-missing-description-fields).

### Adding new translation keys to the language file

1. **Discover the i18n library used in this codebase.** Dispatch a subagent via the Task tool with these instructions verbatim:

   > Determine which i18n library this project uses and how it is invoked.
   >
   > Step 1: Read `package.json` (and any workspace package.json files relevant to where source files live). Examine `dependencies` and `devDependencies`. Common i18n libraries to look for: `next-intl`, `react-intl`, `i18next`, `react-i18next`, `@lingui/react`, `@lingui/macro`, `vue-i18n`, `svelte-i18n`. Note every match.
   >
   > Step 2: If multiple matches exist (or to confirm a single match), grep the source tree for established usage. Try patterns such as `useTranslations\(`, `useIntl\(`, `FormattedMessage`, `useLingui\(`, `\\bi18n\\.t\\(`, `getTranslations\(`, `useT\(`, restricted to `*.tsx`, `*.jsx`, `*.ts`, `*.js`. Identify the library that is actually wired up in the source.
   >
   > Step 3: Open 1–3 representative source files that already use the library — ideally one server component and one client component, if the framework distinguishes — and read their import lines and initialization lines verbatim.
   >
   > Output only a single JSON object — no prose, no code fences:
   >
   > - `library`: the detected package name as it appears in `package.json` (e.g. `"next-intl"`), or `null` if nothing is wired up.
   > - `serverImport`: the exact import statement to add in a server component file, copied from a real example in the codebase (e.g. `import { getTranslations } from "next-intl/server";`). `null` if the library does not distinguish server/client, or if no such file exists in this project.
   > - `clientImport`: the exact import statement to add in a client component file (e.g. `import { useTranslations } from "next-intl";` or `import { useIntl } from "react-intl";`). `null` if N/A.
   > - `serverInit`: the line(s) needed inside a server component to obtain the translator (e.g. `const t = await getTranslations();`). `null` if N/A.
   > - `clientInit`: the line(s) needed inside a client component to obtain the translator (e.g. `const t = useTranslations();` or `const intl = useIntl();`). `null` if N/A.
   > - `callExample`: a representative example showing how a hardcoded string in JSX is replaced using this library — copy the style actually used in the codebase. Examples: `{t("work_in_progress")}`, `{intl.formatMessage({ id: "workInProgress" })}`, `<FormattedMessage id="workInProgress" defaultMessage="Work in progress" />`, `<Trans id="work_in_progress">Work in progress</Trans>`.
   > - `notes`: any caveats — for example, "react-intl: codebase mixes `useIntl` and `<FormattedMessage>`; prefer the latter for static strings", or `null`.

   Call the result `<libraryConfig>`. If `library` is `null`, notify the user — the project has no i18n library installed even though there is an `en.json`, and no source-level changes can be made until one is set up.

2. Decide on new translation keys following `<schema>.namingConvention`. For example, the text "Work in progress" becomes:
   - `work_in_progress` if the convention is `snake_case`
   - `workInProgress` if the convention is `camelCase`
   - `work-in-progress` if the convention is `kebab-case`
   - `WorkInProgress` if the convention is `PascalCase`
   - …and so on.

   If `<schema>.keyShape` is `"nested"`, also pick an appropriate namespace path. The namespace segments themselves should follow `<schema>.namespaceConvention`, while the leaf segment follows `<schema>.namingConvention` (these are often, but not always, the same).

3. Build a single JSON object containing every new entry, **shaped according to `<schema>`**:
   - If `entryShape` is `"string"`: `{"<key>": "<text>", ...}` — no description involved.
   - If `entryShape` is `"object"` and `descriptionField` is non-null: `{"<key>": {"<textField>": "<text>", "<descField>": "<description>"}, ...}`.
   - If `entryShape` is `"object"` but `descriptionField` is `null`: `{"<key>": {"<textField>": "<text>"}, ...}`.
   - If `keyShape` is `"nested"`: nest the new entries under their namespace paths in the same object literal you would build by hand (e.g. `{"common": {"work_in_progress": {...}}}`).

4. Dispatch a subagent via the Task tool with these instructions verbatim:

   > Read the JSON file at `<path>`. Its schema is described by this object:
   >
   > ```
   > <schema>
   > ```
   >
   > Here is a JSON object of new entries to deep-merge in, shaped to match the schema:
   >
   > ```
   > <new entries object>
   > ```
   >
   > Step 1: For each leaf entry in the new entries (resolved by dot-path if the schema is nested), check whether an entry already exists at that path in the file. If any do, output exactly `CONFLICT: <comma-separated list of conflicting paths>` and stop — do not modify the file.
   >
   > Step 2: Otherwise, deep-merge the new entries into the existing object (preserving every existing entry unchanged, creating intermediate namespace objects as needed for nested schemas), write the result to `<path>.tmp`, then rename `<path>.tmp` to `<path>` so the write is atomic. After a successful write, output exactly `OK`.
   >
   > Output only `OK` or the `CONFLICT:` line — no other prose.

   If the subagent returns `CONFLICT:`, rename the conflicting keys (still respecting `<schema>.namingConvention`) and retry with a fresh subagent call.

5. Replace the original hardcoded strings in the source files using `<libraryConfig>`:
   - For each source file you need to edit, check whether it already imports and initializes the translator. If not, add `<libraryConfig>.serverImport` or `<libraryConfig>.clientImport` (whichever applies — pick based on whether the file is a server component or a client component, e.g. via the `"use client"` directive in Next.js codebases) and add `<libraryConfig>.serverInit` or `<libraryConfig>.clientInit` near the top of the component body.
   - At each call site, replace the hardcoded string following the form of `<libraryConfig>.callExample`. Some libraries use a `t()` function, others use a JSX component such as `<FormattedMessage>` or `<Trans>` — follow whatever pattern the example shows.
   - If `<libraryConfig>.notes` is non-null, take it into account.

### Fix missing description fields

This section only runs if `<schema>.descriptionField` is non-null.

1. For each identifier that needs a description, search the source tree for a call site referencing it (e.g. `t("<key>")`, `id="<key>"`, `<Trans id="<key>">`) to understand the context, then decide on an appropriate description. For nested schemas, the leaf key is usually the relevant search term, but namespace-aware projects often pass the full dot-path.

2. Build a single JSON object mapping every identifier to its new description, e.g. `{"common.save": "Submit button on the settings form", "errors.network": "Shown when fetch fails"}`. Use the same identifier shape (flat key or dot-path) that the discovery subagent returned.

3. Dispatch a subagent via the Task tool with these instructions verbatim, substituting `<descField>` with `<schema>.descriptionField` and `<keyShape>` with `<schema>.keyShape`:

   > Read the JSON file at `<path>`.
   >
   > Here is a JSON object mapping identifiers to new description values:
   >
   > ```
   > <updates object>
   > ```
   >
   > Identifiers are <`flat top-level keys` or `dot-paths into nested namespace objects`, as given by `<keyShape>`>. For every `(identifier, description)` pair, set the `<descField>` field of the entry at that location to the given value, leaving every other field on that entry untouched and leaving entries not mentioned untouched. If any identifier does not resolve to an existing entry, do not create it; instead collect it into a `missing` list.
   >
   > Write the resulting object to `<path>.tmp`, then rename `<path>.tmp` to `<path>` so the write is atomic.
   >
   > Output exactly `OK` if every identifier resolved, or `MISSING: <comma-separated list>` if any were not present in the file. No other prose.

   If the subagent reports missing identifiers, those were never added to the language file in the first place — investigate before retrying.