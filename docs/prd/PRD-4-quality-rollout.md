# PRD-4 — Kualitas, Security, Rollout & Future

| | |
|---|---|
| Feature | NFR, keamanan/privasi, strategi testing, milestone, risiko, future work |
| Depends on | PRD-1, PRD-2, PRD-3 |

## 1. Non-functional requirements

| Area | Requirement |
|---|---|
| Kinerja | emit < 5 ms p95; read endpoint < 500 ms untuk 1.000 event (paging wajib); polling hanya saat tab aktif |
| Ukuran | ~2–5 KB/run (tanpa per-delta). Per-session file; purge manual core. Rotasi = future. |
| Reliabilitas | emit gagal tidak pernah menggagalkan chat; reader toleran baris rusak; purge atomik (`os.replace`), tidak ada file temp sisa |
| Maintainability | 1 modul baru (`trace.py`) + wiring point di `service.py`/`health.py` + pane UI; tanpa dependensi; tanpa perubahan `src/openharness/**`; semua kontrak didokumentasikan di PRD-1 §5 |
| Observability | kegagalan emit/purge → stderr (pattern `--debug`), tidak silent |
| Portabilitas | path via `get_logs_dir()`; Windows & POSIX (atomic rename via `os.replace` aman di keduanya) |

## 2. Security & privacy

- **Auth**: semua endpoint di belakang bearer token `_requires_auth` — tidak ada endpoint
  trace/audit publik.
- **Data minimization**: trace hanya menyimpan `summary` (≤160 char) + metadata; tool input
  mentah & output mentah tidak pernah disimpan; sender selalu hash (warisan `audit.py`,
  FR-SEC-6 existing).
- **Tidak ada secret** (API key/token provider) yang masuk trace; prompt_summary dipotong
  sebelum emit.
- **Purge irreversible** → wajib modal konfirmasi (PRD-3 §3.4); jejak `trail.purged`
  mencatat siapa-scope-kapan (scope + ts), tanpa isi data yang dihapus.
- **No path leakage**: API tidak mengembalikan path absolut server.
- **XSS**: semua rendering via `escHtml`; detail JSON dirender sebagai teks di `<pre>`.

## 3. Strategi testing (peta penuh)

| Layer | File test | Cakupan |
|---|---|---|
| Unit store | `tests/test_pha/test_trace.py` (baru) | PRD-1 §8 (1–9) |
| API | `tests/test_pha/test_webui_api.py` (extend) | PRD-2 §6 (1–8) |
| UI smoke | `test_webui_api.py::…smoke` (extend) | id elemen + path API + zero URL eksternal |
| Lint | `ruff check src/pha` | wajib bersih (config pyproject:69-86) |
| Manual QA | checklist di bawah | alur end-to-end |

**Manual QA checklist** (jalankan sekali sebelum selesai):
1. Jalankan gateway (`pha` gateway foreground) → buka `/ui` → kirim 1 prompt berisi tool call.
2. Tab Trace → session muncul → run `ok` dengan ≥1 tool.completed.
3. Hentikan gateway di tengah run → restart → run tampil `interrupted` + tool `? unknown`.
4. Clear per run → run hilang, lain utuh, `trail.purged scope=run` terlihat.
5. Clear per session → runs kosong, meta 1 event.
6. Matikan token auth (salah token) → semua endpoint baru 401 & UI memunculkan flow
   unauthorized existing.
7. Corrupt satu baris JSONL manual → UI tetap render + warning corrupted di meta.

## 4. Milestone rollout (definition of done per milestone)

| Milestone | Isi | DoD |
|---|---|---|
| **M1 — Capture & Store** | PRD-1 penuh (modul `trace.py` + instrumentasi adapter) | test_trace hijau; AC-1.1–1.9 |
| **M2 — API** | PRD-2 penuh (runs, events, audit, purge) | test_webui_api hijau; AC-2.1–2.7 |
| **M3 — WebUI** | PRD-3 penuh (tab + purge UX) | smoke test + manual QA 1–7; AC-3.1–3.8 |
| **M4 — Hardening** | PRD-4: security pass, lint, docs (`pha/README.md` bagian Trace) | ruff bersih, README update, review |

Urutan ketat M1 → M2 → M3 (kontrak berantai); M4 setelah semuanya. Tiap milestone di-commit
terpisah agar mudah di-review/rollback.

## 5. Risiko & mitigasi

| Risiko | Mitigasi |
|---|---|
| Engine tidak mengekspos usage/retry terstruktur | Field opsional; progress teks tetap terlog; kind baru bisa ditambah tanpa breaking (tolerant reader) |
| Trail tumbuh tak terbatas di session panjang | Tanpa per-delta logging; purge UX; rotasi jadi future item cepat ditambah (satu fungsi) |
| Purge keliru (salah session) | Modal menampilkan nama session + jumlah event; DELETE idempotent; jejak trail.purged |
| Drift fork terhadap upstream OpenHarness | Nol perubahan engine; hanya adapter layer pha |
| `WEBUI_HTML` module-level (perlu restart) | Didokumentasikan di PRD-3 + DEVLOG; tidak dianggap bug |

## 6. Future work (non-core, dokumen terpisah bila dijalankan)

1. **Live tail SSE** (`/v1/sessions/{id}/trace/stream`) mengikuti `_SseWriter` pattern.
2. **Rotasi & retensi**: `trace_max_bytes` + `trace_retention_days` di `gateway.json`.
3. **Export/share** trail (markdown/json) — reuse `transcript.md` pattern OpenHarness.
4. **Trace subagent/swarm & channel replay** (Telegram/Slack) di tab Trace.
5. **Search** lintas session (grep index sederhana).
6. **Paritas TUI**: panel trace di OpenHarness TUI membaca store yang sama bila engine
   di-upstream-kan (store sudah dirancang netral terhadap consumer).

## 7. Open questions (perlu keputusan sebelum/di awal M1)

1. `usage` (token) dari stream handler: apakah `final` SSE sudah membawa usage hari ini?
   (Jika belum → field opsional, tidak memblok M1.)
2. Session channel (Telegram/dll) ikut ke tab Trace di core, atau hanya `api:*`?
   — Rekomendasi: semua session yang punya `session_key`, selector menampilkan semua.
3. Batas bawah versi skema: kapan `v` dinaikkan? — Hanya saat field wajib berubah makna.
