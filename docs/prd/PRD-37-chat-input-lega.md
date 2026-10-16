# PRD-37 — Chat Input Lega (Naik ke 100rb)

| | |
|---|---|
| Feature | Composer lega: `maxlength` 20rb → 100rb + auto-grow 320px + tombol expand + counter 3-tier + warning konteks — tanpa ubah limit body server |
| Depends on | PRD-11 (palette composer), PRD-26 (ctx-bar effort/context) |
| Files | `pha/gateway/webui/{markup,app_js,i18n_js,style}.py`, `pha/gateway/health.py` (pesan error saja), `tests/test_pha/test_webui_api.py`, `docs/prd/README.md` |
| Status | Implemented (2026-10-14) |

## 1. Ringkasan

Composer saat ini sempit untuk pesan panjang: `maxlength="20000"` (`markup.py:103`), `autoGrow` mentok `160px` (`app_js.py:2006-2008`), `char-count` baru tampil `>2000` dengan teks `n / 20.000` (`app_js.py:2013-2017`). User minta "input unlimit"; keputusan lean: naik ke **100.000 karakter** (≈25rb token) — muat aman di `_MAX_BODY_BYTES=1_000_000` (`health.py:63,306-308`), tanpa bengkakkan snapshot `session-{sid}.json` secara liar. Unlimit mutlak ditolak (risiko `413`, konteks LLM jebol diam-diam).

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Batas klien | `maxlength=100000`; placeholder beri tahu Shift+Enter = baris baru |
| Auto-grow | Cap `160→320px` + tombol expand (toggle tinggi penuh); paste besar via `requestAnimationFrame` agar tak freeze |
| Counter 3-tier | `>2000` netral (existing) → `>50000` kuning (pesan panjang, jawaban bisa terpotong konteks) → `>90000` merah + saran (pecah pesan / lampirkan sebagai .txt/.md) |
| Warning konteks | Reuse `ctx-bar` PRD-26 bila sesi hampir penuh; tanpa counter token presisi (non-goal) |
| Server | `_MAX_BODY_BYTES` tetap 1MB; `413` diterjemahkan klien ke bahasa manusia |
| Keyboard | Enter=kirim / Shift+Enter=baris baru tetap (`app_js.py:2027`); palette `/` `@` tak terganggu |
| Tanpa knob baru | Tanpa field `gateway.json` baru (ikuti PRD-36) |

## 3. Call-site

| # | Lokasi | Guna | Fallback |
|---|---|---|---|
| 1 | `markup.py:103` | `maxlength` 100000 + tombol expand | Batas lama bila JS mati (server tetap 1MB) |
| 2 | `app_js.py:2006-2019` | grow 320px + counter 3-tier + expand | Counter lama bila selector gagal |
| 3 | `app_js.py:1891` `streamPost` | Map `413` → pesan Indonesia | Teks error mentah |
| 4 | `i18n_js.py` | String baru 3 bahasa (id/en/ja) | Bahasa Indonesia bila kunci hilang (pola existing) |

## 4. Goals

- **G1** — Pesan sampai 100rb karakter lolos klien dan terkirim tanpa freeze.
- **G2** — User tahu posisi panjang pesan + risiko konteks sebelum kirim (counter/warning).
- **G3** — Tanpa dep/build baru; `escHtml` semua dinamis; endpoint tak berubah.

## 5. Non-goals

- Hapus batas total; ubah `_MAX_BODY_BYTES`; field config baru.
- Counter token presisi per model; auto-split pesan otomatis (cukup saran).
- Ubah alur SSE/streaming.

## 6. Acceptance criteria

- **AC-37.1** Suite + lint hijau; tanpa dep/build baru; string baru ada di 3 bahasa.
- **AC-37.2** Ketik/paste 100rb karakter: terkirim, textarea tak freeze, expand tampil penuh.
- **AC-37.3** Tier `50rb` tampil kuning, tier `90rb` tampil merah + saran pecah/lampir.
- **AC-37.4** Enter kirim, Shift+Enter baris baru, palette `/` `@` tetap bekerja.
- **AC-37.5** Body >1MB dari klien nakal tetap `413`, klien tampilkan saran pecah pesan (bukan "request body too large").

## 7. Testing

1. `pytest tests/test_pha/test_webui_api.py -q` (413 + pesan ramah).
2. `ruff check src/pha` bersih + `node --check` JS terekstrak (`APP_JS`, `I18N_JS`).
3. Manual: paste 100rb char → kirim → counter tier → expand; kirim >1MB via curl → pesan Indonesia.
