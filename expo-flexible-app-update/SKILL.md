---
name: expo-flexible-app-update
description: Adds PostHog-driven flexible store update prompts (not OTA) with semver gating, dismiss cooldown, and App Store / Play Store links. Use when adding app update modals, flexible-app-update flags, store version prompts, or replicating the wapda-bill-check update flow in Expo apps.
---

# Expo Flexible App Store Update

PostHog-controlled **store update** modal — guides users to App Store / Play Store. **Not** Expo OTA (`expo-updates`). Always dismissible; never forced.

Reference implementation: wapda-bill-check. Full templates: [reference.md](reference.md)

## Before you start

| Value | Where |
|-------|--------|
| `STORAGE_PREFIX` | e.g. `@my-app-name` |
| `IOS_ITUNES_ITEM_ID` | App Store Connect → Apple ID |
| `ANDROID_PACKAGE_NAME` | `app.json` → `android.package` |
| PostHog flag key | `flexible-app-update` |

```bash
npx expo install expo-application @react-native-async-storage/async-storage
```

Requires `posthog-react-native` if using the hook pattern.

---

## File manifest

| File | Role |
|------|------|
| `constants/posthog-feature-flags.ts` | `FLEXIBLE_APP_UPDATE_FLAG` + defaults |
| `lib/semver-compare.ts` | Numeric semver compare (no npm dep) |
| `lib/flexible-app-update-storage.ts` | AsyncStorage dismissal state |
| `lib/flexible-app-update.ts` | Parse payload, eligibility, store URL |
| `hooks/use-flexible-app-update.ts` | PostHog hook (swap point for hardcode) |
| `lib/flexible-app-update-navigation.ts` | Open/close presentation |
| `components/flexible-app-update-content.tsx` | Shared UI (logo, text, buttons) |
| `components/flexible-app-update-controller.tsx` | Auto-prompt orchestrator |
| `components/flexible-app-update-modal.tsx` | Android centered modal |
| `app/flexible-app-update.tsx` | iOS route (Expo 56 form sheet only) |
| `app/_layout.tsx` | Mount controller + optional sheet route |

---

## Integration checklist

```
- [ ] Add FLEXIBLE_APP_UPDATE_FLAG to posthog-feature-flags.ts
- [ ] Copy lib + hook files; replace STORAGE_PREFIX and store IDs
- [ ] Add ios.appStoreUrl + android.playStoreUrl to app.json
- [ ] Build UI (see generic prompt below)
- [ ] Wire controller in root layout
- [ ] Create PostHog flag with per-platform payload
- [ ] Add dev preview + clear dismissal in Settings (__DEV__)
```

---

## PostHog flag

**Key:** `flexible-app-update` · **Type:** boolean · **Rollout:** 100% when live

```json
{
  "android": {
    "enabled": true,
    "version": "1.2.0",
    "url": null,
    "remindAfterDays": 3,
    "title": "Update Available",
    "message": "A newer version is available with improvements and fixes.",
    "primaryButtonText": "Update Now",
    "secondaryButtonText": "Later",
    "showAfterLaunchDelayMs": 1000
  },
  "ios": {
    "enabled": true,
    "version": "1.2.0",
    "url": null,
    "remindAfterDays": 3,
    "title": "Update Available",
    "message": "A newer version is available with improvements and fixes.",
    "primaryButtonText": "Update Now",
    "secondaryButtonText": "Later",
    "showAfterLaunchDelayMs": 1000
  }
}
```

Flag must be **enabled** (`true`). App reads per-platform object via `Platform.OS`.

---

## Behavior rules

| Rule | Detail |
|------|--------|
| Show when | `enabled` + installed version **<** payload `version` (semver) |
| Dismiss | Save version + timestamp; hide for `remindAfterDays` (default 3) |
| New payload version | Show again even during cooldown |
| Store URL | payload `url` → `app.json` store URLs → constructed fallback |
| Store open fails | Small alert; user keeps using app |
| Web | No modal |
| `__DEV__` | Skip `remindAfterDays`; don't persist dismissals |

**Not forced** — user can always tap Later, close, or swipe dismiss.

---

## UI (generic prompt)

Use app theme tokens + dark mode. **Do not use `flex: 1` on sheet content** — wrap intrinsic height only.

**Content (top → bottom, centered):**
- App logo (`require("@/assets/icons/...")`)
- App name (`Constants.expoConfig?.name`)
- Title + message from payload
- Full-width primary button (Update Now)
- Full-width secondary button (Later)

**Platform presentation:**

| Expo SDK | iOS | Android |
|----------|-----|---------|
| **≤54 (legacy)** | RN `Modal` transparent — centered card, dimmed backdrop | Same |
| **56+** | Expo Router `formSheet` route, `sheetAllowedDetents: "fitToContents"`, grabber | Centered card modal |

For Expo 56 native sheets, read `building-native-ui/references/form-sheet.md` in expo-56 projects (Forex Factory pattern). **Do not** use fixed detent fractions (e.g. `0.58`) for compact content — use `fitToContents`.

---

## Root layout

```tsx
<FlexibleAppUpdateController />
```

Separate from OTA `UpdateBanner` / `ota-forced-update` flag.

---

## Dev testing

Settings → Developer ( `__DEV__` only ):
- **Preview flexible update** → `openFlexibleAppUpdatePresentation()`
- **Clear update dismissal** → `clearFlexibleUpdateDismissal()` + reload

Auto-prompt still needs payload `version` > installed. Preview bypasses eligibility.

---

## PostHog events (optional)

`flexible_update_shown`, `flexible_update_dismissed`, `flexible_update_store_opened`, `flexible_update_store_failed`

---

## Hardcode swap

Replace `useFlexibleAppUpdate` hook body:

```typescript
return {
  ready: true,
  config: resolvePlatformUpdateConfig({ ios: { enabled: true, version: "2.0.0", ... } }),
};
```
