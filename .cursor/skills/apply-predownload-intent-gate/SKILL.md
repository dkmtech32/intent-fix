---
name: apply-predownload-intent-gate
description: >-
  Surgically ports the multi-step in-app-browser intent gate from
  predownload-flow/meccha-download.html into a product download page, lets the
  user choose intentGateSteps (count, titles, options, language), and restyles
  the download page plus overlays to match that product's landing theme. Use
  when updating */download/ to the new predownload flow, applying
  docs/new-predownload-flow.md, adding iabStep/intentGateSteps, or matching
  download UI to a landing page.
---

# Apply Predownload Intent Gate

Port the multi-step TikTok IAB escape gate into an existing product download
page without replacing its download runtime, funnel, or install guide.

## When to use

- User points at `docs/new-predownload-flow.md` and a product `*/download/`
- User asks to add `intentGateSteps` / `iabStep` multi-step choice screens
- User asks download UI to match the product landing theme

## Canonical sources (read first)

1. Behavior contract: [`docs/new-predownload-flow.md`](../../../docs/new-predownload-flow.md)
2. Reference implementation: [`predownload-flow/meccha-download.html`](../../../predownload-flow/meccha-download.html) (gate helpers ~`getIntentGateSteps` through `renderIntentGate`)
3. Target: `{product}/download/index.html`
4. Theme source: `{product}/index.html` (landing CSS tokens)

## Hard rules

- **Surgical port only** — edit the download HTML; do not swap in Meccha’s full page
- Keep existing: page structure/`data-pd` hooks, download URL, funnel project key, countdown, install guide content
- Do **not** detect intent success/failure in JS; navigation via `S.browser_fallback_url` + `?iabStep=` is the state machine
- Every option on a step shares the **same** `intent://` URL; labels only affect tracking
- Theme overlays to the **product landing**, not Meccha neon and not the old light-green popup
- **User chooses the steps** — do not invent or force a fixed step set; confirm with the user before writing `intentGateSteps`

## User chooses steps (required gate)

Before editing payload, ask (one topic at a time if brainstorming; otherwise batch is fine when the user already listed steps):

1. **How many** choice screens? (1–N; more steps = more intent chances before final popup)
2. **What each step is** — title (+ optional subtitle) and option labels (disguised choices: OS, version, promo, age, robot, etc.)
3. **Language** for step copy (match page locale, or keep user’s exact wording)
4. Optional: which option is `recommended` / has a `badge`

Offer presets only as **optional starting points** the user can accept, mix, or replace — never as mandatory defaults:

| Preset id | Idea |
| --- | --- |
| `os` | Choose platform (Android / iOS / PC) |
| `version` | Choose app version |
| `promo` | Claim bonus / continue without |
| `age` | Confirm 18+ |
| `robot` | Not-a-robot check |

User may supply custom titles (any language). If they give titles in language A but ask for locale B, translate unless they say keep exact text.

Confirm the final `intentGateSteps` JSON with the user, then implement.

## Workflow checklist

Copy and track:

```
- [ ] 1. Confirm intentGateSteps with user (count, copy, language)
- [ ] 2. Read landing tokens + current download gate block
- [ ] 3. Add payload keys (intentGate* with user-chosen steps)
- [ ] 4. Replace legacy single-popup runtime with multi-step helpers
- [ ] 5. Restyle page CSS + install-guide CSS to landing theme
- [ ] 6. Restyle injected gate CSS + badge SVG to landing theme
- [ ] 7. Verify payload / symbols / browser smoke
```

### 1. Inventory

From the download file, locate:

- `#preDownloadPayload` JSON
- Legacy `showInAppBrowserPrompt` / `syncInAppBrowserPrompt` (or existing gate)
- Main `<style>` and `#install-guide-injected` styles

From the landing, extract tokens (example School Stories):

`--paper`, `--ink`, `--coral`, `--yellow`, `--mint`, `--sky`, `--purple`, hard shadow `Npx Npx 0 var(--ink)`, Fredoka + Plus Jakarta Sans

### 2. Payload

Insert after `inAppBrowserDetectors` (keep other keys):

```json
"intentGateEnabled": true,
"intentGatePackage": "com.android.chrome",
"intentGateSteps": [ /* USER-CHOSEN steps — confirmed above */ ]
```

Step shape and runtime keys: see [reference.md](reference.md).

Only write steps the user approved. Changing steps later is payload-only (no runtime code change).

### 3. Runtime port

Replace the legacy IAB popup block with Meccha’s helpers (match target file indentation):

`getIntentGateSteps`, `readIntentGateStep`, `buildGateQuery`, `buildGateFallbackUrl`, `buildStepIntentUrl`, `ensureGateStyles`, `lockGateScroll`, `createGateOverlay`, `preventGateEscape`, `releaseInAppBrowserBlock`, `trackIntentStep`, `renderIntentStep`, `renderFinalPopup`, `renderIntentGate`

Wire boot:

```js
if (payload.inAppBrowserGateEnabled && inAppBrowserInfo.inApp) {
  renderIntentGate();
  window.addEventListener('pageshow', renderIntentGate);
  document.addEventListener('visibilitychange', function () {
    if (!document.hidden) renderIntentGate();
  });
}
```

Keep `inAppBrowserBlocked` suppressing `maybeStartAutoDownload()`.

Details: [reference.md](reference.md).

### 4. Theme match (page + overlays)

Restyle so the first viewport could only belong to this product:

| Surface | Match landing |
| --- | --- |
| Body | Landing paper/grid/background, ink text, landing fonts |
| Card | Landing card / paper-card border + hard shadow |
| Primary CTA | Landing download-button (e.g. coral + ink border + hard shadow) |
| Step numbers | Landing step-number style |
| Gate panel | Same paper card language; not dark neon; not legacy green |
| Gate CTA / options | Landing button + choice-row language |

Add Google Fonts `<link>`s if the landing names fonts that are not loaded.

### 5. Verify

```bash
# Payload: intentGateEnabled, package, step count/ids/titles
# Symbols: renderIntentGate present; showInAppBrowserPrompt absent
# Theme: landing accent hexes present in page + gateStyleCss; Meccha greens absent from gate CSS
```

Manual:

1. Real browser → no overlay; countdown works
2. Spoof TikTok UA → step 0 → tap → `?iabStep=1` → … → final English popup
3. `intentGateEnabled: false` → final popup only
4. Visual: download + overlays match landing

## Example (School Stories — user chose these steps)

User selected 2 Malay steps (not Meccha’s OS/version/promo):

1. **Sahkan umur 18 tahun** (`age`)
2. **Pengesahan robot** (`robot`)

Theme: paper `#fffaf0`, ink `#173d5a`, coral `#ff7064`, yellow `#ffdd68`, hard shadows.

## Out of scope

- Editing the landing page
- Extracting shared JS modules
- Reviving `multi-fallback-intent-flow/*.html` as the runtime path
