# Panduan Konfigurasi `gateway.json`

Semua variabel PHA gateway berada di satu file: `~/.pha/gateway.json`
(atau `<workspace>/gateway.json` bila memakai `--workspace`). Cara paling
mudah meng-generate/memperbarui file dengan **semua variabel**:

```bash
pha config --full     # tulis gateway.json berisi SEMUA variabel (nilai existing dipertahankan)
pha config --show     # print config efektif sebagai JSON
```

Atau edit langsung dari WebUI: buka `http://localhost:18790/ui` → panel **SETUP**
(kanan bawah) → ubah → **SAVE**. Field berpilihan sudah memakai select box;
secret (`api_key`, `vision_api_key`, `webui_token`) ditampilkan ter-mask —
kosongkan berarti tidak diubah.

> Setelah mengubah field yang ditandai 🔁 (hanya terbaca saat start), restart gateway:
> `pha gateway restart` (atau stop + start).

## Daftar variabel

| Variabel | Tipe | Default | Opsi | Keterangan |
|---|---|---|---|---|
| `provider_profile` | string | `claude-api` | profil auth terpasang (lihat `pha doctor`) | Profil kredensial LLM yang aktif. 🔁 |
| `model` | string\|null | `null` | — | Override model, mis. `gpt-4o-mini`. Kosong = default profil (atau preset aktif bila diset). |
| `model_preset` | string\|null | `null` | nama di `model_presets` | Preset model aktif gateway. Kosong = default. Hot-apply (tanpa restart). |
| `model_presets` | object | `{}` | mis. `{"hemat": {"model": "gpt-4o-mini"}}` | Katalog preset inline (PRD-34). Hot-apply (tanpa restart). |
| `api_key` | string | `""` | — | API key LLM (disimpan lokal; ter-mask di WebUI). |
| `base_url` | string\|null | `null` | — | Base URL API kompatibel-OpenAI/Anthropic. |
| `effort` | string\|null | `null` | `off` \| `minimal` \| `low` \| `medium` \| `high` \| `xhigh` \| `max` | Default global tiap sesi; bisa dioverride per sesi dari dropdown chat header. |
| `permission_mode` | string | `auto` | `auto` \| `default` \| `plan` | `auto` = semua tool diizinkan; `default` = tool sensitif butuh konfirmasi (popup di WebUI, ditolak di channel); `plan` = mode rencana. |
| `group_permission_mode` | string | `default` | sama di atas | Mode permission khusus chat grup. |
| `enabled_channels` | list | `[]` | `telegram`, `discord`, `slack`, … | Channel chat yang aktif. |
| `session_routing` | string | `chat-thread` | — | Strategi routing session channel. |
| `send_progress` | bool | `true` | — | Kirim update progress ke channel. |
| `send_tool_hints` | bool | `true` | — | Kirim hint "Using tool" ke channel. |
| `allow_remote_admin_commands` | bool | `false` | — | Izinkan slash command admin dari channel remote. |
| `allowed_remote_admin_commands` | list | `[]` | contoh: `permissions`, `plan` | Whitelist command admin remote (butuh flag di atas). |
| `log_level` | string | `INFO` | `DEBUG` \| `INFO` \| `WARNING` \| `ERROR` | Level log gateway. 🔁 |
| `health_port` | int\|null | `18790` | — | Port server WebUI/API; `null` = matikan. |
| `health_host` | string | `127.0.0.1` | — | Bind address. Non-loopback **wajib** set `webui_token`. |
| `webui_token` | string | `""` | — | Token WebUI/API (`Authorization: Bearer` / `?token=`). 🔁 |
| `vision_model` | string | `google/gemini-2.5-flash` | — | Fallback bila provider menolak gambar native; `"off"` = matikan. |
| `vision_api_key` | string | `""` | — | API key vision (kosong = pakai `api_key`). Ter-mask. |
| `vision_base_url` | string | `""` | — | Base URL khusus vision. |
| `decision_enabled` | bool | `false` | — | System One aktif (PRD-33). Mati = perilaku lama persis. |
| `decision_base_url` | string | `http://127.0.0.1:8001` | — | Endpoint `laya-serve` (`POST /v1/systemone`). |
| `decision_api_key` | string | `""` | — | API key System One (kosong = tanpa auth). Ter-mask. |
| `decision_model` | string | `laya` | — | Checkpoint decision. |
| `decision_timeout` | float | `3.0` | — | Timeout detik; gagal = fallback diam-diam. |
| `decision_conf_high` | float | `0.8` | 0–1 (WebUI: 0–100%) | Keyakinan ≥ ini = auto-execute tanpa konfirmasi. Harus ≥ `decision_conf_mid`. |
| `decision_conf_mid` | float | `0.5` | 0–1 (WebUI: 0–100%) | Keyakinan ≥ ini = jalur hati-hati (tool sensitif tetap konfirmasi); di bawah ini = eskalasi ke LLM/human. Harus ≤ `decision_conf_high`. |
| `language` | string | `bahasa` | `bahasa` \| `english` \| `japanese` | Bahasa antarmuka WebUI. Hot-apply (tanpa restart). |
| `allowed_working_dirs` | list | `[]` | — | Daftar folder yang boleh dipilih sebagai working directory (wcd) di WebUI; kosong = hanya cwd gateway. |
| `permission_timeout_seconds` | int | `120` | — | Batas tunggu popup permission (PRD-6) sebelum auto-deny. |
| `sensitive_path_deny` | list | `[]` | cth `["*/rahasia/*"]` | Pola path tambahan yang selalu ditolak agent (semua mode, tanpa confirm). Melengkapi pola bawaan: `*/.ssh/*`, `*.aws`, `gcloud/azure/gnupg/docker/kube`, `.openharness`, `gateway.json`, `config.json`, `cron.json`, `.env`. Berlaku untuk file DAN direktori (pola dir seperti `*/rahasia/*` cocok dengan root dir-nya); basename literal ikut disapu dari command shell (`cat`). Hot-apply (tanpa restart). |
| `channel_configs` | object | `{}` | — | Konfigurasi per channel, mis. `{"telegram": {"allow_from": [123]}}`. |
| `mcp_servers` | object | `{}` | stdio/http/ws (PRD-40) | Server MCP eksternal. Kelola lewat halaman **MCP** (tambah/test/enable). Secret (`env`/`headers`) ter-mask. 🔁 |

## Resep setup

### 1. Minimal (LLM + WebUI saja)

```json
{
  "provider_profile": "openai-compatible",
  "model": "gpt-4o-mini",
  "api_key": "sk-...",
  "base_url": "https://api.openai.com/v1"
}
```

Buka `http://localhost:18790/ui` — semua field lain memakai default.

### 2. WebUI aman di jaringan lokal

```json
{ "health_host": "0.0.0.0", "webui_token": "token-rahasia-ku" }
```

Akses: `http://<ip>:18790/ui` → masukkan token saat diminta.

### 3. Telegram

```json
{
  "enabled_channels": ["telegram"],
  "channel_configs": { "telegram": { "allow_from": [123456789] } },
  "group_permission_mode": "default"
}
```

`allow_from` = user id yang boleh memakai bot (kosong = tolak semua).

### 4. Human-in-the-loop (konfirmasi tool sensitif)

```json
{ "permission_mode": "default", "permission_timeout_seconds": 120 }
```

Setiap tool write/execute memunculkan popup **PERMISSION REQUIRED** di WebUI
(Allow/Deny, auto-deny 120s tanpa jawaban). Mode `auto` = tanpa popup.

### 5. Multi-proyek (wcd)

```json
{ "allowed_working_dirs": ["D:\\Projek\\A", "D:\\Projek\\B"] }
```

Selector working directory muncul di chat header WebUI; agent bekerja di
folder terpilih.

### 6. Fallback gambar (provider non-vision)

```json
{ "vision_model": "google/gemini-2.5-flash" }
```

Gambar selalu dikirim native dulu; bila provider menolak, vision model
menerjemahkannya menjadi teks sekali lalu request diulang. Set `"off"` untuk matikan fallback.

### 7. System One (decision cepat, PRD-33 + action gate PRD-35)

```json
{
  "decision_enabled": true,
  "decision_base_url": "http://127.0.0.1:8001",
  "decision_model": "laya"
}
```

Jalankan `laya-serve` lokal dulu (Jev-compatible `POST /v1/systemone`).
Mati default = zero diff perilaku; nyala = triage pesan, cek bahaya bash,
hint agent, gate compact, skor groundedness — semua tanpa LLM call.
Gagal/timeout/conf rendah = fallback diam-diam ke jalur lama.

PRD-35 (tanpa variabel baru): tiap tool call dinilai **sebelum jalan**
(state kaya: cwd, input, session) — danger + conf tinggi = force-confirm
(HITL); hard-deny deterministik (`*/.ssh/*`) menang tanpa HTTP call.
Output tool dinilai **sesudah jalan** — secret/sensitif + conf tinggi =
di-redact sebelum disajikan. Verdict tercatat di tab Trace
(`system_one.decision`: `permission` + `tool_output`).

Ambang yang sama berlaku di semua titik System One via helper
`gate()` (`services/system_one.py`): `permission`, `tool_output`,
`compact`, `triage`, `agent_hint`, `judge`. Di bawah Batas confirm =
**eskalasi**: hasil dibuang (`dropped: true` di trace), caller fallback
seolah System One mati — System One yang tidak yakin tidak mengotorisasi
apa pun. `confirm`–`auto` = dipakai tapi ditandai `branch: confirm`;
`>=` Batas auto = dipakai penuh. `agent_hint` yang dulu hardcode `0.8`
kini ikut Batas confirm dari config.

### 8. Jaringan korporat / TLS interception (fallback `requests`)

Fallback provider dikonfigurasi di `~/.pha/config.json`
(`provider_fallbacks`). Tiap kandidat menerima dua field tambahan:

| Field | Default | Keterangan |
|---|---|---|
| `transport` | `"sdk"` | `"sdk"` = klien `openai` streaming (default); `"requests"` = klien non-streaming berbasis `requests` (patch P4). |
| `verify_ssl` | `true` | `false` menonaktifkan verifikasi sertifikat TLS — untuk proxy korporat yang meng-intercept TLS (mis. Zscaler). Hanya berlaku saat `transport: "requests"`. |

```json
{
  "provider_fallbacks": [
    {
      "model": "gpt-4o-mini",
      "base_url": "https://api.openai.com/v1",
      "api_key_env": "OPENAI_API_KEY",
      "transport": "requests",
      "verify_ssl": false
    }
  ]
}
```

Kandidat tanpa kedua field itu berperilaku persis seperti sebelumnya.

## Catatan

- Semua variabel di atas juga dapat dilihat/diubah live lewat tab **SETUP**;
  nilai validasi pydantic — salah tipe ditolak dengan pesan jelas.
- Snapshot session tersimpan di `~/.pha/sessions/`, trace di
  `~/.pha/logs/traces/`, audit di `~/.pha/logs/audit.jsonl`.
