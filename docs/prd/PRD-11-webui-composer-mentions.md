# PRD-11 — WebUI: Composer Palette `/` & `@` Mention

| | |
|---|---|
| Feature | Autocomplete composer: `/` slash command (formalkan) + `@` file/dir mention (baru), pola TUI |
| Depends on | PRD-8, PRD-9 |
| Files | `pha/src/pha/gateway/webui/{app_js.py, markup.py, style.py}`, `pha/src/pha/gateway/{service.py, health.py}` |

## 1. Kondisi sekarang

| Hal | Status | Bukti |
|---|---|---|
| `/` slash palette | **Sudah ada** (belum ditata rapi): hanya aktif bila `value[0] === "/"`, tanpa grouping, tanpa empty-state | `app_js.py:1080-1129`, `#palette` `markup.py:45`, CSS `style.py:392-402` |
| `@` mention | **Belum ada** | tak ada handler `@` di `app_js.py` |
| Sumber file | **Belum ada** — `/v1/dir/list` hanya me-list direktori (`if not child.is_dir(): continue`) | `service.py:942-947`, `health.py:606-613` |
| Dir picker | Ada, tapi lewat tombol `DIR ⌕` (bukan inline di composer) | `markup.py:32`, PRD-5 |

Tujuan: meniru kebiasaan agent-harness TUI — `/` untuk perintah, `@` untuk menyebut file/folder lewat pop-up, langsung dari composer.

## 2. Prinsip (ponytail)

- **Satu komponen popup** dipakai ulang untuk `/` dan `@` (bukan dua markup) — reuse `#palette` + fungsi navigasi existing.
- **Tanpa endpoint baru** jika bisa: perluas `/v1/dir/list` dengan flag `include_files=1` (default off, perilaku lama utuh).
- **Insert path sebagai teks**, bukan upload. Agent sudah punya tool baca file. Attach-otomatis = Future (butuh base64 dari server, endpoint baru).

## 3. API — perluasan `/v1/dir/list`

`GET /v1/dir/list?path=<dir>&include_files=1&q=<filter>&limit=<n>`

- Default (tanpa flag) = perilaku sekarang: hanya direktori.
- `include_files=1`: tambah file di level itu, direktori dulu lalu file A–Z.
- Batas aman (wajib, trust boundary): kontainmen allowlist tetap (`_is_within`, `service.py:911-916`); tolak symlink/junction keluar root; skip `.git`, `node_modules`, `__pycache__`, `.venv`, file tersembunyi; `limit` default 200; tak ada rekursi.
- Respons entry: `{"name", "path", "type": "dir"|"file"}` (entry lama tanpa `type` dianggap `dir` — tolerant).

## 4. Perilaku composer

- **Trigger**: `/` di awal input (existing); `@` di awal input atau setelah spasi. Tidak trigger di dalam ```` ``` ```` fence.
- **Popup**: satu `#palette`, menampilkan daftar + ikon (`/` perintah, `📄` file, `📁` folder) + deskripsi singkat.
- **Filter**: ketik lanjutan menyaring (`@app` → `app.py`, `app/`); kosong → tampilkan cwd saat itu (`state.wcd` atau default gateway).
- **Navigasi**: `↑`/`↓` pindah, `Enter`/`Tab` pilih, `Esc` tutup (fungsi existing `movePalette`/`choosePalette`/`hidePalette`).
- **Pilih folder** (`@src/`) → popup tetap terbuka, masuk ke subfolder (path completion); **pilih file** → sisipkan path relatif terhadap wcd ke prompt + spasi, tutup popup.
- **Pilih perintah** (`/`): perilaku existing dipertahankan (isi `nama + " "`).
- **Empty state**: "Tidak ada hasil" (bukan popup hilang tanpa penjelasan).
- **A11y**: `#palette` `role="listbox"`, item `role="option"` + `aria-selected`; navigasi tetap bisa via keyboard.

## 5. Copy

- Placeholder composer: `Tulis pesan…  "/" perintah · "@" sebut file/folder`.
- Judul kecil di popup: `Perintah` / `File & folder`.

## 6. Acceptance criteria

- **AC-11.1** `/` tetap bekerja seperti sekarang; daftar punya empty-state & navigasi keyboard.
- **AC-11.2** Ketik `@` → popup muncul dengan isi folder kerja aktif; `@` di tengah kata biasa tidak memicu.
- **AC-11.3** Pilih file → path relatif tersisip ke prompt; pilih folder → masuk subfolder; `Esc` menutup.
- **AC-11.4** `/v1/dir/list` tanpa `include_files` berperilaku persis seperti sebelum (test `test_dir_picker.py` hijau).
- **AC-11.5** Kontainmen allowlist, skip symlink, dan filter folder berat tetap berlaku untuk `include_files=1`.
- **AC-11.6** Popup pakai `#palette` existing; tak ada URL eksternal; smoke test hijau (tambah `/v1/dir/list` ke daftar path di `test_webui_api.py:1592-1596` bila belum ada).
- **AC-11.7** Semua label dinamis lolos `escHtml`.

## 7. Non-goals

- Auto-attach file sebagai lampiran (butuh base64 server-side — Future).
- Fuzzy search lintas-subfolder/rekursif, indexing, atau ripgrep.
- `@` untuk mention agent/skill (hanya file & folder).
- Perubahan TUI upstream.

## 8. Testing

1. API: perluas `tests/test_pha/test_dir_picker.py` — `include_files=1` mengembalikan file, menghormati limit, menolak path di luar allowlist; tanpa flag = tak berubah.
2. Smoke HTML: `#palette` tetap ada; tambah `/v1/dir/list` ke assertion path.
3. Manual: ketik `@` dan `/`, filter, ↑/↓/Enter/Tab/Esc; uji folder kosong & folder besar (limit); uji `@` setelah spasi vs di tengah kata.
