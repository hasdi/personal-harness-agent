# PRD-0 — Detailed Trace & Audit Trail di WebUI (Overview & Index)

| | |
|---|---|
| Feature | Detailed Trace + Audit Trail per chat session, ditampilkan sebagai tab WebUI |
| Scope | **Core functions first** — capture, storage, API, tab UI, clear/purge trail |
| Projects | sumber data: engine OpenHarness (vendored di `pha-agent/src/openharness`); target UI: `pha-agent/src/pha/gateway/webui` |
| Status | Draft v1.0 (2026-08-29) |
| Dokumen | PRD-0 (ini) · [PRD-1](./PRD-1-trace-core.md) · [PRD-2](./PRD-2-trace-api.md) · [PRD-3](./PRD-3-trace-webui.md) · [PRD-4](./PRD-4-quality-rollout.md) |

---

## 1. Jabaran singkat — apa yang dimau

Fitur **Detailed Trace & Audit Trail** di WebUI pha-agent:

1. **Setiap "process" (run) tercatat append-only.** Satu run = satu eksekusi prompt end-to-end
   (user submit → LLM call(s) → tool call(s) → jawaban final). Setiap run punya `run_id` dan
   timeline event: LLM response, tool started/completed, progress, retry, error.
2. **Per chat session.** Trace dikelompokkan per `session_key`; di WebUI ada tab **Trace** dengan
   session selector (reuse daftar session yang sudah ada) → daftar run → timeline event per run.
3. **Tab Audit terpisah.** Menampilkan audit global gateway yang sudah ada
   (`~/.pha/logs/audit.jsonl`, `src/pha/audit.py`) — event channel (Telegram/Slack/dll) per
   `session_key`, sender di-hash (privacy, tetap dipertahankan).
4. **Clear audit trail.** Pengguna dapat membersihkan trail **per run** atau **per session**
   lewat tombol di tab Trace (modal konfirmasi), via `DELETE` endpoint. Purge sendiri
   tercatat sebagai event `trail.purged` (jejak audit atas penghapusan).
5. **Core dulu, tanpa dependensi baru.** Capture dilakukan di adapter layer pha
   (`gateway/service.py` + `gateway/runtime.py`), **engine OpenHarness tidak diubah** → bebas
   drift fork, low-maintenance. WebUI tetap HTML/CSS/JS murni tanpa framework/build step.

### Masalah saat ini (bukti dari code)

| Gap | Bukti |
|---|---|
| Trace di TUI OpenHarness bersifat transient — stream event tanpa ID/timestamp, tidak dipersisten | `openharness/engine/stream_events.py`, `openharness/ui/protocol.py` (`TranscriptItem`) |
| Session snapshot hanya menyimpan `messages` + `usage` agregat — tool result & error tidak tersimpan | `openharness/services/session_storage.py:85-96` |
| pha sudah punya audit JSONL, tapi tidak per-run, tidak tampil di WebUI, tidak bisa dibersihkan per session | `pha/src/pha/audit.py:26-54`, `~/.pha/logs/audit.jsonl` |
| WebUI baru punya satu view (Chat), tanpa tab | `pha/src/pha/gateway/webui/markup.py:5-56` |

## 2. Arsitektur ringkas

```
pha gateway
  handle_api_prompt_stream (service.py)   ──emit──▶  TraceStore
      │  (mirror tiap event SSE + boundary run)      ~/.pha/logs/traces/<session>.jsonl
      │  runtime tool events (runtime.py:420-457)         (append-only, per session)
      └──▶ audit.jsonl (existing, global gateway audit)
                     │
  Gateway HTTP  _route (health.py)  ◀── provider injection (pattern yang sama
      GET  /v1/sessions/{id}/runs            dengan sessions_provider hari ini)
      GET  /v1/sessions/{id}/runs/{run_id}
      DEL  /v1/sessions/{id}/trace            auth: bearer token (_requires_auth)
                     │
  WebUI  Navbar: [ Chat | Trace | Audit ]
      Trace tab : session selector → runs list → event timeline → detail JSON
                  + tombol Clear (per run / per session)
      Audit tab : tail audit.jsonl + filter session_key
```

## 3. Terminologi

| Istilah | Definisi |
|---|---|
| **Session** | Chat session (`session_key`, format `api:<uuid>` untuk WebUI; channel sessions memakai key channel). Satu session = satu file trace. |
| **Run / Process** | Satu eksekusi prompt end-to-end. Punya `run_id` (`uuid4().hex[:12]`, konvensi sama dengan session id OpenHarness). |
| **TraceEvent** | Satu baris JSONL di trace log: `{v, seq, ts, mono, session_key, run_id, kind, data}`. |
| **Trail** | Kumpulan TraceEvent milik satu session (atau satu run). |
| **Purge / Clear** | Penghapusan trail secara eksplisit (per run / per session). Log **tidak pernah** diedit diam-diam. |
| **Proyeksi derived** | Daftar run, status, durasi, jumlah tool — semuanya *dihitung* dari event, tidak disimpan terpisah. |

## 4. Prinsip desain (diadopsi & disederhanakan dari deepseek-harness `dsh-session`)

1. **Append-only log sebagai single source of truth.** Tidak ada state mutable; daftar run =
   proyeksi dari event log (dsh: `deriveMessages()` dari log, bukan disimpan).
2. **Invariant pairing tool call/result.** Setiap `tool.started` harus punya `tool.completed`;
   jika tidak ada (crash), proyeksi menandai **outcome unknown** — derive-only, log tidak
   ditulis ulang (dsh: crash repair + `TOOL_OUTCOME_UNKNOWN`).
3. **Lossless JSON di titik append.** Payload divalidasi (tanpa `BigInt`/cycle/objek eksotis)
   sebelum ditulis; event yang gagal validasi tidak pernah masuk log.
4. **Tolerant reader.** `kind` tak dikenal tetap tersimpan & dirender generik; viewer tidak
   pernah crash karena event baru (dsh: unknown events preserved verbatim).
5. **Capture di adapter layer, bukan engine.** Semua instrumentasi ada di `pha/gateway/*`
   (mirror event SSE yang sudah mengalir + boundary run). Engine OpenHarness upstream tidak
   disentuh → upgrade vendored engine tanpa merge conflict.
6. **Zero-dependency & konsisten pattern existing.** Tanpa framework baru (mengikuti gaya
   hand-rolled server `health.py` dan WebUI tanpa build step); ruff + pytest tetap gerbang kualitas.

## 5. Goals

- **G1** — 100% run yang dieksekusi WebUI tercatat `run.started` + `run.completed` (atau
  terdeteksi `interrupted` setelah crash/restart).
- **G2** — Detail per run dapat dilihat di WebUI ≤ 3 klik dari buka aplikasi.
- **G3** — Clear trail per run & per session bekerja atomik, terkonfirmasi, dan meninggalkan
  jejak `trail.purged`.
- **G4** — Zero dependensi baru; tidak ada perubahan pada `src/openharness/**`.
- **G5** — Semua endpoint & store teruji (pytest, pattern `tests/test_pha/test_webui_api.py`);
  lulus `ruff check src/pha`.

## 6. Non-goals (core)

- Live SSE tail trace di WebUI (polling 30 s dulu; live tail = PRD-4 §Future).
- Full-text search / export / share trail.
- Trace untuk subagent/swarm & channel replay ke WebUI.
- Perubahan pada TUI OpenHarness (fitur trace TUI upstream tidak disentuh).
- Redaction engine tingkat lanjut (core cukup: `summary` terpotong 160 char + sender hash,
  mengikuti `audit.py` yang sudah ada).

## 7. Metrik sukses

| Metrik | Target |
|---|---|
| Kelengkapan capture | ≥ 99% run punya pasangan `run.started`/`run.completed` atau status `interrupted` |
| Overhead per event emit | < 5 ms p95 (append file kecil + validasi) |
| Ukuran log | ~2–5 KB per run (tanpa per-delta logging); 1.000 run ≈ 2–5 MB per session |
| Latensi UI membuka trace session | < 500 ms untuk 1.000 event (paging) |
| Kualitas | pytest hijau (unit + API + UI smoke), `ruff check` bersih |

## 8. Peta dokumen & ketergantungan

| Dokumen | Isi | Depends on |
|---|---|---|
| [PRD-1](./PRD-1-trace-core.md) | Core: TraceEvent schema, TraceStore (append, seq, purge, integrity), titik instrumentasi | — |
| [PRD-2](./PRD-2-trace-api.md) | Gateway HTTP API (runs, events, purge, audit tail), auth, error contract | PRD-1 |
| [PRD-3](./PRD-3-trace-webui.md) | Sidebar collapsible + side panel Trace/Audit (+ SETUP, rev 1.1), purge trail UX | PRD-2 |
| [PRD-4](./PRD-4-quality-rollout.md) | NFR, security/privacy, strategi test, milestone rollout, risiko, future | semua |
| [PRD-5](./PRD-5-working-directory.md) | Working directory selector (wcd) per chat session: allowlist, API `cwd`, restore guard | PRD-2, PRD-3 |
| [PRD-6](./PRD-6-human-in-the-loop.md) | Human-in-the-loop permission popup (mode ≠ auto): callback engine, SSE, resolve API, timeout | PRD-3, PRD-1 |
| [PRD-7](./PRD-7-webui-uiux-overview.md) | UI/UX overhaul: prinsip, masalah, goals, peta dokumen | — |
| [PRD-8](./PRD-8-webui-design-system.md) | Design system "calm professional": token, tipografi, komponen, focus/motion | PRD-7 |
| [PRD-9](./PRD-9-webui-ux-interaction.md) | UX/IA/copy/empty-state/toast/a11y | PRD-8 |
| [PRD-10](./PRD-10-webui-setup-panel.md) | Panel SETUP: grup, label manusia, help, validasi inline | PRD-8, PRD-9 |
| [PRD-11](./PRD-11-webui-composer-mentions.md) | Composer `/` perintah + `@` sebut file/folder (popup ala TUI) | PRD-8, PRD-9 |
| [PRD-12](./PRD-12-webui-language.md) | Bahasa antarmuka WebUI: bahasa / english / japanese | PRD-8, PRD-9 |
| [PRD-13](./PRD-13-webui-spa.md) | Empat view in-place (Chat/Setup/Skills/Cron) + endpoint skills/cron | PRD-8, PRD-9, PRD-12 |
| [PRD-14](./PRD-14-webui-cron-delivery.md) | Hasil cron sampai & tersimpan di session WebUI (channel outbound `api` + history) | PRD-3, PRD-5, PRD-13 |
| [PRD-15](./PRD-15-webui-skills-crud.md) | Skills: select + preview SKILL.md + CRUD + enable/disable | PRD-8, PRD-9, PRD-12, PRD-13 |
| [PRD-16](./PRD-16-webui-cron-crud.md) | Cron: CRUD penuh (create/edit/validate/run-now) + preview next-run | PRD-8, PRD-9, PRD-12, PRD-13, PRD-14 |
| [PRD-17](./PRD-17-webui-slash-palette.md) | Palette `/` lengkap: fix `available_commands` + `/plan` dari WebUI | PRD-8, PRD-9, PRD-11 |
| [PRD-18](./PRD-18-webui-room-improvement.md) | Room improvement: hash routing, feedback spesifik, polish chat | PRD-8, PRD-9, PRD-12, PRD-13 |
| [PRD-19](./PRD-19-webui-worldclass.md) | World-class: token Claude×Notion + dark/light + toggle | PRD-7, PRD-8 |
| [PRD-20](./PRD-20-webui-production.md) | Production-grade: datetime greeting, skill split kanan, polish | PRD-19 |
| [PRD-21](./PRD-21-webui-mobile-motion-chat.md) | Mobile-friendly + animasi minimalist + bubble world-class | PRD-19, PRD-20 |
| [PRD-22](./PRD-22-rca-mobile-wrap.md) | RCA mobile: wrap normal + tabel scroll-x + resize guard | PRD-21 |
| [PRD-23](./PRD-23-hero-greeting-copy.md) | Hero greeting + copy bubble user | PRD-20, PRD-21 |
| [PRD-24](./PRD-24-greeting-fresh-token.md) | Greeting aesthetic + fresh card + token TTL | PRD-20, PRD-23 |
| [PRD-25](./PRD-25-image-native-first.md) | Gambar native-first + fallback vision sekali | — |
| [PRD-26](./PRD-26-session-effort-context.md) | Effort per-sesi + context progress bar | PRD-5, PRD-10 |
| [PRD-27](./PRD-27-pha-tool-compatibility-subagent.md) | Tools PHA-centric + `subagent` WebUI (worker PHA, auth gateway) | — |
| [PRD-28](./PRD-28-todo-refactor-opencode.md) | Refactor `todo_write` ala OpenCode (state-machine per-session) | — |
| [PRD-29](./PRD-29-webui-tools-toggle.md) | Tools: enable/disable + page Tools + filter runtime nyata | PRD-15 |
| [PRD-30](./PRD-30-agent-profile.md) | Agent profile: capability tools/skills + mandatory + effort default | PRD-29, PRD-15, PRD-26 |
| [PRD-31](./PRD-31-session-agent-effort-picker.md) | Picker agent + effort per sesi di bawah composer | PRD-30, PRD-26 |
| [PRD-32](./PRD-32-session-grouping.md) | Grouping sesi manual di panel sessions | PRD-13 |
| [PRD-33](./PRD-33-system-one.md) | System One decision layer (Laya): triage, permission refine, agent-hint, compact gate, judge | PRD-25, PRD-30 |
| [PRD-34](./PRD-34-token-usage-model-preset.md) | Token usage (reported) + model preset inline | PRD-26, PRD-30, PRD-1 |
| [PRD-35](./PRD-35-system-one-action-gate.md) | System One action gate: pre-tool verdict state kaya + output screen redact | PRD-33, PRD-6 |
| [PRD-36](./PRD-36-webui-setup-cron-friendly.md) | Setup Basic + search + editor list; Cron preset + builder + picker target | PRD-10, PRD-13, PRD-16, PRD-12 |
| [PRD-37](./PRD-37-chat-input-lega.md) | Chat input lega: maxlength 100rb + auto-grow 320px + counter 3-tier + pesan 413 ramah | PRD-11, PRD-26 |
| [PRD-38](./PRD-38-lampiran-multi-format-stdlib.md) | Lampiran multi-format stdlib: teks di-inject (50KB/file, 200KB total) + `.log` + tolak docx ramah | PRD-25, PRD-37 |
| [PRD-39](./PRD-39-ux-ramah-awam.md) | UX ramah awam menyeluruh: composer-first + error-map Indonesia + empty-state 6 halaman | PRD-7, PRD-8, PRD-9, PRD-12, PRD-36, PRD-37, PRD-38 |
| [PRD-40](./PRD-40-webui-mcp.md) | Halaman MCP: kelola server + koneksi live (config gateway.json + wiring pool + test) | PRD-12, PRD-13, PRD-29, PRD-36, PRD-39 |

## 9. Referensi

- deepseek-harness `dsh-session` (event-sourced session log):
  https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/session/README.md
- DeepWiki — Session Log & Persistence:
  https://deepwiki.com/deepseek-ai/deepseek-harness/3.3-session-log-and-persistence
- Kode terkait (baca sebelum implementasi): `pha/src/pha/audit.py`, `pha/src/pha/gateway/health.py`,
  `pha/src/pha/gateway/service.py` (`run_foreground`, `list_webui_sessions`),
  `pha/src/pha/gateway/webui/{markup,style,app_js}.py`, `pha/tests/test_pha/test_webui_api.py`,
  `openharness/engine/stream_events.py`, `openharness/ui/protocol.py`.
