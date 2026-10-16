# PRD-32 — Grouping Sesi Manual di Panel Sessions

| | |
|---|---|
| Feature | Grup manual + assign via menu, persist workspace |
| Depends on | PRD-13 (sidebar) |
| Files | `pha/session_groups.py` (baru), `pha/gateway/{service,health}.py`, `pha/gateway/webui/{markup,app_js,spa_js,style,i18n_js}.py`, `tests/test_pha/test_session_groups.py` (baru) |
| Status | Draft (2026-09-24) |

## 1. Keputusan

| Aspek | Keputusan |
|---|---|
| Store | `<ws>/session_groups.json` `{groups: [{id, name}], assign: {session_id: group_id}}`; modul `pha/session_groups.py` reuse `atomic_write` + `exclusive_file_lock`; hapus sesi → assignment orphan dibersihkan lazy |
| API | `GET /v1/session-groups`, `POST /v1/session-groups {name}`, `DELETE /v1/session-groups/{id}` (sesi jadi unassigned), `POST /v1/sessions/{id}/group {group_id\|null}` |
| UI | Sidebar (`markup.py:37-46`) render grup collapsible + section `Tanpa grup`; tiap item sesi menu `⋯` (pindah grup); header grup: rename/hapus; search (`#chat-search`) filter lintas grup; collapse state di `localStorage` |

## 2. Goals

- **G1** — Sesi bisa dikelompokkan manual, bertahan refresh + restart.

## 3. Non-goals

- Auto-group by cwd/channel/tanggal; drag-drop (menu cukup); paginasi sidebar.

## 4. Acceptance criteria

- **AC-32.1** Bikin/rename/hapus grup round-trip persist; assign sesi → refresh bertahan.
- **AC-32.2** Hapus grup → sesi jadi unassigned, tak ada sesi hilang.
- **AC-32.3** Search filter tetap jalan lintas grup; collapse tak reset saat refresh view.
- **AC-32.4** Suite baru + existing hijau; `ruff` bersih.

## 5. Testing

1. `pytest tests/test_pha/test_session_groups.py tests/test_pha/test_webui_api.py -q` (baru: CRUD grup + assign + orphan cleanup).
2. Manual: bikin 2 grup, pindah sesi, refresh, hapus grup.
