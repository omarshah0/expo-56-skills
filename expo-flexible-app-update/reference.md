# Flexible App Store Update — Reference

Replace placeholders before copying:

- `{STORAGE_PREFIX}` — e.g. `@my-app-name`
- `{IOS_ITUNES_ITEM_ID}` — numeric App Store Apple ID
- `{ANDROID_PACKAGE_NAME}` — e.g. `com.example.app`

Source of truth: wapda-bill-check project files listed below.

---

## File layout

```
app/
├── _layout.tsx                          ← mount controller + iOS sheet route (Expo 56+)
└── flexible-app-update.tsx              ← iOS sheet screen (Expo 56+ only)
components/
├── flexible-app-update-content.tsx      ← shared UI
├── flexible-app-update-controller.tsx   ← auto-prompt
└── flexible-app-update-modal.tsx        ← Android modal
constants/
└── posthog-feature-flags.ts             ← FLEXIBLE_APP_UPDATE_FLAG
hooks/
└── use-flexible-app-update.ts
lib/
├── semver-compare.ts
├── flexible-app-update-storage.ts
├── flexible-app-update.ts
└── flexible-app-update-navigation.ts
```

---

## posthog-feature-flags.ts

```typescript
export const FLEXIBLE_APP_UPDATE_FLAG = "flexible-app-update";

export const FLEXIBLE_APP_UPDATE_DEFAULTS = {
  enabled: false,
  version: "0.0.0",
  url: null,
  remindAfterDays: 3,
  title: "Update Available",
  message: "A newer version is available with improvements and fixes.",
  primaryButtonText: "Update Now",
  secondaryButtonText: "Later",
  showAfterLaunchDelayMs: 1_000,
} as const;
```

---

## app.json store URLs

```json
{
  "expo": {
    "ios": {
      "appStoreUrl": "https://apps.apple.com/app/apple-store/id{IOS_ITUNES_ITEM_ID}"
    },
    "android": {
      "package": "{ANDROID_PACKAGE_NAME}",
      "playStoreUrl": "https://play.google.com/store/apps/details?id={ANDROID_PACKAGE_NAME}"
    }
  }
}
```

---

## lib/semver-compare.ts

Copy from wapda-bill-check. Exports `compareSemver(a, b)` and `isValidSemver(version)`.

Show modal when: `compareSemver(installed, payloadVersion) === -1`.

Installed version: `Application.nativeApplicationVersion` → fallback `Constants.expoConfig?.version`.

---

## lib/flexible-app-update-storage.ts

AsyncStorage keys:
- `{STORAGE_PREFIX}/flexible-update-dismissed-version`
- `{STORAGE_PREFIX}/flexible-update-dismissed-at`

Exports: `getFlexibleUpdateDismissal`, `recordFlexibleUpdateDismissal`, `clearFlexibleUpdateDismissal`.

---

## lib/flexible-app-update.ts

Core exports:
- `resolvePlatformUpdateConfig(payload)` — reads `ios` / `android` from PostHog payload
- `shouldShowFlexibleAppUpdate(config)` — semver + dismissal cooldown (`__DEV__` skips cooldown)
- `resolveStoreUpdateUrl(config)` — url → app.json → constructed
- `openStoreUpdateUrl(config)` — Linking + non-blocking Alert on failure

Parsing: defensive; invalid fields fall back to `FLEXIBLE_APP_UPDATE_DEFAULTS`.

---

## hooks/use-flexible-app-update.ts

Pattern matches `use-forced-ota-update.ts`:
- `useFeatureFlagWithPayload(FLEXIBLE_APP_UPDATE_FLAG)`
- Reload flags on `AppState` active
- Return `{ ready, config }`; safe default when flag off/not ready

---

## lib/flexible-app-update-navigation.ts

```typescript
openFlexibleAppUpdatePresentation();   // iOS: router.push sheet route; Android: show modal
closeFlexibleAppUpdatePresentation();    // router.back() / hide modal
markFlexibleAppUpdateSheetClosed();      // iOS route unmount
```

---

## components/flexible-app-update-controller.tsx

1. Wait PostHog `ready`
2. `shouldShowFlexibleAppUpdate(config)`
3. Delay `showAfterLaunchDelayMs`
4. Re-check eligibility
5. `openFlexibleAppUpdatePresentation()` once per session
6. Android: render `FlexibleAppUpdateModal` via `useSyncExternalStore`

Dev helpers: `resetFlexibleAppUpdateHandledSession()`.

---

## UI — legacy Expo (≤54)

Both platforms: RN `Modal` `transparent`, dimmed backdrop, centered card (~340px max width).

Shared `FlexibleAppUpdateContent` — **no `flex: 1`** on root; intrinsic height only.

---

## UI — Expo 56+ iOS form sheet

`app/_layout.tsx`:

```tsx
<Stack.Screen
  name="flexible-app-update"
  options={{
    presentation: "formSheet",
    headerShown: false,
    sheetGrabberVisible: true,
    sheetAllowedDetents: "fitToContents",
    contentStyle: { backgroundColor: "transparent" },
  }}
/>
```

`app/flexible-app-update.tsx`:
- Render `FlexibleAppUpdateContent` with safe-area bottom padding
- `beforeRemove` → record dismissal
- `onUpdate` → `openStoreUpdateUrl`

See `building-native-ui/references/form-sheet.md` in expo-56 Forex Factory project for full form-sheet patterns.

---

## Settings dev snippet

```typescript
import { resetFlexibleAppUpdateHandledSession } from '@/components/flexible-app-update-controller';
import { openFlexibleAppUpdatePresentation } from '@/lib/flexible-app-update-navigation';
import { clearFlexibleUpdateDismissal } from '@/lib/flexible-app-update-storage';

// Preview
resetFlexibleAppUpdateHandledSession();
openFlexibleAppUpdatePresentation();

// Clear dismissal
await clearFlexibleUpdateDismissal();
resetFlexibleAppUpdateHandledSession();
```

---

## vs OTA update banner

| | Flexible store update | OTA UpdateBanner |
|--|----------------------|------------------|
| Flag | `flexible-app-update` | `ota-forced-update` |
| Destination | App Store / Play Store | `expo-updates` reload |
| Forced | Never | Optional |
| Version source | PostHog payload semver | Expo Updates manifest |

Keep both systems independent.

---

## wapda-bill-check mapping

| File | Notes |
|------|-------|
| `hooks/use-flexible-app-update.ts` | PostHog hook |
| `app/flexible-app-update.tsx` | iOS form sheet (SDK 54 project — migrate UI to 56 pattern when upgrading) |
| `components/flexible-app-update-modal.tsx` | Android only |
