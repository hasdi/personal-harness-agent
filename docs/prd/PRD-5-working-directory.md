# PRD-5 — WebUI: Working Directory Selector (wcd)

| | |
|---|---|
| Feature | Memilih **working directory (wcd)** proyek langsung dari WebUI, per chat session |
| Scope | Core: allowlist config, API `cwd` per-request, selector UI, aturan pergantian cwd |
| Status | Draft v1.0 (2026-08-29) |
| Depends on | PRD-2 (API pattern), PRD-3 (WebUI panel/selector pattern) |

## 1. Ringkasan

User dapat memilih **direktori kerja proyek** yang menjadi `cwd` agent untuk sebuah chat
session — langsung dari WebUI, tanpa restart gateway. Selector menampilkan daftar direktori
yang diizinkan (dari `gateway.json` → `allowed_working_dirs`); setiap request chat membawa
`cwd` terpilih; server memvalidasi terhadap allowlist lalu meneruskannya ke runtime pool
yang **sudah mendukung cwd per-session** sejak awal.

## 2. Kondisi saat ini (bukti kode)

| Fakta | Lokasi |
|---|---|
| Pool **sudah** menerima `cwd` per-request & me-recreate bundle saat cwd berubah | `gateway/runtime.py` `get_bundle(session_key, latest_user_prompt, cwd=None)` — `session_cwd = Path(cwd or self._cwd)`; cache-hit dengan cwd berbeda → close + recreate (log "recreating session for cwd change") |
| `stream_message` belum meneruskan cwd (selalu `self._cwd`) | `gateway/runtime.py` `stream_message` |
| cwd tersimpan di snapshot (per session) | `session_storage.py` payload `"cwd"` |
| Gateway cwd ditentukan saat start (`--cwd` / default CWD) | `service.py` `resolve_gateway_cwd` |
| WebUI belum punya konsep cwd | `webui/app_js.py` |

Artinya: upaya implementasi kecil; risiko utama adalah **keamanan path** dan **UX pergantian
cwd** — keduanya dispesifikasikan di bawah.

## 3. Terminologi

| Istilah | Definisi |
|---|---|
| **wcd** | Working directory aktif untuk sebuah session (tempat agent membaca/menulis file). |
| **Default wcd** | `gateway.json` cwd gateway saat start (`self._cwd`) — selalu tersedia & diizinkan. |
| **Allowlist** | `allowed_working_dirs` — daftar absolut path yang boleh dipilih user. Kosong = hanya default wcd. |

## 4. Konfigurasi (`gateway.json`)

```json
{ "allowed_working_dirs": ["D:\\Projek\\A", "D:\\Projek\\B"] }
```

- Field baru `allowed_working_dirs: list[str]` di `GatewayConfig` (default `[]`).
- Dinormalisasi saat validasi request: `Path(x).resolve()`; perbandingan pakai
  `os.path.normcase` (Windows case-insensitive, Linux case-sensitive — konsisten platform).
- Dapat diedit lewat halaman **SETUP** yang sudah ada (textarea JSON, sudah generic).
- `pha config --full` otomatis menyertakan field baru (pydantic dump).

## 5. UX (WebUI)

- **Selector** `<select id="wcd-select">` di **chat header** (sebelah kiri judul, sebelum
  tombol TRACE/AUDIT): opsi = `Default (<basename gateway cwd>)` + satu opsi per allowlist
  entry (label = basename, `title` = path lengkap).
- State baru `state.wcd` (path string terpilih; `""` = default).
- **Aturan pergantian cwd** (diputuskan sekali, dijalankan konsisten):
  1. Ganti selector saat session kosong (NEW CHAT) → langsung dipakai untuk prompt berikutnya.
  2. Ganti selector saat session **berisi pesan** → modal konfirmasi
     *"Ganti working directory memulai chat baru dengan konteks baru."* → OK = `newChat()`
     (sessionKey baru) + `state.wcd` baru. Context lama tidak ikut ke proyek baru.
  3. Membuka session lama dari sidebar → selector otomatis mengikuti `cwd` session tersebut
     (response history mengembalikan `cwd`); jika cwd-nya tidak ada di allowlist, opsi
     tetap ditampilkan (tanda `•`) dan **server tetap menerimanya** untuk session itu —
     aturan "cwd sebelumnya milik session ini" mencegah session lama mati setelah
     allowlist diedit.
- Restriksi tampilan: selector disabled saat `state.streaming`.

## 6. API contract

### 6.1 `GET /v1/workdirs` (auth, provider pattern)

```json
{"current": "D:\\Projek\\A", "dirs": ["D:\\Projek\\A", "D:\\Projek\\B"]}
```

- `current` = gateway cwd (default wcd). `dirs` = allowlist ternormalisasi yang **exists**
  sebagai directory (entry tak valid di-skip, bukan error).

### 6.2 Chat request membawa `cwd` (opsional)

- `POST /v1/chat/stream` dan `/v1/chat/completions`: field `cwd` opsional di body.
- Validasi server (`_resolve_request_cwd`): kosong/tidak ada → default; harus sama
  (normcase) dengan `current` atau salah satu allowlist, **atau** sama dengan cwd
  snapshot session tersebut (kelanjutan session); selain itu → `400 {"error":
  "working directory not allowed"}`.
- `StreamHandler` signature berevolusi: dipanggil `(prompt, session_key, attachments, cwd)`
  dengan fallback TypeError ke 3-arg lama (pola yang sudah ada di `_handle_chat_stream`).
- `handle_api_prompt_stream(prompt, session_key, media=None, cwd=None)` →
  `stream_message(message, keyed, cwd=cwd)` → `get_bundle(..., cwd=cwd)`.

### 6.3 History mengembalikan cwd session

- `GET /v1/sessions/{id}` tambah `"cwd": "<snapshot cwd>"` (sudah tersimpan di snapshot).

## 7. Server flow (perubahan kecil)

1. `runtime.py`: `stream_message(message, session_key, cwd=None)` → `get_bundle(..., cwd=cwd)`
   — sisanya (recreate-on-change) **sudah berjalan**; tidak diubah.
2. **Restore guard (baru, penting)**: di `get_bundle` creation path, history snapshot hanya
   di-restore bila `os.path.normcase(snapshot["cwd"]) == session_cwd`; jika berbeda → session
   fresh di cwd baru (mencegah konteks proyek lama bocor ke proyek baru). Snapshot id/session
   tetap mengikuti session_key.
3. `session_id`/file snapshot tidak berubah — satu session_key = satu riwayat, cwd bagian
   dari metadata.
4. Trace: `run.started` data tambah `"cwd": session_cwd` (muncul otomatis di tab Trace).

## 8. Keamanan

- **Tidak ada path arbitrer dari client** — hanya `current` atau anggota allowlist
  (perbandingan normcase); pelanggaran → 400.
- Endpoint tetap di belakang bearer token; SETUP page (editor allowlist) juga authed.
- Allowlist kosong secara efektif mematikan fitur (hanya default) — operasi aman default-on.
- Attachments tetap disimpan di workspace (bukan di cwd proyek) — tidak berubah.

## 9. Acceptance criteria

- **AC-5.1** `GET /v1/workdirs` mengembalikan `current` + allowlist ternormalisasi; 503 bila provider belum disuntik; 401 tanpa token.
- **AC-5.2** Chat dengan `cwd` valid diteruskan ke pool; `get_bundle` menerima cwd tersebut (bundle dibuat di cwd itu — terverifikasi via `snapshot["cwd"]`).
- **AC-5.3** Chat dengan `cwd` di luar allowlist → 400, tanpa eksekusi agent.
- **AC-5.4** `cwd` kosong/absen = default (backward compatible; request lama tanpa `cwd` tetap jalan).
- **AC-5.5** Restore guard: session_key sama tapi cwd berbeda → conversation fresh (tanpa pesan lama), snapshot baru ber-cwd baru.
- **AC-5.6** UI: selector tampil di chat header; ganti cwd pada session berisi pesan → modal + `newChat()`; buka session lama → selector mengikuti `cwd` session.
- **AC-5.7** Trace run mencatat `cwd` (terlihat di tab Trace).
- **AC-5.8** Smoke test HTML diperbarui (`wcd-select` + `/v1/workdirs`); ruff + pytest hijau.

## 10. Testing

1. API: workdirs happy/503/401; chat stream dengan cwd valid/invalid/kosong (fake stream handler menangkap cwd); completions idem.
2. Runtime: `get_bundle(cwd=...)` recreate path — stub engine, assert bundle baru dibuat dengan cwd baru (existing log path); restore guard unit test (snapshot cwd ≠ request cwd → restore_messages kosong).
3. History: `load_webui_session` mengembalikan `cwd` snapshot.
4. UI smoke: id `wcd-select` + path `/v1/workdirs`.
5. Manual QA: ganti cwd → chat baru → `pwd`-style tool menunjukkan cwd baru; buka session lama proyek lain → selector mengikuti.

## 11. Rollout

| Milestone | Isi |
|---|---|
| **M1** | Config field + `GET /v1/workdirs` + validasi cwd di chat endpoints + provider wiring |
| **M2** | Runtime: `stream_message(cwd)` + restore guard + trace `cwd` |
| **M3** | WebUI: selector + modal + sync dari history + smoke test |
| **M4** | README + manual QA |

Semua milestone kecil; total perubahan < ±200 baris efektif.

## 12. Out of scope / Future

- Browse/tambah folder dari UI (harus tetap lewat allowlist di SETUP — keamanan).
- Grouping sidebar per proyek; badge proyek per session.
- `--workdir` wizard di `pha config` interaktif.
- Multi-window WebUI dengan session paralel per proyek.
