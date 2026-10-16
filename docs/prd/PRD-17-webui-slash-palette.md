# PRD-17 — WebUI: Palette `/` Lengkap (+ `/plan`)

| | |
|---|---|
| Feature | Fungsi `/` bekerja penuh: daftar command lengkap termasuk `/plan`, eksekusi dari WebUI |
| Depends on | PRD-8, PRD-9, PRD-11 |
| Files | `pha/src/pha/gateway/{service,runtime}.py`, `webui/app_js.py` |
| Status | Draft v1.0 (2026-09-12) |

## 1. Root cause (bukti dari code)

- `service.py:available_commands` melakukan `for command in bundle.commands` — `CommandRegistry` tidak iterable (API benar: `list_commands()`/`lookup()`). Selalu `TypeError` → `except` → `return []` → palette selalu "no commands".
- Andaipun iterasi diperbaiki, filter salah: `SlashCommand` punya `remote_invocable`/`remote_admin_opt_in`, bukan `.metadata["user_invocable"]`.
- `/plan` (`registry.py`) = `remote_invocable=False, remote_admin_opt_in=True` → `runtime.py:_stream_message_inner` menolak dari channel remote; `gateway.json` default `allow_remote_admin_commands=false`.

## 2. Perbaikan

1. `service.py:available_commands` → `list_commands()`, filter `remote_invocable` + `_remote_admin_allowed`; gabung skill-commands (`user_invocable`) agar `/skill-creator` muncul. Hindari `get_bundle("api:__commands__")` penuh bila berat — cache atau `create_default_command_registry()` langsung.
2. `runtime.py:_remote_admin_allowed` + `_stream_message_inner`: kecualikan `channel == "api"` dari blokir remote (WebUI = lokal tepercaya, bukan channel remote). `/plan`, `/permissions` jalan tanpa ubah default `gateway.json`.
3. Palette: grouping (umum/admin/skill), empty-state tetap (PRD-11 AC-11.1).

## 3. Acceptance criteria

- **AC-17.1** `GET /v1/commands` > 0 entri (dulu `[]`).
- **AC-17.2** Ketik `/pl` → suggest `/plan`; Enter → jalan (`Plan mode enabled.`), bukan `only available in local PHA UI`.
- **AC-17.3** Skill-commands user-invocable muncul di palette.
- **AC-17.4** Channel chat non-WebUI tetap diblokir untuk command admin (tanpa opt-in) — tidak ada regresi keamanan.
- **AC-17.5** `ruff`/pytest hijau.

## 4. Non-goals

- Command baru; ubah perilaku TUI lokal.

## 5. Testing

1. Test baru: `available_commands` > 0 & memuat `/plan`; simulasi eksekusi `/plan` via `channel="api"` lolos, via `channel="telegram"` tetap ditolak.
2. Manual: `/`, `/pl`, Enter; cek grouping.
