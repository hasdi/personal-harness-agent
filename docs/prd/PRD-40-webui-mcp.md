# PRD-40 — Halaman MCP: Kelola Server + Koneksi Live

| | |
|---|---|
| Feature | `mcp_servers` di `gateway.json` + wiring pool + halaman MCP WebUI (list/tambah/ubah/hapus/enable/test) — common practice ala Claude Desktop |
| Depends on | PRD-12 (i18n), PRD-13 (SPA), PRD-29 (pola halaman Tools), PRD-36 (pola ramah), PRD-39 (error-map, empty-state) |
| Files | `pha/gateway/{models,service,session_pool}.py`, `pha/gateway/health.py`, `pha/gateway/webui/{markup,spa_js,i18n_js}.py`, `tests/test_pha/test_mcp_webui.py`, `docs/gateway-config.md`, `docs/prd/README.md` |
| Status | Implemented (2026-10-14) |

## 1. Ringkasan

MCP (Model Context Protocol) adalah cara standar menyambungkan agent ke tool eksternal (filesystem, database, API) — common practice di Claude Desktop/CLI. Kondisi existing: MCP hanya hidup di CLI openharness via `settings.mcp_servers` (`settings.json`); `PhaSessionRuntimePool.get_bundle` tidak meneruskan config MCP ke `build_runtime` (`session_pool.py:386-411`), sehingga server MCP tak pernah tersambung di gateway; WebUI tak punya halaman MCP (PRD-13 §Future menyebut "Plugins/MCP" sebagai halaman yang belum ada). PRD ini menutup gap itu: config `mcp_servers` pindah/bertambah di `gateway.json`, pool meneruskannya ke runtime, halaman MCP mengelola lifecycle + test koneksi.

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Bentuk config | `GatewayConfig.mcp_servers: dict[str, McpServerConfig]` (reuse tipe `openharness.mcp.types`: `stdio`/`http`/`ws`); `McpServerConfig = McpStdioServerConfig \| McpHttpServerConfig \| McpWebSocketServerConfig`. Validasi pydantic; `gateway.json` lama (tanpa key) tetap load (default `{}`) |
| Enable/disable | Flag `enabled: bool` per server di lapisan PHA (bukan tipe upstream — jangan ubah `src/openharness/**`); disabled = tidak diteruskan ke `build_runtime` |
| Secret | `env` stdio + `headers` http boleh berisi secret → mask di `GET` (pola `_CONFIG_MASK`), kosong/mask saat save = pertahankan (pola `save_webui_config`) |
| Wiring pool | Pool teruskan `mcp_servers` (yang enabled) ke `build_runtime` via override settings — tanpa ubah signature `openharness/ui/runtime.build_runtime` (terlarang: vendored). Cara: `load_settings().merge_cli_overrides(mcp_servers=...)` di dalam pool sebelum panggil `_compat.build_runtime`, atau env-override yang dibaca `load_settings`. Pilih yang paling kecil saat implementasi |
| Status live | `GET /v1/mcp` kembalikan config (mask) + status koneksi per server dari bundle aktif bila ada (`bundle.mcp_manager.list_statuses()`), else `state: "pending"` + catatan "perlu restart" |
| Test koneksi | `POST /v1/mcp/test` coba konek satu server (timeout 15 dtk, `connect_all` pada manager sekali-pakai lalu `close`) → `{ok, tools, resources}` atau `{error ramah}`. Tanpa simpan config |
| Restart | Tambah/ubah/hapus/enable = restart agar koneksi live tersambung ulang (masuk `_CONFIG_RESTART_FIELDS` via key `mcp_servers`); halaman tampilkan banner restart (reuse pola PRD-36) |
| Halaman | View `mcp` di navbar setelah Tools: list (nama, transport, target, badge status, jumlah tools) + tombol Tambah/Ubah/Test/Hapus/Enable + empty-state + CTA + search + konfirmasi hapus sebut nama. Preset umum (filesystem, sqlite, github, postgres) sebagai tombol isi-cepat — tanpa dep baru, hanya template config |
| Tanpa | OAuth flow MCP (`McpAuthTool` tetap via chat bila server butuh); auto-reconnect live tanpa restart; log kredensial |

## 3. Call-site

| # | Lokasi | Guna | Fallback |
|---|---|---|---|
| 1 | `models.py` `GatewayConfig` | Field `mcp_servers` + validasi | `{}` bila tak ada (kompat) |
| 2 | `service.py` | CRUD + mask secret + status + test koneksi | Error ramah bila validasi gagal |
| 3 | `session_pool.py` `get_bundle` | Teruskan enabled-servers ke runtime | Tanpa MCP bila kosong (existing) |
| 4 | `health.py` | Route `/v1/mcp*` via provider injection (pola tools/agents) | `503` bila provider mati |
| 5 | `webui/{markup,spa_js,i18n_js}` | View + list + editor + test + i18n 3 bahasa | JSON mentah di advanced bila editor gagal |
| 6 | `config.py`/`gateway-config.md` | Field baru terdokumentasi | — |

## 4. Goals

- **G1** — User awam tambah server MCP umum (stdio/http) tanpa edit JSON manual.
- **G2** — Status koneksi + test terlihat sebelum dipakai agent; tools `mcp__*` muncul di halaman Tools setelah restart.
- **G3** — Secret tak pernah bocor ke browser mentah; tanpa dep/build baru; `src/openharness/**` tak diubah.

## 5. Non-goals

- OAuth/autentikasi interaktif MCP; hot-reload koneksi tanpa restart; marketplace server.
- Ubah tipe upstream `McpServerConfig`; ubah signature `build_runtime`.

## 6. Acceptance criteria

- **AC-40.1** Suite + lint hijau; tanpa dep/build baru; string baru 3 bahasa; smoke HTML `external == []`.
- **AC-40.2** Tambah server stdio valid → tersimpan di `gateway.json`, status `pending` + banner restart; setelah restart tersambung (`connected`, tools ter-list).
- **AC-40.3** Test koneksi ke server salah → error ramah (bukan traceback); secret (`env`/`headers`) termask di GET dan tak berubah bila dikirim mask.
- **AC-40.4** Disable server → tools `mcp__<nama>__*` hilang dari Tools setelah restart; hapus konfirmasi sebut nama.
- **AC-40.5** `gateway.json` lama tanpa `mcp_servers` tetap load; config invalid ditolak dengan pesan jelas.

## 7. Testing

1. `pytest tests/test_pha/test_mcp_webui.py -q` (CRUD, mask, validasi, status, test-koneksi fake-stdio).
2. `ruff check src/pha` bersih + `node --check` bundle.
3. Manual: tambah `filesystem` via preset → restart → tools `mcp__filesystem__*` ada di Tools → test koneksi OK.
