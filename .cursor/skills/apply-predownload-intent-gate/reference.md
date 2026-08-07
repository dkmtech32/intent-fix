# Predownload Intent Gate — Reference

## Payload keys

| Key | Type | Purpose |
| --- | --- | --- |
| `inAppBrowserGateEnabled` | boolean | Master switch (already present on most pages) |
| `inAppBrowserDetectors` | array | UA regex detectors (e.g. tiktok) |
| `intentGateEnabled` | boolean | Multi-step choice flow; if false/empty steps → final popup only |
| `intentGatePackage` | string | Usually `com.android.chrome`; `""` omits `package=` |
| `intentGateSteps` | array | Ordered choice screens — **chosen by the user** per product |
| `inAppBrowserTitle` / `Message` / `ManualHeading` / `StepMenu` / `StepOpenBrowser` / `OpenLabel` | string | **Final** manual popup copy (often left English) |

### `intentGateSteps` item

```json
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
}
```

- `id` → tracking `step_id`
- `badge` optional; if omitted, show `→`
- `recommended` highlights the row

### User-chosen steps

There is no required step set. Get the user’s step list (or an accepted preset mix) before writing JSON.

**Example A — user chose age + robot (School Stories, Malay):**

1. `age` — Sahkan umur 18 tahun  
2. `robot` — Pengesahan robot  

**Example B — Meccha-style (Romanian) presets:**

1. `os` — Alege platforma  
2. `version` — Alege versiunea  
3. `promo` — Revendică bonusul  

Add/remove/reorder freely; only `intentGateSteps` in the payload needs to change.

## Flow

```text
In-app + iabStep < steps.length → locked choice screen N
  tap any option → intent://
    success → same page in real browser → normal countdown/download
    failure → ?iabStep=N+1 → next screen
iabStep >= steps.length → locked final "Open in browser" popup
```

Intent URL shape:

```text
intent://<host><path><cleanQuery>#Intent;scheme=https;package=com.android.chrome;S.browser_fallback_url=<samePage?iabStep=N+1>;end
```

Clean success target drops `iabStep` but preserves tracking query params.

## Functions to port

From `predownload-flow/meccha-download.html` runtime IIFE:

| Function | Role |
| --- | --- |
| `detectInAppBrowser` | Keep existing / same UA loop |
| `getIntentGateSteps` | Read `payload.intentGateSteps` |
| `readIntentGateStep` | Parse `?iabStep` (default 0) |
| `buildGateQuery` | Rebuild query; drop/set `iabStep` |
| `buildGateFallbackUrl` | Origin + path + query for next step |
| `buildStepIntentUrl` | `intent://` + package + fallback |
| `ensureGateStyles` / `gateStyleCss` | Inject themed overlay CSS once |
| `lockGateScroll` / `preventGateEscape` | Lock page while gated |
| `createGateOverlay` | Overlay + panel + badge shell |
| `releaseInAppBrowserBlock` | Unlock, remove overlay/styles, `maybeStartAutoDownload` |
| `trackIntentStep` | `downloadFunnel.track('predownload_intent_step', { step_id, option_label, step_index })` — never `preventDefault` |
| `renderIntentStep` | Choice screen for step N |
| `renderFinalPopup` | Manual open-in-browser final step |
| `renderIntentGate` | Orchestrator |

Remove: `showInAppBrowserPrompt`, `syncInAppBrowserPrompt`.

## Theme mapping checklist

Map landing → download:

| Landing cue | Download target |
| --- | --- |
| `:root` CSS variables | Copy into download `:root` |
| Body background / grid | `body` |
| Paper card / bordered panel | `.card` |
| Primary download button | `.btn` + `.pd-iab-prompt__open` |
| Step circles | `.steps b` |
| Choice rows | `.pd-iab-option` / `.is-recommended` |
| Accent strip | `.pd-iab-prompt__panel::before` gradient from landing accents |
| Fonts | `<link>` + `font-family` on body / headings / CTA |

### Avoid in gate CSS

- Meccha neon greens: `#42ff9d`, `#c7ff4f`, `#8cc63f`, `#168a2f`
- Legacy light panel: `#f3ffe8`, `#e5f8d5`
- Unrelated dark-cyber download look if landing is paper/coral (or vice versa)

## Tracking inventory

Download pages use **two** trackers. Keep both when applying the intent gate.

### 1. PostHog (page analytics)

| Item | Value |
| --- | --- |
| Placement | `<head>`, before `</head>` |
| Project key | `phc_QvB9SGMiDA8dFUIy69OPF9n4YayJG81RWbf62mrsSvm` |
| `api_host` | `https://z.vainglory24.site` (managed reverse proxy) |
| `ui_host` | `https://us.posthog.com` (required with proxy) |
| `defaults` | `2026-05-30` |
| `person_profiles` | `identified_only` |

- Autocapture / pageviews via PostHog defaults — **no custom `posthog.capture` in the gate runtime**
- Canonical snippet: `meccha-chameleon/download/index.html` (also school-stories, gta-fivem)
- Do not replace with a different PostHog project unless the user asks
- Note: some older pages may still have a second/legacy PostHog init; prefer the proxy key above as the skill standard

### 2. downloadFunnel (funnel events)

| Item | Value |
| --- | --- |
| Placement | Body, stub + `load` immediately before the predownload runtime IIFE |
| Project key | `dfpk_t1k9su6ium5n7ztlqxfyevz6nscdagde` |
| `api_host` | `https://onesignal5.fastmart24.store` |
| `asset_host` | `https://cdn.fastmart24.store` |
| Script | `{asset_host}/js/df-helper.min.js` |

Stub methods: `init`, `load`, `track`, `trackOnce`, `getIds`, `createDownloadClickId`, `getDownloadClickId`, `trackAutoRedirect`, `debug`.

#### Explicit events fired by the download runtime

| Event | When | Props |
| --- | --- | --- |
| `predownload_button_click` | Manual download / again button | `button_id`, `download_click_id`, `dedupe_key` |
| `predownload_auto_redirect` | Countdown auto-start (via `trackAutoRedirect()` or fallback `track`) | `dedupe_key: predownload_auto_redirect:<pathname>` |
| `predownload_intent_step` | Gate option tap or final “Open in browser” | `step_id`, `option_label`, `step_index` |

`predownload_intent_step` details:

- Choice screens: `step_id` = step `id`, `step_index` = 0-based index, `option_label` = tapped label
- Final popup: `step_id` = `'final'`, `option_label` = open label, `step_index` = `intentGateSteps.length`
- Tracking is best-effort; never `preventDefault` on the intent link

#### Helpers (not events)

| Helper | Role |
| --- | --- |
| `getIds()` | Read funnel / click ids for URL params |
| `createDownloadClickId()` / `getDownloadClickId()` | Stable click id for button tracking |
| `trackAutoRedirect()` | Preferred auto-redirect track; falls back to `predownload_auto_redirect` |

Clean success URLs drop `iabStep` but **preserve** tracking query params from `buildGateQuery`.

## Verification snippets

Payload:

```js
// node: parse #preDownloadPayload JSON
// require intentGateEnabled, intentGatePackage, intentGateSteps length/ids/titles
```

Symbols:

```js
// require: renderIntentGate, buildStepIntentUrl, renderIntentStep, renderFinalPopup, trackIntentStep
// forbid: showInAppBrowserPrompt, syncInAppBrowserPrompt
// require head: posthog.init with api_host z.vainglory24.site
// require body: downloadFunnel.load + track('predownload_intent_step'…)
```

Browser:

1. No TikTok UA → no `#PreDownloadInAppBrowserPrompt`
2. TikTok UA → step 0 title visible; option `href` starts with `intent://` and fallback decodes to `iabStep=1`
3. `?iabStep=<steps.length>` → final popup title from `inAppBrowserTitle`
