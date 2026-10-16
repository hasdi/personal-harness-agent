# PRD-44 — WebUI v2: Chat Modern (Streaming + Composer + Render Aman)

| | |
|---|---|
| Feature | Chat modern 2026: optimistic bubble + SSE throttled + virtualisasi + re-attach + composer lega + upload + render `marked+DOMPurify+highlight+diff+KaTeX+mermaid-vendor` |
| Depends on | PRD-42 (token/komponen), PRD-43 (shell/store/api/a11y), PRD-11/17 (palette `/`@), PRD-25/38 (gambar/lampiran), PRD-26/31/37 (effort/konteks/composer) |
| Files | `src/pha/gateway/webui/v2/js/chat.js+render.js` (baru), `src/pha/gateway/webui/v2/vendor/*` (baru, committed), `src/pha/gateway/webui/v2/static.py` (baru, daftar vendor), `tests/test_pha/test_webui_v2.py` (tambah: XSS + error-map + vendor) |
| Status | Draft v1.0 (2026-10-15) |

## 1. Ringkasan

Chat v1 sudah lengkap (`runStream` SSE + multi-run + re-attach + STOP + fallback blocking + upload 10MB + palette `/`@ + mermaid zoom/pan) tapi: `mdToHtml` per `delta` + `scrollBottom` tiap chunk (berat sesi panjang), tanpa virtualisasi, render tanpa sanitasi terpusat, mermaid via CDN (gagal offline), `#char-count` menimpa `#palette`, codeblock selalu dark, copy/regen/fork hover-reveal sulit ditemukan.

PRD-44 membangun ulang chat di atas store/api v2 dengan **kontrak SSE tak berubah** (`run/delta/progress/tool_hint/file_edit/error/cron/final/done`, `attach_run`, `cancel_run`, `session_key api:<uuid>`) + pola UX nanobot yang diport (streaming render, optimistic bubble, reconnect/hydration — bukan copy komponen).

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Optimistic + throttle | Bubble user langsung; delta diakumulasi + render via `requestAnimationFrame` (bukan per chunk); `scrollBottom` hanya bila user di bawah (jaga posisi baca); REGEN hanya balasan terakhir; fork di titik pesan |
| Virtualisasi/paging | Paging `GET /v1/sessions/:id?limit=100&before=` jaga `scrollTop` (reuse `loadOlderHistory`); render markdown final per bubble, streaming hanya bubble aktif; `IntersectionObserver` untuk mermaid/highlight tunda |
| Re-attach | Reload → `GET /v1/runs/active` + `attach` replay+live (pola `maybeReattach/attachRun`); `409` → attach; `404/no-body` → fallback blocking `/v1/chat/completions`; STOP → `POST /v1/runs/:id/cancel` (tombol kirim jadi Stop merah) |
| Composer lega | Auto-grow ≤320px, `maxlength` 100rb + counter 3-tier (PRD-37), hint `/` perintah · `@` file · 📎 lampir; tombol `Lampirkan` teks (bukan ikon saja); kirim fokus balik ke composer; picker agent + effort + preset per sesi (reuse `POST /v1/sessions/:id/{agent,effort,preset}`, PRD-31) |
| Upload | `POST /v1/upload` (limit + allowlist existing): chip nama + ukuran + hapus, progress, sukses sebut nama file, gagal sebut sebab + solusi; drag-drop/paste/klik; teks di-inject (50KB/file, 200KB total, PRD-38) + preview N baris; gambar native-first (PRD-25) → lightbox (zoom + unduh) |
| Render aman | `marked` vendored → **DOMPurify** → `highlight.js` (code) → `diff` (LCS cap 400) → KaTeX (matematika) → `mermaid` vendored lazy + zoom/pan; tabel/kode/diff collapsible + tombol salin + pratinjau `html/md/diff`; `escHtml` semua dinamis |
| Vendor offline | `marked.min.js + dompurify.min.js + highlight.min.js + diff.min.js + katex/ + mermaid.min.js` di-commit; `static.py` jadi single source daftar file (tanpa CWD-relatif); cek size/head saat commit (cegah HTML-error-as-binary, taste workflow) |
| Error-map | Tabel teknis → Indonesia (PRD-39): `413` → "Pesan terlalu besar — pecah atau lampirkan file"; `run already in progress` → "Masih menjawab — tunggu atau Stop"; tanpa kata `413/400/--flag/mime` |
| Context bar | Progress konteks + effort per sesi (PRD-26) + `hist-count`; header tak sembunyikan title di mobile (perbaiki v1) |
| Tier lean | **High-rigor** untuk upload + token/attachments (keamanan); Standard untuk sisanya |

## 3. Call-site

| # | Lokasi | Guna | Fallback |
|---|---|---|---|
| 1 | `v2/js/chat.js` | Sesi/history/paging, `runStream` SSE, composer/palette/upload, STOP/re-attach/fallback, context bar | Error mentah existing bila map hilang |
| 2 | `v2/js/render.js` | marked→sanitize→highlight→diff→KaTeX→mermaid + lightbox + copy/preview | Source mentah bila vendor gagal load |
| 3 | `v2/vendor/*` | Lib committed offline | — (wajib ada; smoke gagal bila hilang) |
| 4 | `v2/static.py` | `ROOT`, daftar vendor, serve bytes | — |
| 5 | Endpoint reuse | `POST /v1/chat/stream`, `/v1/chat/completions`, `/v1/runs/active`, `/v1/runs/:id/cancel`, `/v1/sessions*`, `/v1/upload`, `/v1/attachments?token=`, `/v1/commands`, `/v1/dir/*`, `/v1/notifications` | Tanpa endpoint baru |

## 4. Goals

- **G1** — Sesi panjang (1000+ pesan) tetap halus: throttle rAF + paging + render tunda.
- **G2** — Nol XSS: semua output model lewat DOMPurify + test injeksi.
- **G3** — Offline penuh: tanpa CDN/URL eksternal (`external == []`).

## 5. Non-goals

- Ubah kontrak SSE/endpoint; editor view lain (PRD-45 s.d. 47); voice-input; pencarian full-text riwayat (backlog).

## 6. Acceptance criteria

- **AC-44.1** Suite + lint hijau; `node --check` JS; string baru 3 bahasa; tanpa dep/build baru.
- **AC-44.2** Kirim → bubble optimistis; streaming halus tanpa jank; user di tengah bacaan tak dipaksa scroll ke bawah; REGEN hanya balasan terakhir.
- **AC-44.3** Reload saat run aktif → re-attach lanjut; STOP hentikan; gagal stream → fallback blocking berhasil.
- **AC-44.4** Upload: progress + chip nama/ukuran/hapus; txt + docx (sukses + gagal ramah); paste/drag-drop/klik semua jalan; lampiran teks terinject + preview.
- **AC-44.5** Injeksi `<script>`/`<img onerror>` model tampil netral (test XSS); kode tersorot + salin; mermaid render offline + zoom/pan; diff collapsible.
- **AC-44.6** Error-map: tiap error tampil Indonesia tanpa `413/400/--flag/mime`; fokus balik ke composer setelah kirim.
- **AC-44.7** Vendor: semua file ada + size wajar + head bukan HTML-error; `external == []`.

## 7. Testing

1. `pytest tests/test_pha/test_webui_v2.py -q` (smoke shell + XSS + error-map + daftar vendor).
2. `ruff check src/pha/gateway/webui/v2` + `node --check` (`chat.js`, `render.js`).
3. Manual sebagai user awam: kirim + lampir + STOP + reload-reattach + fork + regen; uji 360px + screen-reader; uji offline (putus internet → mermaid tetap render).

## 8. Out of scope / backlog

- Virtualisasi windowing penuh (overscan dinamis); pencarian riwayat; voice-input — backlog bila ada demand.

## 9. Referensi

- `src/pha/gateway/webui/app_js.py:1160-2358` (`runStream`, history, upload, palette), `markdown_js.py`, `mermaid_js.py`, `health.py:336-522` (SSE), `service.py` (`handle_api_prompt_stream`, `attach/detach_run`)
- `docs/prd/PRD-11-webui-composer-mentions.md`, `PRD-17-webui-slash-palette.md`, `PRD-25-image-native-first.md`, `PRD-26-session-effort-context.md`, `PRD-31-session-agent-effort-picker.md`, `PRD-37-chat-input-lega.md`, `PRD-38-lampiran-multi-format-stdlib.md`, `PRD-39-ux-ramah-awam.md`
- `.commandcode/skills/frontend-ui-engineering/SKILL.md` (§Loading: skeleton + optimistic)
