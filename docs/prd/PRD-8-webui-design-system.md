# PRD-8 — WebUI: Design System (Calm Professional)

| | |
|---|---|
| Feature | Token visual baru + restyle komponen `nb-*` |
| Depends on | PRD-7 |
| Files | `pha/src/pha/gateway/webui/style.py` (token + `APP_CSS`), dikonsumsi `markup.py`/`app_js.py` |
| Test | `pha-agent/tests/test_pha/test_webui_api.py:1570-1596` |

## 1. Ringkasan

Ganti isi `NEUBRUTAL_CSS` (nama dipertahankan agar import stabil) menjadi tema tenang. **Nol JS baru.** Seluruh perubahan = redefinisi nilai token + sentuhan pada aturan komponen. Kelas `.nb-card`/`.nb-btn--yellow` tetap ada (di-assert test).

## 2. Token (`:root`)

| Token | Nilai | Ganti dari |
|---|---|---|
| `--bg` | `#f7f7f5` | `#fdf6ec` |
| `--surface` | `#ffffff` | (tetap) |
| `--surface-2` | `#f1f2f4` | baru (area sekunder) |
| `--ink` | `#1f2937` | `#171412` |
| `--ink-2` | `#5b6472` | baru (teks sekunder) |
| `--line` | `#e3e4e8` | baru (border tipis) |
| `--accent` | `#2563eb` | `--nb-blue` |
| `--accent-soft` | `#e8effd` | baru |
| `--ok` / `--warn` / `--danger` | `#15803d` / `#b45309` / `#b91c1c` | hijau/kuning/merah cerah |
| `--font` | `system-ui, -apple-system, "Segoe UI", Roboto, Arial, sans-serif` | `Helvetica Neue` |
| `--font-mono` | `ui-monospace, "Cascadia Mono", Consolas, monospace` | `Courier New` |
| `--border` | `1px` | `3px` |
| `--radius` | `10px` (sm 8, lg 14) | `0px` |
| `--shadow-sm` | `0 1px 2px rgba(16,24,40,.06)` | hard offset 6px |
| `--shadow-md` | `0 4px 12px rgba(16,24,40,.08)` | hard offset 8px |
| `--space` | skala 4/8/12/16/24/32 | — |
| `--speed` | `150ms ease` | 80ms |

Alias lama (`--nb-*`) dipertahankan sebagai pointer ke token baru supaya aturan yang belum disentuh tetap valid — ponytail: hindari sapu-bersih total.

## 3. Tipografi

- Base 16px, `line-height: 1.55`.
- Skala: meta 13px · kecil 14px · body 16px · h3 18px · h2 22px · h1 28px.
- Weight: 400 body, 600 heading, 700 maksimum.
- `text-transform: uppercase` **hanya** untuk label pendek (badge/chip ≤2 kata), `letter-spacing: 0.02em`.
- Hapus `letter-spacing: -0.02em` pada heading dan greeting miring.

## 4. Komponen (restyle, kelas sama)

| Komponen | Perubahan |
|---|---|
| `body` | font `--font`, `line-height: 1.55`, latar `--bg` |
| `.nb-card` | `border: 1px solid var(--line); radius: 10px; box-shadow: var(--shadow-sm)` |
| `.nb-btn` | radius 8px, border 1px, tanpa shadow keras; hover = naikkan 1px + `shadow-sm`; variant = `background` lembut, teks `--ink` (bukan putih kecuali danger) |
| `.nb-input/.nb-select/.nb-textarea` | border 1px `--line`, radius 8px, fokus = ring 2px `--accent-soft` + border `--accent` |
| `.nb-badge` | pil radius 999px, border 1px, teks 13px |
| `.nb-chip` | radius 999px, fokus-visible jelas; aktif = `accent-soft` + teks `--accent` |
| `.nb-tag` | tag tool: mono 13px, latar `surface-2`, border `--line` |
| `.nb-alert` | border 1px, latar lembut (bukan warna pekat), teks `--ink` |
| `.nb-modal` | radius 14px, `shadow-md`, overlay `rgba(16,24,40,.45)` |
| `.nb-navbar` | border-bottom 1px `--line`, tinggi 56px |
| `.nb-greeting` | hilangkan rotasi; jadi teks kecil netral `--ink-2` |

Bubble chat: user = `accent-soft` + border `--accent` tipis; assistant = `surface` + border `--line`; meta 13px `--ink-2`. Status "THINKING…" → teks kecil `--ink-2` + spinner CSS (opsional, tanpa aset).

## 5. Focus, motion, responsive

- **Focus:** `:focus-visible { outline: 2px solid var(--accent); outline-offset: 2px }` di semua kontrol; hapus ketergantungan `transform` untuk indikator fokus.
- **Motion:** transisi ≤150ms; `@media (prefers-reduced-motion: reduce) { * { animation: none !important; transition: none !important } }`; hapus `nb-pulse` tak berujung (ganti indikator progres statis/dot).
- **Responsive:** breakpoint `860px` dipertahankan; tambah target sentuh ≥40px; panel kanan jadi overlay penuh di layar sempit.
- **Color scheme:** `:root { color-scheme: light }`; opsional `@media (prefers-color-scheme: dark)` dengan token gelap (nilai tambah, boleh ditunda).

## 6. Acceptance criteria

- **AC-8.1** `--nb-*` di `style.py` memetakan ke token baru; tak ada `font-size` <13px di `APP_CSS`.
- **AC-8.2** Body 16px, meta ≥13px; heading tanpa `letter-spacing` negatif.
- **AC-8.3** Semua kontrol punya `:focus-visible`; tak ada elemen `.nb-greeting` berotasi.
- **AC-8.4** `prefers-reduced-motion` mematikan animasi; tak ada animasi tak berujung.
- **AC-8.5** Kelas `nb-card`, `nb-btn--yellow` tetap ada; smoke test `test_webui_api.py:1570-1596` hijau; `external == []`.

## 7. Testing

1. Smoke test HTML existing (id + kelas + `external == []`).
2. Cek kontras token dengan hitungan manual/WCAG (dokumentasikan pasangan di PR).
3. `node --check` atas `APP_JS` hasil ekstraksi (pola existing).
4. QA visual manual: chat, sidebar, panel trace/audit/setup, modal, drawer permission, mobile <860px.
