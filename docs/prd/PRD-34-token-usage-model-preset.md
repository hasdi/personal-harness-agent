# PRD-34 — Token Usage (Reported) + Model Preset Inline

| | |
|---|---|
| Feature | `model_presets` inline di `gateway.json` + switch per sesi + catat usage reported + API + badge sesi |
| Depends on | PRD-26 (pola `session_effort`), PRD-30 (pola binding via `tool_metadata`), PRD-1 (pola `trace.emit` fail-open) |
| Files | `pha/usage.py` (baru), `pha/presets.py` (baru), `pha/gateway/{models,provider_auth,service,session_pool,stream_updates,health}.py`, `pha/gateway/webui/{markup,app_js,i18n_js}.py`, `pha/workspace.py`, `openharness/services/session_storage.py` (1 baris), `docs/gateway-config.md`, `tests/test_pha/test_usage_presets.py` (baru) |
| Status | Implemented (2026-10-09) |

## 1. Ringkasan

Satu field `model_presets` inline di `gateway.json` (tanpa file/CRUD baru), switch per sesi via `tool_metadata.model_preset` (reuse pola `session_effort`/`agent_name`), dan satu modul `pha/usage.py` yang mencatat `UsageSnapshot` reported dari `ApiMessageCompleteEvent` ke JSONL secara fail-open. Tanpa SQLite, tanpa estimasi, tanpa grafik.

```json
"model_preset": "hemat",
"model_presets": {
  "hemat": {"model": "gpt-4o-mini", "max_tokens": 4096, "temperature": 0.1},
  "kuat": {"model": "gpt-4o", "max_tokens": 8192, "temperature": 0.1}
}
```

## 2. Acuan nanobot (reverse engineering)

| Nanobot | Diambil polanya | Dibuang |
|---|---|---|
| `providers/base.py:41,723` `LLMCallObserver` + `set_llm_call_observer()` fail-open | Hook di handler `ApiMessageCompleteEvent`, try/except never-raise | `ContextVar` source, `_usage_for_call` estimasi |
| `providers/base.py:279` `LLMUsage`, `openai_compat_provider.py:1464` `_extract_usage()` | Pakai `UsageSnapshot` existing (`openharness/api/usage.py:8`), reported saja | `estimated/cache_read/write`, `ttft/generation_ms`, tiktoken |
| `llm_usage/store.py:159` SQLite WAL + `usage_payload()` | `pha/usage.py` JSONL append-only per sesi, reuse pola `pha/trace.py:10` | SQLite, agregasi SQL, grafik 371 hari, streak |
| `config/schema.py:97` `ModelPresetConfig`, `agent/model_presets.py:78` `normalize_preset_name()` | `ModelPreset` Pydantic inline + match case-insensitive ~10 baris | `presets.json`, CRUD, `resolve_default_preset`, ContextVar |
| `command/builtin.py:266,405` `cmd_status` + `cmd_model` | Badge sesi + endpoint switch | `/status` full, `context_window_tokens`, reasoning_effort |

## 3. Skema

```python
class ModelPreset(BaseModel):
    model: str
    provider_profile: str | None = None
    max_tokens: int | None = None
    temperature: float | None = None
    extra_body: dict = {}
# GatewayConfig += model_preset: str | None = None; model_presets: dict[str, ModelPreset] = {}
```

Preseden: `tool_metadata.model_preset` sesi > `config.model_preset` > legacy `config.model`. Unknown → 404 + list available. Tambah `"model_preset"` ke `_PERSISTED_TOOL_METADATA_KEYS` (pola `agent_name`, 1 baris).

Resolve tetap di `resolve_runtime_settings()` (`provider_auth.py:37`), merge `max_tokens/temperature/extra_body` preset. Tanpa modul resolver baru.

## 4. Alur usage

`openai_client.py:504` `ApiMessageCompleteEvent(usage)` → `record_usage(session_key, provider, model, in, out)` → append `~/.pha/logs/usage/<safe>.jsonl` + snapshot `usage` existing (`session_storage.py:116`) tetap sumber sesi. Fail-open: hook gagal → chat tetap jalan, log saja.

`usage.py` (~80 baris): `record_usage()`, `session_totals()`, `global_totals()`.
`# ponytail: scan O(n) per read, SQLite bila sesi >10k run`

## 5. API

| Endpoint | Keterangan |
|---|---|
| `GET /v1/presets` | `{active, presets}` |
| `POST /v1/sessions/{id}/preset {preset}` | binding sesi (pola `_session_agent_for_snapshot`); unknown → 404 |
| `GET /v1/sessions/{id}/usage` | `{input_tokens, output_tokens, total_tokens, requests}` |
| `GET /v1/usage` | `{total_tokens, total_requests, sessions}` |

Session list payload ikut sertakan `usage` + `preset` di sebelah `effort/agent` (`service.py:938-941`).

## 6. UI

Badge `preset` + `tokens sesi` di header session (reuse payload §5). Tanpa halaman Usage, tanpa grafik.

## 7. Goals

- **G1** — Tiap sesi ganti model tanpa restart (chat berikut langsung pakai preset baru).
- **G2** — Tiap chat tercatat usage reported, bisa dibaca per sesi + global.
- **G3** — Tanpa ubah `src/openharness/**` selain 1 baris metadata; tanpa dependensi baru.

## 8. Non-goals

- SQLite/WAL, grafik harian, streak, breakdown provider/model.
- Estimasi lokal, tiktoken, cache split, timing, cost/rupiah.
- CRUD preset via API, versioning, import/export.

## 9. Acceptance criteria

- **AC-34.1** Switch preset → chat berikut pakai `model` preset; refresh bertahan (snapshot `tool_metadata.model_preset`).
- **AC-34.2** Preset unknown → 404 jelas + daftar available (case-insensitive).
- **AC-34.3** `GET usage` sesi = jumlah reported provider, bukan estimasi.
- **AC-34.4** Hook `record_usage` lempar error → chat tetap sukses.

## 10. Testing

1. `pytest tests/test_pha/test_usage_presets.py -q` (baru: record+totals, preseden preset atas legacy, switch tanpa restart).
2. `ruff check src/pha/usage.py src/pha/gateway/`.
3. Manual: 2 preset di `gateway.json` → switch via API → chat 2 turn → `GET /v1/sessions/{id}/usage` naik.

→ skipped: grafik/streak/SQLite, estimasi/tiktoken, CRUD preset. Add when: analitik 30 hari diminta, provider tak lapor usage, preset >5.
