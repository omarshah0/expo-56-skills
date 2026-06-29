# Reference: NativeAd (cross-project)

Use this document as the **source of truth** when porting native ad UI to another Expo app. Replace placeholders (`YOUR_*`, `CHANGE_ME_*`) with project-specific values. **Path alias** `@/*` → repo root must match `tsconfig.json`.

**Semantic theme contract:** `NativeAdComponent` expects `Colors.light` / `Colors.dark` to include at least: `text`, `surface`, `muted`, `tint`, `onPrimary`, `tabIconSelected`, `border`, `adMuted`, `adSurface`, `adBorder`. You may replace hex values and fonts; keep those keys (or update the components accordingly).

---

## Table of contents

- [Native ad: `components/native-ad.tsx`](#native-ad-componentsnative-adtsx)
- [Native ad web stub: `components/native-ad.web.tsx`](#native-ad-web-stub-componentsnative-adwebtsx)
- [Metro / resolution](#metro--resolution)

---

## Native ad: `components/native-ad.tsx`

Keep **`NATIVE_AD_UNIT_ID`** in this file (export optional). Avoid a separate `constants/const-config.ts` that imports `react-native-google-mobile-ads` — that can pull the native SDK into shared bundles and break **web** or **static** builds.

```tsx
import { Colors } from "@/constants/theme";
import { useColorScheme } from "@/hooks/use-color-scheme";
import { useCallback, useEffect, useRef, useState } from "react";
import { Platform, StyleSheet, Text, View, type StyleProp, type ViewStyle } from "react-native";
import { NativeAd, NativeAdView, NativeAsset, NativeAssetType, NativeMediaView, TestIds, useForeground } from "react-native-google-mobile-ads";

export const NATIVE_AD_UNIT_ID = __DEV__
  ? TestIds.NATIVE
  : (Platform.select({
      ios: "CHANGE_ME_NATIVE_AD_UNIT_ID_IOS",
      android: "CHANGE_ME_NATIVE_AD_UNIT_ID_ANDROID",
    }) ?? TestIds.NATIVE);

const MAX_RETRIES = 3;
const RETRY_DELAYS = [2000, 5000, 10_000] as const;
const AD_COOLDOWN_MS = 100_000;

type NativeAdSlot = {
  nativeAd: NativeAd | null;
  refreshAd: () => void;
  isLoading: boolean;
};

function useNativeAdSlot(): NativeAdSlot {
  const [nativeAd, setNativeAd] = useState<NativeAd | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const adRef = useRef<NativeAd | null>(null);
  const retryTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  const lastLoadedAtRef = useRef<number>(0);
  const mountedRef = useRef(true);

  const loadAd = useCallback(async (retryCount = 0) => {
    try {
      const ad = await NativeAd.createForAdRequest(NATIVE_AD_UNIT_ID);
      if (!mountedRef.current) {
        ad.destroy();
        return;
      }
      adRef.current?.destroy();
      adRef.current = ad;
      lastLoadedAtRef.current = Date.now();
      setNativeAd(ad);
      setIsLoading(false);
    } catch (e) {
      console.warn("Native ad failed to load", e);
      if (!mountedRef.current) {
        return;
      }
      if (retryCount < MAX_RETRIES) {
        retryTimerRef.current = setTimeout(() => {
          if (mountedRef.current) {
            void loadAd(retryCount + 1);
          }
        }, RETRY_DELAYS[retryCount]);
      } else {
        setIsLoading(false);
      }
    }
  }, []);

  const refreshAd = useCallback(() => {
    const elapsed = Date.now() - lastLoadedAtRef.current;
    if (elapsed < AD_COOLDOWN_MS) {
      return;
    }
    if (retryTimerRef.current) {
      clearTimeout(retryTimerRef.current);
      retryTimerRef.current = null;
    }
    adRef.current?.destroy();
    adRef.current = null;
    setNativeAd(null);
    setIsLoading(true);
    void loadAd();
  }, [loadAd]);

  useEffect(() => {
    mountedRef.current = true;
    setIsLoading(true);
    void loadAd();
    return () => {
      mountedRef.current = false;
      if (retryTimerRef.current) {
        clearTimeout(retryTimerRef.current);
        retryTimerRef.current = null;
      }
      adRef.current?.destroy();
      adRef.current = null;
    };
  }, [loadAd]);

  useForeground(() => {
    refreshAd();
  });

  return { nativeAd, refreshAd, isLoading };
}

export type NativeAdComponentProps = {
  style?: StyleProp<ViewStyle>;
};

export function NativeAdComponent({ style }: NativeAdComponentProps) {
  const colorScheme = useColorScheme();
  const palette = Colors[colorScheme];
  const { nativeAd } = useNativeAdSlot();

  if (!nativeAd) {
    return null;
  }

  return (
    <View style={[styles.adSlot, style]}>
      <Text style={[styles.adLabel, { color: palette.adMuted }]}>Advertisement</Text>

      <NativeAdView
        nativeAd={nativeAd}
        style={[
          styles.adContainer,
          {
            backgroundColor: palette.adSurface,
            borderColor: palette.adBorder,
          },
        ]}
      >
        <View style={styles.adContent}>
          {nativeAd.advertiser ? (
            <NativeAsset assetType={NativeAssetType.ADVERTISER}>
              <Text style={[styles.advertiser, { color: palette.adMuted }]}>{nativeAd.advertiser}</Text>
            </NativeAsset>
          ) : null}

          <NativeAsset assetType={NativeAssetType.HEADLINE}>
            <Text style={[styles.headline, { color: palette.text }]}>{nativeAd.headline}</Text>
          </NativeAsset>

          <NativeMediaView style={styles.media} />

          <NativeAsset assetType={NativeAssetType.BODY}>
            <Text style={[styles.body, { color: palette.text }]}>{nativeAd.body}</Text>
          </NativeAsset>

          <NativeAsset assetType={NativeAssetType.CALL_TO_ACTION}>
              <Text style={[styles.cta, { backgroundColor: adCtaBackground, color: adCtaText, fontFamily: AppFonts.bodySemi }]}>{nativeAd.callToAction}</Text>
            </NativeAsset>
        </View>
      </NativeAdView>
    </View>
  );
}

const styles = StyleSheet.create({
  adSlot: {
    alignSelf: "stretch",
    width: "100%",
    marginVertical: 24,
    paddingVertical: 4,
  },
  adLabel: {
    fontSize: 11,
    fontWeight: "700",
    letterSpacing: 0.4,
    textTransform: "uppercase",
    marginBottom: 6,
    paddingHorizontal: 4,
  },
  adContainer: {
    alignSelf: "stretch",
    width: "100%",
    borderWidth: 1,
    borderStyle: "dashed",
  },
  adContent: {
    padding: 12,
  },
  advertiser: {
    fontSize: 12,
    fontWeight: "500",
    marginBottom: 2,
  },
  headline: {
    fontSize: 15,
    fontWeight: "700",
  },
  body: {
    fontSize: 14,
    marginBottom: 8,
  },
  // OLD - DONOT USE THIS
  // media: {
  //   height: 150,
  //   marginVertical: 8,
  // },
  // USE THIS INSTEAD
  media: {
    width: "100%",
    height: 180,
    marginVertical: 6,
  },
  cta: {
    paddingVertical: 12,
    paddingHorizontal: 10
    justifyContent: "center",
    alignItems: "center",
    textAlign: "center",
    fontWeight: "700",
    fontSize: 15,
  },
});
```

---

## Native ad web stub: `components/native-ad.web.tsx`

```tsx
import type { StyleProp, ViewStyle } from "react-native";

export type NativeAdComponentProps = {
  style?: StyleProp<ViewStyle>;
};

export function NativeAdComponent(_props: NativeAdComponentProps): null {
  return null;
}
```

---

## Metro / resolution

- `*.web.tsx` overrides `*.tsx` for the same import path on web (`native-ad`).
- No extra `metro.config.js` is required beyond standard Expo for this pattern.
