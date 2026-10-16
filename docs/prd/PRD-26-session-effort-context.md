# PRD-26 — Effort Per-Session + Context Progress Bar

| | |
|---|---|
| Feature | Pilih effort per sesi chat (global standar) + progress bar context per sesi |
| Depends on | PRD-5 (wcd/cwd per sesi), PRD-10 (SETUP `effort`), auto-compact engine |
| Files | `pha/gateway/{models,runtime,service,health}.py`, `pha/session_storage.py`, `pha/gateway/webui/{markup,style,app_js,i18n_js}.py`, `openharness/services/compact/__init__.py`, `tests/test_pha/{test_webui_api,test_health_config,test_session_effort_context}.py` |
| Status | Implemented (2026-09-15) |

## 1. Ringkasan

`effort: max` global membuat tugas ringan ikut over-engineer (kasus PRD-25:
gambar sederhana → 13 turn). Perlu pilihan effort per sesi (default = global
`gateway.json`), plus bar context per sesi karena auto-compact sudah ada tapi
tak terlihat.

## 2. Level effort (global standar)

`off, minimal, low, medium, high, xhigh, max`. `off` = tanpa reasoning
tambahan; `max` dinormalisasi ke `xhigh` mengikuti `merge_cli_overrides`
(`settings.py:893`) dan codex client (`codex_client.py:136-142`).
Perilaku per model tetap milik provider: `reasoning_effort` hanya dikirim ke
family yang mendukung (`openai_client.py:388`), prompt `# Reasoning Settings`
selalu mencantumkan effort aktif walau provider mengabaikannya.

## 3. Keputusan

| Aspek | Keputusan |
|---|---|
| Sumber effort | Per request (dropdown tiap chat) → per sesi (`tool_metadata.session_effort`) → global `gateway.json.effort` → `medium` |
| Persistensi | Snapshot bawa `effort`; restore → engine `set_effort` + prompt rebuild |
| Ubah effort | Dropdown chat-header tiap chat; berlaku chat itu juga (per-request override) lalu tersimpan; tanpa restart; saat run aktif → disabled |
| Context bar | Di bawah chat-header; `tokens / threshold` dari estimator existing; warna: <70% tenang, 70–90% waspada, ≥90% danger + label "compact soon" |
| Data bar | `load_webui_session` hitung `context_tokens`, `context_threshold`, `context_ratio`; sidebar tetap tampilkan badge token existing |
| Estimator | `estimate_message_tokens` (+system+tools heuristic governor); `get_autocompact_threshold(model, context_window_tokens)` |

## 4. Goals

- **G1** — Tugas ringan bisa `low` per sesi tanpa ubah global.
- **G2** — User melihat sisa context sebelum compact menendang.
- **G3** — Tanpa migrasi snapshot lama (field absen = fallback global).

## 5. Non-goals

- Auto-turun effort; `passes` per sesi; bar live per token saat streaming.

## 6. Acceptance criteria

- **AC-26.1** Ganti effort di sesi A → sesi B tetap; refresh → pilihan A bertahan; snapshot `effort` = pilihan.
- **AC-26.2** Bar tampil `tokens/threshold` + %; sesi 858KB gambar menunjukkan rasio tinggi; ≥90% label waspada.
- **AC-26.3** Effort tak valid dari client → fallback global, tanpa 500.
- **AC-26.4** Suite baru + existing hijau; `ruff`; `node --check` JS.

## 7. Testing

1. `pytest tests/test_pha/test_session_effort_context.py tests/test_pha/test_webui_api.py tests/test_pha/test_health_config.py -q`.
2. `ruff check src/pha/gateway/ src/pha/session_storage.py`.
3. Manual: AC-26.1, AC-26.2 (bandingkan low vs max pada prompt gambar yang sama).
