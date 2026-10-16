# PRD-13 — WebUI: Halaman Chat / Setup / Skills / Cron

| | |
|---|---|
| Feature | View switching (tanpa reload), Setup jadi halaman sendiri, menu Skills & Cron |
| Depends on | PRD-8, PRD-9, PRD-12 |
| Files | `pha/src/pha/gateway/webui/{markup,spa_js,app_js,i18n_js,style,__init__}.py`, `pha/src/pha/gateway/{service,health}.py` |
| Referensi IA | `nanobot-main/webui` (settings/skills/automations) |

## 1. Ringkasan

WebUI punya empat **view** dalam satu dokumen: **Chat, Setup, Skills, Cron**.
Perpindahan view murni show/hide elemen (`hidden`) di sisi klien — **tanpa routing
hash/URL, tanpa reload, tanpa build step**. SETUP yang tadinya tab side-panel jadi
halaman tersendiri, plus dua menu baru.

> **Revisi v1.1.** Rancangan awal memakai hash router (`#/chat`, `#/setup`, …).
> Diganti menjadi tombol view biasa: lebih sederhana, dan `spaInit()` yang dipanggil
> dari `app_js` sempat gagal karena `var` (`VIEWS`) belum ter-assign saat init berjalan
> (hoisting = `undefined`). Init sekarang dijalankan di **akhir** `spa_js.py`.

## 2. View & navigasi

| View | Isi |
|---|---|
| Chat (default) | sidebar sesi + chat + composer + panel kanan **Trace/Audit** (session-scoped) |
| Setup | seluruh panel config (toolbar, banner, form, restart) |
| Skills | skill terpasang (nama, deskripsi, sumber, path) + pencarian |
| Cron | cron job (nama, jadwal, kind, target, next run) + enable/disable + hapus |

- Navbar: `<nav id="pha-nav">` berisi 4 tombol `data-view`; item aktif `is-active` +
  `aria-current="page"`.
- `switchView(name)` men-toggle `hidden` view + highlight nav, lalu `refreshActiveView()`
  memuat data view tersebut (Setup → `loadConfig`, Skills → `/v1/skills`, Cron → `/v1/cron`).
- Tidak ada perubahan URL; tombol back/forward browser tidak dipakai.
- Panel kanan tinggal **Trace + Audit**; `id="pane-config"` pindah ke view Setup
  (id lama dipertahankan agar smoke test tetap hijau).

## 3. API baru

| Endpoint | Provider | Keterangan |
|---|---|---|
| `GET /v1/skills` | `PhaGatewayService.list_skills` | `load_skill_registry` (bundled + `~/.pha/skills` + plugins); field `name/title/description/source/path/argument_hint` |
| `GET /v1/cron` | `list_cron_jobs` | `pha.gateway.cron.load_jobs` (`~/.pha/cron.json`) |
| `POST /v1/cron/enable` | `set_cron_job_enabled` | body `{"name", "enabled": bool}` → 404 bila tak ada |
| `POST /v1/cron/remove` | `remove_cron_job` | body `{"name"}` → 404 bila tak ada |

Auth `/v1` sama seperti endpoint lain (`_requires_auth`). Aksi body-POST dipilih mengikuti
pola existing (`/v1/dir/allow`, `/v1/permission`) agar tanpa parsing path.

## 4. Kualitas & constraint

- Zero dependency, zero build; tidak ada URL eksternal baru (smoke test `external == []`).
- `spa_js.py` dirakit **setelah** `app_js.py`; `spaInit()` dipanggil di **akhir** `spa_js.py`
  (bukan dari `app_js`) supaya semua `var` di modul ini sudah ter-assign. `app_js` hanya
  memanggil `refreshActiveView()` yang aman karena function declaration ter-hoist.
- Semua string dinamis lewat `escHtml`; label statis lewat `data-i18n` (PRD-12) — 3 bahasa.

## 5. Acceptance criteria

- **AC-13.1** Klik nav menampilkan view terkait tanpa reload; URL tidak berubah; default Chat.
- **AC-13.2** Navbar 4 tombol, item aktif ter-highlight (`is-active` + `aria-current`).
- **AC-13.3** Setup halaman sendiri; panel kanan hanya Trace/Audit; semua id config lama tetap ada.
- **AC-13.4** Skills menampilkan skill + sumber + path, ada cari & empty state.
- **AC-13.5** Cron menampilkan job + jadwal + target; enable/disable & hapus (konfirmasi) bekerja.
- **AC-13.6** Endpoint `/v1/skills`, `/v1/cron`, `/v1/cron/enable`, `/v1/cron/remove` teruji.
- **AC-13.7** `ruff` bersih, pytest hijau, `node --check` JS gabungan OK.
- **AC-13.8** Restart layanan dari WebUI bekerja juga saat dijalankan sebagai exe PyInstaller.
  Windows: parent tanpa console (background/service/exe) → spawn **detached**
  (`DETACHED_PROCESS | CREATE_NEW_PROCESS_GROUP`, stdio DEVNULL); parent dengan console
  (`--foreground`) → spawn **inherit console** (tanpa flag, stdio diwarisi) supaya log
  tetap streaming dan Ctrl+C tetap berfungsi. POSIX tetap `os.execv`.

## 6. Testing

1. `tests/test_pha/test_webui_spa.py` — provider unit (`list_skills`, cron toggle/remove) + route
   lewat `start_health_server` (GET skills/cron, POST enable/remove, validasi 400/404).
2. Smoke HTML `test_webui_api.py` — id baru (`pha-nav`, `view-*`, `skills-list`, `cron-list`) + path API.

## 7. Future

- Halaman **Plugins/MCP** dan **Memory** (pola sama).
- Aksi tambahan di Cron: "run now", riwayat eksekusi.
- PWA/manifest agar terasa seperti app desktop.
