# PRD-10 — WebUI: Redesign Panel SETUP (gateway.json)

| | |
|---|---|
| Feature | Panel SETUP: field bergrup berlabel Indonesia + help + validasi inline |
| Depends on | PRD-8, PRD-9 |
| Files | `pha/src/pha/gateway/config.py` (meta field), `pha/src/pha/gateway/health.py` (`/v1/config`), `webui/{app_js.py, style.py}` |

## 1. Masalah

`renderConfigForm` (`app_js.py:1583-1654`) menampilkan **semua** field mentah berurutan: label mono = nama field, tanpa grup, tanpa penjelasan; list/dict sebagai textarea JSON; error hanya setelah SAVE. Tidak ramah non-teknis.

## 2. Pendekatan (server-driven meta, pattern existing)

Server sudah mengirim `secrets` + `options` di `GET /v1/config`. Tambah `fields`:

```json
"fields": {
  "model": {"group": "koneksi", "label": "Model", "help": "mis. gpt-4o-mini atau @preset/…", "type": "string"},
  "permission_mode": {"group": "keamanan", "label": "Mode izin tool", "help": "auto = semua boleh; default = konfirmasi tool sensitif", "type": "enum"},
  "api_key": {"group": "koneksi", "label": "API key LLM", "type": "secret"},
  "send_progress": {"group": "channel", "label": "Kirim progres ke channel", "type": "bool"}
}
```

- Sumber kebenaran meta = `config.py` (dekat skema pydantic) → satu tempat, tanpa duplikasi label di JS.
- Field tanpa meta → fallback ke perilaku lama (label = nama field) supaya field baru tidak hilang (tolerant).

## 3. Grup

| Grup | Field (contoh) |
|---|---|
| Koneksi & Model | `provider_profile`, `model`, `api_key`, `base_url` |
| Channel Chat | `enabled_channels`, `channel_configs`, `send_progress`, `send_tool_hints`, `session_routing` |
| Keamanan & Izin | `permission_mode`, `group_permission_mode`, `permission_timeout_seconds`, `webui_token`, `allow_remote_admin_commands`, `allowed_remote_admin_commands` |
| Gambar & Vision | `vision_model`, `vision_api_key`, `vision_base_url` |
| Web & Server | `health_port`, `health_host`, `log_level` 🔁 |
| Lanjutan | `allowed_working_dirs`, `session_*`, sisa field |

Grup dirender dengan `<details open>` native (tanpa JS accordion) — grup "Koneksi & Model" terbuka default.

## 4. Render field

- **Label** = `label` bahasa manusia + ikon ⓘ dengan `title`/teks help 13px `--ink-2` di bawah field.
- **bool** → switch (checkbox + label); **enum** → `<select>` (existing); **string** → `<input>`; **number** → `<input type=number>`.
- **list/dict** → textarea JSON **di balik toggle "Mode lanjutan"**; default tampil sebagai chip/tag read-only + hint format, supaya tak menakutkan. Parsing tetap seperti sekarang (`collectConfig`).
- **secret** → `type=password` ter-mask + tombol "Ganti" (kosong = tidak diubah, perilaku existing).
- **🔁 restart** → badge kecil "perlu restart" di label field tersebut (bukan hanya banner global).

## 5. Simpan & validasi

- **Dirty state**: tombol `Simpan` disabled sampai ada perubahan; indikator "Belum disimpan" muncul.
- **Validasi inline**: tampilkan error pydantic per-field bila bisa dipetakan (path field → pesan), fallback banner. `config-error` existing dipertahankan.
- **Guard keluar**: `beforeunload`/konfirmasi bila ada perubahan belum disimpan.
- **Feedback**: toast sukses (PRD-9 §5) + banner sticky "perlu restart" untuk field 🔁.
- Baris tombol jadi sticky di bawah panel: `Muat ulang` · `Simpan` · `Restart layanan`.

## 6. Acceptance criteria

- **AC-10.1** Field tampil bergrup (`<details>`), label bahasa Indonesia + help.
- **AC-10.2** `GET /v1/config` menyertakan `fields`; field tanpa meta tetap dirender (fallback).
- **AC-10.3** Enum = select, bool = switch, secret = password + "Ganti", list/dict = JSON di "mode lanjutan".
- **AC-10.4** Simpan disabled tanpa perubahan; validasi error tampil inline per-field bila terpetakan.
- **AC-10.5** Field 🔁 punya penanda restart individual.
- **AC-10.6** id existing (`config-form`, `config-save`, `config-reload`, `config-restart`, `config-error`, `config-path`, `service-restart`, `pane-config`) tetap ada; smoke test hijau.
- **AC-10.7** Round-trip: nilai tersimpan identik dengan perilaku lama (test API config existing tetap lulus).

## 7. Testing

1. Test API: `GET /v1/config` memuat `fields`; `POST /v1/config` round-trip tidak berubah (perluas `tests/test_pha/test_health_config.py` / `test_webui_api.py`).
2. Smoke HTML: id existing + `fields` direferensikan di `APP_JS`.
3. Manual: ubah model, simpan, restart, reload; uji JSON invalid → error inline; uji secret kosong = tidak berubah.
