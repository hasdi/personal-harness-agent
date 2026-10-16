# PRD-31 — Picker Agent + Effort per Sesi di Bawah Composer

| | |
|---|---|
| Feature | Bar `#agent-bar` bawah composer: `select#agent-select` + effort; effort pindah dari header |
| Depends on | PRD-30 (agent binding), PRD-26 (effort per-sesi/request) |
| Files | `pha/gateway/webui/{markup,app_js,spa_js,style,i18n_js}.py`, `pha/gateway/{health,service,runtime}.py`, `pha/session_storage.py` |
| Status | Draft (2026-09-24) |

## 1. Keputusan

| Aspek | Keputusan |
|---|---|
| Posisi | Bar baru `#agent-bar` dalam `#composer-wrap` bawah `#composer` (`markup.py:87-98`); `effort-select` header (`markup.py:58`) disembunyikan (compat, sinkron dua arah), sumber benar pindah ke bar bawah |
| Opsi | Agent dari `GET /v1/agents` + entry `default`; effort dari `SESSION_EFFORT_LEVELS` (`runtime.py:59`) |
| Apply | Ganti agent → `POST /v1/sessions/{id}/agent` (PRD-30); ganti effort → `POST /v1/sessions/{id}/effort` existing (`health.py:1035`); berlaku chat itu juga via staged per-request pola `stage_request_effort/consume_request_effort` (`runtime.py:379-413`, `_remember_request_effort:488`) — tiru untuk agent (`stage_request_agent`) |
| Guard | Saat run aktif kedua select disabled; invalid dari client → fallback global tanpa 500 (pola AC-26.3) |
| Persist | Snapshot bawa `agent_name` + `effort`; restore → `set_effort` + agent prune + prompt rebuild (pola PRD-26 §3) |

## 2. Goals

- **G1** — Tiap sesi pilih agent + effort tepat sebelum kirim chat.
- **G2** — Pilihan bertahan refresh; antar-sesi independen.

## 3. Non-goals

- Auto-turun effort (PRD-26 non-goal, tetap); passes per sesi.

## 4. Acceptance criteria

- **AC-31.1** Ganti agent/effort di sesi A → sesi B tetap; refresh bertahan; snapshot cocok.
- **AC-31.2** Kirim chat langsung pakai pilihan baru (tanpa kirim ulang); run aktif → select disabled.
- **AC-31.3** Agent/effort invalid → fallback default tanpa 500.
- **AC-31.4** `ruff` + `node --check` OK.

## 5. Testing

1. Existing `test_session_effort_context.py` + binding agent baru di `test_agents.py`.
2. Manual: ganti effort low vs max pada prompt sama, bandingkan hasil.
