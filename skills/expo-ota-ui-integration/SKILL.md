---
name: expo-ota-ui-integration
description: Use when integrating @ddedic/expo-fancy-ota-updates UI into an Expo or React Native app, especially wiring OTAUpdatesProvider, UpdateBanner, OTAInfoScreen, useOTAUpdates, and ota-version.json together.
---

# Expo Fancy OTA UI Integration

## Overview

Use this skill to add the `@ddedic/expo-fancy-ota-updates` runtime UI to an Expo app. The package has two halves that must stay connected:

- `ota-publish` writes an `ota-version.json` file with version, build number, channel, release date, and changelog.
- The React components read that file through `OTAUpdatesProvider` and display update state through `UpdateBanner`, `OTAInfoScreen`, and `useOTAUpdates()`.

The safest integration is provider at the app root, banner near the root UI, and an info screen in settings/debug routes.

## When to Use

Use this skill when the user asks to:

- Add OTA update UI to an Expo app.
- Show an update banner when an EAS update is available.
- Add a settings/debug screen with OTA version, channel, changelog, and actions.
- Connect `ota-version.json` to the app runtime.
- Debug why the banner or info screen is not showing the expected version.

Do not use this for publishing an EAS update itself; use the `expo-ota-publish-workflow` skill for CLI release workflows.

## Prerequisites

Install the package and required peers in the target Expo app:

```bash
pnpm add @ddedic/expo-fancy-ota-updates expo expo-updates expo-device react react-native react-native-reanimated react-native-safe-area-context
```

Optional, recommended visual peers:

```bash
pnpm add expo-linear-gradient lucide-react-native react-native-svg
```

The app must have `expo-updates` configured and must be built as an EAS development/preview/production build. Expo Go and simulators/dev mode can skip real update checks; expose those skip reasons through `lastSkippedReason` instead of assuming the library failed.

## Canonical Root Integration

In Expo Router, wrap the root layout. In a classic app, do this in `App.tsx`.

```tsx
import { Stack } from 'expo-router';
import { OTAUpdatesProvider, UpdateBanner } from '@ddedic/expo-fancy-ota-updates';
import versionData from '../ota-version.json';

export default function RootLayout() {
  return (
    <OTAUpdatesProvider
      config={{
        versionData,
        checkOnMount: true,
        checkOnForeground: true,
        minCheckIntervalMs: 30_000,
        recordSkippedChecks: true,
        autoDownload: false,
        autoReload: false,
        debug: __DEV__,
      }}
    >
      <UpdateBanner />
      <Stack />
    </OTAUpdatesProvider>
  );
}
```

For classic React Native:

```tsx
import { OTAUpdatesProvider, UpdateBanner } from '@ddedic/expo-fancy-ota-updates';
import versionData from './ota-version.json';

export default function App() {
  return (
    <OTAUpdatesProvider config={{ versionData, recordSkippedChecks: true }}>
      <UpdateBanner />
      <YourNavigation />
    </OTAUpdatesProvider>
  );
}
```

## Add an OTA Info Screen

Use `OTAInfoScreen` for a settings route, debug menu, internal build page, or QA screen.

```tsx
import { OTAInfoScreen } from '@ddedic/expo-fancy-ota-updates';
import { router } from 'expo-router';

export default function UpdatesRoute() {
  return <OTAInfoScreen onBack={() => router.back()} mode="developer" />;
}
```

For user-facing settings, prefer `mode="user"` and explicitly hide debug details:

```tsx
<OTAInfoScreen
  mode="user"
  showDebugSection={false}
  showUpdateId={false}
  showRuntimeVersion={false}
/>
```

## Use the Hook for Custom UI

Use `useOTAUpdates()` when the app needs custom buttons, telemetry, or conditional UI.

```tsx
import { Button, Text, View } from 'react-native';
import { useOTAUpdates } from '@ddedic/expo-fancy-ota-updates';

export function UpdatesCard() {
  const ota = useOTAUpdates();

  return (
    <View>
      <Text>Version: {ota.otaVersion}</Text>
      <Text>Channel: {ota.channel ?? 'unknown'}</Text>
      <Text>Status: {ota.status}</Text>
      {ota.lastSkippedReason ? <Text>Skipped: {ota.lastSkippedReason}</Text> : null}
      <Button title="Check" onPress={() => ota.checkForUpdate()} />
      {ota.isUpdateAvailable ? (
        <Button title="Download" onPress={() => ota.downloadUpdate()} />
      ) : null}
      {ota.isDownloaded ? (
        <Button title="Restart" onPress={() => ota.reloadApp()} />
      ) : null}
    </View>
  );
}
```

## Version Data Contract

The provider expects version data shaped like this:

```json
{
  "version": "1.0.0-p42",
  "buildNumber": 42,
  "releaseDate": "2026-01-01T00:00:00.000Z",
  "channel": "production",
  "changelog": ["Improve onboarding", "Fix settings crash"]
}
```

Prefer generating this with `ota-publish`. If hand-writing it for tests, keep `buildNumber` numeric and `changelog` an array of strings.

## Common Pitfalls

1. **Provider missing at the root.** `UpdateBanner`, `OTAInfoScreen`, and `useOTAUpdates()` must render under `OTAUpdatesProvider`.
2. **Forgetting `versionData`.** Without `config.versionData`, UI falls back to defaults and may not show the release users expect.
3. **Testing in Expo Go.** Real update checks require `expo-updates` in an EAS build. In dev/simulator contexts, expect skipped checks and inspect `lastSkippedReason`.
4. **Auto-reload surprises users.** Keep `autoReload: false` for most apps unless the product explicitly accepts automatic restarts.
5. **Double headers in navigation.** When using `OTAInfoScreen` inside native stack or tabs, avoid showing both the navigator header and the screen's internal header. Use `onBack`, custom `renderHeader`, or navigator header options intentionally.

## Verification Checklist

- [ ] `OTAUpdatesProvider` wraps all OTA UI.
- [ ] `UpdateBanner` renders once near the root, not inside every screen.
- [ ] `ota-version.json` is imported and passed through `config.versionData`.
- [ ] `OTAInfoScreen` is reachable from settings/debug navigation.
- [ ] `pnpm typecheck` or the app's TypeScript check passes.
- [ ] Test a development/preview EAS build, not only Expo Go.
