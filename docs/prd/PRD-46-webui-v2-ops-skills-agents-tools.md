# PRD-46 — WebUI v2: Skills / Agents / Tools

| | |
|---|---|
| Feature | Tiga view operasional konsisten: Skills (grid + split-detail + CRUD + enable) + Agents (profil + picker tool/skill + effort) + Tools (katalog read-only + filter + toggle) |
| Depends on | PRD-42 (token/komponen), PRD-43 (shell/api/a11y), PRD-45 (feedback/i18n/modal lebar), PRD-15/29/30 (pola CRUD + toggle + profil agent) |
| Files | `src/pha/gateway/webui/v2/js/views-ops.js` (baru), `tests/test_pha/test_webui_v2.py` (tambah smoke 3 view) |
| Status | Draft v1.0 (2026-10-15) |

## 1. Ringkasan

Tiga view v1 sudah lengkap tapi berpola duplikat: Skills grid + split-view + drag-resize (`--skill-w` 280-640px persisten) + editor + toggle enable; Agents list + editor modal tab picker tool/skill (mandatory ★) + cache picker; Tools katalog read-only + toggle + filter runtime. PRD-46 menyatukan ketiganya dalam satu modul `views-ops.js` berpola sama (list + search + detail + editor + konfirmasi + toast), reuse 100% endpoint existing, dengan editor modal lebar + feedback PRD-45.

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Pola seragam | Tiap view: header (`h1` + search + tombol Buat) + grid/list + empty-state ber-CTA + detail + editor modal 640-720px + hapus konfirmasi sebut-nama + toast spesifik; tanpa pola khusus per view |
| Skills | Reuse `GET /v1/skills`, `GET /v1/skills/:n`, `POST/PUT/DELETE`, `POST /v1/skills/enable {name,enabled}`. Grid kartu (nama + deskripsi + badge enabled) + split-detail (pratinjau SKILL.md terrender aman via `render.js`) + resize persisten (`localStorage pha-v2-skill-w` 280-640px); editor (nama/deskripsi/konten); toggle tanpa reload |
| Agents | Reuse `GET/POST /v1/agents`, `PUT/DELETE /v1/agents/:n`, `POST /v1/sessions/:id/{agent,effort,preset}`. Editor: nama + deskripsi + allowlist/mandatory tools + skills + effort default (picker checkbox + ★ mandatory, cache `ensurePickCaches`); bar composer tampilkan agent/effort/preset aktif (parity PRD-31) |
| Tools | Reuse `GET /v1/tools`, `POST /v1/tools/enable`. Katalog read-only (nama + sumber + status runtime) + search + filter (semua/aktif/nonaktif/mcp) + toggle; tools `mcp__*` muncul setelah restart (parity PRD-40) |
| Search/filter | Client-side per view (nama + deskripsi); hasil kosong → empty-state + tombol reset/buat; skeleton saat load (bukan spinner) |
| i18n/feedback | Semua string baru 3 bahasa; toast + modal + empty-state reuse PRD-45; `escHtml` semua dinamis |
| Tanpa | Endpoint baru; ubah semantik enable/mandatory; marketplace skill (backlog) |

## 3. Call-site

| # | Lokasi | Guna | Fallback |
|---|---|---|---|
| 1 | `v2/js/views-ops.js` | List/detail/editor/toggle 3 view + picker agent | Teks `…` lama bila data gagal |
| 2 | Endpoint reuse | Skills/Agents/Tools CRUD + enable + session agent/effort/preset | Tanpa endpoint baru |

## 4. Goals

- **G1** — Parity penuh v1 untuk 3 view dengan pola konsisten + modal lebar.
- **G2** — Toggle/enable tanpa reload; picker mandatory jelas (★ + label, bukan warna saja).
- **G3** — Gate `frontend-ui-engineering`: skeleton, empty ber-CTA, keyboard, 3 bahasa.

## 5. Non-goals

- Ubah model data agent/skill/tool; marketplace; ubah runtime tool.

## 6. Acceptance criteria

- **AC-46.1** Suite + lint hijau; `node --check`; tanpa dep/build baru; string baru 3 bahasa.
- **AC-46.2** Skills: buat/ubah/hapus/enable + pratinjau terrender aman + resize persisten reload; hapus konfirmasi sebut nama.
- **AC-46.3** Agents: profil tersimpan (allowlist + mandatory + effort default); picker ★ jelas; sesi pakai agent/effort/preset dan tampil di composer.
- **AC-46.4** Tools: katalog + search + filter benar; toggle ber-toast; `mcp__*` muncul setelah restart.
- **AC-46.5** Cari-tak-cocok tiap view ada kalimat + tombol aksi; load tampilkan skeleton; mobile 360px grid 1 kolom tanpa overflow.

## 7. Testing

1. `pytest tests/test_pha/test_webui_v2.py -q` (smoke 3 view + enable + mandatory).
2. `ruff check src/pha/gateway/webui/v2` + `node --check` (`views-ops.js`).
3. Manual: buat skill → pratinjau → enable; buat agent + mandatory tool → pakai di sesi; toggle tool → toast; 360px + keyboard + screen-reader.

## 8. Out of scope / backlog

- Marketplace/berbagi skill; versi skill; dry-run agent — backlog bila ada demand.

## 9. Referensi

- `src/pha/gateway/webui/spa_js.py:94-817` (skills/agents/tools v1), `app_js.py:670-811` (picker), `health.py` + `service.py` (CRUD skills/agents/tools)
- `docs/prd/PRD-15-webui-skills-crud.md`, `PRD-29-webui-tools-toggle.md`, `PRD-30-agent-profile.md`, `PRD-31-session-agent-effort-picker.md`, `PRD-40-webui-mcp.md`
