# PRD-33 — System One Decision Layer (Laya, Jev-compatible)

| | |
|---|---|
| Feature | Decision cepat bertipe (`choice/score/noul` + probabilitas) untuk routing, permission, compact, judge — tanpa LLM call |
| Depends on | PRD-25 (pola fallback vision), PRD-30 (agent profile), gateway-config |
| Files | `openharness/config/settings.py` (`DecisionModelConfig`), `openharness/services/system_one.py` (baru), `openharness/ui/runtime.py`, `openharness/engine/query.py` (permission refine + compact passthrough), `openharness/services/compact/__init__.py` (near-band gate), `pha/gateway/{models,session_pool,provider_auth}.py`, `pha/gateway/stream_updates.py` (triage/agent-hint/judge), `pha/gateway/{service,runtime}.py` (SETUP meta + re-export), `docs/gateway-config.md`, `tests/test_pha/test_system_one.py` (baru) |
| Status | Implemented (2026-09-26) |

## 1. Ringkasan

LLM dipakai untuk hal sepele: triage pesan, cek bahaya bash, pilih agent, cek perlu compact, nilai jawaban. Lambat, mahal, varians tinggi. System One ganti 5 titik itu dengan 1 call decision: input state + pertanyaan bertipe → `{choice/score/noul, prob, conf}` 1 pass (~33-140ms lokal, $0). Conf rendah → fallback jalur lama (LLM/heuristik). Tak pernah error user.

Pilihan model: Laya self-host default (Apache 2.0, data tak keluar, `POST /v1/systemone` Jev-compatible). Jev hosted hanya fallback opsi bila kategori >15 / zero-shot baru.

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Config | Pluggable, tiru pola vision (`VisionModelConfig`): `DecisionModelConfig` di `settings.py` + mirror 3 field di `GatewayConfig`; resolve via `_resolve_decision_config`, lempar via `tool_metadata["decision_model_config"]`. Tanpa hardcode model di call-site |
| Client | 1 modul `services/system_one.py` tipis: `decide(state, questions)` + `branch(conf)`. HTTP ke `laya-serve` (`POST /v1/systemone`), reuse pola `OpenAICompatibleClient`. Tanpa SDK baru |
| Branch policy | `conf>=high` auto; `mid<=conf<high` jalur konservatif/confirm; `conf<mid` eskalasi LLM/human. Default `high=0.8, mid=0.5`, configurable |
| Fallback | Disabled/timeout/conf rendah → jalur lama persis (LLM/heuristik/fnmatch). Never error, log+trace saja |
| Batas Laya | `<20` opsi per `choice`; state truncate ~512 token; batch semua question 1 call |
| Default | `enabled=false`. Opt-in per gateway. Zero ubah perilaku existing bila mati |

## 3. Skema config

```python
class DecisionModelConfig(BaseModel):
    enabled: bool = False
    base_url: str = "http://127.0.0.1:8001"
    api_key: str = ""
    model: str = "laya"
    timeout: float = 3.0
    conf_high: float = 0.8
    conf_mid: float = 0.5
```

`gateway.json`: `decision_enabled, decision_base_url, decision_api_key, decision_model, decision_conf_high, decision_conf_mid`. Env override: `OPENHARNESS_DECISION_*`.

## 4. Modul `services/system_one.py`

```python
async def decide(state: str | dict, questions: dict, *, config: DecisionModelConfig) -> SystemOneResult
def branch(conf: float, *, high: float, mid: float) -> Literal["auto","confirm","escalate"]
def disabled(config) -> bool
```

`SystemOneResult.answers`: `{name: ChoiceAnswer(choice, probabilities) | ScoreAnswer(score, distribution) | NoulAnswer(noul)}` + `usage`. Timeout → raise `SystemOneUnavailable` → caller fallback diam-diam.

## 5. Call-site (5 titik, semua optional + fallback)

| # | Lokasi | Q | Guna | Fallback lama |
|---|---|---|---|---|
| 1 | `router.py` + `session_pool._stream_message_inner` | `choice` intent/agent, `score` urgency, `noul` spam | Triage inbound Telegram/group; ganti 1 LLM call per pesan | Prompt LLM existing |
| 2 | `permissions/checker.py` | `noul` "op destruktif?" + `score` risiko | Ganti fnmatch kaku; `auto/confirm/block` via `branch()` | Glob/deny list existing |
| 3 | `session_pool._apply_agent_profile` | `choice` profil/worker | Routing agent tanpa reasoning LLM | Default profile |
| 4 | `engine/query.py` + `services/compact` | `score` urgency compact, `choice` micro/collapse/memory/full, `noul` retryable | Ganti `chars/4` + string-match error; gabung `_is_prompt_too_long` ganda | Heuristik existing |
| 5 | `stream_updates._convert_stream_event` final | `score` groundedness/policy | Ganti LLM-as-judge | Tanpa judge |

Urutan implementasi: 2 → 1 → 4 → 3 → 5. Tiap titik merge terpisah, bisa dimatikan via `decision_enabled=false`.

## 6. Goals

- **G1** — 5 titik di atas tanpa LLM call bila `conf>=high`.
- **G2** — Mati default = perilaku identik existing; nyala = data chat tak keluar (Laya lokal).
- **G3** — Tanpa ubah `ToolRegistry`/loop signature; cuma tambah config + 1 modul + 5 patch kecil.

## 7. Non-goals

- Training/fine-tune Laya; Jev hosted sebagai default; long-text/codegen via System One (tetap LLM).
- Dashboard kalibrasi; cukup log `conf vs outcome` ke trace existing.
- Ganti `ContextGovernor`/compact penuh (PRD terpisah); sini cuma decision gate.

## 8. Acceptance criteria

- **AC-33.1** `enabled=false` → semua suite hijau, zero diff perilaku (mock decision tak pernah dipanggil).
- **AC-33.2** Titik 2: `rm -rf /` → `noul≈1.0` → block tanpa LLM; `git status` → `auto`; conf mid → confirm prompt.
- **AC-33.3** Titik 1: pesan spam/urgent/router → 1 HTTP `systemOne` batch, latency p95 <500ms lokal, tanpa LLM call bila high.
- **AC-33.4** Timeout/down → fallback diam-diam ke jalur lama + trace `system_one.unavailable`; tanpa ErrorEvent user.
- **AC-33.5** `ruff` + `pytest tests/test_pha/test_system_one.py` hijau (mock server: branch 3 level, fallback, batch).

## 9. Testing

1. `pytest tests/test_pha/test_system_one.py -q` (16: decide mock, branch batas, tiap call-site fallback, disabled=no-call).
2. `ruff check src/pha tests/test_pha/test_system_one.py --ignore E402` bersih (E402 pre-existing circular import).
3. Manual: `laya-serve` lokal → `decision_enabled=true` → chat group + bash destruktif + sesi panjang (compact trigger).

## 10. Hasil verifikasi (2026-09-26)

- `tests/test_pha`: **335 passed**.
- `tests/test_engine/test_query_engine.py`: 32 passed. `test_services`: 87 passed + 2 cron `TestExecuteJob` gagal pre-existing (`/tmp` tak ada di Windows, `cwd` job → status `error`; untouched oleh PRD ini, tak ada referensi system_one di `cron_scheduler.py`).
- AC-33.1 terpenuhi: disabled default, zero diff perilaku (test `disabled_zero_call`, `none_when_disabled`, `judge_none`).
