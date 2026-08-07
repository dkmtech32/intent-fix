# Predownload Intent Gate — Instructions

How to apply the multi-step in-app-browser (TikTok) escape gate to a product download page.

Agent skill: [`SKILL.md`](SKILL.md) · Payload / tracking details: [`reference.md`](reference.md) · Behavior contract: [`docs/new-predownload-flow.md`](../../../docs/new-predownload-flow.md)

## What this does

For a product like `{product}/`:

1. Ensures `{product}/download/index.html` exists (creates it if needed)
2. Adds multi-step `intentGateSteps` choice screens before the final “Open in browser” popup
3. Restyles the download page + overlays to match `{product}/index.html`
4. Keeps PostHog + downloadFunnel tracking

It does **not** replace the download runtime, funnel keys, or install guide content wholesale.

## Ask Cursor

Examples:

```text
Apply the predownload intent gate to school-stories
```

```text
Create download/ for {product} and add the intent gate
```

```text
Update gta-fivem/download to the new predownload flow — match landing theme
```

The agent should follow `.cursor/skills/apply-predownload-intent-gate/`.

## What you will be asked

### Optional (skip anytime)

| Field | Meaning |
| --- | --- |
| `downloadUrl` | APK / getapp / guide URL used after the countdown |
| `appIconUrl` | App icon image URL (`data-pd="appIcon"`) |

Offered when `download/` is missing or those payload fields are empty. Existing values are preserved unless you replace them.

Do not use landing `intent://…/download/` links as `downloadUrl` — those point at the download page itself.

### Required — intent gate steps

Confirm before implementation:

1. How many choice screens (1–N)
2. Each step: `id`, title, optional subtitle, option labels
3. Language for step copy
4. Optional: which option is `recommended` / has a `badge`

Preset ideas (optional starting points only): `os`, `version`, `promo`, `age`, `robot`.

## Checklist (agent)

```
- [ ] 1. Optionally resolve downloadUrl + appIconUrl
- [ ] 2. Confirm intentGateSteps
- [ ] 3. Read landing tokens + download page (or create download/)
- [ ] 4. Write payload (URLs if provided + intentGate*)
- [ ] 5. Port multi-step gate runtime
- [ ] 6. Ensure PostHog + downloadFunnel
- [ ] 7. Restyle page to landing
- [ ] 8. Restyle gate overlays to landing
- [ ] 9. Verify
```

## Payload essentials

In `#preDownloadPayload`:

```json
{
  "downloadUrl": "https://…",
  "appIconUrl": "https://…",
  "intentGateEnabled": true,
  "intentGatePackage": "com.android.chrome",
  "intentGateSteps": [
    {
      "id": "age",
      "title": "…",
      "subtitle": "…",
      "options": [
        { "label": "…", "desc": "…", "badge": "OK", "recommended": true },
        { "label": "…", "desc": "…" }
      ]
    }
  ]
}
```

`downloadUrl` and `appIconUrl` are optional. Gate steps are required for the multi-step flow.

## Tracking (keep both)

| Tracker | Role |
| --- | --- |
| **PostHog** | Page analytics in `<head>` (proxy: `z.vainglory24.site`) |
| **downloadFunnel** | Funnel events: `predownload_button_click`, `predownload_auto_redirect`, `predownload_intent_step` |

Do not remove either when applying the gate.

## How the gate works

```text
In TikTok (or other detected IAB)
  → step 0, 1, … N-1 (choice screens; every option uses the same intent:// URL)
  → step N = final “Open in browser” popup
Success → real browser on the same page (no iabStep) → normal countdown / download
Failure → ?iabStep=+1 → next screen
```

## Verify

1. Normal browser → no overlay; countdown works
2. TikTok UA (or spoof) → step 0 → tap → `?iabStep=1` → … → final popup
3. Download + overlays match the landing theme
4. PostHog and downloadFunnel scripts load; option tap fires `predownload_intent_step`

## Reference products

| Product | Notes |
| --- | --- |
| `meccha-chameleon/download/` | Canonical PostHog + gate runtime shape |
| `school-stories/download/` | Themed gate example (Malay age + robot steps) |
| `predownload-flow/meccha-download.html` | Source helpers to port |

## Out of scope

- Editing the landing page
- Shared JS module extraction
- Using `multi-fallback-intent-flow/*.html` as the live runtime
