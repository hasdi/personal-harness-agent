# PRD-14 — Cron → WebUI Session Delivery

| | |
|---|---|
| Feature | Perbaikan: hasil cron job sampai ke session WebUI (command output & balasan agent) |
| Depends on | PRD-3, PRD-5, PRD-13 |
| Files | `pha/src/pha/gateway/{cron.py, service.py, health.py, webui_channel.py}`, `webui/app_js.py` |

## 1. Masalah (bukti dari code & log)

Cron job berjalan (`pha cron ran command job=... rc=0`) tetapi hasilnya tidak pernah
muncul di session WebUI. Tiga sebab berlapis:

1. **Channel `api` tidak punya handler outbound.** `ChannelManager._dispatch_outbound`
   hanya mengirim ke `self.channels.get(msg.channel)`; tanpa entri `"api"`, semua
   outbound bertarget WebUI dicatat `Unknown channel: api` lalu **dibuang**
   (balasan agent untuk cron `message`, output cron `command`, restart notice).
2. **Dispatcher tidak pernah start saat tidak ada channel chat.** Dengan
   `enabled_channels: []`, `ChannelManager.start_all()` early-return
   (`"No channels enabled"`) sehingga `_dispatch_outbound` tidak dibuat sama sekali.
3. **Gate target API terlalu sempit.** `PhaCronLoop._notify_api_session` hanya cocok
   bila `chat_id.startswith("api:")`, padahal tool cron PHA menulis
   `channel="api"` + `chat_id="<uuid>"` (lihat `~/.pha/cron.json`). Selain itu
   `_deliver_message_job` sempat memakai jalur notifikasi mentah sehingga instruksi
   message job tidak dijalankan agent.

## 2. Perbaikan

- **`webui_channel.py` (baru)** — `WebUISessionChannel(BaseChannel)`, `name="api"`,
  `send()` → `notify_api_session(chat_id, source="", content)`. Frame `_progress`
  (progress/tool hint) dilewati karena WebUI sudah menampilkannya via SSE live.
- **`service.py`** — daftarkan channel ke manager:
  `self._manager.channels["api"] = WebUISessionChannel(self.notify_api_session)`.
  Ini sekaligus menjamin dispatcher outbound start walau tanpa channel chat.
- **`cron.py`**
  - `_notify_api_session` menerima dua bentuk target: `chat_id="api:<key>"` **dan**
    `channel="api"` + key telanjang.
  - `_deliver_message_job` **selalu** lewat `publish_inbound` (agent harus
    menjalankan instruksi); balasan kembali lewat channel `api`.
  - Header `⏰ <job>` tidak lagi dobel: notifier menambahkannya, chat channel dapat
    header lengkap.
- **`service.py` — notifikasi tertunda**: `_pending_notifications` menjadi daftar
  (maks 20/session); `attach_run` mem-flush seluruh daftar. Method baru
  `pop_session_notifications(session_key)`.
- **`health.py`** — `GET /v1/notifications?session_key=` (auth seperti `/v1` lain).
- **`webui/app_js.py`** — `fetchNotifications()` dipanggil bersama poll
  `/v1/runs/active` (20 s) dan saat membuka session; tiap item dirender lewat
  `addCronBubble()` (ekstraksi dari handler event `cron`).

### 2b. Persistensi (revisi v1.1)

Bug lanjutan: bubble cron **hilang saat refresh** dan hanya yang terakhir tampil —
output cron cuma event transient (SSE + antrian in-memory), tidak pernah masuk
snapshot session.

- **`session_storage.append_snapshot_message()`** (baru) — menambahkan satu pesan ke
  snapshot terbaru milik `session_key` (`latest-<token>.json` + `session-<id>.json`),
  mempertahankan pesan lama & `ts`, dan meng-invalidate cache listing sidebar.
- **`PhaSessionRuntimePool.append_assistant_message()`** — menjadikan output cron
  bagian dari **history**: bila bundle hidup (tidak ada run aktif) pesan di-append ke
  engine lalu snapshot disimpan; bila belum ada bundle, tulis langsung ke snapshot.
  `record_message_to_snapshot()` dipakai saat ada run aktif (engine milik run).
- **`notify_api_session`** memanggil keduanya sebelum mengantri event live, sehingga
  setiap respons cron tersimpan (bukan hanya yang terakhir).
- **`load_webui_session`** menandai pesan berawalan `⏰` dengan `cron: true`
  (WebUI mempertahankan gaya bubble setelah reload) dan **mengosongkan** antrian
  pending session itu — history menang, jadi tidak ada duplikasi setelah refresh.

## 3. Alur hasil

| Jenis job | Jalur |
|---|---|
| `message` (target WebUI) | `publish_inbound` → bridge menjalankan agent → outbound `channel="api"` → `WebUISessionChannel` → session |
| `command` (target WebUI) | dijalankan langsung (CLI-only) → `notify_api_session` → SSE/poll |
| channel chat (telegram/…) | seperti sebelumnya (`publish_inbound`/`publish_outbound`) |

### 3b. Job = agent turn (revisi v1.2)

Bug: job `kind=command` yang kompleks (mis. one-liner PowerShell bersarang) gagal
`rc=1` **berulang tiap tick tanpa agent** yang bisa memperbaikinya.

- `cron_create` (tool) kini **selalu** membuat job `kind="message"` (agent turn,
  seperti `CronPayload.kind="agent_turn"` di nanobot). `command` diubah menjadi
  prompt: *"Run the command below and report the result… If it fails, explain why
  and run the corrected version"* + konteks `Host: <OS>; commands run through <shell>`.
- Karena agent menjalankan command lewat tool shell-nya sendiri, tidak ada lagi
  pembungkusan shell ganda (satu-satunya lapisan yang menulis quoting adalah model,
  untuk shell yang memang dilaporkannya).
- `pha cron add --command` (CLI) tetap ada untuk otomasi shell mentah; ditambah guard
  `_command_names_a_shell()` supaya command yang sudah menyebut interpreter-nya
  (`powershell …`) **tidak** dibungkus shell lagi.
- Tool description + `src/pha/skills/cron/SKILL.md` diperbarui: agent turn adalah
  bentuk utama, `command` hanya untuk hal sepele.

### 3c. Delivery tidak boleh me-reset session (revisi v1.3)

Bug: `Network error: Connection error.` tiap tick setelah session di-recreate.

- Inbound cron dulu tanpa `cwd` → agent berjalan di cwd gateway. Bila session
  dibuat dari cwd lain (WebUI memilih folder), setiap delivery dianggap "cwd
  change" → session di-reset (`fresh session for cwd change`, history hilang).
- Fix: job menyimpan `cwd` session saat dibuat (`add_job(cwd=…)` dari
  `ToolExecutionContext.cwd`), delivery menaruh `metadata["cwd"]`, dan bridge
  meneruskannya ke `stream_message(cwd=…)`. Chat biasa tetap tanpa cwd.
- Bug kedua (yang memunculkan pesan network): `close_runtime()` menutup client
  API **bersama** milik gateway saat bundle di-recreate → semua request
  berikutnya gagal sampai gateway restart. Fix: bundle dengan
  `external_api_client=True` tidak menutup client yang bukan miliknya.
  Detail + bukti log: `CHANGELOG-PHA.md` Patch P10.

## 4. Acceptance criteria

- **AC-14.1** Job `command` dengan `channel="api"` + `chat_id="<uuid>"` menghasilkan
  notifikasi di session tersebut (bukan `Unknown channel`).
- **AC-14.2** Job `message` menjalankan agent (bukan mem-posting instruksi mentah)
  dan balasannya sampai ke session.
- **AC-14.3** `GET /v1/notifications` mengembalikan lalu mengosongkan antrian; 400 tanpa `session_key`.
- **AC-14.4** Notifikasi yang datang saat tidak ada run tetap tampil (poll UI).
- **AC-14.5** Frame progress/tool-hint tidak ikut masuk antrian notifikasi.
- **AC-14.6** Dispatcher outbound tetap hidup walau `enabled_channels: []`.
- **AC-14.8** Setiap respons cron tersimpan sebagai pesan assistant di snapshot session:
  tetap tampil setelah refresh dan **semua** respons hadir (bukan hanya terakhir).
- **AC-14.9** Membuka session mengosongkan antrian pending-nya → tidak ada bubble ganda.
- **AC-14.7** `ruff` bersih, pytest hijau (termasuk `tests/test_pha/test_cron_delivery.py`, `test_cron_history.py`), `node --check` OK.

## 5. Testing

`tests/test_pha/test_cron_delivery.py`: channel forward + skip progress; message job
lewat bus; command job sampai ke target api telanjang; kedua bentuk target API; antrian
`notify_api_session`/`pop_session_notifications`; route `/v1/notifications` (200 → kosong → 400).
