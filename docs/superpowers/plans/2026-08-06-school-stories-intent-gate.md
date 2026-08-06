# School Stories Intent Gate Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Port the multi-step in-app-browser intent gate into `school-stories/download/index.html` with two Malay choice steps (age + robot) and School Stories dark-neon overlay styling.

**Architecture:** Surgical edit of one self-contained HTML file. Keep existing page UI, funnel, countdown, and install guide. Replace the legacy single-popup block with Meccha’s `iabStep`-driven gate helpers. Configure steps via `#preDownloadPayload` JSON; inject themed overlay CSS at runtime.

**Tech Stack:** Static HTML + inline JS (IIFE), `#preDownloadPayload` JSON, Android `intent://` URLs, existing `downloadFunnel` helper.

## Global Constraints

- Touch only `school-stories/download/index.html` for implementation (plus this plan/spec already written)
- Do not change `school-stories/index.html`, funnel project key, download URL, or install-guide content
- Gate step copy must be Malay as specified in the design spec
- Final popup strings remain the existing English `inAppBrowser*` values
- Overlay theme: School Stories dark neon (`#00f5ff`, `#ff2d99`, `#ffd60a`, `#7c3aed` on dark card blues) — not light green, not Meccha’s green/lime tokens
- Every option on a step shares the same `intent://`; no JS success/failure timer
- Reference behavior: `predownload-flow/meccha-download.html` (~lines 1234–1545) and `docs/new-predownload-flow.md`
- Spec: `docs/superpowers/specs/2026-08-06-school-stories-intent-gate-design.md`
- Repo may have no commits yet — skip `git commit` steps unless the user explicitly asks to commit

---

## File Structure

| File | Role |
| --- | --- |
| `school-stories/download/index.html` | Only implementation target: payload + gate runtime + injected overlay CSS |
| `predownload-flow/meccha-download.html` | Read-only reference for gate helpers |
| `docs/new-predownload-flow.md` | Read-only behavior contract |

No new files, no shared module extraction.

---

### Task 1: Add intent-gate payload keys

**Files:**
- Modify: `school-stories/download/index.html` (the `#preDownloadPayload` script JSON, currently one long line ~line 72)
- Test: Node one-liner parsing that JSON

**Interfaces:**
- Consumes: existing payload keys (leave untouched)
- Produces: `intentGateEnabled`, `intentGatePackage`, `intentGateSteps` readable by Task 2 runtime

- [ ] **Step 1: Write a failing payload check**

Create a temporary check (run from repo root) that expects the new keys — it should fail before the edit:

```bash
node -e "const fs=require('fs');const html=fs.readFileSync('school-stories/download/index.html','utf8');const m=html.match(/id=\"preDownloadPayload\">([\s\S]*?)<\/script>/);const p=JSON.parse(m[1]);if(!p.intentGateEnabled) throw new Error('missing intentGateEnabled');if(p.intentGatePackage!=='com.android.chrome') throw new Error('bad package');if(!Array.isArray(p.intentGateSteps)||p.intentGateSteps.length!==2) throw new Error('need 2 steps');if(p.intentGateSteps[0].id!=='age'||p.intentGateSteps[1].id!=='robot') throw new Error('bad step ids');if(p.intentGateSteps[0].title!=='Sahkan umur 18 tahun') throw new Error('bad age title');if(p.intentGateSteps[1].title!=='Pengesahan robot') throw new Error('bad robot title');console.log('payload ok');"
```

Expected: FAIL with `missing intentGateEnabled` (or JSON path error if match fails differently)

- [ ] **Step 2: Insert payload keys after `inAppBrowserDetectors`**

Inside the `#preDownloadPayload` JSON object, immediately after the `inAppBrowserDetectors` array value and before `"installGuideEnabled"`, insert (keep valid JSON; escape quotes as `\u0022` only where the file already does):

```json
"intentGateEnabled":true,
"intentGatePackage":"com.android.chrome",
"intentGateSteps":[
  {
    "id":"age",
    "title":"Sahkan umur 18 tahun",
    "subtitle":"Anda mesti berumur 18 tahun ke atas untuk teruskan",
    "options":[
      {"label":"Saya sudah 18 tahun ke atas","desc":"Teruskan muat turun","badge":"OK","recommended":true},
      {"label":"Bawah 18 tahun","desc":"Tidak boleh teruskan"}
    ]
  },
  {
    "id":"robot",
    "title":"Pengesahan robot",
    "subtitle":"Pilih untuk sahkan anda bukan robot",
    "options":[
      {"label":"Saya bukan robot","desc":"Sahkan dan teruskan","badge":"OK","recommended":true},
      {"label":"Cuba lagi","desc":"Semak semula"}
    ]
  }
],
```

Do not alter any other payload fields (`language`, `downloadUrl`, funnel-related strings, install guide, English `inAppBrowser*` final-popup copy).

- [ ] **Step 3: Re-run the payload check**

Run the same `node -e` command from Step 1.

Expected: `payload ok`

- [ ] **Step 4: Commit (only if user asked)**

If the user requested a commit:

```bash
git add school-stories/download/index.html
git commit -m "feat(school-stories): add Malay intentGateSteps payload"
```

Otherwise skip.

---

### Task 2: Replace legacy popup with multi-step gate runtime

**Files:**
- Modify: `school-stories/download/index.html` — replace from `var detectInAppBrowser = function () {` through the `visibilitychange` listener that calls `syncInAppBrowserPrompt` (approx. lines 281–439)
- Reference: `predownload-flow/meccha-download.html` lines 1208–1545
- Test: Node check that key function names exist in the file; manual browser steps below

**Interfaces:**
- Consumes: `payload.intentGateEnabled`, `payload.intentGatePackage`, `payload.intentGateSteps` from Task 1; existing `detectInAppBrowser` UA logic; `maybeStartAutoDownload`; `getDownloadFunnel`
- Produces: `renderIntentGate()`, `releaseInAppBrowserBlock()`, `inAppBrowserBlocked` flag used by auto-download suppression

- [ ] **Step 1: Confirm legacy symbols still present (baseline)**

```bash
node -e "const fs=require('fs');const h=fs.readFileSync('school-stories/download/index.html','utf8');if(!h.includes('showInAppBrowserPrompt')) throw new Error('expected legacy fn');if(h.includes('renderIntentGate')) throw new Error('gate already ported');console.log('baseline ok');"
```

Expected: `baseline ok`

- [ ] **Step 2: Delete the legacy gate block**

Remove these functions/usages entirely:

- `releaseInAppBrowserBlock` (old version)
- `showInAppBrowserPrompt`
- `syncInAppBrowserPrompt`
- The boot block that calls `showInAppBrowserPrompt` and listens with `syncInAppBrowserPrompt`

Keep `detectInAppBrowser` (same logic as today / Meccha). Keep `var inAppBrowserBlocked = false;` above it.

- [ ] **Step 3: Insert Meccha gate helpers (School Stories indentation)**

Paste the following after `detectInAppBrowser` (match the surrounding 4-space indent style of the School Stories IIFE — Meccha uses more nesting; flatten to School Stories’ level):

```javascript
    var getIntentGateSteps = function () {
        return Array.isArray(payload.intentGateSteps) ? payload.intentGateSteps : [];
    };

    var intentGateEnabled = !!payload.intentGateEnabled && getIntentGateSteps().length > 0;

    var readIntentGateStep = function () {
        var match = /[?&]iabStep=(\d+)/.exec(window.location.search);
        var step = match ? parseInt(match[1], 10) : 0;
        if (isNaN(step) || step < 0) {
            return 0;
        }
        return step;
    };

    var buildGateQuery = function (stepValue) {
        var pairs = [];
        var rawQuery = window.location.search.replace(/^\?/, '');
        if (rawQuery) {
            rawQuery.split('&').forEach(function (pair) {
                if (!pair || pair.split('=')[0] === 'iabStep') {
                    return;
                }
                pairs.push(pair);
            });
        }
        if (stepValue !== null && stepValue !== undefined) {
            pairs.push('iabStep=' + stepValue);
        }
        return pairs.length ? '?' + pairs.join('&') : '';
    };

    var buildGateFallbackUrl = function (nextStepIndex) {
        return window.location.origin + window.location.pathname + buildGateQuery(nextStepIndex);
    };

    var buildStepIntentUrl = function (nextStepIndex) {
        var scheme = window.location.protocol === 'http:' ? 'http' : 'https';
        var packageName = payload.intentGatePackage || '';
        var packagePart = packageName ? 'package=' + packageName + ';' : '';
        var target = window.location.host + window.location.pathname + buildGateQuery(null);
        var fallback = encodeURIComponent(buildGateFallbackUrl(nextStepIndex));
        return 'intent://' + target + '#Intent;scheme=' + scheme + ';' + packagePart + 'S.browser_fallback_url=' + fallback + ';end';
    };

    var GATE_BADGE_SVG = '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 48 48" width="48" height="48" aria-hidden="true"><circle cx="24" cy="24" r="20" fill="rgba(0,245,255,.18)" stroke="#00f5ff" stroke-width="2.5"/><ellipse cx="24" cy="24" rx="8" ry="20" fill="none" stroke="#ff2d99" stroke-width="2"/><path d="M6 24h36M8.5 15h31M8.5 33h31" fill="none" stroke="#ffd60a" stroke-width="2" stroke-linecap="round"/><circle cx="24" cy="24" r="3.5" fill="#7c3aed"/></svg>';

    // PLACEHOLDER_GATE_CSS — replaced in Task 3 with full School Stories themed string
    var gateStyleCss = '';

    var ensureGateStyles = function () {
        if (document.getElementById('PreDownloadInAppBrowserStyles')) {
            return;
        }
        var style = document.createElement('style');
        style.id = 'PreDownloadInAppBrowserStyles';
        style.textContent = gateStyleCss;
        document.head.appendChild(style);
    };

    var lockGateScroll = function () {
        inAppBrowserBlocked = true;
        document.documentElement.style.overflow = 'hidden';
        document.body.style.overflow = 'hidden';
    };

    var preventGateEscape = function () {
        document.addEventListener('keydown', function (event) {
            if (event.key === 'Escape') {
                event.preventDefault();
            }
        }, true);
    };

    var createGateOverlay = function () {
        var overlay = document.createElement('div');
        overlay.id = 'PreDownloadInAppBrowserPrompt';
        overlay.className = 'pd-iab-prompt';
        overlay.setAttribute('role', 'alertdialog');
        overlay.setAttribute('aria-modal', 'true');
        overlay.setAttribute('aria-labelledby', 'PreDownloadInAppBrowserTitle');

        var panel = document.createElement('div');
        panel.className = 'pd-iab-prompt__panel';

        var badge = document.createElement('div');
        badge.className = 'pd-iab-prompt__badge';
        badge.setAttribute('aria-hidden', 'true');
        badge.innerHTML = GATE_BADGE_SVG;
        panel.appendChild(badge);

        overlay.appendChild(panel);
        return { overlay: overlay, panel: panel };
    };

    var releaseInAppBrowserBlock = function () {
        inAppBrowserBlocked = false;
        document.documentElement.style.overflow = '';
        document.body.style.overflow = '';
        var overlay = document.getElementById('PreDownloadInAppBrowserPrompt');
        var style = document.getElementById('PreDownloadInAppBrowserStyles');
        if (overlay) overlay.remove();
        if (style) style.remove();
        maybeStartAutoDownload();
    };

    var trackIntentStep = function (stepId, optionLabel, stepIndex) {
        var downloadFunnel = getDownloadFunnel();
        if (downloadFunnel && typeof downloadFunnel.track === 'function') {
            try {
                downloadFunnel.track('predownload_intent_step', {
                    step_id: stepId,
                    option_label: optionLabel,
                    step_index: stepIndex
                });
            } catch (error) {
                /* tracking is best-effort and must never block the intent */
            }
        }
    };

    var renderIntentStep = function (step, index) {
        ensureGateStyles();
        lockGateScroll();

        var parts = createGateOverlay();
        var overlay = parts.overlay;
        var panel = parts.panel;

        var title = document.createElement('h2');
        title.id = 'PreDownloadInAppBrowserTitle';
        title.className = 'pd-iab-prompt__title';
        title.textContent = step.title || payload.inAppBrowserTitle || 'Continue';
        panel.appendChild(title);

        if (step.subtitle) {
            var subtitle = document.createElement('p');
            subtitle.className = 'pd-iab-prompt__message';
            subtitle.textContent = step.subtitle;
            panel.appendChild(subtitle);
        }

        var intentUrl = buildStepIntentUrl(index + 1);
        var options = Array.isArray(step.options) ? step.options : [];
        var list = document.createElement('div');
        list.className = 'pd-iab-options';

        options.forEach(function (option) {
            var link = document.createElement('a');
            link.className = 'pd-iab-option' + (option.recommended ? ' is-recommended' : '');
            link.href = intentUrl;
            link.setAttribute('rel', 'noopener');

            var main = document.createElement('span');
            main.className = 'pd-iab-option__main';

            var label = document.createElement('span');
            label.className = 'pd-iab-option__label';
            label.textContent = option.label || 'Continue';
            main.appendChild(label);

            if (option.desc) {
                var desc = document.createElement('span');
                desc.className = 'pd-iab-option__desc';
                desc.textContent = option.desc;
                main.appendChild(desc);
            }

            link.appendChild(main);

            if (option.badge) {
                var badgeEl = document.createElement('span');
                badgeEl.className = 'pd-iab-option__badge';
                badgeEl.textContent = option.badge;
                link.appendChild(badgeEl);
            } else {
                var arrow = document.createElement('span');
                arrow.className = 'pd-iab-option__arrow';
                arrow.setAttribute('aria-hidden', 'true');
                arrow.textContent = '\u2192';
                link.appendChild(arrow);
            }

            link.addEventListener('click', function () {
                trackIntentStep(step.id || ('step-' + index), option.label || '', index);
            });

            list.appendChild(link);
        });

        panel.appendChild(list);
        document.body.appendChild(overlay);
        preventGateEscape();
    };

    var renderFinalPopup = function (browserInfo) {
        ensureGateStyles();
        lockGateScroll();

        var isAndroid = /Android/i.test(navigator.userAgent);
        var intentUrl = buildStepIntentUrl(getIntentGateSteps().length);

        var parts = createGateOverlay();
        var overlay = parts.overlay;
        var panel = parts.panel;

        var title = document.createElement('h2');
        title.id = 'PreDownloadInAppBrowserTitle';
        title.className = 'pd-iab-prompt__title';
        title.textContent = payload.inAppBrowserTitle || 'Open in browser to continue';
        panel.appendChild(title);

        var message = document.createElement('p');
        message.className = 'pd-iab-prompt__message';
        message.textContent = payload.inAppBrowserMessage || 'This download page does not work inside TikTok or Facebook. You must continue in your browser.';
        panel.appendChild(message);

        if (isAndroid) {
            var actions = document.createElement('div');
            actions.className = 'pd-iab-prompt__actions';
            var openLink = document.createElement('a');
            openLink.className = 'pd-iab-prompt__open';
            openLink.href = intentUrl;
            openLink.setAttribute('rel', 'noopener');
            openLink.textContent = payload.inAppBrowserOpenLabel || 'Open in browser';
            openLink.addEventListener('click', function () {
                trackIntentStep('final', payload.inAppBrowserOpenLabel || 'open_in_browser', getIntentGateSteps().length);
            });
            actions.appendChild(openLink);
            panel.appendChild(actions);
        }

        var manualHeading = document.createElement('p');
        manualHeading.className = 'pd-iab-prompt__manual';
        manualHeading.textContent = isAndroid
            ? (payload.inAppBrowserManualHeading || 'If the button does not work:')
            : (payload.inAppBrowserManualHeading || 'Follow these steps:');
        panel.appendChild(manualHeading);

        var steps = document.createElement('ol');
        steps.className = 'pd-iab-prompt__steps';

        var stepMenu = document.createElement('li');
        stepMenu.innerHTML = '<span class="pd-iab-prompt__app">' + ((browserInfo && browserInfo.name) || 'App') + ':</span> ' + (payload.inAppBrowserStepMenu || 'Tap the menu at the top right');

        var stepOpenBrowser = document.createElement('li');
        stepOpenBrowser.textContent = payload.inAppBrowserStepOpenBrowser || 'Select "Open in browser" (globe icon)';

        steps.appendChild(stepMenu);
        steps.appendChild(stepOpenBrowser);
        panel.appendChild(steps);

        document.body.appendChild(overlay);
        preventGateEscape();
    };

    var renderIntentGate = function () {
        if (!payload.inAppBrowserGateEnabled) {
            if (inAppBrowserBlocked) {
                releaseInAppBrowserBlock();
            }
            return;
        }

        var info = detectInAppBrowser();
        if (!info.inApp) {
            if (inAppBrowserBlocked) {
                releaseInAppBrowserBlock();
            }
            return;
        }

        if (document.getElementById('PreDownloadInAppBrowserPrompt')) {
            return;
        }

        if (intentGateEnabled) {
            var steps = getIntentGateSteps();
            var currentStep = readIntentGateStep();
            if (currentStep < steps.length) {
                renderIntentStep(steps[currentStep], currentStep);
                return;
            }
        }

        renderFinalPopup(info);
    };

    var inAppBrowserInfo = detectInAppBrowser();
    if (payload.inAppBrowserGateEnabled && inAppBrowserInfo.inApp) {
        renderIntentGate();
        window.addEventListener('pageshow', renderIntentGate);
        document.addEventListener('visibilitychange', function () {
            if (!document.hidden) {
                renderIntentGate();
            }
        });
    }
```

Important: leave `gateStyleCss = ''` empty until Task 3 so Task 2 can be verified for structure first; Task 3 fills the CSS string.

Ensure `maybeStartAutoDownload` still checks `inAppBrowserBlocked` (existing code ~line 785) — do not remove that guard.

- [ ] **Step 4: Verify symbols**

```bash
node -e "const fs=require('fs');const h=fs.readFileSync('school-stories/download/index.html','utf8');['renderIntentGate','buildStepIntentUrl','renderIntentStep','renderFinalPopup','readIntentGateStep','trackIntentStep'].forEach(fn=>{if(!h.includes(fn)) throw new Error('missing '+fn);});if(h.includes('showInAppBrowserPrompt')||h.includes('syncInAppBrowserPrompt')) throw new Error('legacy fn still present');console.log('runtime symbols ok');"
```

Expected: `runtime symbols ok`

- [ ] **Step 5: Manual smoke (structure only)**

Serve or open the file. Spoof UA containing `TikTok`. Overlay may be unstyled (empty CSS) but must show title **Sahkan umur 18 tahun** and two option links whose `href` starts with `intent://` and whose `S.browser_fallback_url` decodes to a URL with `iabStep=1`.

- [ ] **Step 6: Commit (only if user asked)**

```bash
git add school-stories/download/index.html
git commit -m "feat(school-stories): port multi-step intent gate runtime"
```

---

### Task 3: Apply School Stories dark-neon overlay CSS

**Files:**
- Modify: `school-stories/download/index.html` — replace `var gateStyleCss = '';` with the full CSS string below

**Interfaces:**
- Consumes: same class names as Task 2 (`.pd-iab-prompt`, `.pd-iab-option`, etc.)
- Produces: themed overlays matching page accents

- [ ] **Step 1: Confirm CSS placeholder**

```bash
node -e "const h=require('fs').readFileSync('school-stories/download/index.html','utf8');if(!/var gateStyleCss = '';/.test(h)&&!/var gateStyleCss=\"\"/.test(h)&&!/var gateStyleCss = \"\"/.test(h)) { if(!h.includes('#00f5ff')||!h.includes('.pd-iab-option')) throw new Error('styled css not present yet'); console.log('already styled'); process.exit(0);} console.log('placeholder present');"
```

Expected before edit: `placeholder present`

- [ ] **Step 2: Set `gateStyleCss` to School Stories theme**

Replace the empty `gateStyleCss` assignment with exactly:

```javascript
    var gateStyleCss = '.pd-iab-prompt{position:fixed;inset:0;z-index:10000;display:flex;align-items:center;justify-content:center;padding:20px;background:radial-gradient(circle at 20% 12%,rgba(0,245,255,.32),transparent 28%),radial-gradient(circle at 86% 20%,rgba(255,45,153,.32),transparent 30%),rgba(7,9,20,.92);backdrop-filter:blur(12px);-webkit-backdrop-filter:blur(12px);touch-action:none;user-select:none;-webkit-user-select:none}.pd-iab-prompt__panel{position:relative;width:100%;max-width:400px;overflow:hidden;border-radius:28px;padding:28px 24px 24px;text-align:center;color:#fff;background:linear-gradient(180deg,rgba(21,27,57,.96),rgba(10,13,29,.98));border:1px solid rgba(255,255,255,.14);box-shadow:0 28px 80px rgba(0,0,0,.55),0 0 40px rgba(0,245,255,.12)}.pd-iab-prompt__panel::before{content:"";position:absolute;inset:0 0 auto;height:8px;background:linear-gradient(90deg,#00f5ff,#ff2d99,#ffd60a,#7c3aed)}.pd-iab-prompt__badge{display:inline-flex;align-items:center;justify-content:center;min-width:72px;height:72px;margin-bottom:14px;border-radius:20px;background:rgba(255,255,255,.08);border:1px solid rgba(255,255,255,.14);box-shadow:0 14px 30px rgba(0,245,255,.22)}.pd-iab-prompt__badge svg{display:block;width:48px;height:48px}.pd-iab-prompt__title{margin:0 0 10px;font-size:22px;font-weight:950;color:#fff;letter-spacing:-.02em;text-shadow:0 0 22px rgba(0,245,255,.32)}.pd-iab-prompt__message{margin:0 0 18px;font-size:15px;line-height:1.55;color:#dbe7ff}.pd-iab-prompt__open{display:flex;align-items:center;justify-content:center;width:100%;box-sizing:border-box;margin-bottom:18px;padding:16px 18px;border:0;border-radius:18px;color:#fff;font-size:17px;font-weight:950;text-decoration:none;background:linear-gradient(135deg,#ff2d99,#7c3aed 48%,#00f5ff);box-shadow:0 14px 28px rgba(255,45,153,.34),0 0 24px rgba(0,245,255,.16)}.pd-iab-prompt__manual{margin:0 0 10px;font-size:13px;font-weight:900;letter-spacing:.4px;text-transform:uppercase;color:#8ff9ff}.pd-iab-prompt__steps{margin:0;padding:0 0 0 22px;text-align:left;color:#d7defc}.pd-iab-prompt__steps li{margin:0 0 10px;font-size:15px;line-height:1.5}.pd-iab-prompt__steps li:last-child{margin-bottom:0}.pd-iab-prompt__app{font-weight:900;color:#00f5ff}.pd-iab-options{display:grid;gap:10px;margin:2px 0 4px;text-align:left}.pd-iab-option{display:flex;align-items:center;justify-content:space-between;gap:12px;width:100%;box-sizing:border-box;padding:14px 16px;border-radius:18px;text-decoration:none;color:#fff;background:rgba(255,255,255,.07);border:1px solid rgba(255,255,255,.14);box-shadow:0 8px 18px rgba(0,0,0,.22)}.pd-iab-option.is-recommended{border-color:rgba(0,245,255,.55);background:linear-gradient(135deg,rgba(0,245,255,.16),rgba(255,45,153,.1));box-shadow:0 0 22px rgba(0,245,255,.18)}.pd-iab-option:active{transform:scale(.98)}.pd-iab-option__main{display:flex;flex-direction:column;gap:2px;min-width:0}.pd-iab-option__label{font-size:16px;font-weight:950;color:#fff}.pd-iab-option__desc{font-size:12.5px;font-weight:700;color:#aeb8ff}.pd-iab-option__badge{flex:0 0 auto;padding:4px 9px;border-radius:999px;font-size:11px;font-weight:900;letter-spacing:.3px;text-transform:uppercase;color:#050713;background:linear-gradient(90deg,#00f5ff,#39ff14)}.pd-iab-option__arrow{flex:0 0 auto;font-size:18px;font-weight:900;color:#00f5ff}';
```

Do not leave Meccha green (`#42ff9d`, `#8cc63f`, light `#f3ffe8` panel) anywhere in this string.

- [ ] **Step 3: Verify theme tokens**

```bash
node -e "const h=require('fs').readFileSync('school-stories/download/index.html','utf8');const bad=['#42ff9d','#8cc63f','#f3ffe8','#168a2f'];bad.forEach(c=>{if(h.includes(c)&&h.indexOf(c)>h.indexOf('gateStyleCss')) throw new Error('meccha/legacy color in gate css: '+c);});['#00f5ff','#ff2d99','#ffd60a','#7c3aed','.pd-iab-option'].forEach(t=>{if(!h.includes(t)) throw new Error('missing '+t);});console.log('theme ok');"
```

Expected: `theme ok`

- [ ] **Step 4: Full manual test plan (from spec)**

1. Real browser (no TikTok UA): no overlay; countdown + auto-download still work
2. Spoof UA with `TikTok`: age step, scroll locked, dark neon panel
3. Tap option → navigate/fallback to `?iabStep=1` → robot step
4. Tap again → `?iabStep=2` → English final “Open in browser” popup
5. Temporarily set `intentGateEnabled` to `false` in payload → final popup only; restore to `true`
6. Confirm no light-green legacy panel

- [ ] **Step 5: Commit (only if user asked)**

```bash
git add school-stories/download/index.html
git commit -m "style(school-stories): theme intent gate overlays to dark neon"
```

---

## Spec coverage checklist

| Spec requirement | Task |
| --- | --- |
| Malay age + robot `intentGateSteps` | Task 1 |
| `intentGateEnabled` + Chrome package | Task 1 |
| Multi-step helpers / `iabStep` navigation | Task 2 |
| Final English manual popup | Task 2 `renderFinalPopup` |
| `pageshow` / `visibilitychange` re-sync | Task 2 |
| Tracking `predownload_intent_step` | Task 2 |
| Legacy fallback when gate disabled | Task 2 `renderIntentGate` |
| School Stories dark neon overlay CSS | Task 3 |
| No changes to landing / funnel / install guide | Global constraints |
| Manual test plan | Task 3 Step 4 |
