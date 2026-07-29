# 04 — Setup Bot Discord untuk Hermes

Menghubungkan Hermes ke Discord: dari membuat server & bot, sampai bot online dan membalas lewat model (9router). Bot connect **keluar** (bot token) — tidak butuh port masuk.

## Prasyarat
- Hermes sudah jalan (`01-install-hermes-dokploy.md`) dan model tersambung (`03-connect-9router.md`).
- Akses Dokploy (composeId service `hermes`) untuk inject env.

---

## Bagian A — Sisi Discord (di browser/app)

### 1. Buat server
Discord → ikon `+` → *Create My Own* → beri nama.

### 2. Buat aplikasi + bot
https://discord.com/developers/applications → *New Application* → beri nama.
- Menu **Bot** → *Reset Token* → **Copy** → ini `<DISCORD_BOT_TOKEN>` (RAHASIA, seperti password).

### 3. Aktifkan Privileged Intents (halaman **Bot**)
- ✅ **Message Content Intent** — WAJIB (tanpa ini bot tidak bisa membaca isi pesan).
- ✅ **Server Members Intent** — disarankan (auth berbasis role/DM).
- *Save Changes*.

### 4. Undang bot ke server
**OAuth2 → URL Generator**: scopes `bot` + `applications.commands`; Bot Permissions — untuk server pribadi centang **Administrator** (paling gampang) atau minimal: View Channels, Send Messages, Read Message History, **Create Public Threads**, Send Messages in Threads, Add Reactions, Embed Links, Attach Files. Copy URL → buka → pilih server → *Authorize*.

Atau langsung (ganti `<CLIENT_ID>` = Application ID di *General Information*):
```
https://discord.com/oauth2/authorize?client_id=<CLIENT_ID>&scope=bot+applications.commands&permissions=8
```
> Permission **Create Public Threads** penting — perilaku default Hermes membuat thread per percakapan.

### 5. Ambil ID (aktifkan Developer Mode dulu: Settings → Advanced → Developer Mode)
- Klik kanan nama Anda → **Copy User ID** → `<YOUR_USER_ID>` (untuk allowlist).
- (Opsional) klik kanan channel notifikasi → **Copy Channel ID** → `<HOME_CHANNEL_ID>`.

---

## Bagian B — Sisi Hermes (inject env + deploy)

Hermes membaca konfigurasi Discord dari **environment variable** service `gateway` (bukan `.env` di data dir). Tambahkan ke blok `environment:` service `gateway` di compose:
```yaml
    environment:
      - HERMES_UID=1000
      - HERMES_GID=1000
      - DISCORD_BOT_TOKEN=<DISCORD_BOT_TOKEN>
      - DISCORD_ALLOWED_USERS=<YOUR_USER_ID>          # hanya ID ini yg bisa perintah bot
      - DISCORD_HOME_CHANNEL=<HOME_CHANNEL_ID>        # target notifikasi/cron
      - DISCORD_HOME_CHANNEL_NAME=asa-agent
```
Lalu update + redeploy compose (lihat `01` Langkah 4 untuk pola `compose.update`/`compose.deploy`).

Platform Discord **aktif otomatis** begitu `DISCORD_BOT_TOKEN` ada — tidak perlu enable plugin manual.

### Verifikasi (log ada di FILE, bukan stdout)
Stdout container hanya menampilkan banner. Log asli:
```bash
ssh home 'grep -iE "discord|connected|platform" /home/lanstheprodigy/.hermes/logs/gateway.log | tail'
```
Sukses jika ada:
```
Connecting to discord...
[Discord] Connected as <botname>#0000
✓ discord connected
Gateway running with 1 platform(s)
```

### Tes notifikasi ke home channel
```bash
ssh home 'docker exec hermes hermes send --to discord --subject "[Hermes]" "notifikasi tes"'
```
Pesan muncul di channel `<HOME_CHANNEL_ID>`.

---

## Cara pakai (model default: auto-thread)
- **@mention bot** di channel → Hermes membuat **thread** untuk topik itu.
- **Di dalam thread, ketik biasa TANPA mention** — bot terus membalas (satu thread = satu percakapan/topik).
- **Topik baru → mention baru** di channel utama → thread baru.
- Balasan diproses agent + tool-calling lewat model (9router `cx/gpt-5.6-sol`).
- Slash command `/skill` tersedia.

## Opsi konfigurasi (env pada service gateway)
| Env | Efek |
|-----|------|
| `DISCORD_ALLOWED_USERS` | Comma-separated user ID yang boleh perintah bot (allowlist). |
| `DISCORD_ALLOW_ALL_USERS=true` | Semua orang boleh (dev only — hindari di server publik). |
| `DISCORD_HOME_CHANNEL` | Channel default untuk cron/notifikasi (`hermes send --to discord`). |
| `DISCORD_AUTO_THREAD=false` | Matikan auto-thread → bot balas **inline** di channel yang sama. |
| `DISCORD_FREE_RESPONSE_CHANNELS=<id>[,<id>]` | Channel di mana bot balas **tanpa** perlu @mention (`*` = semua). |

## Troubleshooting
| Gejala | Sebab / Fix |
|--------|-------------|
| `Auto-thread creation failed ... Too many requests. Retry in N seconds` | **Rate-limit Discord** untuk pembuatan thread (bukan bug). Terjadi kalau mention beruntun cepat di channel utama. Tunggu ~beberapa menit / lanjut di thread yang sudah ada / set `DISCORD_AUTO_THREAD=false`. |
| Bot online tapi tak membalas | **Message Content Intent** belum ON (Bagian A §3), atau pengirim tidak ada di `DISCORD_ALLOWED_USERS`. |
| Bot tidak online | Token salah/di-reset; cek `gateway.log`. Token baru → update env + redeploy. |
| Tak bisa bikin thread sama sekali | Bot kurang permission **Create Public Threads** → perbaiki di role bot / re-invite. |
| Balasan error 403/404 dari model | Lihat `03-connect-9router.md` (User-Agent 403 / prefix `cx/`). |

## Keamanan
- Token tersimpan di env compose (terlihat di Dokploy) — instance pribadi, OK. Kalau bocor, *Reset Token* di Developer Portal lalu update env + redeploy.
- Selalu pakai `DISCORD_ALLOWED_USERS` (bukan allow-all) supaya hanya Anda yang bisa memicu agent (yang bisa menjalankan kode/tool).
