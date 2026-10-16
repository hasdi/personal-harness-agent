# PRD-23 — Hero Greeting + Copy Bubble User

| | |
|---|---|
| Feature | Greeting hero aesthetic + tombol copy di bubble user |
| Depends on | PRD-20, PRD-21 |
| Files | `webui/{markup,style,app_js,i18n_js}.py`, `tests/test_pha/test_webui_api.py` |
| Status | Draft v1.0 (2026-09-13) |

## 1. Ringkasan

Welcome card jadi hero tengah: eyebrow tanggal, judul gradien besar, sapaan
personal ("Halo, Hasdi ✦"), langkah bernomor. Bubble user dapat tombol COPY
(muncul saat hover/fokus, selalu tampil di touch).

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Hero | `#welcome-card` max 560px, margin auto, eyebrow uppercase + tanggal live, `h3` clamp 26–36px gradien ink→accent, langkah jadi kartu bernomor (counter CSS) |
| Personal | `wc-hello` = `greet_hi` tanpa koma, `wc-name` = userName/agent/"kawan", `wc-date` = `greetDateText()` (ikut bahasa) |
| Copy user | `attachCopyAction()` COPY-only (tanpa REGEN); guard anti-duplikat; `_raw` diset untuk live + history user |
| A11y/i18n | `aria-label` via key `copy_msg` (3 bahasa); reuse pola clipboard existing |

## 3. Goals

- **G1** — Hero tampil besar di tengah saat first-run, personal, tanggal live.
- **G2** — Setiap bubble user bisa di-copy (live + history).

## 4. Non-goals

- REGEN untuk user, endpoint baru, perubahan tema.

## 5. Acceptance criteria

- **AC-23.1** First-run: hero tengah + tanggal + nama benar; ganti bahasa → sapaan ikut.
- **AC-23.2** Hover bubble user → COPY; klik → COPIED; history (reload) tetap ada COPY.
- **AC-23.3** Smoke test assert `wc-date/wc-hello/wc-name` + `attachCopyAction` + `copy_msg`.

## 6. Testing

1. `pytest tests/test_pha/test_webui_api.py -q -k webui_html`.
2. `ruff check src/pha/gateway/webui/`.
3. Manual: first-run hero, kirim pesan → COPY user, reload → COPY tetap ada.
