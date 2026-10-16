# PRD-42 — WebUI v2: Design System (Token + Komponen + Motion + A11y)

| | |
|---|---|
| Feature | Design system v2: evolusi token Claude×Notion ke standar 2026 + komponen + motion + fondasi a11y |
| Depends on | PRD-41 (overview & prinsip), PRD-8 (token calm-professional), PRD-19 (dark/light Claude×Notion) |
| Files | `src/pha/gateway/webui/v2/app.css` (baru), `src/pha/gateway/webui/v2/index.html` (baru, `<head>` anti-FOUC), `tests/test_pha/test_webui_v2.py` (baru, smoke token) |
| Status | Draft v1.0 (2026-10-15) |

## 1. Ringkasan

PRD-8/19 memberi fondasi tenang + dark/light, tapi v1 masih campur token legacy `--nb-*` dengan token baru, `!important` mobile, `style=` inline tersebar, codeblock selalu dark `#1e1b18`, dan z-index tak konsisten (lightbox 400 vs toast 300 vs drawer 120). PRD-42 membangun `app.css` + `<head>` v2 dari nol sebagai **sumber tunggal token**, evolusi (bukan ganti total) dari nilai PRD-19 agar v1 dan v2 tetap harmonis. Nol JS perilaku di PRD ini — hanya fondasi visual yang dipakai PRD-43 s.d. 47.

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Evolusi, bukan revolusi | Pertahankan cream `--bg:#faf9f5`, coral `--accent:#a9583e` (light) / `#e08a6d` (dark), dark `--bg:#181715`/`--surface:#252320` (PRD-19); tambah `--surface-2/3`, `--ink-3`, `--accent-soft/strong`, `--ok/--warn/--danger` + versi `-soft` agar status tak andalkan warna saja |
| Alias kompatibel | `:root` definisikan token baru + alias `--nb-* → token baru` (satu arah); v2 tidak boleh pakai hex mentah di luar `:root` |
| Spacing 2026 | Skala `0.25rem`: `--sp-1..8` = 4/8/12/16/24/32/48/64; radius `--r-sm:8 --r:12 --r-lg:14 --r-full:999px`; shadow `--shadow-sm/md/lg` lembut; border `1px var(--line)`; transisi `--speed:150ms ease` GPU-only (`transform/opacity`) |
| Tipografi | Font sistem (`system-ui, -apple-system, "Segoe UI", Roboto, Arial`) + mono sistem; base 16px/1.55; skala 13 meta / 14 kecil / 16 body / 18 h3 / 22 h2 / 28 h1; `h1` satu per view; `uppercase` hanya badge/chip ≤2 kata |
| Codeblock dua-tema | Code punya pasangan light (`--code-bg`) + dark, bukan selalu `#1e1b18`; kontras AA dua arah |
| Z-index baku | `--z-base:0 --z-sticky:50 --z-panel:100 --z-drawer:150 --z-modal:200 --z-palette:250 --z-toast:300 --z-lightbox:400`; overlay drawer/modal satu elemen `.scrim`, bukan `::before` rapuh (ganti pola `permission-drawer::before` v1) |
| Tema | Toggle manual + `localStorage pha-theme` + auto OS (`prefers-color-scheme`) saat kosong + anti-flash inline di `<head>` sebelum paint; `color-scheme` ikut tema; `:focus-visible` jelas dua tema; `prefers-reduced-motion` mematikan animasi |
| Tanpa AI-aesthetic | Tanpa ungu/indigo default, tanpa gradasi berlebihan, tanpa `rounded-2xl` massal, tanpa hero generik, tanpa padding raksasa seragam, tanpa shadow tebal |
| Komponen baku | `btn` (primary/ghost/danger, ≥40px), `input/select/textarea` (ring fokus `accent-soft` + border `accent`), `card`, `badge`, `chip`, `tag` (mono 13px), `alert` (latar lembut + ikon + teks, bukan warna pekat), `modal` (radius 14 + `shadow-md`, overlay `rgba(16,24,40,.45)`), `skeleton` (shimmer, bukan spinner), `empty-state` (kalimat + 1 CTA), `toast` (`aria-live`), tabel (wrap normal + scroll-x aman ala PRD-22) |

## 3. Call-site

| # | Lokasi | Guna | Fallback |
|---|---|---|---|
| 1 | `v2/app.css` | Seluruh token + komponen + motion + responsif dasar | — (file baru, tanpa fallback) |
| 2 | `v2/index.html` `<head>` | Anti-FOUC tema + `color-scheme` + font sistem (tanpa CDN font) | Light bila `localStorage` kosong + OS tak terbaca |
| 3 | `tests/test_pha/test_webui_v2.py` | Smoke: token ada, alias `--nb-*` valid, tanpa hex di luar `:root`, tanpa `!important` baru | — |

## 4. Goals

- **G1** — Satu sumber token; alias lama tetap valid; tanpa hex mentah di luar `:root`.
- **G2** — Kontras AA dua tema; meta ≥13px; sentuh ≥40px; `h1` satu per view.
- **G3** — Gate `frontend-ui-engineering` §Design/Adherence lolos sejak fondasi.

## 5. Non-goals

- Perilaku JS, layout shell, chat, view apa pun (PRD-43 s.d. 47). Perubahan token v1. Font/CDN eksternal.

## 6. Acceptance criteria

- **AC-42.1** `pytest tests/test_pha/test_webui_v2.py -q` + `ruff check src/pha` hijau; tanpa dep/build baru.
- **AC-42.2** Semua token semantik dipakai via `var()`; tak ada `font-size` <13px; tak ada `!important` baru; z-index hanya via `--z-*`.
- **AC-42.3** Toggle 2× + reload → tema bertahan; hapus `pha-theme` + OS dark → dark otomatis; tanpa flash terang saat load dark.
- **AC-42.4** `:focus-visible` terlihat di light + dark; `prefers-reduced-motion` mematikan animasi/transisi.
- **AC-42.5** Codeblock terbaca AA di dua tema; alert/toast/badge tak andalkan warna saja (ada ikon/teks).

## 7. Testing

1. `pytest tests/test_pha/test_webui_v2.py -q` (token, alias, larangan hex/`!important`, smoke `<head>`).
2. `ruff check src/pha/gateway/webui/v2` bersih.
3. Manual gate `frontend-ui-engineering`: toggle + reload + OS dark/light, 320/768/1024/1440, Tab keyboard di halaman contoh, cek kontras pasangan token.

## 8. Out of scope / backlog

- Density switch (comfortable/compact); token high-contrast tambahan — backlog bila ada demand.

## 9. Referensi

- `src/pha/gateway/webui/style.py:11-58` (token kini), `markup.py:16-24` (navbar), `app_js.py:2139-2160` (init tema), `i18n_js.py:526-547` (`applyLanguage`)
- `docs/prd/PRD-8-webui-design-system.md`, `PRD-19-webui-worldclass.md`, `PRD-22-rca-mobile-wrap.md`
- `.commandcode/skills/frontend-ui-engineering/SKILL.md` (§Design System Adherence, §Accessibility, §Verification)
