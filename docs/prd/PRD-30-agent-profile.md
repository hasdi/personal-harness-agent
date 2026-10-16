# PRD-30 — Agent Profile: Capability Tools/Skills + Mandatory + Effort Default

| | |
|---|---|
| Feature | `agents.json` + CRUD + resolusi capability per sesi (allowed/mandatory tools/skills, effort default, extra prompt, permission override) |
| Depends on | PRD-29 (disabled global), PRD-15 (pola page), PRD-26 (effort) |
| Files | `pha/agents.py` (baru), `pha/gateway/{models,service,health,runtime}.py`, `pha/session_storage.py`, `pha/gateway/webui/{markup,spa_js,app_js,i18n_js,style}.py`, `tests/test_pha/test_agents.py` (baru) |
| Status | Draft (2026-09-24) |

## 1. Ringkasan

Satu file `<ws>/agents.json` (pola `cron.json`), satu modul `pha/agents.py` (load/save/add/update/remove/get, reuse `atomic_write` + `exclusive_file_lock`).

## 2. Skema `AgentProfile`

```json
{"name": "riset", "description": "...", "allowed_tools": ["*"],
 "mandatory_tools": ["web_search"], "allowed_skills": ["*"],
 "mandatory_skills": [], "default_effort": "low",
 "extra_prompt": "Jawab ringkas.", "permission_mode": null}
```

| Field | Arti |
|---|---|
| `allowed_*` | `["*"]` = semua (default); null/absen = ikut global |
| `mandatory_*` | Subset selalu on, menang atas global disabled |
| `default_effort` | Validasi `normalize_session_effort` (`runtime.py:63`); null = ikut sesi/global |
| `permission_mode` | Null = ikut gateway; isi = override per sesi (`_apply_permission_mode:575-593`) |
| `extra_prompt` | Append ke system prompt rebuild (`prompts.py`, `_runtime_system_prompt:899-916`) |

Preseden final per sesi: mandatory > agent allowlist > global `disabled_*` > default on. Skill mandatory tak bisa di-disable (API tolak + UI badge).

## 3. API

| Endpoint | Keterangan |
|---|---|
| `GET /v1/agents` | list profile |
| `POST /v1/agents` | create (nama duplikat → 400) |
| `PUT /v1/agents/{name}` | update |
| `DELETE /v1/agents/{name}` | delete; sesi terikat fallback `default` |
| `POST /v1/sessions/{id}/agent {agent}` | binding sesi, simpan `tool_metadata.agent_name` (ikut pola `session_effort`); unknown → 404 |

Enforcement reuse choke point PRD-29 (`_install_pha_session_tools:137`): prune registry + filter `extra_skill_dirs` per agent.

## 4. UI

View Agents klon skills page + modal editor (nama/desc/tools/skills comma-list/mandatory/effort/extra_prompt) + badge mandatory. i18n 3 bahasa.

## 5. Goals

- **G1** — Tiap sesi bisa jalan dengan capability beda tanpa restart.
- **G2** — Mandatory selalu tersedia walau global mati.
- **G3** — Tanpa ubah `src/openharness/**`.

## 6. Non-goals

- Versioning agent; import/export; builtin read-only (semua user-editable dulu).

## 7. Acceptance criteria

- **AC-30.1** Agent `riset` (tools cuma web+read, mandatory `web_search`) → sesi itu `registry.get("bash")` None, `web_search` ada walau global disabled.
- **AC-30.2** Ganti agent sesi → berlaku chat berikut tanpa restart; refresh bertahan (snapshot `tool_metadata.agent_name`).
- **AC-30.3** Nama duplikat/unknown → 400/404 jelas; mandatory di-disable via tools API → ditolak.
- **AC-30.4** Suite baru + existing hijau; `ruff` bersih.

## 8. Testing

1. `pytest tests/test_pha/test_agents.py -q` (baru: CRUD file, preseden resolusi, binding snapshot).
2. `ruff check src/pha/agents.py src/pha/gateway/`.
3. Manual: bikin agent minimal → chat di luar capability ditolak jelas.
