# PRD-20 — WebUI Production-Grade: Datetime, Skill Split, Polish

| | |
|---|---|
| Feature | Greeting + datetime live, skill preview kanan resizable, polish production-grade |
| Depends on | PRD-19 |
| Files | `webui/{markup,style,app_js,spa_js,i18n_js,__init__}.py`, `tests/test_pha/test_webui_api.py` |
| Status | Draft v1.0 (2026-09-12) |

## 1. Ringkasan

Bawa WebUI ke production-grade: font Inter/JetBrains Mono via Google Fonts CDN
(system fallback bila offline), datetime live di bawah greeting, preview skill
sebagai panel kanan resizable, skeleton loading, char counter composer, polish
kartu/navbar/composer.

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| CDN | Google Fonts (Inter 400-700 + JetBrains Mono) + Mermaid jsDelivr tetap lazy; tanpa framework JS |
| Datetime | `#greet-date` di bawah greeting, `toLocaleString` ikut bahasa UI, refresh 30 dtk |
| Skill split | `#skills-wrap` flex: list kiri + `#skill-detail` kanan (280–640px, default 420px), drag via pointer + keyboard ←/→, `localStorage pha-skill-w`, collapse <1100px |
| Loading | `.skel` shimmer untuk skills/cron/detail; nonaktif saat `prefers-reduced-motion` |
| Composer | `maxlength=20000`, counter tampil >2000 char, focus ring + shadow |
| Aksesibilitas | Detail = `role=dialog`, close `Esc` saat fokus di dalam, resize handle keyboard-focusable |

## 3. Goals

- **G1** — Datetime terlihat di bawah "Halo …", format ikut bahasa, update tiap 30 dtk.
- **G2** — Preview skill di kanan, bisa di-resize, lebar bertahan reload.
- **G3** — Smoke test hijau; eksternal hanya fonts + mermaid; kontras tetap AA.

## 4. Non-goals

- Framework CSS/JS, endpoint API baru, engine `openharness/**`, CLI/TUI, PWA.

## 5. Acceptance criteria

- **AC-20.1** Greeting tampil → datetime di bawahnya; ganti bahasa → format tanggal ikut.
- **AC-20.2** Klik skill → panel kanan muncul + handle resize; drag → lebar berubah & bertahan reload; <1100px → detail pindah ke bawah.
- **AC-20.3** Smoke test assert `greet-date/skills-wrap/skills-resize/char-count` + CDN fonts; `external` non-font/mermaid == [].
- **AC-20.4** `ruff` hijau; `node --check` blok JS utama lolos.

## 6. Testing

1. `pytest tests/test_pha/test_webui_api.py -q -k webui_html`.
2. `ruff check src/pha/gateway/webui/`.
3. Manual: toggle tema, resize panel, reload, 360px & 1280px, offline (fallback font).
