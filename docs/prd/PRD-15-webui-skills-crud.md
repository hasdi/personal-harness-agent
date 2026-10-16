# PRD-15 — WebUI: Skills Select, Preview & CRUD

| | |
|---|---|
| Feature | Menu Skills bisa diklik (select), preview isi SKILL.md, plus CRUD + enable/disable |
| Depends on | PRD-8, PRD-9, PRD-12, PRD-13 |
| Files | `pha/src/pha/gateway/webui/{markup,spa_js,app_js,i18n_js,style}.py`, `pha/src/pha/gateway/{service,health}.py` |
| Status | Draft v1.0 (2026-09-12) |

## 1. Kondisi sekarang

| Hal | Status | Bukti |
|---|---|---|
| View Skills | Read-only list (nama, deskripsi, sumber, path) + cari | `spa_js.py:renderSkillsView/renderSkillsList`, `GET /v1/skills` |
| Preview isi | Tidak ada (`content` dibuang di `list_skills`) | `service.py:list_skills` |
| Create/Update/Delete | Tidak ada endpoint | `health.py` cuma `GET /v1/skills` |
| Enable/disable | Tidak ada (cuma frontmatter statis) | `openharness/skills/*` |
| Label `builtin` ikut `user`, plugin/project tak muncul (`cwd` tak diteruskan) | Bug kecil | `service.py:list_skills` vs `loader.py` |

## 2. API baru

| Endpoint | Provider | Keterangan |
|---|---|---|
| `GET /v1/skills/<name>` | `service.get_skill` via `registry.get()` | `content` + frontmatter (`model`, `argument_hint`, `user_invocable`) |
| `POST /v1/skills` | `service.create_skill` | tulis `<workspace>/skills/<slug>/SKILL.md` reuse template `skill-creator/scripts/init_skill.py`, validasi `parse_skill_metadata()` |
| `PUT /v1/skills/<name>` | `service.update_skill` | sama, tolak `source == bundled/plugin` (read-only) |
| `DELETE /v1/skills/<name>` | `service.delete_skill` | hanya user-workspace, containment check ala `_read_attachment`/`_is_within` |
| `POST /v1/skills/enable` | `service.set_skill_enabled` | denylist di `gateway.json` (pola `save_webui_config`), filter di `list_skills` |

Bonus: `list_skills()` teruskan `cwd=self._cwd`, label `builtin` terpisah.

## 3. UI

- Klik kartu → pane detail: metadata + preview SKILL.md (`mdToHtml` existing) + tombol Edit/Hapus/Enable.
- Modal editor (reuse `openModal` + `toast`): nama/deskripsi/content; tombol "+ Skill Baru".
- Tampilkan `argument_hint`; search tetap client-side; empty-state + CTA "Buat skill".

## 4. Acceptance criteria

- **AC-15.1** Klik skill → detail + preview isi tampil.
- **AC-15.2** Buat/edit/hapus skill user-workspace round-trip; bundled/plugin ditolak jelas.
- **AC-15.3** Enable/disable persist (`gateway.json`) dan memengaruhi `list_skills`.
- **AC-15.4** Label 3 bahasa (PRD-12); string dinamis `escHtml`.
- **AC-15.5** `ruff` bersih, pytest hijau, `node --check` OK, `external == []`.

## 5. Non-goals

- Install skill dari registry luar, paginasi/virtualisasi, versioning.

## 6. Testing

1. `tests/test_pha/test_webui_skills_crud.py` — roundtrip create/update/delete/enable di `tmp_path`, tolak bundled/plugin.
2. Smoke HTML: id detail/editor baru; route baru di daftar path.
3. Manual: preview, edit, hapus, enable, cari.
