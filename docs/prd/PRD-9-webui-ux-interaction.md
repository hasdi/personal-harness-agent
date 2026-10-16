# PRD-9 — WebUI: UX, IA, Copy & Aksesibilitas

| | |
|---|---|
| Feature | Perbaikan alur, penamaan, empty state, feedback, aksesibilitas |
| Depends on | PRD-8 |
| Files | `pha/src/pha/gateway/webui/{markup.py, app_js.py, style.py}` |

## 1. Informasi & navigasi

- Navbar: tetap `PHA / personal harness`; badge (`ONLINE`, model, provider, channel) → pil tenang 13px, ikon status titik + teks (bukan warna saja).
- Greeting: teks netral `Halo, <nama>.` — tanpa emoji berlebih/rotasi (a11y: teks tetap terbaca tanpa warna).
- Tombol chat header: label jelas + `title`. `DIR ⌕` → `Pilih folder`; `TRACE`/`AUDIT` → `Trace`/`Audit` (kapital awal, bukan all-caps).

## 2. Sidebar Sessions

- Item: judul 1 baris + meta 13px `--ink-2`: waktu relatif ("2 jam lalu", helper `relTime()` lokal) + jumlah pesan + token.
- Aksi `RENAME`/`DELETE` hanya tampil saat hover/fokus item (kurangi noise); tetap keyboard-reachable.
- Tambah kotak **cari sesi** (filter client-side pada `state.sessions`) + pengelompokan "Hari ini / 7 hari / Lebih lama".
- Empty state: "Belum ada sesi. Mulai chat baru →" dengan tombol aksi, bukan "belum ada sesi tersimpan".

## 3. Chat & composer

- Status run: teks tenang "Berpikir…", "Menulis…", "Menjalankan tool…", "Menunggu persetujuan…" (huruf normal, bukan KAPITAL); waktu berjalan `12s` di kanan.
- Bubble: pemisah jelas user/asisten + label peran kecil (13px `--ink-2`).
- Tombol scroll-ke-bawah jadi pil "Ke bawah ↓" dengan `aria-label`.
- Composer: placeholder "Tulis pesan… `/` perintah · `@` sebut file/folder" (detail: PRD-11); tombol kirim/stop jelas teks (`Kirim` / `Stop`); lampiran punya `title`.
- **Error tidak lagi disuntik ke log chat** sebagai bubble merah untuk kegagalan UI (upload gagal, load gagal) → pakai toast (§5). Error *run* tetap bubble di posisinya.

## 4. First-run & empty state

- Saat belum ada sesi/`state.sessions` kosong dan belum ada pesan: tampilkan **kartu sambutan** di `#log`:
  1. "Atur model & API key" → tombol buka panel SETUP (tab config).
  2. "Pilih folder kerja" → tombol buka dir-picker.
  3. "Mulai chat" → fokus composer.
- Kartu hilang setelah pesan pertama. Tanpa dependency, HTML statis di `markup.py` + toggle `hidden` di `app_js.py`.

## 5. Umpan balik (toast)

- Komponen `#toast` (satu kontainer `role="status"` `aria-live="polite"`, tumpuk maks 3, auto-dismiss 4s).
- Dipakai untuk: simpan/restart config berhasil-gagal, upload gagal, muat sesi gagal, purge selesai.
- Alert banner config dipertahankan di panel (sticky), toast untuk ringkasan.

## 6. Copy audit (contoh; bahasa manusia)

| Lama | Baru |
|---|---|
| `+ NEW` | `Chat baru` |
| `DIR ⌕` | `Pilih folder` |
| `⟳ REFRESH` | `Muat ulang` |
| `CLEAR TRAIL` | `Hapus trail` |
| `ADD TO ALLOWLIST & USE` | `Izinkan & pakai folder ini` |
| `SESSION` / `ALL` | `Sesi ini` / `Semua` |
| `PERMISSION REQUIRED` | `Izin diperlukan` |
| `THINKING…` | `Berpikir…` |
| `nb-greeting` "Halo… Aku … 👋" | "Halo, <nama>." |

Aturan: kapital hanya di awal kalimat; hindari singkatan tanpa arti; setiap tombol ikon wajib `title` + `aria-label`.

## 7. Aksesibilitas

- Landmark: `<header>`, `<aside>`, `<main>`; `#log` `role="log"` `aria-live="polite"`.
- Ikon dekoratif `aria-hidden="true"`; tombol ikon punya `aria-label`.
- Target sentuh ≥40×40px; jarak antar kontrol ≥8px.
- Fokus trap di modal/drawer + `Esc` menutup; fokus kembali ke pemicu.
- Status tidak hanya warna (tambah teks/ikon), kontras ≥4.5:1.

## 8. Acceptance criteria

- **AC-9.1** Tiap tombol ikon punya `title`/`aria-label`; tak ada aksi tanpa label teks terlihat.
- **AC-9.2** Empty state & kartu sambutan tampil saat belum ada sesi/pesan; hilang setelah pesan pertama.
- **AC-9.3** Kegagalan UI muncul sebagai toast, bukan bubble error di log.
- **AC-9.4** Aksi sidebar hanya muncul saat hover/fokus, tetap bisa diakses keyboard.
- **AC-9.5** `#log` `aria-live`; modal/drawer mengembalikan fokus; `Esc` menutup.
- **AC-9.6** Semua string dinamis lolos `escHtml`; smoke test tetap hijau.

## 9. Testing

1. Perluas smoke test HTML: id baru (`#toast`, `#chat-search`, `#welcome-card`) + assertion `aria-live`/`aria-label` pada tombol ikon kunci.
2. Manual: keyboard-only (Tab/Enter/Esc), screen-reader spot-check `#log`, lebar <860px.
