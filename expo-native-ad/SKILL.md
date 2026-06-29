---
name: expo-native-ad
description: Wires Expo NativeAdComponent with AdHeader icon row, 180px NativeMediaView, dev TestIds, prod platform unit IDs, retry/backoff, useForeground refresh, and web stub. Includes AdMob policy rules for ad visual separation. Use when adding native ads, NativeAdView, or NativeAdComponent to an Expo app.
---

# Expo: NativeAd

Functional wiring only. Preserve the **platform file splits** below.

## Stack assumptions

- Expo SDK 54+, **New Architecture** enabled, **expo-router** file-based routes.
- Path alias `@/*` → project root (see `tsconfig.json`).

---

## Native ad UI component (native + web)

- `components/native-ad.tsx` (native): `NativeAd.createForAdRequest` with **dev** `TestIds.NATIVE` and **prod** platform-specific unit IDs; retry with backoff; `useForeground` refresh with cooldown; render `NativeAdView` + `NativeAsset` + `NativeMediaView`. Use **`Colors` from `@/constants/theme`** (and `useThemeColor` where appropriate); do not introduce a second palette file.
- `components/native-ad.web.tsx`: export same `NativeAdComponent` name returning `null` so screens can import one module.

**IDs**: define `NATIVE_AD_UNIT_ID` **in** `components/native-ad.tsx` (alongside the component). Do **not** put it in a shared `constants/*.ts` that imports `react-native-google-mobile-ads`—that can break **web** / **static** builds. Keep `THEME_STORAGE_KEY` only in `app-theme-provider.tsx`.

Requires **`AdsInitProvider`** to have initialized AdMob before ads load (see **expo-admob-att** skill).

### UI layout (icon + media)

- **`AdHeader`**: if `nativeAd.icon?.url`, render 40×40 icon in a row beside advertiser + headline; otherwise stack them.
- **`NativeMediaView`**: `width: "100%"`, **`height: 180`**, `marginVertical: 6`.

---

## Agent checklist

- [ ] Add `NativeAdComponent` + `.web` stub; `NATIVE_AD_UNIT_ID` defined in `native-ad.tsx` (no shared const module importing AdMob).
- [ ] Implement `AdHeader` with icon row when icon URL present.
- [ ] Use `NativeMediaView` at **180px** height, full width.

**AdSense / AdMob policy safety:** Do not apply `borderRadius`, rounded wrappers, clipping masks, or `overflow: 'hidden'` to `NativeAdView`, `NativeMediaView`, CTA wrappers, asset wrappers, or any parent container that may contain AdMob disclosure / attribution icons. These icons must remain fully visible.

**Native ad visual separation / format-mimicking safety:**

Native ads must never look like normal app content, form sections, calculator cards, settings cards, list items, game UI, forum posts, or primary app actions.

When generating or modifying `NativeAdComponent`:

- Always render a clear text label such as `Advertisement` or `Ad` directly above or inside the ad area.
- The ad container must use ad-specific theme tokens, not the same `surface`, border, spacing, or card style used by normal app content.
- Add clear vertical separation before and after the ad slot, usually at least 16–24px/dp.
- Use a visible boundary such as `borderWidth`, `borderColor`, or `borderStyle: 'dashed'`.
- Do not make the advertiser name look like an app section header.
- Do not use the same CTA button style as the app’s main action buttons.
- Do not place native ads inside app content cards, calculator result cards, form groups, or components that imply app functionality.
- Keep AdChoices/disclosure icons fully visible. Do not use `overflow: 'hidden'`, clipping masks, rounded wrappers, or border radius on `NativeAdView`, `NativeMediaView`, CTA wrappers, asset wrappers, or parent containers containing the ad.

---

## References

- **Full copy-paste snippets** (native ad component + web stub): [reference.md](reference.md)

---

## Install in other projects

```bash
cp -R skills/expo-native-ad /path/to/your-app/.cursor/skills/
```
