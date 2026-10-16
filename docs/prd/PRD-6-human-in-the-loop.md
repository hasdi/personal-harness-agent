# PRD-6 — WebUI: Human-in-the-Loop Permission Popup

| | |
|---|---|
| Feature | Saat `permission_mode` ≠ `auto`, tool yang butuh konfirmasi memunculkan **popup Allow/Deny** di WebUI; engine menunggu keputusan human |
| Scope | Core: callback gateway untuk session WebUI, popup + resolve API, timeout fail-safe |
| Status | Draft v1.0 (2026-08-29) |
| Depends on | PRD-3 (modal/WebUI), PRD-1 (trace kinds, tolerant reader) |

## 1. Konfirmasi pemahaman

Ya — dimaksud: ketika session berjalan dengan `permission_mode = "default"` (bukan `"auto"`),
setiap tool yang memerlukan konfirmasi (write/execute/destination di luar workspace) **tidak
langsung ditolak diam-diam**, melainkan memunculkan popup konfirmasi di WebUI. User memilih
Allow/Deny; keputusan itu dikembalikan ke engine sebagai hasil permission. Channel (Telegram
dll.) tetap silent-deny — tidak ada human yang bisa menjawab di sana.

## 2. Kontrak yang sudah ada di engine (tanpa perubahan engine)

| Fakta | Lokasi |
|---|---|
| `PermissionPrompt = Callable[[str, str], Awaitable[bool]]` — `(tool_name, reason) → bool` | `engine/query.py:69` |
| Engine: bila `decision.requires_confirmation` **dan** `permission_prompt` terdaftar → hook NOTIFICATION → `confirmed = await context.permission_prompt(tool_name, decision.reason)`; deny → ToolResultBlock `is_error` | `engine/query.py:1070-1090` |
| `permission_prompt` belum terdaftar → **silent deny** (perilaku gateway saat ini) | `query.py:1091-1097` |
| `build_runtime(..., permission_prompt=...)` sudah menerima param | `ui/runtime.py:287,414` |
| Gateway tidak meneruskannya | `gateway/runtime.py` (`build_runtime(...)` tanpa permission_prompt) |
| Mode per-message dari `permission_mode`/`group_permission_mode` → checker di-set di `_apply_permission_mode` | `gateway/runtime.py:305-325` |

Artinya: gateway hanya perlu **mendaftarkan callback** untuk session WebUI dan menyediakan
transport pertanyaan → popup → jawaban. Engine sudah menangani sisanya (deny = error result
yang terlihat agent).

## 3. Desain

### 3.1 Registrasi callback (server)

- `get_bundle(...)` menerima `permission_prompt` (bundle-level, dibuat sekali per bundle).
- Aturan registrasi: **hanya session `api:*`** (WebUI/API) → callback gateway. Session
  channel (`telegram:*` dst.) → `None` (silent deny, perilaku existing).
- Callback tidak perlu tahu mode: `requires_confirmation` hanya bernilai True di mode
  ≠ auto — di mode `auto` callback tidak pernah dipanggil → **default config tidak berubah**.
- Implementasi callback (per session_key):
  1. Buat `request_id = uuid4().hex[:12]`, simpan `asyncio.Future` di
     `_pending_permissions[request_id]` beserta meta `{tool_name, reason, session_key, run_id, ts}`.
  2. Emit SSE event `permission_request` (§4) + trace event `permission.requested`.
  3. `return await asyncio.wait_for(future, timeout=permission_timeout_seconds)`.
  4. `TimeoutError` → resolve deny + SSE `permission_expired` + trace `permission.resolved
     {decision:"timeout"}`.
- Pending map dibersikakan saat run selesai/gateway shutdown (deny semua — fail-safe).

### 3.2 API

| Method | Path | Body | Response |
|---|---|---|---|
| GET | `/v1/permission/pending` | — | `{"requests":[{request_id, tool_name, reason, session_key, run_id, ts}]}` |
| POST | `/v1/permission` | `{"request_id": "...", "approved": true}` | `200 {"ok": true}`; `404` request tak dikenal/stale; `400` body salah |

- Semua authed (bearer). POST dari tab/browser lain juga bisa menjawab (multi-tab aman:
  resolve idempotent — request kedua untuk id yang sama → 404).
- Provider pattern seperti endpoint lain (`permission_provider`, `permission_resolver`
  disuntik dari service).

### 3.3 SSE events (channel stream session terkait)

| Event | Payload |
|---|---|
| `permission_request` | `{request_id, tool_name, reason, session_key, run_id}` |
| `permission_resolved` | `{request_id, decision: "approved"\|"denied"\|"expired"}` |

- UI status row menampilkan `MENUNGGU PERSETUJUAN…` saat ada pending (phase dari
  `permission_request`), kembali ke fase sebelumnya setelah resolve.

## 4. UX (WebUI)

- Popup memakai `#modal-overlay` existing: judul **PERMISSION REQUIRED**, isi =
  tool name (mono, besar) + `reason` + session context; tombol **ALLOW** (green) /
  **DENY** (red).
- Bila ada >1 pending → antrean: popup menampilkan "1 dari N", tombol membawa request_id
  terdepan, setelah resolve otomatis menampilkan berikutnya.
- **Survive refresh**: saat load/reconnect, `GET /v1/permission/pending` dirender ulang —
  popup tidak hilang bila user refresh halaman.
- Saat user tidak membuka WebUI sama sekali → timeout auto-deny (§3.1) — perilaku akhir
  sama dengan silent deny hari ini, hanya terlambat.

## 5. Konfigurasi (`gateway.json`)

| Field | Default | Arti |
|---|---|---|
| `permission_timeout_seconds` | `120` | Batas tunggu jawaban human sebelum auto-deny |
| `permission_mode` | `"auto"` | Existing — `"auto"` = tanpa popup sama sekali (perilaku default tidak berubah) |

## 6. Trace & audit

- Trace kinds baru (tolerant reader aman): `permission.requested`
  `{tool, reason}`, `permission.resolved` `{tool, decision}` — muncul di tab Trace.
- Tool result deny tetap tercatat sebagai tool.completed `is_error` (existing).

## 7. Keamanan

- Popup hanya untuk session `api:*` + bearer auth untuk resolve.
- Timeout fail-safe default deny; shutdown gateway = deny semua pending.
- Tidak ada bypass: mode `plan`/`default` tetap memakai `PermissionChecker` existing —
  popup hanya menggantikan silent-deny untuk kasus `requires_confirmation`.
- Reason dari engine sudah terbatas panjangnya; tanpa isi file utuh (tool input ringkas).

## 8. Acceptance criteria

- **AC-6.1** Mode `auto` → tidak ada popup (callback tidak pernah dipanggil) — perilaku default unchanged.
- **AC-6.2** Mode `default` + WebUI tool write → SSE `permission_request` + popup; ALLOW → tool dieksekusi; DENY → tool result error `Permission denied`.
- **AC-6.3** Timeout tanpa jawaban → auto-deny + event `permission_resolved{expired}` + tool error result.
- **AC-6.4** Refresh halaman saat pending → popup muncul lagi (dari `GET /v1/permission/pending`).
- **AC-6.5** Channel session (`telegram:*`) di mode `default` → tetap silent deny (tanpa callback).
- **AC-6.6** Trace memuat `permission.requested`/`permission.resolved`.
- **AC-6.7** Gateway shutdown saat pending → semua future di-resolve deny, tidak ada hang.
- **AC-6.8** Smoke test HTML (`permission-` ids) + ruff + pytest hijau.

## 9. Testing

1. Unit callback: pending map register/resolve/timeout (mock futures, tanpa sleep — timeout 0.01s).
2. API: POST resolve 200/404/400; pending list; auth 401; 503 provider None.
3. Runtime: stub `build_runtime` — assert `permission_prompt` diteruskan hanya untuk `api:*`.
4. SSE: event `permission_request` terlihat di stream session terkait (fake stream handler).
5. Manual QA: mode default → minta agent menulis file → popup muncul → ALLOW → file tertulis; DENY → agent menerima error; refresh halaman → popup kembali.

## 10. Rollout

| Milestone | Isi |
|---|---|
| **M1** | Callback + pending map + timeout + trace kinds (server core) |
| **M2** | API pending/resolve + SSE events |
| **M3** | WebUI popup + antrean + refresh-resilience + status row |
| **M4** | README + manual QA |

## 11. Out of scope / Future

- "Always allow tool ini untuk session/mode ini" (memory allowlist per tool+path).
- HITL untuk channel (reply bot Telegram sebagai jawaban permission).
- Mode `plan` interaktif (persetujuan plan via popup).
- Detail diff file pada popup untuk `edit_file` (menampilkan before/after).
