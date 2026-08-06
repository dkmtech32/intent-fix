# School Stories Predownload Intent Gate

## Goal

Update `school-stories/download/index.html` to use the multi-step in-app-browser intent gate documented in [`docs/new-predownload-flow.md`](../../new-predownload-flow.md), without replacing the School Stories page layout or download runtime.

## Context

- Reference implementation: `predownload-flow/meccha-download.html`
- Current School Stories download page still uses the legacy single locked “Open in browser” popup
- School Stories page language is Malay (`ms` / `my`); existing final IAB popup strings stay English

## Approach

Surgical port into the existing file only:

1. Keep School Stories HTML structure, Malay page copy, download funnel config, countdown, auto-download, and install guide
2. Replace the legacy in-app popup block with the Meccha multi-step gate helpers
3. Configure two Malay disguised choice steps (age + robot), then the final manual popup
4. Style injected overlays to the School Stories dark neon theme (cyan / pink / yellow on dark), not Meccha’s green-white legacy panel and not a blind CSS copy of Meccha tokens

## Payload changes

Add to `#preDownloadPayload`:

| Key | Value |
| --- | --- |
| `intentGateEnabled` | `true` |
| `intentGatePackage` | `"com.android.chrome"` |
| `intentGateSteps` | two Malay steps below |

```json
"intentGateSteps": [
  {
    "id": "age",
    "title": "Sahkan umur 18 tahun",
    "subtitle": "Anda mesti berumur 18 tahun ke atas untuk teruskan",
    "options": [
      {
        "label": "Saya sudah 18 tahun ke atas",
        "desc": "Teruskan muat turun",
        "badge": "OK",
        "recommended": true
      },
      {
        "label": "Bawah 18 tahun",
        "desc": "Tidak boleh teruskan"
      }
    ]
  },
  {
    "id": "robot",
    "title": "Pengesahan robot",
    "subtitle": "Pilih untuk sahkan anda bukan robot",
    "options": [
      {
        "label": "Saya bukan robot",
        "desc": "Sahkan dan teruskan",
        "badge": "OK",
        "recommended": true
      },
      {
        "label": "Cuba lagi",
        "desc": "Semak semula"
      }
    ]
  }
]
```

Leave existing keys unchanged (`inAppBrowserGateEnabled`, detectors, final popup copy, install guide, download URL, funnel project key).

## Runtime behavior

Port these helpers from Meccha into the School Stories runtime IIFE, replacing `showInAppBrowserPrompt` / `syncInAppBrowserPrompt`:

- `getIntentGateSteps`
- `readIntentGateStep` (`?iabStep`, default `0`)
- `buildGateQuery` / `buildGateFallbackUrl` / `buildStepIntentUrl`
- `ensureGateStyles` / `lockGateScroll` / `createGateOverlay` / `preventGateEscape`
- `trackIntentStep` → `predownload_intent_step`
- `renderIntentStep`
- `renderFinalPopup`
- `renderIntentGate`
- `releaseInAppBrowserBlock` (updated to clear gate overlay/styles and resume download)

Wire `pageshow` and `visibilitychange` to `renderIntentGate` when the gate is active.

### Flow

```text
In-app + iabStep=0 → age screen
Tap any option → intent://
  success → same page in real browser → normal countdown/download
  failure → ?iabStep=1 → robot screen
Tap any option → intent://
  success → real browser
  failure → ?iabStep=2 → final locked manual popup
```

Every option on a step shares the same intent URL. Option labels only affect tracking. No JS success/failure timer; navigation via `S.browser_fallback_url` is the state machine.

When `intentGateEnabled` is false or `intentGateSteps` is empty, fall back to the single final popup immediately (legacy-compatible).

## Overlay styling

Injected `.pd-iab-*` CSS should match School Stories tokens already on the page:

- Dark panel (`#090b18` / card blues), not light green
- Accents: cyan `#00f5ff`, pink `#ff2d99`, yellow `#ffd60a`, purple `#7c3aed`
- Recommended option highlight uses cyan/lime edge similar to the download card
- CTA gradient mirrors the existing `.btn` gradient
- Keep Meccha’s overlay structure/classes so behavior stays drop-in

## Out of scope

- No changes to `school-stories/index.html`
- No shared extracted JS module
- No server/template changes
- No rewrite of countdown / install-guide / funnel wiring
- Final popup copy remains the current English strings unless changed later

## Manual test plan

1. Real browser / desktop: no overlay; countdown + auto-download unchanged
2. Spoof TikTok UA: step “Sahkan umur 18 tahun” appears, scroll locked
3. Tap either option in emulator → `?iabStep=1` → “Pengesahan robot”
4. Tap again → `?iabStep=2` → final “Open in browser” popup
5. Set `intentGateEnabled` false (or empty steps): final popup only
6. Confirm overlays visually match School Stories dark neon, not light green
