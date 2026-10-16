# PRD-12 — WebUI: Bahasa Antarmuka (bahasa / english / japanese)

| | |
|---|---|
| Feature | Pilihan bahasa antarmuka WebUI di panel SETUP |
| Depends on | PRD-8, PRD-9 |
| Files | `pha/src/pha/gateway/{models.py, service.py}`, `webui/{i18n_js.py, markup.py, app_js.py, __init__.py}` |

## 1. Ringkasan

Field config `language` (`"bahasa"` \| `"english"` \| `"japanese"`, default
`"bahasa"`) mengubah bahasa seluruh label WebUI. Nilai tersimpan di
`gateway.json`, tampil sebagai **select** di grup **Tampilan & Bahasa**, dan
hot-apply (tanpa restart) lewat `/status`.

## 2. Desain

- **Sumber string**: satu kamus `I18N` di `webui/i18n_js.py` (3 bahasa).
  Kunci yang tidak ada jatuh balik ke bahasa Indonesia — UI tidak pernah
  menampilkan kunci mentah.
- **Label statis**: atribut `data-i18n` / `data-i18n-ph` /
  `data-i18n-title` / `data-i18n-aria` di `markup.py`; `applyLanguage()`
  menimpa `textContent`/`placeholder`/`title`/`aria-label`.
- **String dinamis**: `t(key)` (+ `tn(key, n)` untuk angka) dipakai di
  `app_js.py` (fase run, empty state, toast, dialog konfirmasi, waktu relatif,
  grup sesi).
- **Deteksi bahasa**: `/status` menyertakan `language`; `refreshStatus()`
  memanggil `applyLanguage()` saat nilainya berbeda, lalu me-render ulang
  greeting + daftar sesi.
- **Elemen yang tidak boleh ditimpa**: jangan pasang `data-i18n` pada elemen
  yang `textContent`-nya dikelola JS (mis. `#send-btn` tetap boleh karena
  `syncComposer()` memakai `t()` yang sama) atau yang berisi child element.

## 3. Acceptance criteria

- **AC-12.1** SETUP → grup "Tampilan & Bahasa" → select bahasa/english/japanese.
- **AC-12.2** Simpan bahasa → UI berganti tanpa restart (`restart_required: false`).
- **AC-12.3** Pilihan persist di `gateway.json` dan bertahan setelah reload.
- **AC-12.4** Kunci tak dikenal jatuh balik ke bahasa Indonesia (tidak ada kunci mentah).
- **AC-12.5** Greeting, fase run, dan daftar sesi ikut berganti bahasa.
- **AC-12.6** `lang` dokumen (`<html lang>`) mengikuti pilihan (id/en/ja).
- **AC-12.7** Zero dependency; tanpa URL eksternal baru; smoke test hijau.

## 4. Testing

1. `tests/test_pha/test_health_config.py` — `get_webui_config()` memuat
   `language` di `config`/`options`/`fields`/`groups`; `save_webui_config`
   hot-apply dan persist.
2. Smoke HTML: `data-i18n` hadir; `I18N_JS` ter-rakit sebelum `APP_JS`.
3. `node --check` atas JS gabungan.

## 5. Future

- Bahasa untuk balasan agent (system prompt) — terpisah dari bahasa UI.
- Deteksi otomatis dari `Accept-Language` sebagai default pertama.
