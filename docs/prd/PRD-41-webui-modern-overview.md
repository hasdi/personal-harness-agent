# PRD-41 — WebUI Modern 2026: Overview & Index (v2 Vanilla Multi-file)

| | |
|---|---|
| Feature | Modernisasi UI/UX WebUI ala modern webapp 2026: app-shell, command palette, chat modern, 7 view konsisten — vanilla multi-file tanpa build step |
| Depends on | PRD-7/8/9 (fondasi UI/UX), PRD-12 (i18n), PRD-19 (token Claude×Notion), PRD-36/37/38/39 (pola ramah awam) |
| Files | `docs/prd/PRD-41-*.md` (ini) s.d. `PRD-48-*.md`, `docs/prd/README.md` (index) |
| Status | Draft v1.0 (2026-10-15) |

## 1. Ringkasan

WebUI PHA hari ini **berfungsi penuh** (chat SSE, setup, skills, agents, tools, MCP, cron, trace/audit, HITL, upload, i18n 3 bahasa) tapi strukturnya single-file Python-string (~7000 baris di `src/pha/gateway/webui/`, HTML final ~350-400 KB) dan visualnya masih warisan calm-professional PRD-7/8/19 yang belum setara webapp 2026.

PRD-41 s.d. PRD-48 memecah modernisasi menjadi **8 PR kecil berurutan**, masing-masing satu sesi implementasi pendek (taste workflow: satu PRD = satu PR reviewable):

| PRD | Isi | Depends |
|---|---|---|
| [PRD-41](./PRD-41-webui-modern-overview.md) (ini) | Overview, prinsip 2026, metrik, rollout flag, parity checklist | PRD-7/8/19/39 |
| [PRD-42](./PRD-42-webui-v2-design-system.md) | Design system v2: token + komponen + motion + a11y | PRD-41 |
| [PRD-43](./PRD-43-webui-v2-shell-palette.md) | App shell + navigasi + command palette | PRD-42 |
| [PRD-44](./PRD-44-webui-v2-chat-streaming.md) | Chat modern: streaming + composer + render aman | PRD-42, PRD-43 |
| [PRD-45](./PRD-45-webui-v2-setup-i18n-feedback.md) | Setup + i18n + sistem feedback | PRD-42, PRD-43 |
| [PRD-46](./PRD-46-webui-v2-ops-skills-agents-tools.md) | Skills / Agents / Tools | PRD-42, PRD-43 |
| [PRD-47](./PRD-47-webui-v2-auto-cron-mcp-trace.md) | Cron / MCP / Trace-Audit / HITL | PRD-42, PRD-43 |
| [PRD-48](./PRD-48-webui-v2-wiring-rollout.md) | Wiring, flag, vendor-audit, parity & rollout | semua 41-47 |

Keputusan stack yang sudah dikunci user: **vanilla multi-file v2** (`src/pha/gateway/webui/v2/`: `index.html` + `app.css` + `js/` + `vendor/` committed, tanpa bundler/Node), **scope penuh 7 view + sistem desain**, **evolusi Claude×Notion** (bukan gaya baru total, bukan kloning mentah nanobot).

## 2. Prinsip 2026 (mengikat semua PRD 42-48)

1. **Vanilla tanpa build** — tanpa `package.json`, tanpa bundler, tanpa dep Node; vendor minified (marked, DOMPurify, highlight.js, diff, KaTeX, mermaid) di-commit untuk offline (taste frontend).
2. **Sanitasi wajib** — tiap render markdown/HTML model lewat DOMPurify sebelum `innerHTML` (taste frontend, XSS).
3. **Jangan sentuh engine** — `src/openharness/**` beku; fitur baru di edge `webui/v2/`, edit file lama minimal (wiring saja). Reuse seams: `RunsApi` detached-run, `session_key api:<uuid>`, endpoint existing (taste architecture).
4. **Reuse endpoint** — tanpa endpoint baru kecuali `/ui/v2` + `/v2/*` static + field flag; kontrak SSE (`run/delta/progress/tool_hint/file_edit/error/cron/final/done`) tak berubah.
5. **Flag + parity** — `webui_v2: Literal["v1","v2"] = "v1"` default v1; v1 dibekukan; flip hanya setelah parity checklist (§5) lolos.
6. **Gate UI tiap PRD** — checklist `frontend-ui-engineering`: token semantik, spacing `0.25rem`, tanpa AI-aesthetic, kontras AA (4.5:1 / 3:1), keyboard penuh, `aria-live`, focus-trap + focus-return, `prefers-reduced-motion`, responsif 320/768/1024/1440, skeleton bukan spinner, empty/error/success/permission bermakna + CTA.

## 3. Metrik sukses global

| Metrik | Target |
|---|---|
| Parity 7 view | Chat/Setup/Skills/Agents/Tools/MCP/Cron + Trace/Audit + HITL setara v1 |
| External URL | `external == []` (semua vendor lokal; font sistem) |
| Kualitas | `pytest tests/test_pha/test_webui_*.py -q` + `ruff check src/pha` + `node --check` JS hijau |
| Kontras | Teks normal ≥4.5:1, teks besar ≥3:1 di light + dark |
| Responsif | Rapi di 320/768/1024/1440, tanpa scroll horizontal tak disengaja |
| Aksesibilitas | Tab keyboard penuh, screen-reader (toast/status `aria-live`), axe bersih |
| Performa sesi panjang | Delta throttled rAF + paging `before/limit=100`, tanpa jank |

## 4. Rollout

1. PRD-42 s.d. 47 bangun `v2/` berdampingan v1 (v1 tak diubah kecuali wiring).
2. PRD-48 tambah rute + flag (default `v1`), smoke test, audit vendor.
3. Parity checklist (§5) → flip default ke `v2` di PR terpisah (bukan di PRD-48).

## 5. Parity checklist (syarat flip default)

- [ ] 7 view + trace/audit + HITL + upload + i18n 3 bahasa setara v1
- [ ] SSE: optimistic bubble, throttle rAF, re-attach reload, STOP/cancel, fallback blocking
- [ ] Setup: grup + search + validasi inline + banner restart; cron: preset/builder/preview/target-picker
- [ ] Error-map Indonesia tanpa istilah HTTP/flag CLI; empty-state ber-CTA; hapus konfirmasi sebut nama
- [ ] Mobile 320 + desktop 1440; keyboard + screen-reader; `external == []`; kontras AA

## 6. Goals

- **G1** — 8 PRD tertulis + index update, tiap PRD satu PR reviewable.
- **G2** — `/ui/v2` jalan berdampingan `/ui` tanpa regresi v1.
- **G3** — Semua PRD lolos gate `frontend-ui-engineering` + `lean-verification`.

## 7. Non-goals

- Rewrite React/Vite/Tailwind; Electron `desktop/`; PWA/mobile app; ubah SSE/engine; ubah kontrak API existing.
- Flip default ke v2 (dilakukan PR terpisah setelah parity).

## 8. Acceptance criteria

- **AC-41.1** PRD-41 s.d. PRD-48 ada di `docs/prd/` + index `README.md` update, Bahasa Indonesia, format baku (tabel → Ringkasan → Keputusan → Goals → Non-goals → AC → Testing → Referensi).
- **AC-41.2** Tiap PRD 42-48 cantumkan gate `frontend-ui-engineering` yang relevan + tier `lean-engineering-workflow` (Standard; High-rigor untuk auth/upload/config/HITL/purge).
- **AC-41.3** Tanpa kode di PRD-41 (dokumen saja); tanpa dep/build baru.

## 9. Testing

1. Daftar file `docs/prd/PRD-4[1-8]-*.md` lengkap + link index valid.
2. Ejaan/label Indonesia konsisten; nomor AC unik per PRD.

## 10. Referensi

- Struktur v1: `src/pha/gateway/webui/{__init__,markup,style,app_js,spa_js,markdown_js,mermaid_js,i18n_js}.py`
- Backend: `src/pha/gateway/health.py` (~1617 baris), `service.py`, `models.py:84-86`, `config.py`
- Pola PRD: `docs/prd/PRD-7-webui-uiux-overview.md`, `PRD-8-webui-design-system.md`, `PRD-19-webui-worldclass.md`, `PRD-39-ux-ramah-awam.md`
- Skills: `.commandcode/skills/frontend-ui-engineering/SKILL.md`; `.commandcode/plugins/lean-engineering/skills/lean-{engineering-workflow,planning,minimal-implementation,tdd,code-review,verification}/SKILL.md`
- Taste: `.commandcode/taste/{frontend,architecture,workflow}/taste.md`
