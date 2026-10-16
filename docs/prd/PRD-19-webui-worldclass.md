# PRD-19 — WebUI World-Class: Claude×Notion + Dark/Light

| | |
|---|---|
| Feature | Tema world-class WebUI: token Claude×Notion + dark/light + toggle |
| Depends on | PRD-7, PRD-8 |
| Files | `webui/style.py`, `webui/markup.py`, `webui/app_js.py`, `webui/i18n_js.py`, `webui/__init__.py`, `tests/test_pha/test_webui_api.py` |
| Status | Draft v1.0 (2026-09-12) |

## 1. Ringkasan

WebUI light-only (`:root{color-scheme:light}`, `style.py:12`). PRD ini menambah dark mode + toggle + token Claude×Notion. Nol dep, nol build, nol font eksternal (ponytail full).

## 2. Keputusan desain

| Aspek | Keputusan |
|---|---|
| Acuan | Claude (cream hangat + coral + dark hangat) × Notion (radius 8/12, hairline 1px, input 44px, body 16px/1.55) |
| Aksen tunggal | Coral gelap `#a9583e` (light) / `#e08a6d` (dark) — digeser dari `#cc785c` agar kontras AA dua arah |
| Light | `--bg:#faf9f5`, `--surface:#fff`, `--ink:#141413`, `--ink-2:#57534e`, `--line:#e6dfd8`, `--surface-2:#f5f0e8` |
| Dark | `--bg:#181715`, `--surface:#252320`, `--surface-2:#1f1e1b`, `--ink:#faf9f5`, `--ink-2:#a09d96`, `--line:#3a3733` |
| Radius | `--radius:12px` (kartu), `--radius-sm:8px` (tombol/input) |
| Perilaku tema | Toggle manual `#theme-toggle` + `localStorage pha-theme` + auto OS (`prefers-color-scheme`) saat kosong + anti-flash inline di `<head>` |
| Stabilitas | Nama kelas `.nb-*` dan semua `id` existing tidak berubah; alias `--nb-*` tetap |

## 3. Goals

- **G1** — Toggle dark/light jelas di navbar, pilihan bertahan reload, auto OS saat pertama kali.
- **G2** — Kontras teks utama ≥4.5:1 di kedua tema; meta ≥13px; sentuh ≥40px.
- **G3** — Smoke test + `ruff` hijau; `external == []`.

## 4. Non-goals

- Framework CSS, font/CDN eksternal, perubahan endpoint API, engine `openharness/**`, CLI/TUI.

## 5. Acceptance criteria

- **AC-19.1** Klik toggle 2× + reload → tema bertahan; hapus `pha-theme` + OS dark → dark otomatis.
- **AC-19.2** `color-scheme` ikut tema; `:focus-visible` jelas di dark; `prefers-reduced-motion` tetap.
- **AC-19.3** `nb-card`, `nb-btn--yellow` tetap ada; tidak ada `font-size` <13px baru; tombol kirim tetap satu aksen.

## 6. Testing

1. `pytest tests/test_pha/test_webui_api.py -q` (smoke termasuk `theme-toggle`).
2. `ruff check src/pha/gateway/webui/`.
3. Manual: `/ui` toggle + reload, OS dark/light, 360px & 1280px (chat/composer/setup/skills/cron).

## 7. Out of scope

- PWA, animasi baru, redesign alur SSE/trace/HITL.

## 8. Referensi

- `style.py:11-58` token kini; `markup.py:16-24` navbar; `app_js.py:2139-2160` init; `i18n_js.py:526-547` `applyLanguage`.
- DESIGN.md: Claude (cream `#faf9f5`, coral `#cc785c`, dark `#181715`) + Notion (radius 8/12, input 44px) via awesome-design-md.
