![PHA — Personal Harness Agent](docs/assets/banner.png)

# PHA — Personal Harness Agent

Agent pribadi 24/7: chat **Telegram/Discord/Slack**, WebUI lokal, cron, memori. Engine OpenHarness vendored (`src/openharness`) + lapis PHA (`src/pha`).

![Python >= 3.11](https://img.shields.io/badge/python-%3E%3D3.11-blue)
![License MIT](https://img.shields.io/badge/license-MIT-green)
![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20Linux%20%7C%20macOS-lightgrey)

Oleh **Hasdi Sasandi** — [github.com/hasdisasandi/pha-agent](https://github.com/hasdisasandi/pha-agent).

> 🇬🇧 Read in [English](README.md).

## Start 5 menit

```bash
pip install pha-agent
pha init        # wizard: profil LLM, API key, channel
pha doctor      # semua ok / configured
pha gateway start --foreground  # Ctrl+C berhenti
```

Buka `http://localhost:18790/ui`. Selesai.

## WebUI

![WebUI PHA — chat, sesi, tool chip, todo, diff file](docs/assets/webui.png)

Chat multi-sesi dengan sidebar, streaming SSE, markdown, tool chip, todo card, diff file, panel Trace/Audit — plus halaman Setup, Skills, Agents, Tools, MCP, dan Cron. Satu file HTML tanpa build step.

## System One — keputusan cepat, opsional

![System One — lapisan keputusan cepat](docs/assets/system-one.png)

Lima keputusan kecil (triage pesan, cek bahaya bash, pilih agent, cek compact, nilai jawaban) biasanya memakan satu call LLM penuh. System One menggantinya dengan **1 call decision terstruktur** ke `laya-serve` lokal (`POST /v1/systemone`, Jev-compatible): ~33–140 ms, $0, data tidak keluar mesin. Conf rendah, timeout, atau server mati → **fallback diam-diam** ke jalur lama, tidak pernah error ke user; semuanya tercatat di Trace.

```bash
laya-serve --port 8001
```

```json
{ "decision_enabled": true, "decision_base_url": "http://127.0.0.1:8001" }
```

Mati secara default = zero diff. Detail: [`docs/prd/PRD-33-system-one.md`](docs/prd/PRD-33-system-one.md).

## Bisa apa lagi

- **Chat multi-channel** — Telegram, Discord, Slack. Satu agent semua channel.
- **WebUI lokal** — Chat/Setup/Skills/Agents/Tools/MCP/Cron + Trace/Audit per sesi. SSE streaming, markdown, mermaid, upload/paste/drag-drop, palette `/` + `@` file, picker dir (wcd), picker agent + effort, context bar, todo card, diff file.
- **Cron** — `pha cron add/list/enable/remove`, hasil masuk chat / sesi WebUI.
- **Memori** — `~/.pha/memory/`, `soul.md`/`user.md`/`identity.md`, awet antar sesi.
- **10 skill bawaan** — cron, github, image-generation, memory, weather, summarize, tmux, clawhub, skill-creator, update.
- **Fallback provider** — pindah otomatis saat utama gagal (`provider_fallbacks` di `gateway.json`, legacy `config.json`).
- **Desktop GUI** — `pha gui [--engine webview|chrome|tk]`, `pha gateway start --gui`.

## Harian

```bash
pha -p "ringkas laporan.md ini"
pha --continue -p "lanjutkan" / pha --resume <id> -p "ulangi detail"
pha memory add prefs "Jawab ringkas, Bahasa Indonesia" / pha memory list / pha soul show
pha provider status / pha provider use openrouter / pha provider set-key / pha provider set-model gpt-4o-mini
pha cron add pagi "0 8 * * *" --message "Susun rencana" --channel telegram --chat-id 123
pha gateway allow telegram 123456789
```

## Gateway

```bash
pha gateway start [--foreground|--verbose|--gui] / pha gateway status|stop|restart
pha gateway install-service / pha gateway uninstall-service   # auto-start boot
pha gateway start --workspace "D:\data\pha-kerja" --foreground --gui
```

| Flag | Arti |
|---|---|
| `--workspace W` | Root data (`soul.md`, `sessions/`, `gateway.json`, `cron.json`, `agents.json`, `session_groups.json`, `engine/`…). Tanpa `--cwd`, workspace = folder kerja agent. Default `~/.pha`. |
| `--cwd D` | Folder proyek agent. Selalu menang atas `--workspace`. |
| `--gui` / `--gui-engine chrome\|webview\|tk` | Buka WebUI desktop setelah start. |

## WebUI / API

Default `http://localhost:18790/ui` (`health_port`/`health_host` di `gateway.json`). API: `POST /v1/chat/completions`, `POST /v1/chat/stream` (SSE), `GET /health`, `GET /status`. Full endpoint: baca `src/pha/gateway/health.py`.

- **Trace/Audit** — tiap run tercatat (`~/.pha/logs/traces/`), audit global (`logs/audit.jsonl`, sender hash). Hapus per run / sesi.
- **Setup** — edit `gateway.json` dari browser (secret mask, validasi pydantic, banner bila butuh restart).
- **Effort** — `off|minimal|low|medium|high|xhigh|max`, global + per sesi. **Agent** — profil di `agents.json` (allowlist/mandatory tools+skills, effort default).
- `health_host` non-loopback wajib `webui_token`.

## Config

Satu file: `~/.pha/gateway.json` (atau `<workspace>/gateway.json`). Jangan commit.

```bash
pha config --show   # JSON efektif
pha config --full   # tulis SEMUA variabel
```

Knob utama: `model/api_key/base_url/headers`, `effort/max_tokens/context_window_tokens/temperature/extra_body/max_turns`, `permission_mode/group_permission_mode` (`auto|default|plan`), `vision_model` (default `google/gemini-2.5-flash`, `"off"` matikan), `disabled_tools/disabled_skills/allowed_tools/denied_tools/bash_allow_patterns/denied_commands`, `allowed_working_dirs`, `language` (`bahasa|english|japanese`), `webui_token_ttl_hours`, `decision_*`. Tabel + resep: [`docs/gateway-config.md`](docs/gateway-config.md).

Fallback (`gateway.json`, `config.json` legacy menang bila dua-duanya ada):

```json
{ "provider_fallbacks": [{ "model": "gpt-4o-mini", "base_url": "https://api.openai.com/v1", "api_key_env": "OPENAI_API_KEY", "transport": "requests", "verify_ssl": false }] }
```

Docker: `docker compose up -d` → `http://localhost:8787/ui` (mapping compose; native default 18790). Exe: `python scripts/build_exe.py` → `dist\pha.exe`.

## Security

- `allow_from` kosong = tolak semua. `pha gateway allow <channel> <id>`.
- `permission_mode: default` = tool tulis/eksekusi popup di WebUI, ditolak di channel. `plan` = mode rencana.
- Secret (`gateway.json`, `config.json`, `cron.json`, `*.env`) di `.gitignore`.

## Struktur

```
src/openharness/   engine vendored (upstream + patch PHA)
src/pha/           lapisan produk: workspace, persona, gateway, cron, WebUI, agents, trace
  gateway/session_pool.py|stream_updates.py|snapshots.py  runtime split
  gateway/webui/   frontend tanpa build (satu file HTML)
  skills/          10 skill bawaan
tests/test_pha/    ~385 test PHA (+ suite engine upstream)
docs/              gateway-config.md + docs/prd/ (PRD-1..48)
scripts/           build_exe.py + checker dev
```


## Dev

```bash
pip install -e ".[dev]"
python -m pytest tests/test_pha -q
ruff check src/pha
```

16 test upstream gagal Windows murni (shell Unix) — bukan regresi. `tests/test_pha` hijau.

## Troubleshoot

| Gejala | Obat |
|---|---|
| `missing auth` | `pha provider set-key` / `use <profil>` |
| Port dipakai | `pha gateway status`, ganti `health_port`, restart |
| Channel diam | `allow_from` terisi? allow + restart |
| Zscaler/TLS-intercept | `transport: "requests"`, `verify_ssl: false` |
| Cron tak sampai | `pha cron list` enabled? gateway jalan? cek Trace |

## Lisensi

MIT — [`LICENSE`](LICENSE). Engine muat kode upstream OpenHarness (MIT).
