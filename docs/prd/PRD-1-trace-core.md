# PRD-1 — Core: Trace Capture & Storage (TraceStore)

| | |
|---|---|
| Feature | Event-sourced trace store per chat session + titik instrumentasi di adapter layer |
| Depends on | — (fondasi) |
| Consumers | PRD-2 (API), PRD-3 (WebUI) |

## 1. Ringkasan

Satu modul baru `pha/src/pha/trace.py` (paralel dengan `pha/src/pha/audit.py`):
append-only JSONL **satu file per session** di `~/.pha/logs/traces/`, satu fungsi `emit()`,
validasi lossless-JSON, `seq` monotonik per file, purge per run/per session, dan proyeksi
derived (daftar run + status + pairing invariant). Tidak ada dependensi baru; pola kode
mengikuti `audit.py` yang sudah ada.

## 2. Sumber data & titik instrumentasi

Capture dilakukan **hanya di adapter layer pha** — mirror event yang sudah mengalir, tanpa
mengubah `src/openharness/**`.

| Titik | Lokasi (referensi) | Event yang di-emit |
|---|---|---|
| Boundary run (mulai/akhir eksekusi prompt) | `gateway/service.py` `handle_api_prompt_stream` (+ fallback blocking `handle_api_prompt`) | `run.started`, `run.completed`, `run.error` |
| Tool lifecycle | `gateway/runtime.py:420-429` (start) & `:448-457` (completion) — hari ini sudah memanggil `audit_tool_event`; tambah panggilan `trace` di sampingnya | `tool.started`, `tool.completed` |
| Progress & hint dari engine | mirror event SSE `progress` / `tool_hint` di `_SseWriter` (`gateway/health.py:56-86`) | `progress` |
| Akhir jawaban LLM | event SSE `final` | `llm.response` (panjang teks + usage **bila tersedia**) |
| Retry LLM | belum ada event struktural dari engine (engine hanya StatusEvent teks, `openharness/api/client.py:98-113`) → **core: tidak ada kind khusus**; jika `progress` membawa teks retry, cukup tersimpan sebagai `progress` | — |

Aturan instrumentasi:
- **Jangan log per-delta LLM** (sumber noise utama; satu run bisa ratusan delta). Cukup boundary.
- `tool_input`/`output` TIDAK di-log mentah. Log ringkasan (`summary` ≤ 160 char, mengikuti
  `audit.py`), panjang output, dan `is_error`.
- Emit gagal (disk penuh dsb.) **tidak boleh** menggagalkan chat: try/except + log ke stderr.

## 3. Skema TraceEvent (v1)

Satu baris JSON per event, field tetap + `data` per kind:

```json
{"v":1,"seq":42,"ts":"2026-08-29T03:12:45.123456+00:00","mono":98123456,
 "session_key":"api:3f2c…","run_id":"a1b2c3d4e5f6","kind":"tool.completed",
 "data":{"tool":"bash","summary":"pytest -q","is_error":false,"duration_ms":1420,"output_lines":31}}
```

| Field | Tipe | Aturan |
|---|---|---|
| `v` | int | Skema versi — selalu `1` di core. Reader menolak `v` > 1? **Tidak** — reader tetap render (tolerant reader), hanya tandai `unknown_version`. |
| `seq` | int | Monotonik naik per file, mulai 1. Di-derive saat load bila gap (§6). |
| `ts` | ISO8601 UTC | `datetime.now(timezone.utc)` saat emit. |
| `mono` | int | `time.monotonic_ns()` — untuk durasi presisi dalam satu proses. |
| `session_key` | str | Kunci session pemilik event (join ke `/v1/sessions`). |
| `run_id` | str \| null | `uuid4().hex[:12]`. Null untuk event lintas-run (`trail.purged`, `session.rotated` bila ada). |
| `kind` | str | Salah satu dari tabel kind di bawah. Unknown kind tetap valid (tolerant reader). |
| `data` | object | Payload lossless-JSON saja: string/number/bool/null/list/dict primitif (tanpa BigInt, cycle, objek eksotis — divalidasi di `emit`, invalid → `TypeError`, event dibuang + catat stderr). |

### 3.1 Kind minimum (core)

| Kind | data (field kunci) | Emit dari |
|---|---|---|
| `run.started` | `prompt_summary` ≤ 120 char, `channel`, `model`, `provider` | boundary run |
| `tool.started` | `tool`, `summary` | `runtime.py:420-429` |
| `tool.completed` | `tool`, `summary`, `is_error`, `duration_ms`, `output_lines` | `runtime.py:448-457` |
| `progress` | `message` (teks progress/hint engine) | mirror SSE `progress`/`tool_hint` |
| `llm.response` | `text_len`, `usage` (opsional `{input_tokens,output_tokens}` bila tersedia) | SSE `final` |
| `run.error` | `message`, `recoverable` | error path stream handler |
| `run.completed` | `status` = `ok` \| `error` \| `aborted`, `duration_ms`, `tool_count`, `error_count` | boundary run (finally) |
| `trail.purged` | `scope` = `run` \| `session`, `run_id?`, `purged_events` int | TraceStore.purge |

### 3.2 Pairing invariant & status run (proyeksi derived, tidak disimpan)

Saat load trail (semua event satu session):

1. Kelompokkan per `run_id` (abaikan `run_id` null).
2. `run.started` tanpa `run.completed` → **status `error`** bila ada event `run.error`,
   selain itu **`interrupted`** (crash/kill/restart).
3. `tool.started` tanpa `tool.completed` → tool row dengan **`outcome: "unknown"`**
   (jangan pernah menulis synthetic event ke log — derive-only, berbeda dari dsh yang
   menulis synthetic result saat resume).
4. `run.completed` menang atas semua event lain dalam run tsb untuk status.
5. Run tanpa `run.started` (sisa legacy) → dikecualikan dari daftar run.

## 4. Storage

| Aspek | Keputusan |
|---|---|
| Lokasi | `~/.pha/logs/traces/<sanitized_session_key>.jsonl` — via `get_logs_dir()` (`workspace.py:216`) + subfolder `traces/`. Sanitasi: `session_key.replace(":", "_")` + whitelist `[A-Za-z0-9._-]`, maks 120 char. |
| Append | Satu `emit()` = validasi payload → `json.dumps(..., ensure_ascii=False, separators=(",",":"))` → append + `flush` (pola sama dengan `audit.py`; file dibuka append per emit, tanpa handle long-lived → aman terhadap rotasi manual). |
| Ukuran | Per-delta tidak dilog → ~2–5 KB/run. **Rotasi otomatis = non-core** (PRD-4 §Future); core cukup purge manual. |
| Konkurensi | Gateway single event loop; emit sinkron cepat (append kecil). Tidak perlu lock file. |
| Baris rusak | Reader me-skip baris yang gagal `json.loads`, hitung `corrupted_lines`, tetap lanjut (tolerant reader). |
| Privacy | `sender` TIDAK masuk trace (hanya di audit.jsonl, sudah hashed). `prompt_summary` dipotong; tidak ada credential yang dilog. |

## 5. API modul `trace.py` (kontrak internal)

```python
def emit(session_key: str, run_id: str | None, kind: str, data: dict) -> None  # append; never raises
def read_events(session_key: str, *, run_id: str | None = None,
                kinds: list[str] | None = None, after_seq: int = 0,
                limit: int = 500) -> list[dict]            # oldest→newest, paging by seq
def derive_runs(session_key: str) -> list[dict]            # proyeksi §3.2
def purge_run(session_key: str, run_id: str) -> int        # rewrite atomik, return jumlah event dihapus
def purge_session(session_key: str) -> int                 # unlink file; buat file baru berisi 1 event trail.purged(scope=session)
def trail_meta(session_key: str) -> dict                   # {exists, size_bytes, event_count, corrupted_lines, first_ts, last_seq}
```

Aturan penting:
- **Purge per run = rewrite atomik**: tulis file sementara (event minus run_id tsb) → `os.replace()`.
  Setelah itu append `trail.purged {scope:"run", run_id}`.
- **Purge session**: hapus file, lalu buat file baru yang isinya tepat satu event
  `trail.purged {scope:"session", purged_events:N}` — penghapusan selalu meninggalkan jejak.
- `seq` setelah purge per-run **tidak direset** (lanjut dari max seq tersisa). Setelah purge
  session, seq mulai dari 2 (setelah event `trail.purged`). Didokumentasikan; `seq` bukan
  janji kontinuitas lintas purge.
- Semua fungsi toleran terhadap session yang belum punya file (return empty / no-op).

## 6. Seq repair saat load

Jika baris terakhir file rusak (crash di tengah append): baris di-skip (§4), dan `seq` event
berikutnya = `last_valid_seq + 1` (dari hasil read). Tidak ada penulisan balik ke log.
Sifat dsh yang diadopsi: **log tidak pernah diperbaiki dengan menulis ulang; proyeksi yang
menoleransi**.

## 7. Acceptance criteria

- **AC-1.1** `emit()` menambah tepat 1 baris JSON valid per panggilan; field wajib lengkap; `seq` naik 1 per file.
- **AC-1.2** `emit()` dengan payload non-lossless (mis. `set()`) tidak menulis baris dan tidak melempar exception ke caller.
- **AC-1.3** `read_events` mem-filter `run_id`, `kinds`, `after_seq`, dan menghormati `limit` (default 500).
- **AC-1.4** `derive_runs` mengklasifikasi status `ok|error|aborted|interrupted` sesuai §3.2; tool tanpa result → `outcome:"unknown"`.
- **AC-1.5** `purge_run` menghapus semua event run tsb, menyisakan event run lain utuh (urutan & seq terjaga), lalu menambah `trail.purged scope=run`.
- **AC-1.6** `purge_session` menghapus file dan menghasilkan file baru dengan tepat 1 event `trail.purged scope=session`.
- **AC-1.7** File berisi baris rusak → `read_events`/`derive_runs` tetap bekerja; `trail_meta.corrupted_lines` > 0.
- **AC-1.8** Satu prompt end-to-end via gateway menghasilkan minimal `run.started` + `run.completed` (atau `run.error`) dengan `run_id` sama dan `session_key` benar.
- **AC-1.9** Tidak ada perubahan file di bawah `src/openharness/**`; tidak ada dependensi baru di `pyproject.toml`.

## 8. Testing (unit — `tests/test_pha/test_trace.py`)

Pola mengikuti test existing (pytest + tmp_path; tanpa jaringan):

1. round-trip emit→read; urutan & seq benar.
2. filter run_id/kinds/after_seq/limit.
3. derive_runs: skenario ok / error / aborted / interrupted (file dipotong manual).
4. pairing invariant: tool.started tanpa completed → outcome unknown.
5. purge_run: event run hilang, run lain utuh, event trail.purged ada, file atomik (tidak ada file temp sisa).
6. purge_session: file baru hanya trail.purged.
7. baris rusak (tulis teks sampah) → read tetap jalan, corrupted_lines terhitung.
8. sanitasi nama file session_key (`:` → `_`, karakter ilegal dibuang).
9. emit ke disk penuh disimulasikan (mock `open` melempar OSError) → tidak exception, chat tetap jalan (caller-side try/except teruji di test gateway).

## 9. Out of scope (PRD ini)

Rotasi otomatis, kompresi, live tail, search, export, sinkronisasi lintas mesin, event kind
subagent/swarm — lihat PRD-4 §Future.
