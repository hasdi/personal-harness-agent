# PRD-22 — RCA Mobile: Teks Wrap + Tabel Remuk + Drawer Terkunci

| | |
|---|---|
| Feature | Root-cause & fix mobile: wrap normal, tabel scroll-x, resize guard |
| Depends on | PRD-21 |
| Files | `webui/{style,spa_js}.py`, `tests/test_pha/test_webui_api.py` |
| Status | Draft v1.0 (2026-09-13) |

## 1. Root-cause analysis (bukti CDP, viewport 390)

Ukur via Chrome CDP `setDeviceMetricsOverride(390x844, mobile)` + screenshot:

| Gejala di screenshot | Root cause | Bukti |
|---|---|---|
| Teks chat pecah per kata ("mo-dul", "au-tenti-kasi") | `word-break: break-all` + `overflow-wrap: anywhere` di `.bubble p/a/code/td` memecah kata di spasi normal | `style.py` bubble markdown; screenshot `rca2-shrunk.png` baris 1: `bubW344 bubSW342` |
| Tabel remuk (kolom path pecah per segmen) | Sel `white-space: normal` + wrap agresif | Eksperimen A_wrap: `tableSW311`, sel 191px remuk |
| Drawer skill "terkunci" 340px saat shrink browser | `--skill-w` dari `localStorage` + `applySkillW()` tanpa cek viewport; `resize` tidak reset | `spa_js.py:applySkillW` versi lama |
| Toolbar terlihat padat (bukan overflow) | `docSW == viewport == 390`, `offenders == []` — tidak ada overflow page-level | CDP `SHRUNK` measurement |

Kesimpulan: **bukan overflow struktural** (rantai `min-width:0` + `overflow-x:hidden`
sudah benar) — melainkan **fragmentasi kata** + **tabel wrap** + **resize listener**.

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Teks normal | `word-break: normal; overflow-wrap: break-word` untuk p/li/h/code/a/td — pecah hanya saat darurat |
| Exception tanpa-spasi | `break-all` HANYA untuk URL, inline code, codeblock `pre`, path/mono (trace, skill, cron, audit, dir, config) |
| Tabel mobile | `white-space: nowrap` semua sel + scroll-x di dalam bubble; hasil eksperimen B: `tableSW390`, sel utuh 242px |
| Resize guard | `skillResizeActive()` (≥1100px): `applySkillW` + pointerdown + keydown nonaktif di mobile; `resize` listener reset `--skill-w`; CSS `#skills-resize{display:none!important}` |

## 3. Goals

- **G1** — Teks chat wrap di batas kata normal; URL/path panjang tetap tidak overflow.
- **G2** — Tabel mobile utuh + scroll-x di dalam bubble.
- **G3** — Shrink desktop→mobile tanpa drawer terkunci.

## 4. Non-goals

- Mengubah struktur grid/flex desktop; endpoint API; engine.

## 5. Acceptance criteria

- **AC-22.1** Screenshot 390px: tidak ada kata terpotong ("mendukung", "autentikasi" utuh).
- **AC-22.2** Tabel: `scrollWidth > clientWidth` tapi `docSW == viewport`.
- **AC-22.3** Smoke test assert rule tabel + `skillResizeActive`; `ruff` + `node --check` hijau.

## 6. Testing

1. `pytest tests/test_pha/test_webui_api.py -q -k webui_html`.
2. `ruff check src/pha/gateway/webui/`.
3. CDP: desktop 1280 → shrink 390 tanpa reload + screenshot; cek `docSW == viewport`.
