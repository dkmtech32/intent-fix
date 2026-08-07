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

## Hướng dẫn luồng xử lý (landing → CMS)

Luồng chuẩn khi ads gửi yêu cầu landing + predownload. Ví dụ:

**Ads yêu cầu**

| Mục | Giá trị ví dụ |
| --- | --- |
| Kênh / thị trường | TikTok — Thailand |
| Product | GTA FiveM |
| Intent gate steps | Xác minh robot · Chọn thiết bị / hệ điều hành |
| Landing | https://akpgame.com/gta-fivem/ |

### Các bước

```text
1. Tải landing từ CMS
2. Chạy skill tạo / cập nhật predownload → check lại
3. Bật predownload + Manual HTML trên CMS (site akpgame)
4. Paste HTML predownload vào Manual HTML
5. Deploy → check landing gốc đã gắn /download
```

#### 1. Tải folder landing từ CMS

- Vào server / CMS, tải folder landing của product về máy (chỉ cần đúng product, ví dụ `gta-fivem/`).
- Đặt vào repo: `gta-fivem/index.html` (landing). Chưa có `download/` cũng được — skill sẽ tạo.

#### 2. Chạy skill tạo trang predownload

Trong Cursor, ví dụ:

```text
Apply the predownload intent gate to gta-fivem
```

Cung cấp theo yêu cầu ads (có thể bỏ qua URL nếu chưa có):

- Steps: xác minh robot, chọn thiết bị / hệ điều hành (tiếng Thái nếu thị trường TH)
- Optional: `downloadUrl`, `appIconUrl`

Skill sẽ tạo/cập nhật `gta-fivem/download/index.html`, match theme landing, gắn PostHog + downloadFunnel.

**Check lại 1 lượt trước khi lên CMS:**

| Check | Ghi chú |
| --- | --- |
| `intentGateSteps` | Đúng số bước + copy theo ads |
| `downloadUrl` | Có URL APK/getapp (nếu đã có) |
| `appIconUrl` | Icon đúng product |
| PostHog | Có snippet trong `<head>` |
| downloadFunnel | Có `load` + events (`predownload_*`) |
| Theme | Download + overlay giống landing |
| Locale | `language` / `htmlLang` / copy bước khớp thị trường |

Chi tiết checklist: [Skill README](.cursor/skills/apply-predownload-intent-gate/README.md).

#### 3. Lên CMS — bật predownload + Manual HTML

1. Mở CMS → site **akpgame** (đúng domain landing, ví dụ akpgame.com).
2. Bật setting **predownload**.
3. Kéo xuống cuối trang setting.
4. Bật **Manual HTML**.

#### 4. Copy predownload vào Manual HTML

- Mở `gta-fivem/download/index.html` đã tạo ở bước 2.
- Copy toàn bộ HTML → dán vào box **Manual HTML** trên CMS.
- Lưu.

#### 5. Deploy và kiểm tra landing gốc

1. Deploy site.
2. Mở landing gốc: https://akpgame.com/gta-fivem/
3. Xác nhận CTA / nút tải đã gắn link tới `/download` (hoặc `…/gta-fivem/download/`).
4. Smoke test: mở `/download` trên browser thường (countdown OK); nếu có thể, test UA TikTok (hiện gate steps).

### Sơ đồ tóm tắt

```text
Ads brief (market, product, steps, URL)
        │
        ▼
CMS → tải folder landing (gta-fivem/)
        │
        ▼
Cursor skill → gta-fivem/download/index.html
        │
        ▼
Tự check: steps, downloadUrl, icon, tracking, theme
        │
        ▼
CMS akpgame → Predownload ON → Manual HTML → paste
        │
        ▼
Deploy → check landing → /download
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
