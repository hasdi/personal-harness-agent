# PRD-39 — UX Ramah Awam Menyeluruh (Composer-first, 6 Halaman Konsisten)

| | |
|---|---|
| Feature | Composer + lampiran + empty-state/onboarding + error bahasa manusia, plus sapuan copy konsisten di Setup/Skills/Agents/Tools/Cron — tanpa framework/dep baru |
| Depends on | PRD-7/8/9 (fondasi UI/UX), PRD-12 (i18n 3 bahasa), PRD-36 (pola ramah Setup/Cron), PRD-37 (counter input), PRD-38 (pesan lampiran) |
| Files | `pha/gateway/webui/{markup,style,app_js,spa_js,i18n_js}.py`, `pha/gateway/service.py` (meta label SETUP bila perlu, tanpa field baru), `tests/test_pha/test_webui_spa.py`, `tests/test_pha/test_webui_crud.py`, `docs/prd/README.md` |
| Status | Implemented (2026-10-14) |

## 1. Ringkasan

Fondasi ramah-awam sudah ada: onboarding 3 langkah (`welcome_1/2/3`, `i18n_js.py:56-59`), placeholder composer (`prompt_ph`), pola Setup Basic + Cron preset (PRD-36), toast + modal konfirmasi. Yang kurang: (1) composer belum memandu lampiran file (user tak tahu bisa drag-drop/paste, format apa saja); (2) error masih ada yang mentah/teknis (`res.error`, `413 request body too large`, `file type not allowed` — `health.py:1426`, `app_js.py:2233`); (3) copy 6 halaman (Chat/Setup/Skills/Agents/Tools/Cron) belum diaudit konsisten — pelajaran PRD-7 (`DIR ⌕`, `ADD TO ALLOWLIST & USE`, `CLEAR TRAIL`) jangan terulang. PRD ini composer-first, lalu sapuan ringan per halaman. Tanpa wizard berlangkah, tanpa redesign token, tanpa endpoint baru.

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Composer | Placeholder sebut lampir juga (`/` perintah · `@` file · 📎 lampir/data); hint satu-baris di bawah composer saat kosong (bukan tooltip tersembunyi); tombol `Lampirkan` tetap teks (jangan ikon saja) |
| Lampiran UX | Status terlihat: chip nama + ukuran (KB/MB) + hapus; progress saat upload; pesan sukses/gagal spesifik (nama file disebut); preview N baris untuk teks (PRD-38) |
| Error-map Indonesia | Tabel pesan teknis → manusia (tanpa `--flag`, tanpa istilah HTTP): `413` → "Pesan terlalu besar — pecah jadi beberapa pesan atau lampirkan sebagai file"; `file type not allowed` → dari PRD-38; `upload unavailable` → "Fitur lampiran belum aktif — restart layanan"; `working directory not allowed` → "Folder ini di luar daftar izin — pilih via Pilih folder"; `run already in progress` → "Masih menjawab — tunggu atau tekan Stop" |
| Empty-state per halaman | Tiap halaman sepi (Skills/Agents/Tools/Cron/sesi kosong/hasil cari kosong) tampil: kalimat + 1 tombol aksi ("Buat cron pertama", "Tambah skill", bukan "kosong") |
| Onboarding | Reuse `welcome-card/fresh-card` (`markup.py:79-94`); tambah langkah lampir file (langkah ke-4 singkat) + sebut batas 10MB dengan bahasa santai |
| Sapuan copy | Audit 6 halaman: ganti jargon/singkatan kapital tanpa arti; samakan pola tombol (Buat/Ubah/Hapus/Muat ulang), pola toast (sukses hijau spesifik, gagal merah + solusi), pola hapus (modal konfirmasi sebut nama item) |
| Aksesibilitas | `aria-live` toast + status upload; fokus kembali ke composer setelah kirim; kontras/target sentuh ikut token PRD-8 (tanpa token baru); mobile reuse PRD-21/22 |
| Tanpa knob baru | Tanpa field `gateway.json` baru; tanpa endpoint baru; pesan error ramah dibuat di server (sumber tunggal) + klien hanya tampilkan |

## 3. Call-site

| # | Lokasi | Guna | Fallback |
|---|---|---|---|
| 1 | `markup.py:97-113` composer | Hint lampir + struktur status upload | Placeholder lama bila string hilang |
| 2 | `app_js.py` composer/upload/toast | Error-map + status/progress + fokus balik | Error mentah (existing) |
| 3 | `spa_js.py` 5 halaman non-chat | Empty-state + toast/konfirmasi konsisten | Teks `…` lama |
| 4 | `service.py` meta SETUP | Label/help ramah bila ada yang masih teknis; tanpa field baru | Label lama |
| 5 | `i18n_js.py` | Semua string baru 3 bahasa (id/en/ja) | Indonesia bila kunci hilang |
| 6 | `style.py` | Reuse token PRD-8/19; gaya hint/chip ringan | Tanpa gaya baru bila bentrok |

## 4. Goals

- **G1** — User awam kirim pesan + lampir file + paham error tanpa bertanya ke teknisi (happy-path tanpa bantuan).
- **G2** — 6 halaman konsisten: empty-state beraksi, toast seragam, hapus selalu konfirmasi sebut nama.
- **G3** — Tanpa dep/build/endpoint baru; `escHtml` semua dinamis; 3 bahasa; smoke HTML hijau, `external == []`.

## 5. Non-goals

- Wizard setup berlangkah; grid visual; redesign token visual; dark-mode baru; PWA/mobile app.
- Tur produk interaktif (coach-mark); cukup hint + onboarding statis yang jelas.
- Perubahan alur data (SSE, trace, HITL) — hanya presentasi.

## 6. Acceptance criteria

- **AC-39.1** Suite + lint hijau; tanpa dep/build baru; string baru 3 bahasa; smoke HTML `external == []`.
- **AC-39.2** Composer kosong memandu `/`, `@`, lampir (drag-drop/paste disebut); kirim fokus balik ke composer.
- **AC-39.3** Upload: ada progress, chip nama + ukuran + hapus, sukses sebut nama file, gagal sebut sebab + solusi.
- **AC-39.4** Tiap error di error-map tampil bahasa manusia (ID/EN/JA), tanpa kata `413/400/--flag/mime`.
- **AC-39.5** 6 halaman: kondisi kosong & cari-tak-cocok ada kalimat + tombol aksi; hapus selalu modal sebut nama item; toast sukses/gagal konsisten.
- **AC-39.6** Onboarding sebut lampir file + batas 10MB santai; mobile 360px composer + chip tidak overflow; toast terbaca screen-reader (`aria-live`).

## 7. Testing

1. `pytest tests/test_pha/test_webui_spa.py tests/test_pha/test_webui_crud.py tests/test_pha/test_webui_api.py -q`.
2. `ruff check src/pha` bersih + `node --check` JS terekstrak (`APP_JS`, `SPA_JS`, `I18N_JS`).
3. Manual sebagai user awam: kirim pesan, lampir txt + docx (sukses + gagal ramah), buat cron pertama dari empty-state, hapus skill (modal sebut nama), cari tak-cocok di tiap halaman, uji di 360px + screen-reader.
