# Hermes Agent — Runbook Pribadi

Dokumentasi deploy **Hermes Agent** (Nous Research) di home server, dikelola Dokploy dan diekspos lewat Cloudflare Tunnel. Baca berurutan untuk memasang di device/server lain.

> Semua secret/token/API key/password di sini adalah **placeholder** (`<...>`). Nilai asli tidak ditulis.

## Apa itu Hermes
Agen AI pribadi open-source (image `nousresearch/hermes-agent`). Terdiri dari **2 proses**:
- **gateway** (`gateway run`) — inti agen + integrasi platform (**Discord**, Telegram, Slack, WhatsApp, Signal, Email) + cron scheduler. Connect **keluar** (mis. Discord via bot token) — tidak butuh port masuk.
- **dashboard** — web UI (port **9119**) untuk kelola config, API key, sesi.

## Arsitektur setup ini
```
Browser  ──►  Cloudflare Access (opsional, disarankan)  ──►  Cloudflare Tunnel
                                                                   │  origin: http://172.17.0.1:9119
                                                                   ▼
                                                     cloudflared (container bridge)
                                                                   │
                                                                   ▼
                                    home server ──► Dokploy compose "hermes"
                                       ├─ hermes            (gateway, network_mode: host)
                                       └─ hermes-dashboard  (bridge, publish 9119:9119, --host 0.0.0.0)
                                             volume: /home/<user>/.hermes → /opt/data  (state.db, config.yaml, dst)
```

## 3 hal yang WAJIB dipahami (kalau salah, gagal)
1. **Dashboard menolak bind non-loopback tanpa auth.** Sejak hardening Juni 2026, `--insecure` = no-op. Untuk expose (bind ke `0.0.0.0`/non-127.0.0.1), WAJIB set auth: `dashboard.basic_auth` (password) atau OAuth Nous Portal. Lihat `01`.
2. **`network_mode: host` TIDAK bisa dijangkau cloudflared (container bridge)** karena firewall host memblokir trafik container→host di port yang tidak di-*publish*. Solusi: jalankan dashboard sebagai **published port** (`ports: 9119:9119`) supaya Docker menambah aturan iptables sendiri (persis pola 9router). Lihat `02`.
3. **gateway ↔ dashboard koordinasi via file/SQLite di `/opt/data`** (`state.db`, `gateway_state.json`, `gateway.lock`), bukan TCP — jadi dashboard aman dijalankan di network berbeda dari gateway.

## Daftar isi
| File | Isi |
|------|-----|
| [01-install-hermes-dokploy.md](01-install-hermes-dokploy.md) | Deploy Hermes (Dokploy API + auth dashboard + Cloudflare tunnel/Access) |
| [02-troubleshooting.md](02-troubleshooting.md) | 502/networking, crash-loop auth-bind, cek cepat |
| [03-connect-9router.md](03-connect-9router.md) | Sambungkan model Hermes ke 9router (provider custom + gotcha User-Agent 403) |
| [04-setup-discord.md](04-setup-discord.md) | Setup bot Discord: buat server/bot → inject env → cara pakai (thread) + gotcha rate-limit |
| [05-tts.md](05-tts.md) | Aktifkan Text-to-Speech (Edge TTS) — install ke durable lazy target (persisten), voice Jepang |
| [06-9router-monitoring.md](06-9router-monitoring.md) | Agent baca usage & limit 9Router (read-only): exporter+cron, skill, quota per akun, kredensial host-only |

## Instance yang sudah jalan (referensi)
- URL: `https://hermes.lans.my.id` (login Hermes; Cloudflare Access opsional)
- Server: `ssh home` · Dokploy project **AI-Workspace** → compose **hermes**
- Data: `/home/lanstheprodigy/.hermes`

## Langkah berikutnya
- Sambungkan model ke **9router**: [03-connect-9router.md](03-connect-9router.md).
- Bot **Discord**: [04-setup-discord.md](04-setup-discord.md).
- Pemantauan 9Router oleh agent (read-only): [06-9router-monitoring.md](06-9router-monitoring.md).
- Automation/cron (`hermes cron`) — belum didokumentasikan.
