# PRD-36 — WebUI Setup + Cron Ramah End User

| | |
|---|---|
| Feature | Setup Basic + Lanjutan + search + editor list; Cron preset + builder + preview live + picker target — tanpa dep/build baru |
| Depends on | PRD-10 (panel SETUP), PRD-13 (SPA), PRD-16 (cron CRUD), PRD-12 (i18n), PRD-29 (pola halaman) |
| Files | `pha/gateway/webui/{spa_js,app_js,i18n_js,markup,style}.py`, `pha/gateway/{service,cron}.py`, `pha/gateway/health.py`, `openharness/services/cron.py` (reuse), `tests/test_pha/test_webui_crud.py` |
| Status | Implemented (2026-10-10) |

## 1. Ringkasan

Setup menampilkan ~47 field sekaligus dengan 10+ field JSON mentah; Cron pakai ketik cron 5-field (default `* * * * *` = tiap menit) plus `chat_id` kosong yang pasti gagal save. PRD ini meramahkan keduanya murni di lapisan UI + fasad tipis: Setup dapat seksi Basic (~9 field populer) + search + editor list + konfirmasi diff restart + dirty-guard; Cron dapat preset + builder + preview live 3 run + picker target (sesi WebUI + channel) + list manusiawi + countdown. Tanpa field `gateway.json` baru, endpoint lama kompatibel.

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Setup | Basic + Lanjutan (9 field populer di atas), grup collapsible, search client-side (nama+label+help). Tanpa wizard — rombak besar, non-goal |
| Field array string | Editor list add/remove (`enabled_channels`, `allowed/denied/disabled_tools`, `disabled_skills`, `bash_allow_patterns`, `denied_commands`). Objek (`channel_configs`, `model_presets`) tetap JSON + validasi ramah |
| Secret | Show/hide 👁 (nilai tetap mask di server) |
| Save restart | Konfirmasi diff sebelum save (nilai lama → baru per field restart) |
| Dirty | Guard pindah view (modal Buang/Batal); reload toolbar tetap reset (eksplisit) |
| Cron jadwal | 5 preset + builder (jam/menit/pola) + preview live debounce 400ms, 3 next-run |
| Cron target | Picker sesi WebUI + channel aktif + "Gunakan sesi ini" (prefill sesi aktif); manual di advanced. Fasad `GET /v1/cron/targets` reuse `list_webui_sessions` + `enabled_channels` |
| Cron list | Label manusiawi (pola umum, sisanya mentah) + countdown + target nama sesi; toggle ber-toast; hapus tampilkan jadwal/target; run-now tawarkan buka sesi |
| Error | Pesan Indonesia tanpa flag CLI (`--channel` → "Pilih tujuan kirim dulu") |
| Tanpa knob baru | Tanpa field `gateway.json` baru; payload `POST/PUT /v1/cron` + `POST /v1/config` tak berubah |

## 3. Skema config / API

Tanpa field baru. Satu endpoint baru: `GET /v1/cron/targets` → `{sessions: [{label, channel, chat_id}], channels: [...]}`. Validator `GET /v1/cron/validate` kini kembalikan 3 `next_runs` (base berantai, tanpa dep baru).

## 4. Call-site

| # | Lokasi | Guna | Fallback |
|---|---|---|---|
| 1 | `spa_js.openCronEditor` | Preset/builder/preview/target-picker | Field mentah di advanced |
| 2 | `spa_js.renderCronList` | Label + countdown + toast/konfirmasi | Teks mentah bila pola asing |
| 3 | `app_js.renderConfigForm` | Basic + search + strlist + secret peek + diff/dirty | Grup lama bila search kosong |
| 4 | `service.validate_cron_schedule` | 3 run | 1 run (lama) |
| 5 | `service.list_cron_targets` + `health` | Picker target | Ketik manual |

## 5. Goals

- **G1** — User awam ubah model/channel/izin/bahasa tanpa buka JSON.
- **G2** — Buat cron tanpa ketik cron mentah; save tak pernah gagal karena target kosong.
- **G3** — Tanpa dep/build baru; 3 bahasa; `escHtml` semua dinamis; endpoint lama kompatibel.

## 6. Non-goals

- Wizard setup berlangkah; grid visual menit-jam penuh; auto-default sesi aktif; status last-run persisten. Add when ada demand.

## 7. Acceptance criteria

- **AC-36.1** Suite + lint hijau; tanpa dep/build baru; string baru di 3 bahasa.
- **AC-36.2** Setup: Basic tampil; search temukan field; array string tanpa JSON; secret bisa intip; save restart tampilkan diff; pindah view dirty ada prompt.
- **AC-36.3** Cron: preset/builder isi field; preview live 3 run; picker target prefill sesi aktif; save tanpa `chat_id` manual.
- **AC-36.4** List manusiawi + countdown; toggle toast; hapus ada jadwal/target; run-now tawarkan buka sesi.
- **AC-36.5** `POST/PUT /v1/cron` + `POST /v1/config` lama tetap valid.

## 8. Testing

1. `pytest tests/test_pha/test_webui_crud.py tests/test_pha/test_webui_api.py tests/test_pha/test_webui_spa.py -q` (validator 3 run, error ramah tanpa `--`, fasad target).
2. `ruff check src/pha` bersih + `node --check` JS terekstrak (`APP_JS`, `SPA_JS`, `I18N_JS`).
3. Manual: Setup ubah bahasa (hot-apply) + port (diff + restart); Cron "Harian 09:00" via preset → preview 3 run → save → run-now → buka sesi.
