# PRD-2 — Gateway HTTP API (Trace & Audit)

| | |
|---|---|
| Feature | Endpoint REST untuk membaca proyeksi run, event per run, purge trail, dan tail audit global |
| Depends on | PRD-1 (`pha/src/pha/trace.py`) |
| Consumers | PRD-3 (WebUI) |

## 1. Ringkasan

Menambah endpoint di server gateway yang sudah ada (`gateway/health.py`, hand-rolled asyncio,
routing murni via `_route()` `health.py:311-421`) mengikuti pattern yang sudah terbukti:
**provider injection** (seperti `sessions_provider` di `start_health_server` `health.py:88`)
sehingga server tetap bisa dites tanpa gateway nyata. Tanpa framework baru.

## 2. Wiring (service → server)

`PhaGatewayService.run_foreground()` (`gateway/service.py:357-381`) menambah:

```python
trace_runs_provider=self.list_session_runs,        # (session_id) -> {runs, meta}
trace_events_provider=self.get_session_run_events, # (session_id, run_id, kinds, after_seq, limit)
trace_purge_provider=self.purge_session_trace,     # (session_id, run_id | None) -> {deleted, ...}
audit_provider=self.list_audit_events,             # (limit) -> {events}
```

Implementasi provider di `service.py` (pola sama dengan `list_webui_sessions` `:255-274`):
resolusi `session_id` → `session_key` via `PhaSessionBackend`, lalu delegasi ke modul `trace.py`
/ tail `audit.jsonl`. Toleran terhadap baris rusak; **tidak pernah melempar** ke router
(return `None` → router menjawab 503, konsisten dengan guard provider lain).

## 3. Endpoint

Semua endpoint (kecuali `/health`, `/ui`, `/`) **wajib bearer token** — otomatis via
`_requires_auth` (`health.py:191-192`). Semua response JSON.

| Method | Path | Param | Response 200 |
|---|---|---|---|
| GET | `/v1/sessions/{id}/runs` | — | `{"runs":[RunSummary], "meta":TrailMeta}` |
| GET | `/v1/sessions/{id}/runs/{run_id}` | `?kinds=tool.started,tool.completed&after_seq=0&limit=500` | `{"run":RunSummary, "events":[TraceEvent], "meta":TrailMeta}` |
| GET | `/v1/audit` | `?limit=100&session_key=api:…` | `{"events":[AuditEvent]}` (newest-first) |
| DELETE | `/v1/sessions/{id}/trace` | `?run_id=<id>` (opsional) | `{"deleted":N, "scope":"run"\|"session"}` |

### 3.1 RunSummary (proyeksi derived — field dihitung, bukan disimpan)

```json
{"run_id":"a1b2c3d4e5f6","status":"ok","started_at":"2026-08-29T03:12:40.1+00:00",
 "duration_ms":18400,"prompt_summary":"jalankan test webui",
 "model":"kimi-k2","provider":"moonshot","channel":"webui",
 "tool_count":4,"error_count":1,"usage":{"input_tokens":1200,"output_tokens":480},
 "interruptions":[{"tool":"bash","outcome":"unknown"}]}
```

`status` ∈ `ok | error | aborted | interrupted` (aturan PRD-1 §3.2). `interruptions` hanya
muncul bila ada tool tanpa result (pairing invariant).

### 3.2 Aturan query

| Aturan | Nilai |
|---|---|
| `limit` | default 500, maks 2000; > 2000 → 413 |
| `after_seq` | 0 default; paging: kirim `seq` event terakhir yang sudah dirender |
| `kinds` | comma-separated; kind tak dikenal diabaikan (bukan error) |
| `run_id` tak dikenal | 404 `{"error":"run not found"}` |
| `session_id` tak valid/tak ada | 400 (pola salah, `_SESSION_ID_PATTERN`) / 404 `{"error":"session not found"}` |
| provider `None` | 503 `{"error":"trace unavailable"}` / `{"error":"audit unavailable"}` |
| tanpa/invalid token | 401 (sudah otomatis) |

### 3.3 DELETE semantics

- Tanpa `run_id` → `purge_session`: hapus seluruh trail session; response `scope:"session"`.
- Dengan `run_id` → `purge_run`: hapus event run tsb saja; `scope:"run"`.
- Keduanya idempotent: purge run yang sudah tak ada → `deleted:0` + tetap 200 (UI tidak perlu
  membedakan); session tanpa trail → `deleted:0`.
- Server TIDAK menerima body untuk DELETE (router existing sudah begitu) — konfirmasi dua
  langkah adalah tanggung jawab UI (PRD-3).

## 4. Contract & konsistensi

- JSON key mengikuti konvensi existing (`snake_case`, `{"error": "..."}` untuk gagal).
- Tidak ada path absolut server dalam response (nama file internal tidak bocor).
- Urutan: `runs` terbaru dulu (`started_at` desc); `events` oldest→newest (timeline natural).
- Tanggal selalu ISO8601 UTC; UI yang mengonversi ke lokal.

## 5. Acceptance criteria

- **AC-2.1** `GET /v1/sessions/{id}/runs` mengembalikan proyeksi run sesuai PRD-1 §3.2 + `meta` (`size_bytes`, `event_count`, `corrupted_lines`).
- **AC-2.2** `GET .../runs/{run_id}` menghormati `kinds`, `after_seq`, `limit`; `limit` > 2000 → 413.
- **AC-2.3** Semua endpoint 401 tanpa token; 503 bila provider belum disuntik; 404 session/run tak dikenal; 400 session id melanggar pola.
- **AC-2.4** `DELETE .../trace?run_id=` hanya menghapus run tsb (verifikasi: run lain masih ada di response GET berikutnya) dan trail memuat `trail.purged`.
- **AC-2.5** `DELETE .../trace` (tanpa run_id) mengosongkan trail; GET berikutnya `runs:[]`, `meta.event_count == 1` (event trail.purged).
- **AC-2.6** `GET /v1/audit` mengembalikan audit newest-first, filter `session_key` bekerja, `limit` default 100 maks 1000.
- **AC-2.7** `audit.jsonl` tidak pernah dimodifikasi oleh endpoint ini (read-only tail).

## 6. Testing (`tests/test_pha/test_webui_api.py` — extend, pattern existing)

Server start di port ephemeral (0) dengan fake provider callables, request via
`asyncio.to_thread(http.client...)` — persis template existing:

1. happy path runs/run/audit dengan fake data.
2. auth: 401 tanpa token (semua endpoint baru).
3. provider None → 503 untuk runs & audit.
4. 404 session tidak ada; 404 run tidak ada; 400 session id salah pola.
5. limit 2001 → 413; `kinds` aneh → diabaikan, tetap 200.
6. DELETE per-run: hanya run tsb terhapus; event `trail.purged` muncul di GET.
7. DELETE per-session: runs kosong setelahnya.
8. smoke test HTML: tambah element id & path baru ke daftar assertions yang ada
   (`test_webui_api.py:414-427`) — setiap path `/v1/…` di atas harus direferensikan
   `WEBUI_HTML`, dan setiap id baru ada.
9. `ruff check src/pha` bersih.
