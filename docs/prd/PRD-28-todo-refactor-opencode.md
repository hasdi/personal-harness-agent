# PRD-28 — Refactor Todo ala OpenCode

| | |
|---|---|
| Feature | `todo_write` state-machine per-session (`pending/in_progress/completed/cancelled`) menggantikan append `TODO.md` |
| Depends on | — (independen dari PRD-27) |
| Files | `openharness/tools/todo_write_tool.py`, `openharness/engine/query.py` (`_record_tool_carryover`), `openharness/prompts/system_prompt.py`, `openharness/ui/backend_host.py`, `tests/test_tools/test_core_tools.py` + baru |
| Status | Implemented (2026-09-16) |

## 1. Ringkasan

`todo_write` hari ini (`item/checked/path="TODO.md"`) bukan TodoWrite ala OpenCode: satu item per call, boolean saja, tulis file `TODO.md` ke cwd proyek (polusi folder user; melanggar PHA-centric: state sesi harus di snapshot, bukan di folder kerja). Akibatnya: model tidak bisa me-replace satu daftar utuh, tidak ada aturan "tepat satu `in_progress`", tidak ada `activeForm`, dan `backend_host.py:335` yang mengharapkan input `todos` list tidak pernah cocok (jalur `todo_update` mati, cuma fallback parse output).

Port **pola** dari `todowrite.txt` OpenCode (bukan kode TS): satu call = satu daftar penuh, status `pending/in_progress/completed/cancelled`, tepat satu `in_progress` saat kerja tersisa, update real-time antar langkah.

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Skema | `TodoWriteToolInput(todos: list[TodoItem])`; `TodoItem(content, status, activeForm="")`. Nama tool tetap `todo_write` (tanpa rename/break) |
| Semantik | Satu call me-replace seluruh daftar. Validasi: tolak bila >1 `in_progress`. `completed` hanya setelah kerja+verifikasi benar selesai; blocked → tetap `in_progress` + tambah follow-up |
| Simpan | `tool_metadata["todos"]` via `_record_tool_carryover` (ikut snapshot/restore sesi otomatis; tanpa file baru). Hapus tulis `TODO.md` (file lama yang sudah ada dibiarkan, tidak dimigrasi) |
| Prompt | Ganti 2 baris `todo_write` di `system_prompt.py` dengan aturan ringkas todowrite.txt (kapan pakai / state / satu in_progress / update tiap langkah) |
| WebUI/host | Perbaiki `backend_host` baca `input.todos` (status + `activeForm` saat `in_progress`); tanpa UI/sidebar baru di scope ini |

## 3. Goals

- **G1** — Daftar todo per-session, persisten lewat snapshot (refresh tidak hilang).
- **G2** — Kontrak sama dengan OpenCode/Claude Code (`todos` + 4 status + `activeForm`) sehingga prompt skill umum langsung bisa dipakai.
- **G3** — Tidak ada lagi file `TODO.md` baru di folder user; tanpa dependensi baru.

## 4. Non-goals

- Sidebar/progress TODO di WebUI; migrasi isi `TODO.md` lama; sinkron todo ke subagent (`enableTodos` OpenCode) — catat future.

## 5. Acceptance criteria

- **AC-28.1** Satu call 3 todos me-replace daftar; call berikut dengan status berubah me-replace tanpa duplikat.
- **AC-28.2** Dua `in_progress` dalam satu call → `is_error` dengan pesan jelas.
- **AC-28.3** `todos` ikut snapshot → restore sesi mengembalikan daftar yang sama.
- **AC-28.4** `backend_host` meng-emit `todo_update` dari `input.todos` (termasuk label `activeForm` untuk yang `in_progress`).
- **AC-28.5** Tidak ada `TODO.md` baru tercipta di cwd selama AC-28.1–28.4.
- **AC-28.6** Suite lama yang diperbarui + baru hijau; `ruff` bersih.

## 6. Testing

1. `pytest tests/test_tools/test_core_tools.py tests/test_tools/test_integration_flows.py tests/test_tools/test_todo_state.py -q` (baru: replace-semantik, tolak 2 in_progress, roundtrip `tool_metadata`, host emit, tanpa file).
2. `ruff check src/openharness/tools/todo_write_tool.py src/openharness/engine/query.py src/openharness/ui/backend_host.py`.
3. Manual: sesi multi-langkah → model memanggil `todo_write` per langkah dengan tepat satu `in_progress`.

## 7. Referensi

- OpenCode `todowrite.txt` (kontrak + state machine): https://raw.githubusercontent.com/Kilo-Org/kilocode/main/packages/opencode/src/tool/todowrite.txt
- Claude Code todo lifecycle + Task tools: https://code.claude.com/docs/en/agent-sdk/todo-tracking
- Kode: `src/openharness/tools/todo_write_tool.py`, `src/openharness/engine/query.py:410-526` (`_record_tool_carryover`), `src/openharness/ui/backend_host.py:334-349`, `src/openharness/prompts/system_prompt.py:544,761`.

## 8. Follow-up 2026-09-16 — checklist tampil di WebUI (ala OpenCode)

Request: todo harus terlihat sebagai use-case spesifik — keluar sebagai list lalu jadi checklist saat selesai, seperti coding agent OpenCode. Implementasi: `ToolExecutionStarted(todo_write)` kini membawa `todos` di metadata update → SSE `tool_hint` membawa payload `todos` (detached-run replay ikut membawa) → WebUI me-render kartu checklist live (satu kartu, di-replace tiap update: `○` pending, `→` in_progress + `activeForm`, `✓` completed, `–` cancelled, header `☑ n/m`). `load_webui_session` mengekspos `todos` per pesan assistant (dari block `tool_use` todo_write) → refresh me-render ulang kartu per pesan (pinned + closable). Chip `Using todo_write: …` diganti ringkasan `todos n/m: [x] … [>] …`. File: `pha/gateway/{runtime,service,health}.py`, `webui/{app_js,style}.py`, `tests/test_pha/test_todo_webui.py`.
