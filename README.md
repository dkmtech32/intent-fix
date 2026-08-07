# Intent Fix

Static product landing + download pages with a multi-step **in-app browser intent gate** so users can escape TikTok (and similar) webviews into a real browser before the APK countdown / download runs.

No build step — each product is plain HTML (plus optional Cloudflare Pages helpers under `maokiwebsite/`).

## How it works

TikTok’s in-app browser often fails silent `intent://` taps and loads the fallback inside the same webview. This repo’s fix:

1. Show several disguised choice screens (OS, version, promo, age, …)
2. Every option on a screen is the **same** `intent://` URL
3. Success → clean URL in Chrome / external browser → normal download
4. Failure → same page with `?iabStep=N+1` → next screen
5. After all steps → locked “Open in browser” popup

State lives in the URL (`iabStep`), not in JS success/failure detection.

Full contract: [`docs/new-predownload-flow.md`](docs/new-predownload-flow.md)

```text
Landing  →  /download/  →  [IAB gate steps…]  →  countdown  →  downloadUrl
```

## Repo layout

```text
intent-fix/
├── docs/                          # Flow docs + design/plans
├── predownload-flow/              # Reference Meccha landing + download
├── multi-fallback-intent-flow/    # Legacy multi-file chain (do not use as runtime)
├── school-stories/                # Product: landing + download/
├── meccha-chameleon/
├── gta-fivem/
├── ea-sports-fc-mobile/
├── maokiwebsite/                  # Additional products + Pages redirects
└── .cursor/skills/apply-predownload-intent-gate/   # Agent skill to port the gate
```

### Typical product folder

| Path | Role |
| --- | --- |
| `{product}/index.html` | Marketing landing (theme source) |
| `{product}/download/index.html` | Predownload page: gate, countdown, install guide, tracking |

Configuration lives in `#preDownloadPayload` JSON inside the download HTML.

## Products

| Folder | Has `download/` with intent gate |
| --- | --- |
| `school-stories` | Yes |
| `meccha-chameleon` | Yes |
| `gta-fivem` | Yes |
| `ea-sports-fc-mobile` | Yes |
| `maokiwebsite/*` | Mixed (several have download pages; some landings only) |

Reference implementation for the gate runtime and PostHog snippet: `meccha-chameleon/download/` (helpers also in `predownload-flow/meccha-download.html`).

## Tracking

Download pages use **two** trackers:

| Tracker | Purpose |
| --- | --- |
| **PostHog** | Page analytics (`<head>`, reverse proxy `z.vainglory24.site`) |
| **downloadFunnel** | Funnel events: `predownload_button_click`, `predownload_auto_redirect`, `predownload_intent_step` |

Keep both when updating a download page. Event details: [`.cursor/skills/apply-predownload-intent-gate/reference.md`](.cursor/skills/apply-predownload-intent-gate/reference.md).

## Apply the gate to a product

Use the Cursor skill (human instructions + agent checklist):

- [Skill README](.cursor/skills/apply-predownload-intent-gate/README.md)
- [SKILL.md](.cursor/skills/apply-predownload-intent-gate/SKILL.md)

In Cursor, for example:

```text
Apply the predownload intent gate to {product}
```

You may optionally provide:

- `downloadUrl` — APK / getapp / guide URL after countdown
- `appIconUrl` — icon image URL

You must confirm **intent gate steps** (count, titles, options, language). The download UI is restyled to match that product’s landing.

## Payload (download page)

Important keys in `#preDownloadPayload`:

| Key | Notes |
| --- | --- |
| `downloadUrl` | Optional until set; required for auto-download to work |
| `appIconUrl` | Optional; syncs to `[data-pd="appIcon"]` |
| `inAppBrowserGateEnabled` | Master IAB gate switch |
| `intentGateEnabled` | Multi-step choice flow |
| `intentGatePackage` | Usually `com.android.chrome` |
| `intentGateSteps` | Ordered choice screens (user-defined) |

## Local preview

Open any HTML file in a browser, or serve the folder:

```bash
npx --yes serve .
```

Then visit e.g. `/school-stories/` and `/school-stories/download/`.

To smoke-test the gate, spoof a TikTok user agent or use a real in-app browser.

## Docs

| Doc | Contents |
| --- | --- |
| [`docs/new-predownload-flow.md`](docs/new-predownload-flow.md) | Behavior contract for the multi-step intent gate |
| [`docs/superpowers/`](docs/superpowers/) | Design / plan notes (e.g. School Stories) |
| [Skill README](.cursor/skills/apply-predownload-intent-gate/README.md) | How to port the gate onto a product |

## Out of scope / legacy

- **`multi-fallback-intent-flow/`** — older multi-file fallback chain; keep for reference only; new work uses a single download page + `?iabStep=`
- Landing pages are not edited by the apply-gate skill (theme is read from landing, written onto download)
