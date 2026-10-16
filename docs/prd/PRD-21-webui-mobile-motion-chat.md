# PRD-21 — WebUI Mobile + Motion + Chat Palette

| | |
|---|---|
| Feature | Mobile-friendly chat, animasi minimalist, bubble world-class |
| Depends on | PRD-19, PRD-20 |
| Files | `webui/{markup,style,app_js}.py`, `tests/test_pha/test_webui_api.py` |
| Status | Draft v1.0 (2026-09-12) |

## 1. Ringkasan

Tiga permintaan: WebUI ramah mobile, sentuhan animasi minimalist, bubble chat
berwarna world-class (tanpa transparan). Semua reuse token `:root` + pola
drawer/animasi existing — nol dep baru.

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Mobile ≤860px | Sidebar jadi drawer slide-over (`#sidebar-scrim` + `#sidebar-close` + tombol `#sessions-btn` di chat-header); tertutup default saat load mobile; auto-tutup saat pilih/buat sesi; Esc menutup |
| Composer mobile | Sticky bottom + `env(safe-area-inset-bottom)`, padding rapat, kirim min 76px |
| Side panel mobile | Bottom-sheet 78dvh, radius atas 16px; skill split collapse (reuse PRD-20) |
| Greeting mobile | Disembunyikan agar navbar muat; badge dibatasi lebar + ellipsis |
| Animasi | Satu kurva `cubic-bezier(0.22,1,0.36,1)`, 140–220ms, transform+opacity saja: msg-in, toast, palette/modal/drawer pop, press-scale, typing dots, dot-pulse; semua mati via `prefers-reduced-motion` |
| Bubble | Radius 18px ala chat (sudut tajam di sisi pengirim); user = coral solid (light) / peach-tint (dark); assistant = paper hangat (light) / lift hangat (dark); log punya `--chat-log-bg`; status "Berpikir…" = chip tinted (bukan transparan) |
| Pending | `pending status` (chip) sejak awal; `textContent` di-escape (XSS-safe); timer tick ikut scroll |

## 3. Goals

- **G1** — Chat nyaman satu tangan di 360px: drawer, composer aman notch, tanpa scroll horizontal.
- **G2** — Animasi terasa hidup tapi tenang; nol animasi saat reduced-motion.
- **G3** — Nuansa chatting jelas: user vs assistant vs status vs error beda warna; kontras AA.

## 4. Non-goals

- Framework/anim lib, endpoint baru, engine `openharness/**`, PWA, haptik.

## 5. Acceptance criteria

- **AC-21.1** 360px: sidebar drawer + scrim; pilih sesi menutup drawer; composer tidak tertutup keyboard inset.
- **AC-21.2** Bubble user/assistant/status/error semua punya background (tidak transparan); dark/light konsisten.
- **AC-21.3** Smoke test assert drawer ids + token `--chat-*`; `ruff` + `node --check` hijau.

## 6. Testing

1. `pytest tests/test_pha/test_webui_api.py -q -k webui_html`.
2. `ruff check src/pha/gateway/webui/`.
3. Manual: 360px vs 1280px, drawer/scrim/Esc, kirim pesan (pending chip → bubble), toggle tema.
