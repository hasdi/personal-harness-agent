# PRD-35 — System One Action Gate (Pre-Tool + Output Screen)

| | |
|---|---|
| Feature | Verdict System One sebelum tiap tool jalan + screening output sebelum disajikan — tanpa LLM call, tanpa knob baru |
| Depends on | PRD-33 (decision layer, `decide_quiet`, `decision_*`), PRD-6 (HITL confirm), gateway-config |
| Files | `openharness/engine/query.py` (`_apply_system_one_permission`, `_apply_system_one_output_verdict` baru, `_execute_tool_call`, `_convert_images_to_text`), `openharness/permissions/checker.py` (pola sensitif + guard shell), `openharness/services/secret_redact.py` (baru), `docs/gateway-config.md` (§7), `tests/test_pha/test_system_one.py` |
| Status | Implemented (2026-10-10), hardened (2026-10-10) |

## 1. Ringkasan

Gate PRD-33 setengah jadi: verdict `risk` hanya kirim `{tool, command, path}` tanpa cwd/input/session, dan output tool tak pernah dinilai — `read_file` berisi AWS key lolos mentah ke chat, `bash` ke direktori kritis (`C:/Windows/System32`) cuma dicek string command. PRD ini menutup keduanya di choke point tunggal `_execute_tool_call`: pre-tool verdict diperkaya state (cwd, input ringkas, session, flag read-only) + short-circuit bila deterministik sudah hard-deny; output verdict baru (`expose`) me-redact output bersecret sebelum disajikan. Semua tool, force-confirm untuk danger, `enabled=false` = zero diff.

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Lapisan (Tiwari) | Deterministik (`PermissionChecker` fnmatch + `SENSITIVE_PATH_PATTERNS`) → probabilistik (System One) → HITL (confirm) → audit (trace tiap verdict). Prompt bukan enforcement (Laclé) |
| Pre-tool | Perkaya **state**, tetap 1 question `risk` — tanpa call tambahan, tanpa question baru |
| Pre enforcement | `risk>=high` + allowed → `requires_confirmation` (force-confirm). Hard-deny deterministik selalu menang + skip System One |
| Output | `_apply_system_one_output_verdict`: 1 question `expose` (noul), state = tool + output truncate ~2000 char |
| Output enforcement | **Redact-and-replace** (confirm tak bermakna — output sudah terjadi): `expose>=high` → notice + `is_error=True` + trace `point=tool_output`. Mid/low → lolos + trace |
| Cakupan | Semua tool bila enabled. Tanpa knob scope — YAGNI. Add when ada keluhan latency |
| Bypass | `_convert_images_to_text` (panggil `execute` langsung) ikut verdict output yang sama |
| Fail | Fail-open ke legacy (konsisten PRD-33); deterministic layer tetap aktif = defense in depth |
| Config | Tanpa knob baru — reuse `decision_*`. `enabled=false` = zero diff |

## 3. Skema config

Tanpa field baru. `gateway.json` tetap `decision_enabled, decision_base_url, decision_api_key, decision_model, decision_timeout, decision_conf_high, decision_conf_mid` (PRD-33). Resolve via `normalize_config(tool_metadata["decision_model_config"])`, dipasang per-session oleh `session_pool._apply_decision_config`.

## 4. Call-site

| # | Lokasi | Q | Guna | Fallback |
|---|---|---|---|---|
| 1 | `_execute_tool_call` pre (`query.py`) | `risk` + state kaya | Force-confirm danger semua tool; hard-deny skip call | Legacy `PermissionDecision` |
| 2 | `_execute_tool_call` post (`query.py`) | `expose` | Redact output bersecret (`is_error=True`) | Output mentah |
| 3 | `_convert_images_to_text` (`query.py`) | `expose` | Tutup bypass vision-fallback | Deskripsi mentah |

```python
async def _apply_system_one_output_verdict(context, tool_name, tool_output) -> tuple[str, bool]
```

State pre: `{tool, command[:500], path, cwd, input[:500], read_only, session}`. Short-circuit: `not allowed and not requires_confirmation` → return tanpa HTTP.

## 5. Goals

- **G1** — `bash "rm -rf C:/Windows/System32/*"` → force-confirm, bukan auto-allow.
- **G2** — `read_file` berisi secret + `expose>=high` → redact, secret tak sampai chat/agent.
- **G3** — Hard-deny (`*/.ssh/*`) tanpa HTTP System One; disabled = zero diff; tanpa ubah signature loop.

## 6. Non-goals

- Deteksi prompt-injection via output (Laya kecil, klaim berlebihan). Add when dibutuhkan.
- Strict fail-closed. Add when ada threat model yang menuntutnya.
- Knob scope per-tool / dashboard kalibrasi. Add when ada keluhan latency.

## 7. Acceptance criteria

- **AC-35.1** `enabled=false` → suite hijau, mock decision tak pernah dipanggil.
- **AC-35.2** `bash` destruktif → `requires_confirmation` + state memuat cwd/session.
- **AC-35.3** Output secret + `expose>=high` → redact + `is_error=True` + trace `point=tool_output`.
- **AC-35.4** Path `~/.ssh/id_rsa` → deny tanpa HTTP System One.
- **AC-35.5** Timeout/down → fallback diam-diam + trace `system_one.unavailable`; tanpa ErrorEvent.
- **AC-35.6** `ruff` bersih.

## 8. Testing

1. `pytest tests/test_pha/test_system_one.py -q` (5 baru: hard-deny skip, state kaya, redact high, lolos low, output disabled; 16 lama).
2. `ruff check src/openharness/engine/query.py tests/test_pha/test_system_one.py` bersih.
3. Manual: `laya-serve` lokal → `decision_enabled=true` → bash direktori sistem (confirm) → read file secret (redact) → tab Trace (`permission` + `tool_output`).

## 9. Hardening (2026-10-10, insiden `gateway.json` bocor ke chat)

Replay: user minta isi `gateway.json` → `glob **/gateway.json` + `read_file` lolos (read-only auto-allow, pola sensitif tak kenal file PHA) → output mentah via System One `expose` yang butuh service up → secret tampil di chat.

- **Lapis deterministik menang dulu**: `SENSITIVE_PATH_PATTERNS` += `*/gateway.json`, `*/config.json`, `*/cron.json`, `*/.env`, `*/.env.*` — deny di semua mode, tanpa confirm, tanpa HTTP (short-circuit existing).
- **Guard shell**: `_command_reads_sensitive_file` — `cat/type/head/... gateway.json|.env|cron.json|config.json` ditolak (command tak lewat normalisasi path; `grep/rg` dikecualikan — ambigu, file tetap ditolak saat dibaca).
- **Redaksi deterministik**: `services/secret_redact.py` (api_key/bearer/AWS key/private key/token telegram/secret=...) jalan SEBELUM System One, tanpa network; trace `point=tool_output, via=deterministic`.

→ skipped: daftar file sensitif via config user. Add when: ada secret store baru di luar pola ini.
