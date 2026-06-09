---
name: expo-ota-publish-workflow
description: Use when configuring, dry-running, publishing, promoting, or reverting Expo EAS OTA updates with the ota-publish CLI from @ddedic/expo-fancy-ota-updates.
---

# Expo OTA Publish Workflow

## Overview

Use this skill to operate the package's `ota-publish` CLI. The CLI automates the normal Expo OTA release loop:

1. Read the current `ota-version.json`.
2. Generate changelog entries from git, a prompt, a file, or a custom hook.
3. Compute the next version/build number.
4. Write the updated `ota-version.json`.
5. Run `eas update --channel <channel> --message <message>` unless in dry-run mode.
6. Run configured hooks.

Default to `--dry-run` first for production-impacting commands.

## When to Use

Use this skill when the user asks to:

- Initialize `ota-updates.config.js`.
- Publish a development, preview, or production OTA update.
- Configure version strategies, compact channel aliases, or EAS message templates.
- Promote a tested update group from one channel to another.
- Revert/roll back a channel to a previous update group.
- Diagnose `ota-publish` config validation errors.

Use `expo-ota-ui-integration` instead when the task is about rendering provider/banner/info-screen UI in the app.

## Prerequisites

From the Expo app root:

```bash
pnpm add @ddedic/expo-fancy-ota-updates
pnpm add -D eas-cli
```

Confirm the project is ready:

```bash
git status --short
eas --version
eas project:info
```

If `changelog.source` is `git`, the app must be in a git repo with meaningful commits.

## Initialize Config

Create a starter config:

```bash
npx ota-publish init
```

Expected config shape:

```js
/** @type {import('@ddedic/expo-fancy-ota-updates').OTAConfig} */
export default {
  versionFile: './ota-version.json',
  baseVersion: 'package.json',
  versionFormat: '{major}.{minor}.{patch}-{channelAlias}.{build}',
  versionFormatByChannel: {
    production: '{major}.{minor}.{patch}-p{build}',
  },
  versionStrategy: 'build',
  channelAliases: {
    development: 'd',
    preview: 'pr',
    production: 'p',
  },
  changelog: {
    source: 'git',
    commitCount: 10,
    format: 'short',
    includeAuthor: false,
  },
  eas: {
    autoPublish: true,
    messageFormat: 'v{version}: {firstChange}',
    platforms: ['ios', 'android'],
  },
  channels: ['development', 'preview', 'production'],
  defaultChannel: 'development',
};
```

Validation rules to remember:

- `defaultChannel` must appear in `channels`.
- `versionFormatByChannel`, `channelAliases`, and `eas.messageFormatByChannel` keys must be known channels.
- `changelog.source: 'file'` requires `changelog.filePath`.
- `versionStrategy: 'custom'` requires `hooks.generateVersion`.
- `changelog.source: 'custom'` requires `hooks.generateChangelog`.

## Publish Safely

Always preview a production or preview release before publishing:

```bash
npx ota-publish --channel production --dry-run
```

Review:

- Next `version` and `buildNumber`.
- Generated changelog.
- Rendered EAS update message.
- Target channel and platform flags.

Then publish:

```bash
npx ota-publish --channel production
```

Platform-scoped publishing:

```bash
npx ota-publish --channel production --platform ios
npx ota-publish --channel production --platform ios --platform android
```

One-off overrides:

```bash
npx ota-publish --channel preview --strategy semver
npx ota-publish --channel production --version-format '{major}.{minor}.{patch}-p{build}'
npx ota-publish --channel production --message 'Fix startup crash on iOS'
npx ota-publish --channel production --no-increment
```

## Recommended Package Scripts

Add these to the app's `package.json` for repeatability:

```json
{
  "scripts": {
    "ota:dev": "ota-publish --channel development",
    "ota:preview": "ota-publish --channel preview",
    "ota:prod": "ota-publish --channel production",
    "ota:prod:dry": "ota-publish --channel production --dry-run",
    "ota:promote:preview-to-prod": "ota-publish promote --from preview --to production",
    "ota:revert:prod": "ota-publish revert --channel production"
  }
}
```

## Promote Between Channels

Use promotion when a preview update has been tested and should be copied to production.

```bash
npx ota-publish promote --from preview --to production --dry-run
npx ota-publish promote --from preview --to production
```

For non-interactive CI or explicit rollback plans, pass the update group:

```bash
npx ota-publish promote --from preview --to production --group <update-group-id> --yes
```

Remember: the source group is resolved from the branch linked to `--from`, and the republished target uses the branch linked to `--to`.

## Revert / Roll Back

Preview the rollback first:

```bash
npx ota-publish revert --channel production --dry-run
```

Then execute:

```bash
npx ota-publish revert --channel production
```

With an explicit group:

```bash
npx ota-publish revert --channel production --group <update-group-id> --message 'Rollback production to stable update' --yes
```

## Hooks Pattern

Use hooks when the release process needs deterministic project-specific logic.

```js
export default {
  // ...base config
  hooks: {
    beforePublish: async ({ changelog, channel, dryRun }) => {
      if (channel === 'production' && changelog.length === 0) {
        throw new Error('Production releases need a changelog');
      }
      return { changelog };
    },
    generateVersion: async ({ defaultVersion }) => defaultVersion,
    generateChangelog: async () => ['Custom release note'],
    afterPublish: async (version, context) => {
      console.log(`Published ${version.version} to ${context.channel}`);
    },
    onError: async (error, context) => {
      console.error(`OTA publish failed in ${context.cwd}:`, error.message);
    },
  },
};
```

## Common Pitfalls

1. **Skipping dry-run for production.** Always run `--dry-run` before production publish, promote, or revert.
2. **Mismatched channel names.** Keep `channels`, `channelAliases`, and per-channel template objects in sync.
3. **Expecting OTA to update native code.** EAS OTA updates JS/assets only. Native changes require a new binary build.
4. **Using `--no-increment` accidentally.** This preserves the existing version/build; only use it for republishing intentionally.
5. **Missing EAS auth or project link.** `eas update` needs logged-in EAS CLI and a linked Expo project.
6. **Uncommitted release file changes.** After a real publish, `ota-version.json` changes. Commit it if the app imports the file from source control.

## Verification Checklist

- [ ] `npx ota-publish --channel <channel> --dry-run` shows the intended channel, version, changelog, and EAS command.
- [ ] `ota-updates.config.js` validates with known channel names.
- [ ] `ota-version.json` contains `version`, numeric `buildNumber`, `releaseDate`, `channel`, and string-array `changelog`.
- [ ] Real publish output confirms `eas update` succeeded.
- [ ] The changed `ota-version.json` is committed when appropriate.
- [ ] A device on the target channel can fetch the update.
