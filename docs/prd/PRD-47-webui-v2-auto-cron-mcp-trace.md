# PRD-47 — WebUI v2: Cron / MCP / Trace-Audit / HITL

| | |
|---|---|
| Feature | Empat area otomatisasi & observabilitas: Cron (preset + builder + preview + target-picker) + MCP (badge + editor + test) + Trace/Audit (accordion + filter + purge) + HITL (permission drawer benar) |
| Depends on | PRD-42 (token/komponen), PRD-43 (shell/api/a11y), PRD-45 (feedback/i18n/modal), PRD-1/2/3 (trace), PRD-6 (HITL), PRD-16/36 (cron ramah), PRD-40 (MCP) |
| Files | `src/pha/gateway/webui/v2/js/views-auto.js+views-trace.js` (baru), `tests/test_pha/test_webui_v2.py` (tambah smoke 4 area) |
| Status | Draft v1.0 (2026-10-15) |

## 1. Ringkasan

Empat area v1 sudah matang: Cron (preset + builder + preview 3 next-run + target-picker + countdown + run-now); MCP (badge connected/failed/pending/disabled + editor stdio/http + preset fs/sqlite/github/postgres + test 15 dtk); Trace (accordion run + filter all/errors/tools/systemone + detail JSON + polling 30 dtk + purge per run/sesi + jejak `trail.purged`); Audit (tail + filter session_key + scope sesi/all); HITL (`permission-drawer` polling 2 dtk + resolve). Masalah sisa: overlay drawer rapuh (`::before`), side-panel sempit di 1024px, tabel mobile paksa scroll.

PRD-47 memindahkan keempatnya ke v2 dengan overlay/scrim baku PRD-42 + feedback PRD-45, reuse 100% endpoint, tanpa ubah semantik cron/trace/HITL.

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Cron | Reuse `GET/POST /v1/cron`, `PUT/DELETE`, `POST /v1/cron/{enable,remove,run-now}`, `GET /v1/cron/{validate,targets}`. Editor: 5 preset + builder jam/menit/pola + preview live 3 run (debounce 400ms) + target-picker (sesi WebUI + channel + "sesi ini"); list: label manusiawi (`cronHumanLabel`) + countdown + nama target; toggle toast; hapus tampilkan jadwal/target; run-now tawarkan buka sesi; error tanpa `--flag` |
| MCP | Reuse `GET/POST /v1/mcp`, `POST /v1/mcp/{enable,remove,test}`. List (nama + transport + target + badge status + jumlah tools) + preset isi-cepat + test tanpa simpan + secret mask (env/headers); tambah/ubah/hapus/enable → banner restart (key `mcp_servers` di restart-fields); badge pakai ikon + teks (bukan warna saja) |
| Trace | Reuse `GET /v1/sessions/:id/runs`, `GET .../runs/:run`, `DELETE .../trace[/:run]`. Panel/split akordeon + filter chips + detail JSON + polling 30 dtk + purge konfirmasi sebut run/sesi + jejak `trail.purged`; ≤3 klik dari buka aplikasi (G2 PRD-0) |
| Audit | Reuse `GET /v1/audit?limit&session_key`. Tail + filter session_key + scope sesi/semua; sender tetap hash; purge tercatat |
| HITL | Reuse `GET /v1/permission/pending` + `POST /v1/permission`. Drawer polling 2 dtk + overlay `.scrim` baku (ganti `::before` rapuh) + z-index `--z-drawer` + focus-trap; timeout/conf rendah = fallback diam (parity System One PRD-33/35) |
| Layout | Side-panel `clamp(320px,36vw,540px)` dievaluasi ulang di ≥1280 agar chat tak sempit; mobile = bottom-sheet `78dvh` + focus-trap |
| Tier lean | **High-rigor** untuk purge trace + permission-resolve (destruktif/keamanan); Standard untuk cron/MCP |

## 3. Call-site

| # | Lokasi | Guna | Fallback |
|---|---|---|---|
| 1 | `v2/js/views-auto.js` | Cron + MCP list/editor/test/picker/banner | Field mentah advanced bila builder gagal |
| 2 | `v2/js/views-trace.js` | Trace/Audit + purge + HITL drawer | Teks mentah bila pola asing |
| 3 | Endpoint reuse | Cron/MCP/Trace/Audit/Permission di atas | Tanpa endpoint baru |

## 4. Goals

- **G1** — Parity penuh 4 area + overlay/focus benar + banner restart MCP.
- **G2** — Purge aman (konfirmasi + jejak audit); permission tak pernah bocor fokus.
- **G3** — Countdown/label manusiawi; badge status berikon + teks.

## 5. Non-goals

- Live SSE tail trace; full-text search/export trail; ubah engine/cron-backend; OAuth MCP interaktif.

## 6. Acceptance criteria

- **AC-47.1** Suite + lint hijau; `node --check`; tanpa dep/build baru; string baru 3 bahasa.
- **AC-47.2** Cron: preset/builder isi field; preview 3 run; picker prefill sesi aktif; save tanpa `chat_id` manual; countdown + run-now tawarkan buka sesi.
- **AC-47.3** MCP: preset isi-cepat; test tanpa simpan (sukses + gagal ramah); secret mask; disable hilangkan `mcp__*` setelah restart + banner.
- **AC-47.4** Trace ≤3 klik; filter chips benar; purge per run/sesi konfirmasi + tinggalkan `trail.purged`; audit scope + sender hash.
- **AC-47.5** HITL: drawer muncul + focus-trap + Esc/scrim; approve/tolak resolve; polling berhenti saat tutup.
- **AC-47.6** Mobile 360px panel bottom-sheet tanpa jebak fokus; tabel/detail wrap aman.

## 7. Testing

1. `pytest tests/test_pha/test_webui_v2.py -q` (smoke cron-validate, mcp-mask, trace-purge-guard, permission).
2. `ruff check src/pha/gateway/webui/v2` + `node --check` (`views-auto.js`, `views-trace.js`).
3. Manual: cron harian 09:00 → preview → save → run-now → buka sesi; MCP filesystem preset → test → restart → tools muncul; purge run + cek `trail.purged`; HITL approve/tolak via keyboard.

## 8. Out of scope / backlog

- Live tail, export/share trail, auto-reconnect MCP tanpa restart — backlog (lihat PRD-4 §Future).

## 9. Referensi

- `src/pha/gateway/webui/spa_js.py:819-1484` (cron/mcp v1), `app_js.py:2360-2738` (trace/audit/HITL), `health.py` + `service.py` (cron/mcp/trace/audit/permission)
- `docs/prd/PRD-1-trace-core.md`, `PRD-2-trace-api.md`, `PRD-3-trace-webui.md`, `PRD-6-human-in-the-loop.md`, `PRD-14-webui-cron-delivery.md`, `PRD-16-webui-cron-crud.md`, `PRD-33-system-one.md`, `PRD-35-system-one-action-gate.md`, `PRD-36-webui-setup-cron-friendly.md`, `PRD-40-webui-mcp.md`
