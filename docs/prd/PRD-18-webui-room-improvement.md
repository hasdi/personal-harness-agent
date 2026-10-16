# PRD-18 — WebUI: Room Improvement (Navigasi, Feedback, Chat)

| | |
|---|---|
| Feature | Kumpulan improvement murah: hash routing, feedback spesifik, polish chat/composer |
| Depends on | PRD-8, PRD-9, PRD-12, PRD-13 |
| Files | `pha/src/pha/gateway/webui/{spa_js,app_js,markup,i18n_js,style}.py` |
| Status | Draft v1.0 (2026-09-12) |

## 1. Navigasi (tanpa router lib)

- Hash routing `#chat/#setup/#skills/#cron` + deep-link item (`#skills/<name>`, `#cron/<name>`): `switchView` tulis `location.hash`, `hashchange` → `switchView` + buka detail. Refresh tidak lagi selalu kembali ke Chat.
- Catatan: PRD-13 v1.1 menolak hash router karena bug init `var` hoisting — pakai pola aman: `spaInit()` tetap di akhir `spa_js.py`, guard `typeof VIEWS`, listener pasif.

## 2. Feedback & loading

- Toast error spesifik per aksi cron/skills (ganti `cron_load_failed` generik): pesan server (`{error}`) diteruskan ke `toast(..., "err")`.
- Loading state per tombol (spinner kecil + disabled), bukan full `renderCronView`/`renderSkillsView` kedip — update row/card saja setelah aksi.
- `applyLanguage()` refetch data view aktif, bukan render cache lama.

## 3. Chat/composer polish

- Composer: hint `/` + `@` tetap (PRD-11); tambah counter karakter? skip — ponytail: hanya bila diminta.
- Sidebar: search + grouping existing dipertahankan; tambah badge "● berjalan" sudah ada — tambah auto-refresh ringan saat run selesai (existing `adoptCurrentSession` cukup, tanpa poll baru).
- Notifikasi cron: batas 20/session dipertahankan; tambah "tandai dibaca" implisit saat buka session (existing `load_webui_session` kosongkan pending — cukup).

## 4. Acceptance criteria

- **AC-18.1** Reload di `#skills/<name>` → kembali ke skill itu (bukan Chat kosong).
- **AC-18.2** Error aksi menampilkan pesan server yang spesifik.
- **AC-18.3** Ganti bahasa me-refresh data view aktif.
- **AC-18.4** Tanpa URL eksternal baru; `node --check` + smoke hijau.

## 5. Non-goals

- Framework router, PWA/manifest (tetap Future PRD-13 §Future), dark mode default.
