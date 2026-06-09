---
name: expo-ota-channel-surfing
description: Use when implementing or troubleshooting runtime update-channel switching with expo-fancy-ota-updates, including preview/production QA flows and safe UI around switchChannel().
---

# Expo OTA Channel Surfing

## Overview

`@ddedic/expo-fancy-ota-updates` exposes runtime channel switching through `useOTAUpdates().switchChannel(channel)`. This is useful for internal QA, preview rings, support diagnostics, and developer builds where testers need to move between EAS update channels without rebuilding the binary.

Channel switching is powerful. Treat it as an internal/debug capability unless the product deliberately exposes it to users.

## When to Use

Use this skill when the user asks to:

- Add a channel switcher UI for development, preview, beta, or production channels.
- Build a QA screen that can move devices between EAS update channels.
- Diagnose `switchChannel` errors or skipped switches.
- Combine `OTAInfoScreen` with channel-switching controls.
- Explain safe product rules for update-channel switching.

Use `expo-ota-publish-workflow` when the task is about creating, promoting, or reverting update groups on EAS.

## Mental Model

There are two separate concerns:

- **Publishing:** `ota-publish` or `eas update` sends bundles to named channels.
- **Runtime selection:** `switchChannel(channel)` changes which channel this installed app checks for next.

Switching channels does not create updates. It only changes the runtime's target channel and then the app must check/download/reload as appropriate.

## Basic Channel Switcher

```tsx
import { Alert, Button, View } from 'react-native';
import { useOTAUpdates } from '@ddedic/expo-fancy-ota-updates';

const CHANNELS = ['development', 'preview', 'production'];

export function ChannelSwitcher() {
  const {
    channel,
    isSwitchingChannel,
    switchChannel,
    checkForUpdate,
    downloadUpdate,
    reloadApp,
  } = useOTAUpdates();

  async function onSwitch(nextChannel: string) {
    const result = await switchChannel(nextChannel);

    if (!result.success) {
      Alert.alert('Channel switch failed', result.reason ?? result.error?.message ?? 'Unknown error');
      return;
    }

    const check = await checkForUpdate();
    if (check.isAvailable) {
      await downloadUpdate();
      Alert.alert('Update ready', 'Restart the app to load this channel.', [
        { text: 'Later' },
        { text: 'Restart', onPress: reloadApp },
      ]);
    } else {
      Alert.alert('Channel switched', `Now using ${nextChannel}. No update available yet.`);
    }
  }

  return (
    <View>
      {CHANNELS.map((next) => (
        <Button
          key={next}
          title={next === channel ? `${next} ✓` : `Switch to ${next}`}
          disabled={isSwitchingChannel || next === channel}
          onPress={() => onSwitch(next)}
        />
      ))}
    </View>
  );
}
```

## Safer QA-Only Pattern

Hide channel switching in production UI unless explicitly enabled:

```tsx
const canSwitchChannels = __DEV__ || process.env.EXPO_PUBLIC_ENABLE_OTA_CHANNEL_SWITCHER === 'true';

return canSwitchChannels ? <ChannelSwitcher /> : null;
```

If exposing to support staff, add a confirmation step:

```tsx
Alert.alert(
  'Switch update channel?',
  `This will move the app from ${channel ?? 'unknown'} to ${nextChannel}.`,
  [
    { text: 'Cancel', style: 'cancel' },
    { text: 'Switch', style: 'destructive', onPress: () => performSwitch(nextChannel) },
  ]
);
```

## Combine with OTAInfoScreen

A good internal updates screen includes:

- Current channel.
- Runtime version.
- Embedded vs OTA update status.
- Last skipped reason.
- Buttons to check/download/reload.
- Channel switcher below the standard info screen.

```tsx
import { ScrollView } from 'react-native';
import { OTAInfoScreen } from '@ddedic/expo-fancy-ota-updates';

export default function InternalUpdatesScreen() {
  return (
    <ScrollView>
      <OTAInfoScreen mode="developer" showDebugSection />
      <ChannelSwitcher />
    </ScrollView>
  );
}
```

Avoid nesting competing scroll views in production screens; if layout gets awkward, customize `renderActions` or build a custom screen from `useOTAUpdates()`.

## Operational Checklist Before Enabling Channels

- Each target channel exists in EAS.
- Each target channel has a compatible runtime version for the installed binary.
- Test devices have a binary configured for `expo-updates`.
- The switcher labels are clear for non-engineers, e.g. `Preview QA` instead of raw `preview` if needed.
- There is a safe path back to the stable channel.

## Common Failure Modes

1. **No compatible update.** The channel may have updates, but none for the installed binary's runtime version.
2. **Testing in dev contexts.** Expo Go, dev clients, simulators, or update-disabled builds may skip checks. Surface `lastSkippedReason` and `SwitchChannelResult.reason`.
3. **Switch without reload.** A downloaded update is not running until `reloadApp()` is called.
4. **Public users on preview channels.** Treat switchers as internal unless there is a strong product reason.
5. **Channel names drift.** Keep switcher constants aligned with `ota-updates.config.js`, EAS channels, and release scripts.

## Troubleshooting Steps

1. Display current `channel`, `runtimeVersion`, `currentUpdateId`, and `isEmbeddedUpdate` from `useOTAUpdates()`.
2. Call `switchChannel(nextChannel)` and log the full `SwitchChannelResult`.
3. Call `checkForUpdate()` and inspect `CheckResult.isAvailable`, `isSkipped`, `reason`, and `error`.
4. Confirm EAS has an update on `nextChannel` for the same runtime version.
5. If downloaded, call `reloadApp()` and re-open the info screen to confirm the new channel/version.

## Verification Checklist

- [ ] The UI disables channel buttons while `isSwitchingChannel` is true.
- [ ] Failures show `reason` or `error.message` to the tester.
- [ ] The app checks for an update after switching.
- [ ] Downloaded updates prompt for a restart or call `reloadApp()` intentionally.
- [ ] The switcher is hidden or guarded outside intended QA/support contexts.
