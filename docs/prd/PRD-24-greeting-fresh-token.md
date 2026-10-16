# PRD-24 — Greeting Aesthetic + Fresh Card + Token TTL

| | |
|---|---|
| Feature | Greeting navbar aesthetic, greeting card tiap new chat, token expiry configurable |
| Depends on | PRD-20, PRD-23 |
| Files | `webui/{markup,style,app_js,i18n_js}.py`, `gateway/{models,service}.py`, `tests/test_pha/{test_webui_api,test_health_config}.py` |
| Status | Draft v1.0 (2026-09-13) |

## 1. Ringkasan

Tiga permintaan: greeting "Halo …" di navbar dibuat aesthetic, greeting card
juga tampil di tiap "new chat" (bukan cuma first-run), dan token browser bisa
kedaluwarsa sesuai config.

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Greeting navbar | Avatar gradien inisial + halo, nama gradien ink→accent; animasi pulse pindah ke avatar; `g-dot` lama dibuang |
| Fresh card | `#fresh-card` compact hero (sapaan + tanggal + hint) tampil di setiap log kosong non-first-run; `updateWelcome()` memilih: first-run → welcome-card, kosong lain → fresh-card, ada pesan → keduanya sembunyi |
| Token TTL | `GatewayConfig.webui_token_ttl_hours: float = 0`; `0` = tidak kedaluwarsa (perilaku lama); hot-apply (tanpa restart); field tampil di SETUP grup keamanan |
| Mekanisme | Browser simpan `pha_token_at` saat token disimpan; `apiFetch` cek `tokenExpired()` sebelum request → hapus token + toast + modal login ulang; TTL dibaca dari `/status token_ttl_hours` tiap `refreshStatus` |
| i18n | `token_expired` + `fresh_hint` 3 bahasa |

## 3. Goals

- **G1** — Greeting navbar aesthetic, konsisten light/dark.
- **G2** — Tiap new chat / sesi kosong menampilkan greeting card personal.
- **G3** — Admin bisa set masa berlaku token (mis. 24 jam) dari SETUP.

## 4. Non-goals

- Rotasi token server-side / refresh token otomatis; expiry per-token ganda.

## 5. Acceptance criteria

- **AC-24.1** Navbar: avatar inisial + nama gradien; tanggal tetap live.
- **AC-24.2** New chat (sesi sudah ada) → fresh card tampil; kirim pesan → hilang; reload sesi kosong → tampil lagi.
- **AC-24.3** TTL 0 → token tidak pernah kedaluwarsa; TTL 24 → token >24 jam memicu login ulang + toast.
- **AC-24.4** Smoke + config test hijau; `ruff` + `node --check` hijau.

## 6. Testing

1. `pytest tests/test_pha/test_webui_api.py -q -k webui_html tests/test_pha/test_health_config.py -q`.
2. `ruff check src/pha/gateway/`.
3. Manual: first-run hero, new chat fresh card, set TTL 0.05 (≈3 mnt) → tunggu → request → modal + toast.
