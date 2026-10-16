# PRD-27 — Tools PHA-centric + Subagent WebUI

| | |
|---|---|
| Feature | Panggilan `subagent` dari WebUI bekerja + worker yang di-spawn adalah PHA (prompt, skill, auth gateway) |
| Depends on | — |
| Files | `pha/gateway/subagent_tools.py` (baru), `pha/task_worker.py` (baru), `pha/cli.py` (`--task-worker`), `pha/gateway/runtime.py`, `tests/test_pha/test_subagent_tools.py` (baru) |
| Status | Implemented (2026-09-16) |

## 1. Ringkasan

Request WebUI "pakai subagent …" gagal. Tiga lapis penyebab, terverifikasi dari kode:

1. **Nama tidak ada.** Registry hanya punya `agent` (`openharness/tools/__init__.py:87`); tidak ada `subagent`. Seluruh kosakata produk (`/subagents`, `subagent_type="worker"`, event `subagent_stop`, prompt "Use subagents…") mengarahkan model menebak tool `subagent` → `Unknown tool: subagent` (`engine/query.py:1069`). `tool_search(query="subagent")` me-return `(no matches)` sehingga model tidak bisa menemukan nama benar sendiri.
2. **Worker bukan PHA.** `AgentTool` selalu spawn `[sys.executable, "-m", "openharness", "--task-worker"]` (`swarm/subprocess_backend.py:83`): tanpa system prompt PHA (soul/identity/memory), tanpa skill/plugin PHA, tanpa fallback chain. Frozen `pha.exe` lebih parah: argv jadi `[pha.exe, --task-worker]` padahal `pha` tidak punya flag itu → child langsung exit.
3. **Auth gateway tidak ikut.** `gateway.json` (`api_key`/`base_url`/`model` dari `pha provider set-key`) hanya hidup di objek `api_client` parent. `build_inherited_env_vars()` tidak meneruskan apa pun soal PHA/gateway, jadi worker yang key-nya hanya ada di `gateway.json` gagal auth / salah provider.

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Alias | `SubagentTool(AgentTool)`, `name="subagent"` + `PhaAgentTool` (`name="agent"`) override upstream khusus sesi gateway (pola sama seperti `install_pha_cron_tools`). `src/openharness/**` tidak diubah |
| Worker PHA | Flag tersembunyi `pha --task-worker` → runtime PHA penuh (prompt `build_pha_system_prompt`, auth `resolve_runtime_api_client`, extra skill/plugin roots, fallback chain). Spawn pakai `pha.exename.child_argv` (frozen-safe), bukan `sys.executable -m openharness` |
| Auth worker | `extra_env` = `build_inherited_env_vars()` + `PHA_WORKSPACE` + profile/model/base_url gateway; key via env (jangan via argv). Worker resolve via jalur gateway yang sama |
| Scope lain | `remote_trigger` (baca store openharness, bukan `cron.json` PHA), tool `config` (tulis settings openharness, bukan `gateway.json`), `enter_plan_mode` global → catat sebagai debt, bukan scope ini |

## 3. Audit singkat tools lain

| Status | Tools |
|---|---|
| OK | `cron_*` (override per-session), `skill`, `ask_user_question` (HITL WebUI), `bash/file_*/glob/grep/web_*`, `task_get/list/output/stop`, `send_message`, `team_*` |
| Debt (tunda) | `remote_trigger` selalu "not found" untuk job PHA; `config set` diabaikan gateway; `enter_plan_mode` no-op di WebUI + efek global |

## 4. Goals

- **G1** — `tool_search("subagent")` menemukan tool; panggilan `subagent` dari WebUI men-spawn worker dan hasilnya bisa dibaca via `task_output`.
- **G2** — Worker adalah PHA: prompt/skill/auth sama dengan sesi induk.
- **G3** — Tanpa ubah `src/openharness/**`; tanpa dependensi baru.

## 5. Non-goals

- Scope `remote_trigger`/`config`/`plan-mode` per-session; sidebar/progress khusus subagent di WebUI; migrasi worker lama yang sedang jalan.

## 6. Acceptance criteria

- **AC-27.1** `tool_search("subagent")` me-return `subagent`; tidak ada lagi `Unknown tool: subagent` di `gateway.log` untuk prompt subagent.
- **AC-27.2** Prompt WebUI "pakai subagent …" → task `local_agent` tercatat di `task_list`, `task_output` berisi jawaban worker (bukan error spawn/auth).
- **AC-27.3** argv worker = `child_argv(…, "--task-worker", …)` (frozen: `[pha.exe, --task-worker]`; source: `[python, -m, pha, --task-worker]`); `env` memuat `PHA_WORKSPACE`.
- **AC-27.4** Worker yang key-nya hanya ada di `gateway.json` tetap bisa memanggil LLM (tanpa fallback ke provider/credential lain).
- **AC-27.5** Suite baru + existing hijau; `ruff check` bersih.

## 7. Testing

1. `pytest tests/test_pha/test_subagent_alias.py tests/test_pha/test_subagent_worker.py -q` (baru: alias terdaftar, argv frozen/source, env PHA, override `agent`+`subagent` di sesi gateway, worker gateway-only-auth via fake client).
2. `ruff check src/pha/gateway/subagent_tools.py src/pha/gateway/runtime.py src/pha/runtime.py`.
3. Manual: WebUI → "pakai subagent worker untuk …" → `task_output` ada isi; cek `gateway.log` tanpa `Unknown tool: subagent`.

## 8. Referensi

- `src/openharness/tools/__init__.py:87`, `src/openharness/tools/agent_tool.py`, `src/openharness/swarm/subprocess_backend.py:83`, `src/openharness/swarm/spawn_utils.py`, `src/openharness/engine/query.py:1069`, `src/pha/gateway/cron_tools.py` (pola override), `src/pha/exename.py`, `src/pha/gateway/provider_auth.py`.

## 9. Follow-up 2026-09-16 — spawn default gagal di lapangan

Laporan sesi `@0-hasdi-agent`: spawn default gagal `No such option: --system-prompt`. Penyebab: `pha_worker_argv` meneruskan `--system-prompt`/`--append-system-prompt` (flag openharness-only, dipakai `build_inherited_cli_flags`) ke child `pha` yang tidak punya flag itu. Fix: worker argv hanya berisi flag yang `pha` terima (`--task-worker` + `--model`); role prompt definisi agent disuntik ke dalam task prompt (`[Worker role] … [Task] …`); param `system_prompt` dihapus dari `run_pha_task_worker` (worker selalu boot prompt PHA penuh sendiri). Workaround manual `command="python -m pha -p …"` tidak lagi diperlukan.
