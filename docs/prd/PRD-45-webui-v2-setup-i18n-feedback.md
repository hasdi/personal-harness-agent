# PRD-45 — WebUI v2: Setup + i18n + Sistem Feedback

| | |
|---|---|
| Feature | Setup ramah (Basic + search + validasi inline + banner restart) + i18n 3 bahasa + toast/modal/empty-state konsisten |
| Depends on | PRD-42 (token/komponen), PRD-43 (shell/api/a11y), PRD-10/36 (pola Setup ramah), PRD-12 (i18n), PRD-39 (error-map + empty-state) |
| Files | `src/pha/gateway/webui/v2/js/views-setup.js+i18n.js` (baru), `tests/test_pha/test_webui_v2.py` (tambah: 3 bahasa + secret-mask) |
| Status | Draft v1.0 (2026-10-15) |

## 1. Ringkasan

Setup v1 sudah ramah (Basic ~9 field + grup collapsible + search + editor list + secret peek + diff restart + dirty-guard, PRD-36) tapi dibangun di atas `renderConfigForm` monolit + `buildConfigRow` multi-kind + modal sempit 480px. PRD-45 memindahkan pola yang terbukti itu ke v2 dengan editor modal lebar + sistem feedback seragam (toast `aria-live` + empty-state ber-CTA + hapus konfirmasi sebut-nama) yang dipakai ulang PRD-46/47, plus kunci i18n baru 3 bahasa sejak awal.

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Setup | Reuse kontrak `GET /v1/config` (mask `_CONFIG_SECRET_FIELDS`) + `POST /v1/config` (merge `model_fields`, `model_validate`, hot-apply) + `POST /v1/service/restart` + polling `/health`. Basic (~9 populer) + Lanjutan grup collapsible + search nama/label/help; editor list untuk array-string; JSON hanya objek + validasi ramah; secret show/hide (nilai tetap mask server); diff konfirmasi restart; dirty-guard pindah rute (Buang/Batal); banner "butuh restart" untuk field restart |
| Kinds field | `bool/strlist/json/enum/secret/nullable/number/conf` (slider 0-100% untuk `decision_conf_*`) — reuse semantik `buildConfigRow`, render dengan komponen PRD-42 |
| i18n | Reuse 200 kunci existing + kunci baru wajib id/en/ja; fallback Indonesia bila kunci hilang; `applyLanguage` sapu ulang shell + view aktif; greeting tanggal 3 bahasa (PRD-20/23/24) |
| Feedback seragam | Toast sukses hijau spesifik / gagal merah + solusi (`aria-live`, max 3, 4 dtk); empty-state (kalimat + 1 CTA: "Buat cron pertama", "Tambah skill"); hapus selalu modal sebut-nama; error-map Indonesia tanpa HTTP/flag CLI; fokus-balik composer/input setelah aksi |
| Modal lebar | Editor form `640-720px`, bukan 480px; mobile full-sheet; focus-trap + Esc + scrim (reuse `a11y.js`) |
| Validasi | Inline per field (bukan alert); `highlightConfigErrors` tunjuk field gagal; pesan tanpa traceback |
| Tier lean | **High-rigor** untuk `POST /v1/config` + secret + restart (keamanan/kontrak); Standard untuk i18n/feedback |

## 3. Call-site

| # | Lokasi | Guna | Fallback |
|---|---|---|---|
| 1 | `v2/js/views-setup.js` | Form setup + search + strlist + secret + diff + dirty + banner | Grup lama bila search kosong |
| 2 | `v2/js/i18n.js` | Kunci id/en/ja + `t()/applyLanguage()` | Indonesia bila kunci hilang |
| 3 | Endpoint reuse | `GET/POST /v1/config`, `POST /v1/service/restart`, `GET /health`, `GET /status` | Tanpa endpoint baru |

## 4. Goals

- **G1** — User awam ubah model/channel/izin/bahasa tanpa buka JSON (parity PRD-36).
- **G2** — Semua string baru 3 bahasa sejak PR; feedback seragam siap dipakai PRD-46/47.
- **G3** — Secret tak bocor; restart terdokumentasi via diff + banner.

## 5. Non-goals

- Field `gateway.json` baru; wizard setup berlangkah; isi view lain (PRD-46/47).

## 6. Acceptance criteria

- **AC-45.1** Suite + lint hijau; `node --check`; tanpa dep/build baru.
- **AC-45.2** Setup: Basic tampil; search temukan field; array tanpa JSON; secret intip; save restart tampilkan diff; dirty ada prompt Buang/Batal.
- **AC-45.3** Secret termask di GET dan tak berubah bila dikirim mask; config invalid ditolak pesan jelas (bukan traceback).
- **AC-45.4** Semua string baru ada di id/en/ja; ganti bahasa sapu ulang tanpa reload; fallback Indonesia bila kunci hilang.
- **AC-45.5** Toast `aria-live` terbaca screen-reader; empty-state + hapus-sebut-nama konsisten; fokus kembali ke input setelah modal tutup.
- **AC-45.6** Mobile 360px: form + modal full-sheet tanpa overflow-x; banner restart jelas.

## 7. Testing

1. `pytest tests/test_pha/test_webui_v2.py -q` (3 bahasa, mask secret, smoke setup).
2. `ruff check src/pha/gateway/webui/v2` + `node --check` (`views-setup.js`, `i18n.js`).
3. Manual: ubah bahasa (hot-apply) + port (diff + restart); secret intip; dirty-guard pindah rute; screen-reader baca toast.

## 8. Out of scope / backlog

- Wizard berlangkah; validasi silang antar-field kompleks — backlog.

## 9. Referensi

- `src/pha/gateway/webui/app_js.py:2740-3264` (`renderConfigForm`, diff, dirty), `health.py:820-1488` (config/restart), `service.py:1392-1412` (mask + save), `i18n_js.py` (~200 kunci)
- `docs/prd/PRD-10-webui-setup-panel.md`, `PRD-12-webui-language.md`, `PRD-20-webui-production.md`, `PRD-23-hero-greeting-copy.md`, `PRD-24-greeting-fresh-token.md`, `PRD-36-webui-setup-cron-friendly.md`, `PRD-39-ux-ramah-awam.md`
