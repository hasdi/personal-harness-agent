# PHA — Personal Harness Agent

Asisten AI pribadi yang jalan 24/7: chat via **Telegram / Discord / Slack**,
WebUI lokal, cron terjadwal, dan memori personal. Dibangun solo oleh
**Hasdi Sasandi** di atas engine OpenHarness (vendored, `src/openharness`)
plus lapisan produk PHA (`src/pha`).

![Python >= 3.11](https://img.shields.io/badge/python-%3E%3D3.11-blue)
![License MIT](https://img.shields.io/badge/license-MIT-green)
![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20Linux%20%7C%20macOS-lightgrey)

## Cara tercepat (5 menit)

```bash
pip install pha-agent            # atau: pip install -e .
pha init                         # wizard: profil LLM, API key, channel
pha doctor                       # pastikan semua "ok" / "configured"
pha gateway start --foreground   # jalan di console ini, Ctrl+C berhenti
```

Buka `http://localhost:18790/ui` → chat dengan agent-mu. Selesai.

## Yang bisa dilakukan PHA

- **Chat multi-channel** — Telegram, Discord, Slack. Satu agent, semua channel.
- **WebUI lokal** — chat + trace run + audit + setup + skills + cron, tanpa build step.
- **Cron proaktif** — "ingatkan saya tiap jam 8 pagi", hasilnya masuk ke chat.
- **Memori personal** — preferensi & konteks awet antar sesi (`~/.pha/memory/`).
- **10 skill bawaan** — cron, github, image-generation, memory, weather,
  summarize, tmux, clawhub, skill-creator, update.
- **Fallback provider** — pindah otomatis ke provider cadangan bila utama gagal.
- **Desktop GUI** — `pha gui` membuka WebUI sebagai jendela desktop.

## Instalasi

Butuh **Python >= 3.11**. Opsional: Node.js >= 18 (hanya untuk TUI React —
bila tidak ada, `pha -p` dan gateway tetap jalan penuh).

```bash
# dari PyPI
pip install "pha-agent[gui]"     # [gui] opsional, untuk `pha gui --engine webview`

# dari source
git clone https://github.com/hasdisasandi/pha-agent
cd pha-agent
pip install -e ".[dev]"          # [dev] = pytest + ruff
```

> Windows: pakai PowerShell. Docker & `pha.exe` single-file ada di bawah.

## Setup LLM provider

Semua kredensial tinggal di **satu file**: `~/.pha/gateway.json`
(atau `<workspace>/gateway.json` bila pakai `--workspace`).
Jangan pernah commit file ini — sudah di `.gitignore`.

```bash
pha provider status              # lihat profil + status auth
pha provider use openrouter      # ganti profil aktif
pha provider set-key             # isi API key (input tersembunyi)
pha provider set-model gpt-4o-mini
pha provider set-url https://api.openai.com/v1
```

Atau edit langsung:

```json
{ "provider_profile": "openai-compatible", "model": "gpt-4o-mini",
  "api_key": "sk-...", "base_url": "https://api.openai.com/v1" }
```

Cek kesiapan kapan saja: `pha doctor`.

## Pemakaian harian

```bash
# sekali jalan (scripting, tanpa interaktif)
pha -p "ringkas file laporan.md ini"

# lanjutkan sesi terakhir / sesi tertentu
pha --continue -p "lanjutkan"
pha --resume <session-id> -p "ulangi dengan detail"

# memori & persona
pha memory add prefs "Jawab ringkas, Bahasa Indonesia"
pha memory list
pha soul show

# cron: pesan proaktif tiap hari jam 8
pha cron add pagi "0 8 * * *" --message "Susun rencana hari ini" \
  --channel telegram --chat-id 123456789
pha cron list
pha cron enable pagi --disable
pha cron remove pagi
```

## Gateway (jalan 24/7)

```bash
pha gateway start                # background (detached)
pha gateway start --verbose      # background + console log live
pha gateway start --foreground   # jalan di console ini (Ctrl+C berhenti)
pha gateway start --gui          # background + buka jendela desktop
pha gateway status               # pid, sesi, channel, error terakhir
pha gateway stop
pha gateway restart

# pairing channel: izinkan pengirim (allowlist, kosong = tolak semua)
pha gateway allow telegram 123456789

# jalan sebagai service OS (auto-start saat boot)
pha gateway install-service
pha gateway uninstall-service

# workspace terpisah (mis. isolasi data kerja vs pribadi)
pha gateway start --workspace "D:\data\pha-kerja" --foreground --gui
```

| Flag | Arti |
|---|---|
| `--workspace W` | Root data PHA (`soul.md`, `sessions/`, `gateway.json`, …). Default `~/.pha`. Tanpa `--cwd`, workspace juga jadi folder kerja agent. |
| `--cwd D` | Folder proyek yang dikerjakan agent. Selalu menang atas `--workspace`. |
| `--gui` / `--gui-engine chrome\|webview\|tk` | Buka WebUI desktop setelah gateway start. |

## WebUI

Aktif by default di `http://localhost:18790/ui`
(`gateway.json`: `health_port`, `health_host`). Satu halaman, empat view:
**Chat / Setup / Skills / Cron** — tanpa reload.

- Chat: streaming SSE, markdown + copy code, diagram mermaid, upload file,
  sidebar sesi (cari/rename/hapus), palet `/` command & `@` file,
  ganti working directory per sesi, 3 bahasa UI.
- Tab **TRACE/AUDIT** per sesi: tiap run tercatat (tool, error, durasi),
  audit tool global (sender di-hash). Bisa hapus per run / per sesi.
- Tab **SETUP**: edit `gateway.json` dari browser (secret ter-mask,
  validasi pydantic, banner bila butuh restart).
- Run **detached**: refresh browser tidak memutus task; buka lagi →
  auto re-attach + tombol STOP.

API OpenAI-compatible: `POST /v1/chat/completions`
(field `user` = session key), `GET /health`, `GET /status`.
Detail endpoint: [`docs/gateway-config.md`](docs/gateway-config.md).

> `health_host` non-loopback (mis. `0.0.0.0`) **wajib** set `webui_token`.
> Lihat [Security](#security).

## Desktop GUI & exe

```bash
pha gui                          # auto: webview → chrome app → tk
pha gui --engine chrome          # jendela app browser, tanpa dependensi
pha gui --start-gateway          # start gateway dulu bila belum jalan
pip install "pha-agent[gui]"     # untuk engine webview (pywebview)
```

Single-file Windows exe (PyInstaller):

```bash
.venv\Scripts\pip install pyinstaller
.venv\Scripts\python scripts\build_exe.py   # → dist\pha.exe
pha.exe gateway start --workspace "D:\data\pha" --verbose --gui
```

## Docker

```bash
docker compose run --rm pha pha init --no-interactive
# edit volume: gateway.json / config.json / cron.json
docker compose up -d             # WebUI: http://localhost:8787/ui
```

## Konfigurasi lanjut

```bash
pha config --show                # tampilkan config efektif (JSON)
pha config --full                # tulis gateway.json berisi SEMUA variabel
pha init --no-interactive        # init workspace + config lengkap
```

Referensi semua variabel + resep (Telegram, HITL, multi-proyek, proxy
korporat Zscaler): [`docs/gateway-config.md`](docs/gateway-config.md).

Fallback provider (pindah otomatis saat utama gagal) di `~/.pha/config.json`:

```json
{ "provider_fallbacks": [
  { "model": "gpt-4o-mini", "base_url": "https://api.openai.com/v1",
    "api_key_env": "OPENAI_API_KEY" } ] }
```

## Security

- **Allowlist channel**: `allow_from` kosong = tolak semua.
  `pha gateway allow <channel> <sender-id>`.
- **Permission mode**: `auto` (semua tool jalan) | `default`
  (tool tulis/eksekusi butuh konfirmasi — popup di WebUI, ditolak di channel)
  | `plan` (mode rencana). Grup default `default` (read-only).
- **Audit**: tiap eksekusi tool tercatat di `~/.pha/logs/audit.jsonl`
  (sender di-hash).
- **Token WebUI**: set `webui_token` bila expose ke jaringan.
- **Secret tidak pernah ke Git**: `gateway.json`, `config.json`, `cron.json`,
  `*.env` sudah di `.gitignore`. Cek ulang sebelum `git push`.

## Struktur repo

```
src/openharness/   engine vendored (upstream OpenHarness + patch PHA)
src/pha/           lapisan produk: workspace, persona, gateway, cron, WebUI
  gateway/webui/   frontend WebUI tanpa build step (satu file HTML)
  skills/          10 skill bawaan
tests/test_pha/    248 test khusus PHA (+ suite engine upstream)
docs/              gateway-config.md + docs/prd/ (24 PRD WebUI, Bahasa Indonesia)
scripts/           build_exe.py + checker dev
Dockerfile / docker-compose.yml / pha.spec   distribusi
```

Engine upstream: [OpenHarness](https://github.com/openharness/openharness) ·
inspirasi lapisan produk: [nanobot](https://github.com/nanobot/nanobot).
Detail patch engine: [`CHANGELOG-PHA.md`](CHANGELOG-PHA.md).

## Development

```bash
pip install -e ".[dev]"
.venv\Scripts\python -m pytest tests/test_pha -q     # suite PHA (~248 test)
.venv\Scripts\python -m pytest tests -q              # semua (beberapa perlu POSIX)
.venv\Scripts\ruff check src/pha                     # lint lapisan PHA
```

Catatan: 16 test upstream gagal di Windows murni karena shell Unix
(`printf`, `while read`) — bukan regresi PHA. Lihat CHANGELOG-PHA
bagian bawah. `tests/test_pha` hijau penuh.

Progres & keputusan desain: [`DEVLOG.md`](DEVLOG.md) (Bahasa Indonesia).

## Troubleshooting

| Gejala | Obat |
|---|---|
| `pha doctor` → "missing auth" | `pha provider set-key` / `use <profil>` |
| Gateway tidak start / port dipakai | `pha gateway status`, ganti `health_port`, `pha gateway restart` |
| Channel tidak merespons | `allow_from` terisi? (`pha gateway allow …`), lalu restart |
| `Network error: Connection error` terus | restart gateway; cek proxy/`base_url` |
| Kantor pakai Zscaler / TLS-intercept | fallback `transport: "requests"`, `verify_ssl: false` (lihat docs) |
| WebUI basi setelah update | hard refresh (server sudah kirim `no-store`) |
| Cron tidak sampai | `pha cron list` → enabled? gateway jalan? cek tab TRACE |

## Lisensi

MIT — lihat [`LICENSE`](LICENSE). Engine memuat kode upstream OpenHarness
(MIT, © OpenHarness Contributors); lapisan produk terinspirasi nanobot
(MIT, © Xubin Ren dkk).
