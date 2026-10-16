# PRD-25 — Image Native-First

| | |
|---|---|
| Feature | Gambar selalu dikirim native (`ImageBlock`) + prompt user apa adanya; fallback `image_to_text` hanya saat provider menolak |
| Depends on | — |
| Files | `openharness/engine/query.py`, `openharness/api/provider.py`, `pha/gateway/{runtime,models,service}.py`, `openharness/prompts/system_prompt.py`, `docs/gateway-config.md`, `tests/test_{tools/test_image_to_text_tool,pha/test_vision_config,pha/test_webui_api}.py` |
| Status | Implemented (2026-09-15) |

## 1. Ringkasan

Prompt+gambar sederhana ("sajikan ke tabel monospace") jadi overengineer: model membaca via path/tool padahal `ImageBlock` native sudah ada. Penyebab: gate heuristik nama model + `multimodal=false`, vision prompt generik hardcoded (`query.py:620`), instruksi "Inspect them by path".

## 2. Keputusan

| Aspek | Keputusan |
|---|---|
| Pengiriman | Selalu kirim `ImageBlock` native + prompt user; tanpa gate nama model |
| Fallback | Hanya saat provider error menyebut image/unsupported → konversi `image_to_text` sekali → retry turn sekali |
| Vision prompt | = teks user (default detail bila kosong), bukan hardcoded generik |
| Attachment note | Baris note dipertahankan (diparse WebUI via regex `_ATTACH_LINE`); instruksi diubah: prefer native, bukan path |
| Hapus | Regex `_MULTIMODAL_MODEL_PATTERNS` + `is_model_multimodal`, field `multimodal`, env `PHA_MULTIMODAL`, meta SETUP + restart field (`service.py:137,183,271-274`); `vision_model` tetap sebagai fallback text-only |
| Base prompt | 1 baris: `image_to_text` hanya bila gambar tak tersedia native |
| Kompat | Key `multimodal` lama di `gateway.json` diabaikan pydantic; aman tanpa migrasi |

## 3. Goals

- **G1** — Tugas gambar sederhana dijawab langsung tanpa tool call.
- **G2** — Model text-only tetap bisa via 1x fallback.
- **G3** — Tanpa dependensi/crop/OCR baru.

## 4. Non-goals

- Crop/OCR, ubah soul.md/base prompt besar; tambah bila fallback terbukti kurang.

## 5. Acceptance criteria

- **AC-25.1** Multimodal + "sajikan ke tabel monospace" → jawab langsung, tanpa tool call.
- **AC-25.2** Text-only → 1x fallback `image_to_text` lalu jawab.
- **AC-25.3** Test heuristik/multimodal lama diperbarui; suite terkait hijau.

## 6. Testing

1. `pytest tests/test_tools/test_image_to_text_tool.py tests/test_pha/test_vision_config.py tests/test_pha/test_webui_api.py -q`.
2. `ruff check src/openharness/engine/query.py src/openharness/api/provider.py src/pha/gateway/`.
3. Manual: AC-25.1 dan AC-25.2.
