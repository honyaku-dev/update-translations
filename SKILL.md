---
name: update-translations
description: Update language files when a new message was added to the codebase. If you have already updated en.json with "description" field filled in, you do not need this skill.
---

# update-translations

Create a language file if it doesn't exist, and update it with the new message key and value.

## Instructions

1. Find a file matching the following pattern `**/en.json` in the entire codebase. If this file doesn't exist, notify the user and ask how to set up i18n. After finding the path of en.json, do not read the content of it, as it can contain a massive amount of text and strain the context window rapidly. Instead, you can assume the schema of en.json is `Record<string, { text: string, description: string }>`, where "description" field describes where and how the text is used.
2. Run `git diff HEAD -- '*.tsx' '*.jsx'` and `git ls-files --others --exclude-standard '*.tsx' '*.jsx'` to check both uncommitted changes and untracked files. If there are new messages within JSX, [add new translation keys](#adding-new-translation-keys-to-enjson).
3. Run `jq -r '[to_entries[] | select(.value.description == "" or .value.description == null) | .key | "\"\(.)\""] | join(",")' <path>` to find translation keys you need to update. If the output is not empty, [add description fields](#fix-missing-description-fields).

### Adding new translation keys to en.json

1. Think of new translation keys. A translation key must be snake_case: `[a-z0-9]+(_[a-z0-9]+)*`. For example, a translation key for the text "Work in progress" should be `work_in_progress`.
2. Verify that the keys do not already exist: `jq -e 'has("<key>")' <path>` for each key. If a key already exists, choose a different name.
3. Build a single JSON object containing all new entries and merge them at once:

```bash
jq --argjson new '{ "<key1>": { "text": "<text1>", "description": "<desc1>" }, "<key2>": { "text": "<text2>", "description": "<desc2>" } }' '. * $new' <path> > <path>.tmp && mv <path>.tmp <path>
```

4. After adding translation keys, replace the original hardcoded strings in the source files with t() calls. If the `t` function is not available in the file, add `import { getTranslations } from "next-intl/server"` if the file is a server component, or `import { useTranslations } from "next-intl"` if the file is a client component, and call the function at the top of the component.

### Fix missing description fields

1. For each key that needs a description, find the pattern of `t\(['"]<key>['"]\)` to understand the context.
2. Build a single jq expression to update all descriptions at once:

```bash
jq '
  .["<key1>"].description = "<description1>" |
  .["<key2>"].description = "<description2>"
' <path> > <path>.tmp && mv <path>.tmp <path>
```
