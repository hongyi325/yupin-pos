# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Language

一律使用繁體中文（zh-TW）回覆使用者。程式碼、識別字與必要的技術名詞可保留英文。

## Overview

御品泡茶體驗 (Yupin) — a point-of-sale (POS) web app for a Taiwanese tea shop. UI text is Traditional Chinese (zh-TW). There are two pages, each a single self-contained file (HTML, CSS, and JavaScript together):

- `index.html` — the staff POS (login required). Almost everything lives here.
- `order.html` — the customer-facing QR ordering page (`order.html?t=<table>`, no login). See "QR table ordering" below.

## Build / run / deploy

There is **no build system, no package manager, no tests, and no lint**. All dependencies are loaded from CDNs at runtime (Firebase 8.10.1 compat SDK, XLSX 0.18.5, SweetAlert2, Notyf, Phosphor Icons, qrcode-generator). `order.html` deliberately loads **no** external scripts or styles at all. **Every CDN tag is pinned to an exact version with an SRI `integrity` hash** — never use floating versions (`@11`, unversioned unpkg); when upgrading a dependency, recompute the hash (`curl -sf <url> | openssl dgst -sha384 -binary | openssl base64 -A`) or the browser will refuse to load it.

- **Develop**: open `index.html` directly in a browser, or serve it (`python3 -m http.server`) to test against the live Firebase database.
- **Deploy**: GitHub Pages serves `main` at the custom domain in `CNAME` (`yupin-pos.com`). Pushing to `main` deploys — there is no staging environment, so `main` is production.

## Architecture

**Client-only, offline-first.** No backend of our own; the app talks directly to a **Firebase Realtime Database** (config is inline near the top of the `<script>` block). All state, business logic, and rendering happen in the browser.

**Firebase data paths** (subscribed via `db.ref(path).on('value', ...)` so the UI reacts live to remote changes):
- `inventory` — array of category objects `{ cat, items: [...] }`. Item fields are terse: `id`, `n` (name), `c` (cost), `p` (price), `s` (stock), `type` (`'item'` or `'combo'`). Combo items carry `comboDeductionIds` — the underlying item ids to deduct stock for.
- `orders` — pushed with generated keys; each order stores `d` (date), `t` (time), `total`, `profit`, `pay`, `table`, `items` (cart snapshot), `discount`, `note`. **Only the current month is subscribed** (`subscribeCurrentMonthOrders()`, range query on `d` with a `"YYYY/M/"` prefix — `d` is unpadded zh-TW format, so use `dateNum()`/`orderSortKey()` for comparisons, never string compare). History search / delete / Excel export / JSON backup fetch on demand (`fetchOrdersByDay/Month/InRange`). **The Rules must declare `".indexOn": ["d"]` on `orders`** or the SDK silently downloads the whole node and filters client-side.
- `active_tables` — keyed by table number; tracks an in-progress table session (`startTime`, accumulated `items`, `ack90`/`ack120` timeout acknowledgements).
- `table_count` — number of tables, synced across devices.
- `public_menu`, `public_tables`, `qr_orders` — the QR ordering paths, see "QR table ordering" below.

**Stock deduction uses `firebase.database.ServerValue.increment(-n)`** in a single batched `db.ref().update(updates)` to avoid race conditions between multiple POS devices. When changing order/checkout logic, preserve this atomic-batch pattern.

**Offline mode.** If Firebase fails to init, `isOnline` is false and the app falls back to `localStorage`. Offline orders queue under `little_nook_offline_orders` and are flushed later via `syncOfflineData()` (the "雲端同步" button pulses when there's a pending queue). Other localStorage keys: `yupin_pos_table_count`, `yupin_pos_active_tables`, `yupin_pos_pwd_hash`. Note the `little_nook_` prefix on the offline queue is a legacy name — keep it consistent or existing devices lose their queue.

**Two main views** toggled by `toggleView('menu'|'tables')`: the menu/ordering grid (`#item-area`) and the table dashboard (`#table-area` / `renderTableDashboard`). Admin editing (products, combos, categories) happens in modals; category/item reordering supports both HTML5 drag-and-drop and touch handlers.

**Auth uses Firebase Email/Password** for the online path. `checkSystemLogin()` calls `signInWithEmailAndPassword(STAFF_EMAIL, password)` (fixed staff email `pos@yupin-pos.com` — an identifier, not a secret; the password is the secret, verified server-side). `onAuthStateChanged` gates the login overlay and only calls `loadData()` once authenticated, so all `db.ref().on(...)` subscriptions run with a valid auth token. Changing the password (5-tap the header) calls `currentUser.updatePassword()`. When Firebase is unavailable (true offline / CDN unreachable), it falls back to the legacy front-end-only check: a SHA-256+salt hash in `localStorage` (`yupin_pos_pwd_hash`, default `1234`) so the already-loaded app can still open.

**Idle auto-logout**: 24 hours without any interaction signs the device out (`IDLE_LOGOUT_MS`, last-activity timestamp in localStorage `yupin_pos_last_active`, per device). Any pointer/key/touch event resets the timer; `onAuthStateChanged` also checks expiry when Firebase restores a persisted session.

**Head fee (人頭費)**: `calcHeadFee()` — $150 per person with a $60 credit against consumption; children of elementary-school age and under are free (`#head-count` is the total party size including children, `#kid-count` is how many of them are children); a combo flagged `exemptFee` offsets one person; add-on orders to an already-open table are never charged again. Orders store `people`, `kids`, `headFee`, `headCredit`. `order.html` mirrors the same rule in `calcTotals()` to show the customer an estimate — change both together.

**QR table ordering (`order.html`)**. Customers scan a per-table QR code (generated in 管理中心 → 桌邊 QR Code), order on their phone, then **pay at the counter**; there is no online payment and no backend of our own, so the project stays on the Firebase Spark plan. The design rule is that *nothing a customer sends is trusted*:
- `public_menu` — a projection of `inventory` that the POS publishes (`publishPublicMenu()`, called from the `inventory` / `table_count` listeners, written only when its JSON changes). It carries names, prices, sold-out flags (`so`, stock ≤ 0), combo groups, the head-fee constants and the table count — **never cost `c` or exact stock `s`**. `public_tables` is the same idea for which tables are open (`publishPublicTables()`), which is how the customer page knows whether to ask for party size (new table) or treat the order as an add-on.
- `qr_orders/<key>` — a pending order written by the customer page: `{ table, people?, kids?, items: [{ id, sel? }], ts }`. It contains item ids and combo option ids only, no names or prices. Staff see these under 掃碼訂單 (nav bar; badge, toast and chime on arrival), tap 載入購物車 to rebuild the cart **from the POS's own `inventory`** (`buildCartItemFromQr()`), adjust if needed, and check out through the normal `submitOrder()`. The pending order is deleted in the same atomic `update()` as the real order (`loadedQrKeys`). Entries older than 12 hours are purged automatically.
- `order.html` uses the Realtime Database **REST API** (`fetch`), not the SDK: no realtime connection is held per customer (Spark allows 100 simultaneous connections) and the page stays tiny. It writes with `PUT` to a client-generated key so a retry after a lost response cannot create a duplicate order.
- New `qr_orders` data is attacker-controlled: always `esc()` it, pass list indices (not keys) into `onclick`, and keep validating table numbers and item ids against POS state before use.

**XSS escaping**: all user-entered text (item names, category names, combo group titles, notes) must go through `esc()` before being interpolated into `innerHTML` (Notyf/Swal HTML too); use `escJsStr()` when embedding into an `onclick="fn('...')"` string, or better, pass indices instead of strings.

**Firebase Database Rules** live in `database.rules.json` (the reference copy — it is **not** deployed automatically; paste it into Firebase Console → Realtime Database → Rules whenever it changes). The code and rules are two halves of the same protection: without the rules, the public `databaseURL` is world-readable/writable regardless of the login UI. What the rules enforce:
- Everything is readable/writable only by the staff account (`auth.token.email === 'pos@yupin-pos.com'`). A bare `auth != null` is **not** enough: the Email/Password provider lets anyone self-register an account with the public apiKey.
- `public_menu` and `public_tables` are world-readable (staff-only write).
- `qr_orders/$id` can be **created** by anyone but never overwritten or deleted by them, must match a strict shape (`.validate`, unknown fields rejected, server timestamp, ≤ 100 items), and can be read back only by exact key (listing is staff-only).
- `orders` keeps `".indexOn": ["d"]`.
- **Never enable Anonymous sign-in** as a shortcut for customers, and keep the staff-email check if you do add providers.

The staff account (Email/Password provider + user) is created in the Firebase Console, not in code.

## Conventions

- Everything is global functions and top-level `let` state (`inv`, `orders`, `cart`, `activeTables`, `currentPay`) — match this style rather than introducing modules or a framework.
- Notifications: `notyf.success/error` for toasts, `Swal.fire` for confirmations/prompts (always with `heightAuto: false` for correct mobile sizing).
- Layout is heavily tuned for iPhone (safe-area insets, dynamic island, `100dvh`) — recent commit history is mostly mobile layout fixes, so verify changes on a narrow mobile viewport.
