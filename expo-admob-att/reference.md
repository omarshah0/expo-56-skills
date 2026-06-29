# Reference: AdMob and ATT (cross-project)

Use this document as the **source of truth** when porting AdMob / ATT / UMP wiring to another Expo app. Replace placeholders (`YOUR_*`, `CHANGE_ME_*`) with project-specific values. **Path alias** `@/*` → repo root must match `tsconfig.json`.

---

## Table of contents

- [Dependencies (`package.json`)](#dependencies-packagejson)
- [Static Expo config (`app.json`)](#static-expo-config-appjson)
- [AdMob: `providers/ads-init-provider.tsx` (native)](#admob-providersads-init-providertsx-native)
- [AdMob: `providers/ads-init-provider.web.tsx` (web)](#admob-providersads-init-providerwebtsx-web)
- [Hooks: AdMob legal](#hooks-admob-legal)
- [UMP reset: `lib/revoke-ads-consent.ts` + web](#ump-reset-librevoke-ads-consentts--web)
- [Root layout: `app/_layout.tsx`](#root-layout-app_layouttsx)
- [Legal URLs + WebView: `constants/legal-config.ts`, `app/legal-webview.tsx`](#legal-urls--webview-constantslegal-configts-applegal-webviewtsx)
- [Metro / resolution](#metro--resolution)

---

## Dependencies (`package.json`)

Install versions appropriate to your Expo SDK (below match **Expo SDK 54** template).

```json
{
  "dependencies": {
    "expo-tracking-transparency": "~6.0.8",
    "react-native-google-mobile-ads": "^16.3.2"
  }
}
```

**Notes**

- `expo-tracking-transparency`: required for ATT APIs used in the native ads provider on iOS.
- `react-native-google-mobile-ads`: native AdMob; web uses `.web.tsx` shims so this is not loaded on web for the provider stub.

---

## Static Expo config (`app.json`)

**Plugins:** `expo-tracking-transparency`, `react-native-google-mobile-ads` (with App IDs). **Android ads:** add `com.google.android.gms.permission.AD_ID` to `android.permissions`.

```json
{
  "expo": {
    "android": {
      "permissions": ["com.google.android.gms.permission.AD_ID"]
    },
    "plugins": [
      "expo-tracking-transparency",
      [
        "react-native-google-mobile-ads",
        {
          "iosAppId": "ca-app-pub-XXXXXXXXXXXXXXXX~YYYYYYYYYY",
          "androidAppId": "ca-app-pub-XXXXXXXXXXXXXXXX~ZZZZZZZZZZ",
          "userTrackingUsageDescription": "This app uses your data to show personalized ads and improve your experience."
        }
      ]
    ]
  }
}
```

**Notes**

- App IDs use format `ca-app-pub-xxx~yyy` (AdMob **app** id, not ad unit id).

---

## AdMob: `providers/ads-init-provider.tsx` (native)

Metro loads this on **iOS and Android**. Early return for `web` is defensive if the file were ever bundled on web without the `.web` override.

```tsx
import { getTrackingPermissionsAsync, requestTrackingPermissionsAsync } from "expo-tracking-transparency";
import React, { createContext, useContext, useEffect, useMemo, useState } from "react";
import { Platform } from "react-native";
import mobileAds, { AdsConsent, AdsConsentDebugGeography } from "react-native-google-mobile-ads";

type AdsInitContextValue = {
  adsReady: boolean;
  isConsentRequired: boolean;
};

const AdsInitContext = createContext<AdsInitContextValue>({ adsReady: false, isConsentRequired: false });

/** Minimum time after the consent flow before showing the iOS ATT prompt (better UX than back-to-back dialogs). */
const ATT_PROMPT_DELAY_MS = 2000;

export function AdsInitProvider({ children }: { children: React.ReactNode }) {
  const [adsReady, setAdsReady] = useState(false);
  const [isConsentRequired, setIsConsentRequired] = useState(false);

  useEffect(() => {
    if (Platform.OS === "web") return;

    let cancelled = false;

    (async () => {
      try {
        // =========================
        // ANDROID FLOW
        // =========================
        if (Platform.OS === "android") {
          const info = await AdsConsent.requestInfoUpdate(__DEV__ ? { debugGeography: AdsConsentDebugGeography.EEA } : {});

          if (__DEV__) {
            console.log("[Android] Consent status:", info.status);
          }

          if (info.isConsentFormAvailable) {
            setIsConsentRequired(true);
            await AdsConsent.loadAndShowConsentFormIfRequired();
          }

          if (cancelled) return;

          await mobileAds().initialize();
          setAdsReady(true);
          return;
        }

        // =========================
        // iOS FLOW
        // =========================
        let attAllowed = false;

        const { status } = await getTrackingPermissionsAsync();

        if (status === "undetermined") {
          await new Promise((resolve) => setTimeout(resolve, ATT_PROMPT_DELAY_MS));
          if (cancelled) return;

          const res = await requestTrackingPermissionsAsync();
          attAllowed = res.status === "granted";
        } else {
          attAllowed = status === "granted";
        }

        if (__DEV__) {
          console.log("[iOS] ATT allowed:", attAllowed);
        }

        if (!attAllowed) {
          if (__DEV__) console.log("[iOS] ATT denied → init ads");

          await mobileAds().initialize();
          if (!cancelled) setAdsReady(true);
          return;
        }

        const info = await AdsConsent.requestInfoUpdate(__DEV__ ? { debugGeography: AdsConsentDebugGeography.EEA } : {});

        if (__DEV__) {
          console.log("[iOS] Consent status:", info.status);
        }

        if (info.isConsentFormAvailable) {
          setIsConsentRequired(true);
          await AdsConsent.loadAndShowConsentFormIfRequired();
        }

        if (cancelled) return;

        await mobileAds().initialize();
        setAdsReady(true);
      } catch (e) {
        console.error("Ads init failed:", e);
        setAdsReady(true);
      }
    })();

    return () => {
      cancelled = true;
    };
  }, []);

  const value = useMemo(() => ({ adsReady, isConsentRequired }), [adsReady, isConsentRequired]);

  return <AdsInitContext.Provider value={value}>{children}</AdsInitContext.Provider>;
}

export function useAdsInit(): AdsInitContextValue {
  return useContext(AdsInitContext);
}
```

**Behavior summary**

| Platform | Order                                                                           |
| -------- | ------------------------------------------------------------------------------- |
| Android  | UMP `requestInfoUpdate` → optional form → `mobileAds().initialize()`            |
| iOS      | ATT (delayed if undetermined) → if denied, init ads; if granted, UMP → init ads |
| Error    | Logs and sets `adsReady: true` so UI does not hang                              |

---

## AdMob: `providers/ads-init-provider.web.tsx` (web)

Same **export names** as native so imports stay `@/providers/ads-init-provider`. No `react-native-google-mobile-ads` import on web.

```tsx
import React, { createContext, useContext, useMemo } from "react";

type AdsInitContextValue = {
  adsReady: boolean;
  isConsentRequired: boolean;
};

const AdsInitContext = createContext<AdsInitContextValue>({
  adsReady: true,
  isConsentRequired: false,
});

/** Web: no AdMob / UMP / ATT — stable context so the bundle never pulls native ad SDKs. */
export function AdsInitProvider({ children }: { children: React.ReactNode }) {
  const value = useMemo(() => ({ adsReady: true, isConsentRequired: false }), []);
  return <AdsInitContext.Provider value={value}>{children}</AdsInitContext.Provider>;
}

export function useAdsInit(): AdsInitContextValue {
  return useContext(AdsInitContext);
}
```

---

## Hooks: AdMob legal

### `hooks/use-admob-legal.ts`

```tsx
/**
 * AdMob legal / consent surface: EU UMP + readiness.
 * @see https://developers.google.com/admob/ios/eu-consent
 * @see https://developers.google.com/admob/android/eu-consent
 */
import { useAdsInit } from "@/providers/ads-init-provider";

export function useAdmobLegal(): { isConsentRequired: boolean; adsReady: boolean } {
  const { isConsentRequired, adsReady } = useAdsInit();
  return { isConsentRequired, adsReady };
}
```

---

## UMP reset: `lib/revoke-ads-consent.ts` + web

### `lib/revoke-ads-consent.ts`

```tsx
import { Platform } from "react-native";
import { AdsConsent, AdsConsentDebugGeography } from "react-native-google-mobile-ads";

const consentOptions = __DEV__ ? { debugGeography: AdsConsentDebugGeography.EEA } : undefined;

/**
 * Resets UMP consent state and presents Google's form again when available.
 * `adsReady` is for call-site parity with settings UI.
 */
export async function revokeAdsConsentAndShowForm(_adsReady: boolean): Promise<void> {
  if (Platform.OS === "web") {
    return;
  }

  AdsConsent.reset();
  const info = await AdsConsent.requestInfoUpdate(consentOptions);

  if (info.isConsentFormAvailable) {
    await AdsConsent.loadAndShowConsentFormIfRequired();
  }
}
```

### `lib/revoke-ads-consent.web.ts`

```tsx
export async function revokeAdsConsentAndShowForm(_adsReady: boolean): Promise<void> {
  // no-op
}
```

---

## Root layout: `app/_layout.tsx`

Provider order: **AppThemeProvider** → **AdsInitProvider** → navigation. Adjust `Stack.Screen` entries to your routes.

```tsx
import { DarkTheme, DefaultTheme, ThemeProvider } from "@react-navigation/native";
import { Stack } from "expo-router";
import { StatusBar } from "expo-status-bar";
import "react-native-reanimated";
import { SafeAreaProvider } from "react-native-safe-area-context";

import { AdsInitProvider } from "@/providers/ads-init-provider";
import { AppThemeProvider, useAppThemeContext } from "@/providers/app-theme-provider";

export const unstable_settings = {
  anchor: "(tabs)",
};

function RootNavigation() {
  const { resolvedColorScheme } = useAppThemeContext();

  return (
    <SafeAreaProvider>
      <ThemeProvider value={resolvedColorScheme === "dark" ? DarkTheme : DefaultTheme}>
        <Stack screenOptions={{ headerBackTitle: "Back" }}>
          <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
          <Stack.Screen name="legal-webview" options={{ title: "", headerBackButtonDisplayMode: "minimal" }} />
        </Stack>
        <StatusBar style={resolvedColorScheme === "dark" ? "light" : "dark"} />
      </ThemeProvider>
    </SafeAreaProvider>
  );
}

export default function RootLayout() {
  return (
    <AppThemeProvider>
      <AdsInitProvider>
        <RootNavigation />
      </AdsInitProvider>
    </AppThemeProvider>
  );
}
```

---

## Legal URLs + WebView: `constants/legal-config.ts`, `app/legal-webview.tsx`

**Dependency:** `react-native-webview`. Only URLs listed in `LEGAL_URLS` (or matching `key`) load — avoids opening arbitrary URLs from query params.

### `constants/legal-config.ts`

```tsx
export const LEGAL_URLS = {
  termsOfService: "https://your-domain.com/terms",
  privacyPolicy: "https://your-domain.com/privacy",
  support: "https://your-domain.com/support",
} as const;

export type LegalDocKey = keyof typeof LEGAL_URLS;

export const LEGAL_SETTINGS_ROWS: { key: LegalDocKey; label: string }[] = [
  { key: "termsOfService", label: "Terms of Service" },
  { key: "privacyPolicy", label: "Privacy Policy" },
  { key: "support", label: "Support" },
];
```

### `app/legal-webview.tsx`

```tsx
import { LEGAL_URLS, type LegalDocKey } from "@/constants/legal-config";
import { useNavigation } from "@react-navigation/native";
import { useLocalSearchParams } from "expo-router";
import { useEffect, useMemo } from "react";
import { ActivityIndicator, Platform, StyleSheet, View } from "react-native";
import { useSafeAreaInsets } from "react-native-safe-area-context";
import { WebView } from "react-native-webview";

const ALLOWED_URLS = new Set<string>(Object.values(LEGAL_URLS));

function resolveLegalUrl(key: string | undefined, fallbackUrl: string | undefined): string | null {
  if (key && key in LEGAL_URLS) {
    return LEGAL_URLS[key as LegalDocKey];
  }
  if (fallbackUrl && ALLOWED_URLS.has(fallbackUrl)) {
    return fallbackUrl;
  }
  return null;
}

export default function LegalWebViewScreen() {
  const navigation = useNavigation();
  const insets = useSafeAreaInsets();
  const { url, title, key } = useLocalSearchParams<{
    url?: string;
    title?: string;
    key?: string;
  }>();

  const resolvedUrl = useMemo(() => resolveLegalUrl(typeof key === "string" ? key : undefined, typeof url === "string" ? url : undefined), [key, url]);

  useEffect(() => {
    if (title) {
      navigation.setOptions({ title: String(title) });
    }
  }, [navigation, title]);

  if (!resolvedUrl) {
    return (
      <View style={[styles.centered, { paddingBottom: insets.bottom }]}>
        <ActivityIndicator />
      </View>
    );
  }

  return (
    <View style={[styles.flex, { paddingBottom: Platform.OS === "ios" ? 0 : insets.bottom }]}>
      <WebView
        source={{ uri: resolvedUrl }}
        style={styles.flex}
        startInLoadingState
        renderLoading={() => (
          <View style={styles.loader}>
            <ActivityIndicator />
          </View>
        )}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  flex: {
    flex: 1,
  },
  centered: {
    flex: 1,
    alignItems: "center",
    justifyContent: "center",
  },
  loader: {
    ...StyleSheet.absoluteFillObject,
    alignItems: "center",
    justifyContent: "center",
  },
});
```

**Navigation example:** `router.push({ pathname: '/legal-webview', params: { key: 'privacyPolicy', title: 'Privacy' } })`.

---

## Metro / resolution

- `*.web.tsx` overrides `*.tsx` for the same import path on web (`ads-init-provider`, `revoke-ads-consent`).
- No extra `metro.config.js` is required beyond standard Expo for this pattern.
