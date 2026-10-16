# PRD-38 — Lampiran Multi-format Stdlib (Teks Di-inject, Tanpa Dep Baru)

| | |
|---|---|
| Feature | File teks umum (`txt/md/csv/json/log`) isi di-inject ke prompt (bukan cuma preview 900 char); `pdf/gambar` tetap jalur existing; `docx/xlsx/pptx` ditolak dengan pesan jelas — stdlib saja |
| Depends on | PRD-25 (image native-first), PRD-37 (batas body/counter) |
| Files | `pha/gateway/{health,service,session_pool}.py`, `pha/gateway/webui/{app_js,markup,i18n_js}.py`, `tests/test_pha/test_webui_api.py`, `docs/prd/README.md` |
| Status | Implemented (2026-10-14) |

## 1. Ringkasan

Upload hari ini hanya allowlist `_UPLOAD_EXTENSIONS={png,jpg,jpeg,gif,webp,pdf,txt,md,csv,json}` (`health.py:68-70`) — belum ada `.log`. Lebih penting: isi file teks **tidak benar-benar dibaca model**. `_summarize_attachment` (`session_pool.py:759-781`) cuma tempel `text preview:` ≤900 karakter satu-baris (`_TEXT_PREVIEW_CHARS=900`, `_decode_text_preview` `session_pool.py:784-801` menormalkan whitespace sehingga struktur markdown/kode hancur). `_split_attachment_note` (`service.py:263-288`) hanya memuat `mime image/*` sebagai data-URL; file teks cuma numpang di note. User awam mengira "file terkirim = AI membacanya" — pengalaman menipu. Keputusan: naikkan injeksi teks stdlib-only dengan batas truncate eksplisit, tanpa `pypdf`/`python-docx` (sesuai arahan user + hindari risiko build `pha.spec`).

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Format didukung penuh (isi di-inject) | `txt/md/csv/json/log` (+ existing `pdf`? tidak — pdf tetap note, lihat baris bawah). `.log` baru masuk allowlist |
| Cara baca | UTF-8, fallback latin-1, `errors="replace"`; struktur baris dipertahankan (jangan normalisasi satu-baris) |
| Batas truncate | Per-file 50KB, total per-pesan 200KB; tiap potongan diakhiri penanda `…[dipotong, sisa N KB — minta bagian lanjutan bila perlu]` |
| Deteksi biner | Tolak NUL-byte di 8KB pertama → `400` "file tampak biner, bukan teks" (ramah, 3 bahasa) |
| Blok injeksi | Format `<attachment name="..." size="...">\n...\n</attachment>` sebelum prompt user; nama di-escape agar tak merusak format |
| `pdf` | Tetap jalur existing (note + baca via tool bila model butuh); tanpa ekstraksi teks pdf (butuh dep baru = non-goal) |
| `gambar` | Tetap PRD-25 (`ImageBlock` native + 1x fallback `image_to_text`) |
| `docx/xlsx/pptx/dll` | Tetap ditolak `400`, tapi pesan ramah 3 bahasa: "format .docx belum didukung — simpan sebagai .txt/.md/.csv lalu lampirkan ulang" |
| Batas upload | Tetap 10MB (`health.py:64`, `app_js.py:2216`); tanpa perubahan |
| Keamanan | Sanitasi nama + dedup existing (`health.py:1471-1492`); path traversal guard existing (`service.py:249-260`); NUL-check baru |
| Klien | `file-input` tambah `accept` longgar (biar file-picker OS memfilter, tapi validasi akhir tetap server); chip tampilkan nama + ukuran + hapus; preview N baris pertama untuk non-gambar; progress upload (spinner di chip); error server ditampilkan apa adanya (sudah ramah dari server) |
| Tanpa knob baru | Tanpa field `gateway.json` baru; konstanta truncate di kode (ikuti PRD-36) |

## 3. Call-site

| # | Lokasi | Guna | Fallback |
|---|---|---|---|
| 1 | `health.py:68-70` | Allowlist + `.log` | Tolak bila ekstensi asing (existing) |
| 2 | `health.py:1404-1428` | Pesan `400` ramah (tipe tak diizinkan, biner, kebesaran) | Kode error mentah bila handler mati |
| 3 | `session_pool.py:728-751` `_build_attachment_notes` | Bangun blok `<attachment>` teks + note existing untuk pdf/gambar/audio/video | Preview 900 char lama bila baca gagal |
| 4 | `session_pool.py:759-801` `_summarize/_decode` | Baca teks multi-KB + NUL-check; preview lama dipertahankan untuk audio/video | `summary: ... unavailable` (existing) |
| 5 | `service.py:211,219` `_ATTACH_NOTE/_ATTACH_LINE` | Regex diperluas tahan blok baru; WebUI render history tetap (appendHistoryEntry) | Teks mentah bila parse gagal (existing: nothing hidden) |
| 6 | `app_js.py:2183-2262` chips/upload/drop/paste | Ukuran + preview teks + progress + `accept` | Chip lama bila metadata tak ada |
| 7 | `i18n_js.py` | String baru 3 bahasa | Indonesia bila kunci hilang |

## 4. Goals

- **G1** — Isi `txt/md/csv/json/log` sampai batas truncate benar-benar terbaca model (struktur baris utuh).
- **G2** — Format tak didukung gagal dengan pesan yang dimengerti non-teknis + solusi konkret.
- **G3** — Tanpa dep/build baru (`pyproject.toml` tak berubah); keamanan existing dipertahankan.

## 5. Non-goals

- OCR, ekstraksi teks `pdf`, parsing `docx/xlsx/pptx` via dep baru; virus-scan.
- Ubah batas 10MB; ubah jalur gambar PRD-25; ubah `src/openharness/**`.
- Preview kaya (syntax highlight) di chip — cukup N baris teks polos.

## 6. Acceptance criteria

- **AC-38.1** Suite + lint hijau; `pyproject.toml` tak berubah; string baru 3 bahasa.
- **AC-38.2** Lampir `txt/md/csv/json/log`: prompt yang sampai ke engine mengandung blok `<attachment>` dengan isi utuh (test assert), struktur baris utuh.
- **AC-38.3** File teks >50KB terpotong di 50KB + penanda potong; 5 file kecil total >200KB terpotong di total + penanda.
- **AC-38.4** File biner rename `.txt` (ada NUL) ditolak dengan pesan biner ramah; `docx/xlsx` ditolak dengan pesan + saran save-as (ID/EN/JA).
- **AC-38.5** Drag-drop + paste + multi-file tetap jalan; chip tampil nama + ukuran; ada status progress saat upload; hapus chip batalkan sebelum kirim.
- **AC-38.6** `pdf` + gambar tidak regresi (note existing, native `ImageBlock`); snapshot lama tetap ke-render (regex toleran).

## 7. Testing

1. `pytest tests/test_pha/test_webui_api.py -q` + fixture baru `txt/md/csv/log` (isi utuh, truncate per-file/total, NUL-check, docx ditolak ramah).
2. `ruff check src/pha` bersih + `node --check` JS terekstrak.
3. Manual: lampir `csv` 100 baris → model sebut isinya; lampir `.docx` → pesan saran; lampir teks 1MB → terpotong + penanda.
