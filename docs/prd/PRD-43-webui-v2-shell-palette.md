# PRD-43 — WebUI v2: App Shell + Navigasi + Command Palette

| | |
|---|---|
| Feature | App-shell modern 2026: topbar ramping + sidebar sesi collapsible/resizable + hash-router + command palette `Ctrl/⌘+K` + fondasi modal/drawer/toast |
| Depends on | PRD-42 (token/komponen/motion/a11y), PRD-13 (pola SPA), PRD-18 (hash routing), PRD-32 (grouping sesi) |
| Files | `src/pha/gateway/webui/v2/index.html` (shell), `src/pha/gateway/webui/v2/js/boot.js+store.js+api.js+a11y.js` (baru), `tests/test_pha/test_webui_v2.py` (tambah smoke shell) |
| Status | Draft v1.0 (2026-10-15) |

## 1. Ringkasan

Navbar v1 padat (7 tombol + badges `max-width:38vw` mudah terpotong; mobile sembunyikan greeting+model), sidebar fixed 300px / collapsed 56px tanpa resize, drawer mobile tanpa focus-trap, `chat-header overflow-x:auto` + title hilang (konteks sesi hilang), z-index tak konsisten, modal generik `max-width:480px`.

PRD-43 membangun shell v2: **topbar** (breadcrumbs sesi + search `Ctrl/⌘+K` + badges ciut + theme-toggle), **sidebar** (daftar sesi + grup + tombol sesi baru, collapsible 56px + resizable `localStorage pha-v2-sidebar-w`, mobile = drawer + focus-trap + scrim), **main** 8 rute hash, **fondasi modal/drawer/toast** dipakai PRD-44 s.d. 47. State global `var state` v1 diganti **store kecil pub/sub** (`session/view/ui`), cache per-view disatukan polanya.

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Rute hash | `#/chat/:id`, `#/setup`, `#/skills`, `#/agents`, `#/tools`, `#/mcp`, `#/cron`, `#/trace`; deep-link + back/forward browser; guard sesi hilang → `#/chat` + toast ramah |
| Topbar | Kiri: toggle sidebar + breadcrumbs (view / sesi aktif); tengah: search semu (tombol `Cari atau perintah… ⌘K`); kanan: badges ciut (conn/model/provider/channels/system-one) + theme-toggle + greeting tanggal (pola PRD-20) |
| Sidebar | Atas: tombol `+ Sesi baru`; tengah: grup (reuse `GET /v1/session-groups`, `POST /v1/sessions/:id/group`) + sesi (reuse `GET /v1/sessions`, rename/fork/hapus); bawah: usage ringkas (`GET /v1/usage`) + status koneksi. Collapsible 56px (ikon + tooltip), resizable 240-420px persisten, mobile drawer 84vw + focus-trap + Esc + scrim |
| Command palette | `Ctrl/⌘+K` atau klik search: daftar aksi (navigasi 8 rute, sesi baru, toggle tema, buka setup/cron) + perintah `/` (reuse `GET /v1/commands`) + sesi (lompat). Filter fuzzy client-side, `↑↓/Enter/Esc`, mouse penuh, focus-return ke pemicu |
| Store | `store.js` pub/sub minimal (`get/sub/emit`, slice `session/view/ui`); ganti mutasi `var state` + `innerHTML` campur; langganan render per slice agar update tak render ulang shell |
| API wrapper | `api.js`: `fetch` + `Bearer`/`X-Pha-Token` + `attachments?token=` untuk `<img>` + error-map Indonesia (reuse PRD-39: tanpa kata `413/400/--flag/mime`) + timeout + `401` → layar token (pola `token TTL` PRD-24) |
| A11y fondasi | `a11y.js`: `trapFocus(container)`, `returnFocus`, `announce(msg)` (`aria-live`), hormat `prefers-reduced-motion`; landmark (`header/nav/main`), `h1` per view, skip-link `Lewati ke konten` |
| Responsif | Desktop: sidebar + main; ≤1024: sidebar overlay; ≤860: drawer + bottom-sheet panel; 320px tanpa scroll-x tak disengaja; tabel wrap + scroll-x aman (PRD-22) |
| Tanpa | Konten chat (PRD-44), isi tiap view (PRD-45 s.d. 47), ubah endpoint (reuse 100%) |

## 3. Call-site

| # | Lokasi | Guna | Fallback |
|---|---|---|---|
| 1 | `v2/index.html` | Shell: skip-link, topbar, sidebar, main 8 view kosong, modal/drawer/toast root, scrim | — |
| 2 | `v2/js/boot.js` | Init tema/auth/router/palette; guard sesi hilang | `#/chat` + toast bila rute asing |
| 3 | `v2/js/store.js` | Pub/sub slice `session/view/ui` | — |
| 4 | `v2/js/api.js` | Wrapper auth + error-map + `401` → layar token | Error mentah bila map hilang (existing) |
| 5 | `v2/js/a11y.js` | Focus-trap, focus-return, announce, reduced-motion | Fokus biasa bila trap gagal |
| 6 | `tests/test_pha/test_webui_v2.py` | Smoke: shell render (id rute, palette, sidebar), tanpa console-error statis | — |

## 4. Goals

- **G1** — Navigasi 8 rute + deep-link + back/forward; konteks sesi tak pernah hilang (breadcrumbs).
- **G2** — Palette `Ctrl/⌘+K` jadi jalan pintas utama (navigasi + perintah + sesi).
- **G3** — Keyboard penuh + focus-trap drawer/palette/modal + `aria-live`; gate `frontend-ui-engineering` §Accessibility/Responsive lolos.

## 5. Non-goals

- Isi chat/view (PRD-44 s.d. 47). Ubah endpoint/kontrak SSE. Sidebar drag di mobile (tap/overlay saja).

## 6. Acceptance criteria

- **AC-43.1** Suite + lint hijau; tanpa dep/build baru; `external == []`.
- **AC-43.2** 8 rute hash jalan + deep-link + back/forward; rute asing → `#/chat` + toast ramah.
- **AC-43.3** Palette: buka via `Ctrl/⌘+K` + klik; filter perintah/sesi/aksi; `↑↓/Enter/Esc`; focus kembali ke pemicu saat tutup.
- **AC-43.4** Sidebar: collapse 56px + resize 240-420px persisten reload; mobile drawer + focus-trap + Esc + scrim; tak ada fokus bocor keluar.
- **AC-43.5** 320/768/1024/1440 rapi; topbar tak terpotong; skip-link + landmark + `h1` per view; toast terbaca screen-reader.
- **AC-43.6** `401` → layar token ramah (bukan halaman putih); error lain tanpa istilah HTTP/flag CLI.

## 7. Testing

1. `pytest tests/test_pha/test_webui_v2.py -q` (smoke shell + rute + palette root).
2. `ruff check src/pha/gateway/webui/v2` + `node --check` 4 file JS.
3. Manual: Tab penuh tanpa mouse; palette di tiap rute; resize + reload; 320px + 1440px; screen-reader baca breadcrumbs + toast.

## 8. Out of scope / backlog

- Sidebar multi-workspace; palette plugin pihak ketiga — backlog.

## 9. Referensi

- `src/pha/gateway/webui/markup.py:16-121` (navbar + chat layout), `spa_js.py` (routing + cache view), `app_js.py:670-811` (picker sesi), `health.py:758-820` (sessions/groups/usage)
- `docs/prd/PRD-13-webui-spa.md`, `PRD-18-webui-room-improvement.md`, `PRD-20-webui-production.md`, `PRD-32-session-grouping.md`
- `.commandcode/skills/frontend-ui-engineering/SKILL.md` (§Keyboard, §ARIA, §Focus, §Responsive)
