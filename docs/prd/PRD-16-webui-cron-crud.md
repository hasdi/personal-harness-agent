# PRD-16 — WebUI: Cron CRUD Penuh + Preview

| | |
|---|---|
| Feature | Menu Cron bisa diklik (select), preview next-run, plus create/edit/run-now |
| Depends on | PRD-8, PRD-9, PRD-12, PRD-13, PRD-14 |
| Files | `pha/src/pha/gateway/{cron,cron_tools,service,health}.py`, `webui/{markup,spa_js,app_js,i18n_js,style}.py` |
| Status | Draft v1.0 (2026-09-12) |

## 1. Kondisi sekarang

| Hal | Status | Bukti |
|---|---|---|
| View Cron | List + toggle + hapus | `spa_js.py:renderCronList`, `GET /v1/cron`, `POST /v1/cron/enable|remove` |
| Create | Hanya via agent tool `cron_create` (chat "buatkan cron…") | `cron_tools.py`, tanpa endpoint |
| Edit | Tidak ada (`delete+recreate`) | `cron.py` tanpa `update_job()` |
| Preview jadwal | Tidak ada | `next_run_time()` tak terekspos |
| Run-now / riwayat per-job | Tidak ada | cuma trace panel global |

## 2. API baru

| Endpoint | Provider | Keterangan |
|---|---|---|
| `POST /v1/cron` | `service.create_cron_job` | bungkus `add_job()` existing |
| `PUT /v1/cron/<name>` | `service.update_cron_job` | `update_job()` baru di `cron.py`: edit schedule/message/command/timezone/target/enabled, hitung ulang `next_run` |
| `GET /v1/cron/validate?schedule=&timezone=` | validator | reuse `validate_cron_expression`/`validate_timezone` + `next_run_time` → N run berikutnya untuk preview |
| `POST /v1/cron/run-now {name}` (opsional murah) | service | panggil path `_deliver_message_job`/`_run_command_job` existing |

Aksi body-POST mengikuti pola existing (`/v1/dir/allow`, `/v1/permission`) agar tanpa parsing path baru bila memungkinkan.

## 3. UI

- Modal create/edit (reuse `openModal` + pola `cron-actions`): nama, schedule + tombol "cek" (preview next runs), kind message/command, target, timezone. Validasi inline sebelum save.
- Klik row → pane detail: payload + `next_run` + N run berikutnya. Log per-job skip — trace panel sudah ada.
- Error spesifik per field (bukan `cron_load_failed` generik); loading per tombol (tanpa full `renderCronView` kedip).
- Empty-state + CTA "Buat cron pertama".

## 4. Acceptance criteria

- **AC-16.1** Buat cron dari WebUI tanpa chat; jadwal tervalidasi + preview next-run sebelum simpan.
- **AC-16.2** Edit job existing (termasuk rename? tidak — nama immutable, cukup field lain).
- **AC-16.3** Toggle/hapus tetap bekerja (existing, AC-13.5).
- **AC-16.4** (Jika run-now) job jalan on-demand, hasil masuk session seperti tick normal.
- **AC-16.5** Label 3 bahasa; `ruff`/pytest/`node --check` hijau.

## 5. Non-goals

- Log/riwayat eksekusi per-job, retry/backoff config, pause-sampai-tanggal.

## 6. Testing

1. Perluas `tests/test_pha/test_cron_delivery.py` — create/update via service, validate schedule, `next_run` terhitung ulang.
2. Smoke HTML: id modal/detail baru.
3. Manual: create→preview→edit→toggle→run-now; jadwal invalid ditolak inline.
