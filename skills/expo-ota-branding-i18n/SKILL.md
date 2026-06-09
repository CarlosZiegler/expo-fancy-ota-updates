---
name: expo-ota-branding-i18n
description: Use when customizing expo-fancy-ota-updates with brand themes, localized copy, custom UpdateBanner rendering, or custom OTAInfoScreen sections.
---

# Expo OTA Branding and i18n

## Overview

Use this skill when the default OTA UI needs to match an app's product design, language, or information architecture. `@ddedic/expo-fancy-ota-updates` supports three levels of customization:

1. **Theme tokens** through `OTAUpdatesProvider theme`.
2. **Translations** through `OTAUpdatesProvider translations`.
3. **Render props** for `UpdateBanner` and `OTAInfoScreen` sections.

Start with theme and translations. Use render props only when the layout or component structure must change.

## When to Use

Use this skill when the user asks to:

- Make the update banner match app branding.
- Localize OTA UI text into German, Portuguese, Spanish, French, or another language.
- Replace the default banner with custom UI.
- Hide debug-heavy fields for user-facing settings.
- Customize `OTAInfoScreen` sections with `renderInfo`, `renderActions`, or `renderChangelog`.

Use `expo-ota-ui-integration` first if the provider/banner/info screen are not integrated at all.

## Theme Customization

Define brand tokens and pass them to the provider. Theme values are merged with defaults, but nested `colors` should include every color you rely on for contrast.

```tsx
import { OTAUpdatesProvider } from '@ddedic/expo-fancy-ota-updates';
import versionData from './ota-version.json';

const otaTheme = {
  colors: {
    primary: '#7C3AED',
    primaryLight: '#A78BFA',
    background: '#0B0B0F',
    backgroundSecondary: '#15151C',
    backgroundTertiary: '#222230',
    text: '#FFFFFF',
    textSecondary: '#D1D5DB',
    textTertiary: '#9CA3AF',
    border: '#2D2D3A',
    error: '#EF4444',
    success: '#10B981',
    warning: '#F59E0B',
  },
  bannerGradient: ['#7C3AED', '#2563EB'] as [string, string],
  borderRadius: 18,
  buttonBorderRadius: 14,
  animation: {
    duration: 250,
    pulseDuration: 1800,
  },
};

export function AppShell({ children }) {
  return (
    <OTAUpdatesProvider theme={otaTheme} config={{ versionData }}>
      {children}
    </OTAUpdatesProvider>
  );
}
```

## Translations

Pass only the strings you need to override, or pass a complete locale object for consistency.

```tsx
const portugueseTranslations = {
  banner: {
    updateAvailable: 'Nova atualização disponível',
    updateReady: 'Atualização pronta',
    downloading: 'Baixando atualização...',
    versionAvailable: 'Uma nova versão está disponível',
    restartToApply: 'Reinicie para aplicar as mudanças',
    updateButton: 'Atualizar',
    restartButton: 'Reiniciar',
  },
  infoScreen: {
    title: 'Atualizações OTA',
    statusTitle: 'Status da atualização',
    embeddedBuild: 'Build embutido',
    otaUpdate: 'Atualização OTA',
    runtimeVersion: 'Versão runtime',
    otaVersion: 'Versão OTA',
    releaseDate: 'Data de lançamento',
    updateId: 'ID da atualização',
    channel: 'Canal',
    whatsNew: 'Novidades',
    checkForUpdates: 'Verificar atualizações',
    downloadUpdate: 'Baixar atualização',
    reloadApp: 'Reiniciar app',
    debugTitle: 'Depuração',
    simulateUpdate: 'Simular atualização',
    hideSimulation: 'Ocultar simulação',
    devMode: 'Modo desenvolvimento',
    notAvailable: 'Indisponível',
    none: 'Nenhum',
  },
};

<OTAUpdatesProvider translations={portugueseTranslations}>
  <YourApp />
</OTAUpdatesProvider>
```

For dynamic locale selection:

```tsx
import * as Localization from 'expo-localization';

const translationsByLocale = {
  en: englishTranslations,
  pt: portugueseTranslations,
  de: germanTranslations,
};

const locale = Localization.locale.split('-')[0];
const translations = translationsByLocale[locale] ?? translationsByLocale.en;
```

## Custom UpdateBanner

Use `renderBanner` when the app needs a fully branded banner but still wants the package's state and actions.

```tsx
import { Pressable, Text, View } from 'react-native';
import { UpdateBanner } from '@ddedic/expo-fancy-ota-updates';

<UpdateBanner
  renderBanner={({
    isUpdateAvailable,
    isDownloading,
    isDownloaded,
    otaVersion,
    onUpdate,
    onRestart,
    onDismiss,
    theme,
    translations,
  }) => {
    if (!isUpdateAvailable && !isDownloaded) return null;

    return (
      <View style={{ padding: 16, backgroundColor: theme.colors.primary, borderRadius: 16 }}>
        <Text style={{ color: 'white', fontWeight: '700' }}>
          {isDownloaded ? translations.updateReady : translations.updateAvailable}
        </Text>
        <Text style={{ color: 'white' }}>Version {otaVersion}</Text>
        <Pressable onPress={isDownloaded ? onRestart : onUpdate} disabled={isDownloading}>
          <Text style={{ color: 'white' }}>
            {isDownloaded ? translations.restartButton : translations.updateButton}
          </Text>
        </Pressable>
        <Pressable onPress={onDismiss}>
          <Text style={{ color: 'white' }}>Dismiss</Text>
        </Pressable>
      </View>
    );
  }}
/>
```

## User-Facing OTAInfoScreen

For production settings screens, reduce noisy diagnostics:

```tsx
<OTAInfoScreen
  mode="user"
  showRuntimeVersion={false}
  showUpdateId={false}
  showDebugSection={false}
  showCheckButton
  showChangelog
/>
```

For custom sections:

```tsx
<OTAInfoScreen
  renderChangelog={({ otaChangelog, theme }) => (
    <View style={{ padding: 16 }}>
      <Text style={{ color: theme.colors.text, fontWeight: '700' }}>Latest changes</Text>
      {otaChangelog.map((item) => (
        <Text key={item} style={{ color: theme.colors.textSecondary }}>• {item}</Text>
      ))}
    </View>
  )}
/>
```

## Accessibility and UX Rules

- Keep update CTAs explicit: `Update`, `Download`, `Restart` should not be ambiguous.
- Avoid auto-reload in user-facing flows unless the user has opted in.
- Preserve high contrast for banner text over gradients.
- Localize `restartToApply` carefully; users need to understand the app will reload.
- Do not show raw update IDs to normal users unless support needs them.

## Common Pitfalls

1. **Partial colors with poor contrast.** If changing backgrounds, verify text, secondary text, border, success, warning, and error colors too.
2. **Render prop loses actions.** Custom banners must still call `onUpdate`, `onRestart`, and `onDismiss` appropriately.
3. **Forgetting `hideSimulation`.** Complete translation objects should include `hideSimulation` along with `simulateUpdate`.
4. **Debug info in production settings.** Prefer `mode="user"`, hide update IDs, and hide debug sections for customer-facing routes.
5. **Locale object recreated every render.** Memoize dynamic translations if they depend on hooks to avoid unnecessary re-renders.

## Verification Checklist

- [ ] Banner text remains readable in light and dark mode.
- [ ] Localized copy covers banner and info-screen labels used by the selected mode.
- [ ] Custom `renderBanner` returns `null` when no banner should be visible.
- [ ] User-facing screens hide debug-only fields.
- [ ] Update/download/restart actions still work after customization.
