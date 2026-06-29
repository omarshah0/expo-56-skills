---
name: expo-admob-att
description: Wires Expo AdMob initialization with iOS ATT delay, Android/iOS UMP consent, web shims, useAdmobLegal hook, and revoke-ads-consent for settings. Use when adding AdMob, react-native-google-mobile-ads, expo-tracking-transparency, AdsInitProvider, ATT prompt flow, or UMP consent reset to an Expo app.
---

# Expo: AdMob and ATT

Functional wiring only. Preserve the **provider order** and **platform file splits** below.

## Stack assumptions

- Expo SDK 54+, **New Architecture** enabled, **expo-router** file-based routes.
- Path alias `@/*` → project root (see `tsconfig.json`).

---

## AdMob initialization

### Dependencies

- `react-native-google-mobile-ads`
- `expo-tracking-transparency` (iOS ATT + aligns with UMP flow)

### Config (`app.json` / `app.config`)

- Add plugin `react-native-google-mobile-ads` with **placeholder** `iosAppId` / `androidAppId` (format `ca-app-pub-xxx~yyy`) and `userTrackingUsageDescription`.
- Keep `expo-tracking-transparency` in the plugins list (order is not critical; match project conventions).
- **Android**: include `com.google.android.gms.permission.AD_ID` in `android.permissions` if you serve ads.

### Provider split (Metro resolves `.web` automatically)

| File | Role |
|------|------|
| `providers/ads-init-provider.tsx` | Native: ATT delay → optional UMP (`AdsConsent`) → `mobileAds().initialize()`. Exposes `useAdsInit()` → `{ adsReady, isConsentRequired }`. |
| `providers/ads-init-provider.web.tsx` | Web: **no** native SDK imports; context `{ adsReady: true, isConsentRequired: false }` so imports never pull AdMob. |

**Native flow summary**

- **Android**: `AdsConsent.requestInfoUpdate` (optional `debugGeography: EEA` in `__DEV__`) → `loadAndShowConsentFormIfRequired` if available → initialize ads.
- **iOS**: Read ATT status; if `undetermined`, wait ~2s then `requestTrackingPermissionsAsync`. If ATT denied, initialize ads immediately. If granted, run UMP then initialize.

On any init failure, still set `adsReady: true` after logging so the app is not stuck (adjust if you prefer a hard gate).

### Optional hook

- `hooks/use-admob-legal.ts`: thin re-export of `useAdsInit()` for settings/legal copy that mentions consent.

### Consent reset (settings)

- `lib/revoke-ads-consent.ts`: `AdsConsent.reset()` → `requestInfoUpdate` → `loadAndShowConsentFormIfRequired` when form available; guard `Platform.OS === 'web'`.
- `lib/revoke-ads-consent.web.ts`: no-op for parity.

### Root layout

Wrap navigation so ads init runs app-wide:

1. Outer: **app theme** provider (or your existing theme system).
2. Inner: **`AdsInitProvider`**.
3. Then: router / `Stack` / `Tabs`.

`unstable_settings` with `anchor: '(tabs)'` is optional but matches expo-router tab anchoring used with tab groups.

---

## Agent checklist

**AdMob**

- [ ] Install packages; configure plugins and Android `AD_ID` if needed.
- [ ] Add `ads-init-provider.tsx` + `ads-init-provider.web.tsx`; wrap root layout.
- [ ] Optional: `use-admob-legal`, `revoke-ads-consent` (+ web no-op), legal WebView routes if required.

---

## References

- **Full copy-paste snippets** (providers, hooks, layouts, EAS config, legal WebView): [reference.md](reference.md)

---
