---
name: apply-predownload-intent-gate
description: >-
  Surgically ports the multi-step in-app-browser intent gate from
  predownload-flow/meccha-download.html into a product download page, lets the
  user choose intentGateSteps (count, titles, options, language), restyles the
  download page plus overlays to match that product's landing theme, and keeps
  PostHog + downloadFunnel tracking. Use when updating */download/ to the new
  predownload flow, applying docs/new-predownload-flow.md, adding
  iabStep/intentGateSteps, matching download UI to a landing page, or ensuring
  PostHog/downloadFunnel on download pages, or creating download/ when missing
  (optionally collecting downloadUrl / appIconUrl).
---

# Apply Predownload Intent Gate

Port the multi-step TikTok IAB escape gate into an existing product download
page without replacing its download runtime, funnel, or install guide.

## When to use

- User points at `docs/new-predownload-flow.md` and a product `*/download/`
- User asks to add `intentGateSteps` / `iabStep` multi-step choice screens
- User asks download UI to match the product landing theme
- Product has a landing but **no** `download/` yet, or payload lacks `downloadUrl` / `appIconUrl`

## Canonical sources (read first)

1. Behavior contract: [`docs/new-predownload-flow.md`](../../../docs/new-predownload-flow.md)
2. Reference implementation: [`predownload-flow/meccha-download.html`](../../../predownload-flow/meccha-download.html) (gate helpers ~`getIntentGateSteps` through `renderIntentGate`)
3. Target: `{product}/download/index.html` (create if missing — see below)
4. Theme source: `{product}/index.html` (landing CSS tokens)

## Hard rules

- **Surgical port only** — edit the download HTML; do not swap in Meccha’s full page
- Keep existing: page structure/`data-pd` hooks, funnel project key, countdown, install guide content
- **Preserve `downloadUrl` / `appIconUrl` when present** — never invent or overwrite; if missing, **optionally ask** the user (see Optional URL options)
- Do **not** detect intent success/failure in JS; navigation via `S.browser_fallback_url` + `?iabStep=` is the state machine
- Every option on a step shares the **same** `intent://` URL; labels only affect tracking
- Theme overlays to the **product landing**, not Meccha neon and not the old light-green popup
- **User chooses the steps** — do not invent or force a fixed step set; confirm with the user before writing `intentGateSteps`
- **Keep both trackers** — PostHog (head) + `downloadFunnel` (body); do not remove or replace either when porting the gate

## Optional URL options (`downloadUrl`, `appIconUrl`)

Both are **optional**. Offer them when creating `download/` or when the payload value is missing/empty — the user may skip.

| Field | Role | If missing / empty |
| --- | --- | --- |
| `downloadUrl` | APK / getapp / guide link after countdown | Optionally ask; if skipped, leave empty/`#` and note download won’t start until set |
| `appIconUrl` | App icon shown on the download page (`data-pd="appIcon"`) | Optionally ask; if skipped, keep existing `<img src>` or infer from landing hero icon when obvious |

| Condition | Action |
| --- | --- |
| `{product}/download/index.html` **does not exist** | Create from a sibling/reference download themed to the landing; **optionally ask** for `downloadUrl` and `appIconUrl` |
| File exists but `downloadUrl` or `appIconUrl` missing / `""` / `"#"` / whitespace | **Optionally ask** for the missing field(s); set only values the user provides |
| Valid value already present | Keep it; change only if the user supplies a replacement |

Example prompts (skip-friendly):

> Optional: what should `downloadUrl` be for `{product}`? (APK / getapp / guide link — skip to leave unset)
> Optional: what should `appIconUrl` be? (CDN/icon image — skip to keep/infer from landing)

Do **not** guess `downloadUrl` from landing `intent://…/download/` links. Do **not** copy another product’s URLs. For `appIconUrl`, preferring the landing’s app icon asset is fine when the user skips and the landing has a clear icon URL.

If creating `download/` from scratch, also confirm (or infer from landing): `appName`, `language` / `htmlLang`, and intent gate steps.

## User chooses steps (required gate)

Before editing payload, ask (one topic at a time if brainstorming; otherwise batch is fine when the user already listed steps):

1. **downloadUrl** (optional) — offer if file missing or URL empty; user may skip
2. **appIconUrl** (optional) — offer if file missing or icon empty; user may skip
3. **How many** choice screens? (1–N; more steps = more intent chances before final popup)
4. **What each step is** — title (+ optional subtitle) and option labels (disguised choices: OS, version, promo, age, robot, etc.)
5. **Language** for step copy (match page locale, or keep user’s exact wording)
6. Optional: which option is `recommended` / has a `badge`

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
- [ ] 1. Optionally resolve downloadUrl + appIconUrl (offer if missing; user may skip)
- [ ] 2. Confirm intentGateSteps with user (count, copy, language)
- [ ] 3. Read landing tokens + current download gate block (or create download/)
- [ ] 4. Add payload keys (optional URLs if provided + intentGate* with user-chosen steps)
- [ ] 5. Replace legacy single-popup runtime with multi-step helpers
- [ ] 6. Ensure PostHog head snippet + downloadFunnel body loader
- [ ] 7. Restyle page CSS + install-guide CSS to landing theme
- [ ] 8. Restyle injected gate CSS + badge SVG to landing theme
- [ ] 9. Verify payload / symbols / tracking / browser smoke
```

### 1. Inventory

Check target path first:

- If `{product}/download/` is missing → create from a known-good sibling download; optionally ask for `downloadUrl` + `appIconUrl`
- If file exists → parse `#preDownloadPayload.downloadUrl` and `appIconUrl`; optionally ask for any that are missing/empty/`#`

From the download file (once it exists), locate:

- `#preDownloadPayload` JSON — especially `downloadUrl`, `appIconUrl`
- Legacy `showInAppBrowserPrompt` / `syncInAppBrowserPrompt` (or existing gate)
- Main `<style>` and `#install-guide-injected` styles
- PostHog `<script>` in `<head>` (`posthog.init`)
- `downloadFunnel` stub + `downloadFunnel.load` before the runtime IIFE

From the landing, extract tokens (example School Stories):

`--paper`, `--ink`, `--coral`, `--yellow`, `--mint`, `--sky`, `--purple`, hard shadow `Npx Npx 0 var(--ink)`, Fredoka + Plus Jakarta Sans

### 2. Payload

If the user provided them, set:

```json
"downloadUrl": "https://…",
"appIconUrl": "https://…",
```

Also sync `appIconUrl` onto `[data-pd="appIcon"]` `src` when writing the icon.

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

### 4. Tracking (PostHog + downloadFunnel)

Ensure both trackers are present. Copy PostHog from a known-good product download
(e.g. `meccha-chameleon/download/index.html`) into `</head>` if missing:

```html
<script>
    !function (t, e) { var o, n, p, r; e.__SV || (window.posthog && window.posthog.__loaded) || (window.posthog = e, e._i = [], e.init = function (i, s, a) { function g(t, e) { var o = e.split("."); 2 == o.length && (t = t[o[0]], e = o[1]), t[e] = function () { t.push([e].concat(Array.prototype.slice.call(arguments, 0))) } } p || ((p = t.createElement("script")).type = "text/javascript", p.crossOrigin = "anonymous", p.async = !0, p.src = s.api_host.replace(".i.posthog.com", "-assets.i.posthog.com") + "/static/array.js", p.onerror = function () { p = null }, (r = t.getElementsByTagName("script")[0]).parentNode.insertBefore(p, r)); var u = e; for (void 0 !== a ? u = e[a] = [] : a = "posthog", u.people = u.people || [], u.toString = function (t) { var e = "posthog"; return "posthog" !== a && (e += "." + a), t || (e += " (stub)"), e }, u.people.toString = function () { return u.toString(1) + ".people (stub)" }, o = "an ln init xn Cn Br kn In capture Fn nn calculateEventProperties On register register_once register_for_session unregister unregister_for_session Ln getFeatureFlag getFeatureFlagPayload getFeatureFlagResult getAllFeatureFlags isFeatureEnabled reloadFeatureFlags updateFlags updateEarlyAccessFeatureEnrollment getEarlyAccessFeatures on onFeatureFlags onSurveysLoaded onSessionId getSurveys getActiveMatchingSurveys renderSurvey displaySurvey cancelPendingSurvey canRenderSurvey canRenderSurveyAsync Dn identify setPersonProperties unsetPersonProperties group resetGroups setPersonPropertiesForFlags resetPersonPropertiesForFlags setGroupPropertiesForFlags resetGroupPropertiesForFlags reset shutdown setIdentity clearIdentity get_distinct_id getGroups get_session_id get_session_replay_url alias set_config startSessionRecording stopSessionRecording sessionRecordingStarted captureException addExceptionStep captureLog startExceptionAutocapture stopExceptionAutocapture loadToolbar get_property getSessionProperty An Rn createPersonProfile setInternalOrTestUser $n yn jn opt_in_capturing opt_out_capturing has_opted_in_capturing has_opted_out_capturing get_explicit_consent_status is_capturing clear_opt_in_out_capturing Tn debug Ur Rt getPageViewId captureTraceFeedback captureTraceMetric pn".split(" "), n = 0; n < o.length; n++)g(u, o[n]); e._i.push([i, s, a]) }, e.__SV = 1) }(document, window.posthog || []);
    posthog.init('phc_QvB9SGMiDA8dFUIy69OPF9n4YayJG81RWbf62mrsSvm', {
        api_host: 'https://z.vainglory24.site', // managed reverse proxy
        ui_host: 'https://us.posthog.com', // required when using a proxy
        defaults: '2026-05-30',
        person_profiles: 'identified_only',
    })
</script>
```

Keep the existing `downloadFunnel` stub + `downloadFunnel.load(...)` before the runtime
IIFE. Do **not** invent custom `posthog.capture` calls in the gate — PostHog is
autocapture / page analytics; funnel events stay on `downloadFunnel.track`.

Full event list: [reference.md](reference.md#tracking-inventory).

### 5. Theme match (page + overlays)

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

### 6. Verify

```bash
# Payload: intentGateEnabled, package, step count/ids/titles
# Optional: downloadUrl / appIconUrl if user provided them
# Symbols: renderIntentGate present; showInAppBrowserPrompt absent
# Theme: landing accent hexes present in page + gateStyleCss; Meccha greens absent from gate CSS
# Tracking: posthog.init (proxy key) in <head>; downloadFunnel.load + trackIntentStep / button / auto_redirect
```

Manual:

1. Real browser → no overlay; countdown works; download uses `downloadUrl` when set
2. Spoof TikTok UA → step 0 → tap → `?iabStep=1` → … → final English popup
3. `intentGateEnabled: false` → final popup only
4. Visual: download + overlays match landing; icon matches `appIconUrl` when set
5. Network: PostHog + downloadFunnel assets load; option tap fires `predownload_intent_step`

## Example (School Stories — user chose these steps)

User selected 2 Malay steps (not Meccha’s OS/version/promo):

1. **Sahkan umur 18 tahun** (`age`)
2. **Pengesahan robot** (`robot`)

Theme: paper `#fffaf0`, ink `#173d5a`, coral `#ff7064`, yellow `#ffdd68`, hard shadows.

## Out of scope

- Editing the landing page
- Extracting shared JS modules
- Reviving `multi-fallback-intent-flow/*.html` as the runtime path
