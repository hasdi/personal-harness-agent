# PRD-7 — WebUI: UI/UX Overhaul (Overview & Index)

| | |
|---|---|
| Feature | Redesign UI/UX WebUI pha-agent: tema tenang, mudah dibaca, ramah user 30+ |
| Scope | Visual design system, IA & interaksi, redesign panel SETUP |
| Target | `pha-agent/src/pha/gateway/webui/{style,markup,app_js}.py` (+ `gateway/config.py` untuk meta SETUP) |
| Status | Draft v1.0 (2026-09-12) |
| Dokumen | PRD-7 (ini) · [PRD-8](./PRD-8-webui-design-system.md) · [PRD-9](./PRD-9-webui-ux-interaction.md) · [PRD-10](./PRD-10-webui-setup-panel.md) |

## 1. Masalah (bukti dari code)

Desain sekarang neubrutalist keras — playful, tapi melelahkan dan sulit dibaca untuk user 30+:

| Gap | Bukti |
|---|---|
| Font terlalu kecil | meta 0.68–0.72rem (~11px), label config 0.7rem (`style.py:293,499,515,525`) |
| Semua all-caps + border 3px + shadow keras 6px + radius 0 | `style.py:7-9,35-44,59-107` |
| Warna cerah bertumpuk (kuning/pink/biru/hijau) + greeting miring −1.2° | `style.py:14-20,558-563` |
| Panel SETUP = dump mentah `gateway.json`: label mono, JSON textarea, tanpa grup/help | `app_js.py:1583-1654` |
| Aksi kriptik & jargon: `DIR ⌕`, `ADD TO ALLOWLIST & USE`, `CLEAR TRAIL`, `SESSION`/`ALL` | `markup.py:32,132`, `app_js.py:397` |
| Empty state minimal ("memuat…", "audit kosong"), error dicampur ke log chat | `markup.py:24,83`, `app_js.py:1193` |

## 2. Prinsip desain

1. **Tenang, bukan ramai** — satu warna aksen, latar netral, shadow lembut, border tipis.
2. **Terbaca untuk 30+** — body ≥16px, meta ≥13px, kontras ≥4.5:1, target sentuh ≥40px.
3. **Bahasa manusia** — label Indonesia, bukan name field mentah atau singkatan kapital.
4. **Rendah noise** — aksi sekunder muncul saat hover/fokus, bukan selalu tampil.
5. **Aksesibel & keyboard-first** — landmark, `:focus-visible`, `aria-live`, `prefers-reduced-motion`.
6. **Zero-dep, zero-build** — reuse kelas `nb-*` + token CSS; tanpa framework/CDN baru.

## 3. Goals

- **G1** — Semua teks utama ≥16px, meta ≥13px, kontras WCAG AA (≥4.5:1).
- **G2** — Panel SETUP tampil sebagai grup berlabel Indonesia + help text, tanpa JSON mentah untuk field umum.
- **G3** — Empty state & first-run menjelaskan langkah berikutnya (bukan "memuat…").
- **G4** — Semua kontrol punya label/tooltip jelas; tanpa singkatan kapital tanpa arti.
- **G5** — Smoke test HTML + `ruff` + pytest tetap hijau; tak ada URL eksternal baru.

## 4. Non-goals

- Framework/library UI, build step, atau dependency baru.
- Perubahan engine OpenHarness (`src/openharness/**`) atau endpoint API (kecuali penambahan meta di `/v1/config`).
- Redesign alur data (SSE, trace, HITL) — hanya presentasinya.
- Dark mode default (opsi via `prefers-color-scheme` = nilai tambah, bukan inti).
- Mobile app / PWA.

## 5. Metrik sukses

| Metrik | Target |
|---|---|
| Ukuran font minimum (meta) | ≥13px |
| Rasio kontras teks utama | ≥4.5:1 (`--ink` di `--bg`/`--surface`) |
| Target sentuh tombol | ≥40×40px |
| Langkah pindah tema | 0 (token `:root` saja, tanpa JS) |
| Smoke test HTML | 100% hijau, `external == []` |

## 6. Peta dokumen

| Dokumen | Isi | Depends |
|---|---|---|
| [PRD-8](./PRD-8-webui-design-system.md) | Token visual, tipografi, komponen, focus, motion, responsive | — |
| [PRD-9](./PRD-9-webui-ux-interaction.md) | IA navbar/sidebar/chat, empty state & onboarding, feedback (toast), copy, a11y | PRD-8 |
| [PRD-10](./PRD-10-webui-setup-panel.md) | Panel SETUP: grup, label manusia, help, validasi inline, restart banner | PRD-8, PRD-9 |
| [PRD-11](./PRD-11-webui-composer-mentions.md) | Composer `/` perintah + `@` sebut file/folder (popup ala TUI) | PRD-8, PRD-9 |
| [PRD-12](./PRD-12-webui-language.md) | Bahasa antarmuka: bahasa / english / japanese | PRD-8, PRD-9 |
| [PRD-13](./PRD-13-webui-spa.md) | Empat view in-place (Chat/Setup/Skills/Cron) + menu Skills & Cron | PRD-8, PRD-9, PRD-12 |
| [PRD-14](./PRD-14-webui-cron-delivery.md) | Hasil cron sampai ke session WebUI (channel outbound `api`) | PRD-3, PRD-5, PRD-13 |

## 7. Referensi kode

- Token & komponen: `pha/src/pha/gateway/webui/style.py:6-214`
- Layout CSS: `style.py:217-613`
- Markah: `pha/src/pha/gateway/webui/markup.py:5-161`
- Logika & render config: `pha/src/pha/gateway/webui/app_js.py:1583-1704`
- Konfig server: `pha/src/pha/gateway/config.py`, `pha/src/pha/gateway/health.py` (`/v1/config`)
- Smoke test: `pha-agent/tests/test_pha/test_webui_api.py:1570-1596`
