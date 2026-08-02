# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Language

一律使用繁體中文（zh-TW）回覆使用者。程式碼、識別字與必要的技術名詞可保留英文。

## Overview

御品泡茶體驗 (Yupin) — a single-page point-of-sale (POS) web app for a Taiwanese tea shop. The entire application (HTML, CSS, and JavaScript) lives in one file: `index.html`. UI text is Traditional Chinese (zh-TW).

## Build / run / deploy

There is **no build system, no package manager, no tests, and no lint**. All dependencies are loaded from CDNs at runtime (Firebase 8.10.1 compat SDK, XLSX 0.18.5, SweetAlert2, Notyf, Phosphor Icons). **Every CDN tag is pinned to an exact version with an SRI `integrity` hash** — never use floating versions (`@11`, unversioned unpkg); when upgrading a dependency, recompute the hash (`curl -sf <url> | openssl dgst -sha384 -binary | openssl base64 -A`) or the browser will refuse to load it.

- **Develop**: open `index.html` directly in a browser, or serve it (`python3 -m http.server`) to test against the live Firebase database.
- **Deploy**: GitHub Pages serves `main` at the custom domain in `CNAME` (`yupin-pos.com`). Pushing to `main` deploys — there is no staging environment, so `main` is production.

## Architecture

**Client-only, offline-first.** No backend of our own; the app talks directly to a **Firebase Realtime Database** (config is inline near the top of the `<script>` block). All state, business logic, and rendering happen in the browser.

**Firebase data paths** (subscribed via `db.ref(path).on('value', ...)` so the UI reacts live to remote changes):
- `inventory` — array of category objects `{ cat, items: [...] }`. Item fields are terse: `id`, `n` (name), `c` (cost), `p` (price), `s` (stock), `type` (`'item'` or `'combo'`). Combo items carry `comboDeductionIds` — the underlying item ids to deduct stock for.
- `orders` — pushed with generated keys; each order stores `d` (date), `t` (time), `total`, `profit`, `pay`, `table`, `items` (cart snapshot), `discount`, `note`.
- `active_tables` — keyed by table number; tracks an in-progress table session (`startTime`, accumulated `items`, `ack90`/`ack120` timeout acknowledgements).
- `table_count` — number of tables, synced across devices.

**Stock deduction uses `firebase.database.ServerValue.increment(-n)`** in a single batched `db.ref().update(updates)` to avoid race conditions between multiple POS devices. When changing order/checkout logic, preserve this atomic-batch pattern.

**Offline mode.** If Firebase fails to init, `isOnline` is false and the app falls back to `localStorage`. Offline orders queue under `little_nook_offline_orders` and are flushed later via `syncOfflineData()` (the "雲端同步" button pulses when there's a pending queue). Other localStorage keys: `yupin_pos_table_count`, `yupin_pos_active_tables`, `yupin_pos_pwd_hash`. Note the `little_nook_` prefix on the offline queue is a legacy name — keep it consistent or existing devices lose their queue.

**Two main views** toggled by `toggleView('menu'|'tables')`: the menu/ordering grid (`#item-area`) and the table dashboard (`#table-area` / `renderTableDashboard`). Admin editing (products, combos, categories) happens in modals; category/item reordering supports both HTML5 drag-and-drop and touch handlers.

**Auth uses Firebase Email/Password** for the online path. `checkSystemLogin()` calls `signInWithEmailAndPassword(STAFF_EMAIL, password)` (fixed staff email `pos@yupin-pos.com` — an identifier, not a secret; the password is the secret, verified server-side). `onAuthStateChanged` gates the login overlay and only calls `loadData()` once authenticated, so all `db.ref().on(...)` subscriptions run with a valid auth token. Changing the password (5-tap the header) calls `currentUser.updatePassword()`. When Firebase is unavailable (true offline / CDN unreachable), it falls back to the legacy front-end-only check: a SHA-256+salt hash in `localStorage` (`yupin_pos_pwd_hash`, default `1234`) so the already-loaded app can still open.

**Idle auto-logout**: 24 hours without any interaction signs the device out (`IDLE_LOGOUT_MS`, last-activity timestamp in localStorage `yupin_pos_last_active`, per device). Any pointer/key/touch event resets the timer; `onAuthStateChanged` also checks expiry when Firebase restores a persisted session.

**XSS escaping**: all user-entered text (item names, category names, combo group titles, notes) must go through `esc()` before being interpolated into `innerHTML` (Notyf/Swal HTML too); use `escJsStr()` when embedding into an `onclick="fn('...')"` string, or better, pass indices instead of strings.

**Firebase Database Rules must require auth** (`{".read":"auth != null",".write":"auth != null"}`), set in the Firebase Console — the code and rules are two halves of the same protection. Without the rules, the public `databaseURL` is world-readable/writable regardless of the login UI. The staff account (Email/Password provider + user) is created in the Firebase Console, not in code.

## Conventions

- Everything is global functions and top-level `let` state (`inv`, `orders`, `cart`, `activeTables`, `currentPay`) — match this style rather than introducing modules or a framework.
- Notifications: `notyf.success/error` for toasts, `Swal.fire` for confirmations/prompts (always with `heightAuto: false` for correct mobile sizing).
- Layout is heavily tuned for iPhone (safe-area insets, dynamic island, `100dvh`) — recent commit history is mostly mobile layout fixes, so verify changes on a narrow mobile viewport.
