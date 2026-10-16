# PRD-48 — WebUI v2: Wiring + Flag + Vendor-Audit + Parity & Rollout

| | |
|---|---|
| Feature | Wiring server (`/ui/v2` + `/v2/*`), flag `webui_v2` default `v1`, audit vendor offline, smoke test, parity checklist, sinkron docs |
| Depends on | PRD-41 s.d. PRD-47 (seluruh v2) |
| Files | `src/pha/gateway/webui/v2/static.py` (serve), `src/pha/gateway/health.py` (rute), `src/pha/gateway/{models,config,service}.py` (flag + status), `tests/test_pha/test_webui_v2.py` (smoke), `docs/prd/README.md`, `README.md`, `DEVLOG.md` |
| Status | Draft v1.0 (2026-10-15) |

## 1. Ringkasan

PRD-42 s.d. 47 membangun v2 berdampingan tanpa menyentuh server. PRD-48 menyambungkannya: serve file statis via `static.py` (tanpa CWD-relatif — impor `ROOT` dari modul, taste workflow), rute `GET /ui/v2` + `GET /v2/*` di `health.py` dengan auth yang sama (`_requires_auth`, `Bearer`/`X-Pha-Token`), flag `GatewayConfig.webui_v2: Literal["v1","v2"] = "v1"` + expose di `GET /status`, audit vendor (size + head cegah HTML-error-as-binary), smoke test, parity checklist, dan sinkron docs. **PR ini tidak flip default** — flip ke `v2` dilakukan PR terpisah setelah parity lolos.

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Serve statis | `v2/static.py`: `ROOT = Path(__file__).parent`, peta `index.html/app.css/js/*/vendor/*`, MIME benar, anti path-traversal (`resolve().is_relative_to(ROOT)`), cache `Cache-Control: no-cache` untuk html + `immutable 1y` untuk vendor berversi; test impor `ROOT`, bukan CWD-relatif |
| Rute | `GET /ui/v2` → `index.html`; `GET /v2/*` → file; auth ikut `_requires_auth` (open hanya `/health`, `/ui`, `/` — `/ui/v2` + `/v2/*` butuh token bila `webui_token` set); `GET /v1/attachments?token=` tetap satu-satunya query-token |
| Flag | `GatewayConfig.webui_v2: Literal["v1","v2"] = "v1"` (pydantic, single source of truth — tanpa env tersembunyi, taste architecture); `config.py` load/save toleran (hilang → `v1`); `service.py` expose `webui_v2` di `GET /status`; v1 dibekukan (tanpa fitur baru) |
| Vendor-audit | Tiap file `vendor/`: catat nama + versi + sumber + sha + size di `v2/vendor/README.md`; smoke tolak file `<1KB` atau head `<!doctype`/`<html` (HTML-error-as-binary); probe runtime: `node --check` tiap `.js`, KaTeX/mermaid load tanpa error di smoke browser manual |
| Smoke test | `tests/test_pha/test_webui_v2.py`: shell render (8 id rute + palette + sidebar), token + alias, `external == []`, flag default `v1`, daftar vendor lengkap, XSS + error-map (reuse AC-44), 3 bahasa (reuse AC-45) |
| Parity gate | Checklist PRD-41 §5 harus hijau sebelum PR flip: 7 view + trace/audit + HITL + upload + i18n; SSE re-attach/STOP/fallback; setup diff/restart; cron preview/picker; error-map; 320/768/1024/1440; keyboard + screen-reader; kontras AA |
| Docs | `docs/prd/README.md` tambah baris PRD-41 s.d. 48; `README.md` seksi WebUI v2 (cara buka `/ui/v2`, flag, status parity); `DEVLOG.md` entri bertanggal; PRD disesuaikan dengan yang benar dibangun (deviation notes, taste workflow) |
| Tier lean | **High-rigor** (auth/token + serve statis + flag kontrak publik): `lean-design` bila ambiguitas MIME/cache; `lean-tdd` untuk traversal-guard + auth + default-flag; `lean-code-review` + `lean-verification` penuh |
| Tanpa | Flip default ke `v2` (PR terpisah); Electron `desktop/`; PWA; ubah SSE/engine; endpoint baru selain `/ui/v2` + `/v2/*` |

## 3. Call-site

| # | Lokasi | Guna | Fallback |
|---|---|---|---|
| 1 | `v2/static.py` | `ROOT`, peta file, MIME, traversal-guard | `404`/`403` ramah (bukan traceback) |
| 2 | `health.py` | Rute `/ui/v2` + `/v2/*` via provider/serve statis (pola existing) | `503` bila provider mati |
| 3 | `models.py` + `config.py` | Field `webui_v2` + load/save kompatibel | `v1` bila key hilang (kompat) |
| 4 | `service.py` | Expose `webui_v2` di `GET /status` | — |
| 5 | `v2/vendor/README.md` | Manifest vendor (versi + sha + size) | — |
| 6 | `tests/test_pha/test_webui_v2.py` | Smoke §Testing | — |

## 4. Goals

- **G1** — `/ui/v2` + `/ui` jalan berdampingan; default `v1`; nol regresi v1.
- **G2** — Offline penuh + aman: traversal-guard, auth konsisten, vendor teraudit.
- **G3** — Parity checklist + docs sinkron jadi syarat flip (PR berikutnya).

## 5. Non-goals

- Flip default; Electron/PWA; ubah kontrak SSE/API; fitur baru v1.

## 6. Acceptance criteria

- **AC-48.1** `pytest tests/test_pha/test_webui_*.py -q` + `ruff check src/pha` + `node --check` semua JS v2 hijau.
- **AC-48.2** `GET /ui/v2` + `/v2/*` hormati `webui_token` (tanpa token → `401` JSON, bukan HTML); traversal `../` → `403/404` tanpa bocor path.
- **AC-48.3** Flag default `v1` (config lama tanpa key tetap load); `GET /status` tampilkan `webui_v2`.
- **AC-48.4** `external == []` di HTML v2; tiap vendor ada di manifest + size wajar + head bukan HTML; `GET /health` tetap open.
- **AC-48.5** Parity checklist PRD-41 §5 terdokumentasi hasilnya (lolos/belum + bukti) sebagai lampiran PR.
- **AC-48.6** `docs/prd/README.md` + `README.md` + `DEVLOG.md` update di sesi yang sama; tiap deviasi dari PRD-42 s.d. 47 dicatat di PRD terkait.

## 7. Testing

1. `pytest tests/test_pha/test_webui_v2.py tests/test_pha/test_webui_api.py tests/test_pha/test_webui_spa.py tests/test_pha/test_webui_crud.py -q`.
2. `ruff check src/pha` bersih + `node --check` tiap file `v2/js/*.js` + `v2/vendor/*.min.js` (parse saja).
3. Manual: `/ui` vs `/ui/v2` berdampingan; token salah → `401` ramah; traversal `curl /v2/../../gateway.json` → `403/404`; offline penuh (putus internet → v2 tetap utuh); QA 320/768/1024/1440 + keyboard + screen-reader; audit kontras AA.

## 8. Out of scope / backlog

- PR flip default `v1 → v2` (syarat: parity hijau + persetujuan user); Electron `desktop/` shell (taste architecture — artefak terpisah, bukan wheel); PWA/service-worker.

## 9. Referensi

- `src/pha/gateway/health.py:214-442` (auth + rute), `service.py:2263-2307` (wiring + status), `models.py:48-86` (`GatewayConfig`), `config.py:20` (load/save)
- `src/pha/gateway/webui/__init__.py:56-66` (assembler v1 — pola serve yang diganti `static.py`)
- `docs/prd/PRD-41-webui-modern-overview.md` (§5 parity checklist)
- `.commandcode/plugins/lean-engineering/skills/lean-{engineering-workflow,planning,minimal-implementation,tdd,code-review,verification}/SKILL.md`
