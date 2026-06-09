# Agent Skills for expo-fancy-ota-updates

This directory contains reusable Agent Skills for working with `@ddedic/expo-fancy-ota-updates` in Expo apps.

They follow the same folder-based `SKILL.md` structure used by the open agent skills ecosystem and directories such as [skills.sh](https://www.skills.sh/):

```text
skills/
  <skill-name>/
    SKILL.md
```

Each `SKILL.md` has YAML frontmatter with a `name` and `description`, followed by task-specific instructions and examples.

## Included skills

- `expo-ota-ui-integration` — integrate `OTAUpdatesProvider`, `UpdateBanner`, `OTAInfoScreen`, `useOTAUpdates`, and `ota-version.json` into an Expo app.
- `expo-ota-publish-workflow` — configure and operate the `ota-publish` CLI for dry runs, publishing, promotion, and reverts.
- `expo-ota-channel-surfing` — implement and troubleshoot runtime channel switching with `switchChannel()`.
- `expo-ota-branding-i18n` — customize themes, translations, render props, and user-facing OTA screens.

## Install with the skills CLI

From another project, list or install these skills from this repository:

```bash
npx skills add CarlosZiegler/expo-fancy-ota-updates --list
npx skills add CarlosZiegler/expo-fancy-ota-updates --skill expo-ota-ui-integration
npx skills add CarlosZiegler/expo-fancy-ota-updates --skill expo-ota-publish-workflow
```

Install all skills:

```bash
npx skills add CarlosZiegler/expo-fancy-ota-updates --skill '*'
```

For Claude Code specifically, project installs are typically placed under `.claude/skills/` in the consuming project. The `skills` CLI handles agent-specific install locations.

## Local development

When editing these skills:

1. Keep `name` identical to the parent directory name.
2. Keep descriptions explicit about when an agent should use the skill.
3. Prefer concrete commands and code snippets over abstract guidance.
4. Validate that each `SKILL.md` starts with YAML frontmatter and has a non-empty Markdown body.
