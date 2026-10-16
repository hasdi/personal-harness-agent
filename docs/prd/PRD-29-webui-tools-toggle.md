# PRD-29 — WebUI: Tools Enable/Disable + Page Tools

| | |
|---|---|
| Feature | Menu Tools bisa enable/disable per tool + page sendiri (klon Skills), denylist `disabled_tools`, filter runtime nyata |
| Depends on | PRD-15 (pola skills CRUD + toggle) |
| Files | `pha/gateway/{models,service,health,runtime}.py`, `pha/gateway/webui/{markup,spa_js,app_js,i18n_js,style}.py`, `tests/test_pha/test_webui_tools.py` (baru) |
| Status | Draft (2026-09-24) |

## 1. Kondisi sekarang

| Hal | Status | Bukti |
|---|---|---|
| Toggle skills | Ada (`disabled_skills` + `POST /v1/skills/enable`) | `service.py:1180-1191`, `health.py:775-841` |
| Toggle tools | Tidak ada (`grep disabled.*tool\|list_tools` nihil) | `gateway/*` |
| Enforcement skills | Cuma flag UI — runtime tak filter (`extra_skill_dirs` mentah) | `service.py:1105` vs `runtime.py:258-274` |
| Page Tools | Tidak ada (nav cuma chat/setup/skills/cron) | `markup.py:9-14`, `spa_js.py:12` |
| Registry tools | 39 builtin + MCP/plug dinamis, tanpa `unregister` | `tools/__init__.py:48`, `tools/base.py`, `ui/runtime.py:333-336` |

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Model | `GatewayConfig.disabled_tools: list[str]` sebelah `disabled_skills` (`models.py:57`); label `_CONFIG_LABELS` pola `service.py:192` |
| Service | `list_tools()/get_tool()/set_tool_enabled()` tiru `service.py:1105-1191`; slug = `lower().strip()` (nama tool udah snake_case) |
| HTTP | `ToolsProvider/ToolToggler` pola `health.py:86-91`; `GET /v1/tools`, `POST /v1/tools/enable` (validasi `enabled must be bool` sama kayak skills); wire `service.py:1603-1607`; update header doc `health.py:1-31` |
| UI | Klon skills: nav `button[data-view=tools]` + `#view-tools` (`markup.py:152-166`); `toolsCache/renderToolsView/List/Detail` + toggle POST (`spa_js.py:73-380`); i18n 3 bahasa; grid `.tool-card` (`style.py:412-426`); tambah ke `APP_VIEWS` + `refreshActiveView` + `state.view` |
| Enforcement | Satu choke point `runtime.py:_install_pha_session_tools:137` + `get_bundle:239,290`: prune registry pasca-`build_runtime` (skip/del bila `name in disabled`), termasuk hasil `install_pha_cron_tools` + `install_pha_agent_tools`. Prune dict langsung — registry tak punya `unregister` (`tools/base.py`), tanpa method baru, tanpa ubah `src/openharness/**` (prinsip PRD-0 G4) |
| MCP/plug | `list_mcp_resources/read_mcp_resource/mcp__*` (`tools/__init__.py:93-97`) + plugin tools ikut ke-list; prefix `mcp__` bisa di-disable per-server |

## 3. Goals

- **G1** — Tiap tool bisa on/off dari WebUI, persist, berlaku sesi baru tanpa restart.
- **G2** — Disable beneran matikan tool di runtime, bukan cuma badge.
- **G3** — Tanpa ubah `src/openharness/**`; tanpa dependensi baru.

## 4. Non-goals

- Per-session tool override (itu PRD-30); paginasi/virtualisasi; versioning.

## 5. Acceptance criteria

- **AC-29.1** Disable `bash` → `list_tools` flag false, persist `gateway.json`, sesi baru `registry.get("bash")` None, chat panggil → error jelas bukan hang.
- **AC-29.2** Refresh WebUI pilihan bertahan; enable lagi balik normal tanpa restart.
- **AC-29.3** `POST /v1/tools/enable` tanpa bool → 400; nama unknown → 404.
- **AC-29.4** Suite baru + existing hijau; `ruff check src/pha`; `node --check` JS.

## 6. Testing

1. `pytest tests/test_pha/test_webui_tools.py tests/test_pha/test_webui_crud.py -q` (baru: roundtrip toggle + filter runtime via fake bundle).
2. `ruff check src/pha/gateway/`.
3. Manual: disable `bash` → chat "pakai bash …" gagal jelas; enable → normal.
