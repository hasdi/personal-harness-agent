# PRD-3 — WebUI: Panel Trace & Audit + Clear Trail

| | |
|---|---|
| Feature | Sidebar collapsible, side-panel Trace/Audit side-by-side dengan chat, purge trail |
| Depends on | PRD-2 (endpoint) |
| Files | `pha/src/pha/gateway/webui/{markup.py, style.py, app_js.py, __init__.py}`; test: `tests/test_pha/test_webui_api.py` |

> **Revision v1.1 (UX review).** Tab atas (Chat/Trace/Audit) diganti: **sidebar Sessions
> permanen + collapsible** (tombol ☰ di chat header, persist di localStorage), dan **Trace +
> Audit menjadi side-panel di kanan chat** (show/hide via tombol `TRACE`/`AUDIT` di chat
> header, tab mini di panel header, tombol `×`). Panel **selalu mengikuti session aktif di
> sidebar** — tidak ada session selector duplikat; Audit punya toggle `SESSION`/`ALL`.
> Polling 30 s hanya saat panel terbuka pada tab terkait.

## 1. Ringkasan

WebUI tetap **satu halaman tanpa router/framework**: panel = toggle `hidden` (pattern
existing; `WEBUI_HTML` dirender ulang hanya setelah restart gateway). Struktur:

- `#pha-layout` grid: `#sidebar` (collapsible) + `#main`.
- `#main` flex: `#chat-pane` (selalu tampil) + `#side-panel` (Trace/Audit, collapsible).
- `#side-panel` berisi tab mini `TRACE | AUDIT` + konten pane masing-masing.

## 2. Struktur UI

```
#pha-navbar   [ PHA ]                          ● ONLINE  model/provider/channel badges
#pha-layout   ┌ #sidebar (collapsible ☰) ┐ ┌ #main ─────────────────────────────────┐
              │ Sessions  [+ NEW]        │ │ #chat-pane        [☰][title] [TRACE|AUDIT] │
              │ • session aktif …        │ │ #log / #composer                            │
              └──────────────────────────┘ │ #side-panel (×) [TRACE|AUDIT]               │
                                           │   trace: chips + CLEAR TRAIL + runs/events  │
                                           │   audit: [SESSION|ALL] + REFRESH + rows     │
                                           └─────────────────────────────────────────────┘
#modal-overlay (existing; dipakai untuk konfirmasi purge)
```

Semantic: `<aside>` sidebar, `<main>` + `<section>`; tab panel memakai
`role="tab"` + `aria-selected`; pane memakai `hidden` attribute.

## 3. Perilaku

### 3.1 Sidebar collapsible
- Tombol `☰` di chat header toggle sidebar; state persist di `localStorage("pha_sidebar")`.
- Mobile (<860 px): sidebar jadi baris atas yang bisa disembunyikan; panel penuh.

### 3.2 Side panel — show/hide
- Tombol `TRACE`/`AUDIT` di chat header: buka panel pada tab tsb; klik lagi = tutup;
  klik tombol satuan = pindah tab. `×` di panel header menutup.
- **Semua query mengikuti session aktif** (`state.sessionId` dari sidebar): belum ada
  session → hint "pilih session di sidebar…".
- `openSession()`/`newChat()` me-refresh panel bila terbuka.

### 3.3 Trace panel — load
1. Panel terbuka (tab Trace) → `GET /v1/sessions/{id}/runs` → meta + kartu run (terbaru dulu).
2. Klik run → `GET .../runs/{run_id}?limit=500` → timeline; polling 30 s.
3. Chips filter: All / Errors / Tools (client-side).

### 3.4 Render event (tolerant reader)
- Setiap event row: **badge kind** (run=yellow, tool=ink, progress=gray, error=red) + `ts`
  lokal (HH:MM:SS) + summary ringkas.
- Summary per kind: `tool.completed` → tool + summary + durasi + ✓/✗; `tool.started` → tool +
  summary; `run.started` → prompt_summary; `progress` → message; `llm.response` → text_len;
  `run.completed` → status + durasi; `trail.purged` → "trail dibersihkan (scope)";
  **kind tak dikenal → `# Unknown <kind>` + payload JSON tetap bisa dibuka** — viewer tidak
  pernah crash.
- Run tanpa `run.completed` → badge `interrupted` (kuning); tool `outcome:"unknown"` →
  ditandai "? unknown" tanpa menghukum status.
- Detail JSON: `<pre class="trace-detail">` — klik lagi untuk menutup.

### 3.5 Clear trail (purge)
1. Tombol `CLEAR TRAIL` (per session) / `CLEAR RUN` (per run) → modal konfirmasi
   (reuse `#modal-overlay` + `nb-modal`): teks eksplisit *"Hapus permanen … tidak bisa
   dibatalkan."*
2. Konfirmasi → `DELETE /v1/sessions/{id}/trace` (+`/{run_id}` bila per run) → reload panel.
3. Setelah purge session, meta menampilkan 1 event `trail.purged` — dirender sebagai baris
   biasa, bukan error.
4. **Tidak ada purge tanpa modal**; tombol tidak pernah melakukan destructive action langsung.

### 3.6 Audit panel
- Default **session-scoped** (`?session_key=` dari session aktif); chip `SESSION`/`ALL`
  membalik cakupan; `REFRESH` manual + polling 30 s saat panel terbuka.
- Row: ts, event, channel, session_key (dipotong), detail; error → merah.
- Sender tidak ditampilkan penuh (sudah hash di sumber — jangan coba "memperkaya").

### 3.7 State & error
| State | Tampilan |
|---|---|
| Empty | "belum ada trail untuk session ini" (Trace) / "audit kosong" |
| Loading | polling silent; list terakhir tetap tampil |
| Error | `#trace-runs`/`#audit-list` menampilkan "gagal memuat …"; `catch` mengikuti pattern `e.message !== "unauthorized"` (`app_js.py:71-81`) |
| Unauthorized | biarkan `handleUnauthorized` existing bekerja |

## 4. Kontrak kode (kualitas & low-maintenance)

- Semua state baru di objek `state` existing (`state.trace = {runs, activeRun, filter}`,
  `state.audit = {events, all}`, `state.panel = {open, tab}`) — tidak ada variabel global baru.
- Satu fungsi render per pane (`renderTracePane()`, `renderAuditPane()`), DOM dibangun dengan
  helper existing; **tidak ada template literal HTML dari data tanpa `escHtml`** (XSS).
- Fungsi kecil & murni: `summaryForEvent(ev)`, `kindBadge(kind)`, `formatTs(iso)`.
- Tidak ada CDN/dependency eksternal (smoke test existing menolak URL eksternal —
  `test_webui_api.py:414-427`).
- CSS: reuse token `neubrutal` (`style.py:6-24`) + class `nb-card/nb-btn/nb-badge/nb-tag`;
  hanya tambah maksimal ~60 baris `APP_CSS` untuk layout timeline.

## 5. Acceptance criteria

- **AC-3.1** Sidebar Sessions tampil permanen dan bisa collapse via `☰` (persist di localStorage); Trace/Audit diakses dari tombol chat header atau tab mini panel.
- **AC-3.2** Side panel tampil side-by-side dengan chat; show/hide tidak me-reload halaman; Trace & Audit **selalu mengikuti session aktif di sidebar** tanpa selector duplikat; `openSession()`/`newChat()` me-refresh panel yang terbuka.
- **AC-3.3** Trace: klik run → event timeline; klik event → detail JSON (klik lagi menutup).
- **AC-3.4** Filter chips All/Errors/Tools mengubah daftar event tanpa request tambahan.
- **AC-3.5** Clear per session & per run selalu melalui modal konfirmasi; setelah sukses data direfresh; hasil purge terlihat sebagai event `trail.purged`.
- **AC-3.6** Audit: tampil newest-first, default mengikuti session aktif dengan toggle `SESSION`/`ALL`, error row merah.
- **AC-3.7** Polling trace/audit hanya jalan saat panel terbuka pada tab terkait (interval dibersihkan saat menutup/pindah tab).
- **AC-3.8** Semua string dinamis lolos `escHtml`; smoke test HTML (id elemen baru + semua path API baru + tetap zero URL eksternal) lulus.
- **AC-3.9** Kind tak dikenal / file dengan event versi lebih baru dirender generik tanpa error di console.

## 6. Testing

1. Extend smoke test HTML: daftar id (`#side-panel`, `#btn-trace`, `#btn-audit`,
   `#sidebar-toggle`, `#trace-runs`, `#audit-list`, …) + path (`/v1/sessions/{id}/runs`,
   `/v1/audit`, DELETE trace) di assertions existing (`test_webui_api.py`).
2. Unit JS-level tidak tersedia (tanpa build) → kover lewat smoke test + test API (PRD-2 §6)
   + `node --check` atas `APP_JS` yang diekstrak.
3. Manual QA checklist (di PRD-4 §4) untuk alur sidebar/panel + purge.
