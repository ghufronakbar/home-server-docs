# Runbook Pribadi — AI-Workspace (Home Server)

Kumpulan runbook untuk layanan AI yang berjalan di **home server** (`ssh home`), dikelola **Dokploy** (project **AI-Workspace**) dan diekspos lewat **Cloudflare Tunnel**. Tujuannya: bisa dipasang ulang / di device lain tinggal baca berurutan.

> Semua secret/token/API key/password di seluruh dokumen ini adalah **placeholder** (`<...>`). Nilai asli tidak ditulis. Aman dibaca/di-share.

## Layanan

| Folder | Layanan | Ringkas | URL |
|--------|---------|---------|-----|
| [9router/](9router/README.md) | **9Router** — AI coding gateway | Endpoint OpenAI-compatible ke banyak akun/model (Codex `cx/*`, dll). Dipakai Codex CLI & Hermes. | `https://9router.lans.my.id` |
| [hermes/](hermes/README.md) | **Hermes Agent** (Nous Research) | Agen AI multi-platform (Discord, dll) + dashboard + TTS. Modelnya lewat 9Router. | `https://hermes.lans.my.id` |

## Bagaimana semuanya terhubung
```
Codex CLI (MacBook) ─┐
                     ├─►  9Router (:20128)  ─►  akun GPT Codex (cx/gpt-5.6-*)
Hermes Agent ────────┘        │
   ├─ gateway (Discord, cron) │  provider: custom → https://9router.lans.my.id/v1
   └─ dashboard (:9119)       │
                              ▼
             (semua di home server, Dokploy AI-Workspace,
              publik via Cloudflare Tunnel + cloudflared container)
```
- **Hermes memakai 9Router** sebagai provider model (`provider: custom`) → satu sumber akun/model untuk CLI maupun agen.
- Codex CLI di MacBook juga menunjuk ke 9Router.

## Referensi infrastruktur bersama
- **Server**: `ssh home` (host `lans-hp`), user uid/gid `1000`. Docker + Dokploy (`:3000`).
- **Dokploy**: project **AI-Workspace**. Kelola via API: `curl -H "x-api-key: <DOKPLOY_API_TOKEN>" http://localhost:3000/api/...` (`compose.create` / `compose.update` / `compose.deploy`).
- **Cloudflare Tunnel**: container `cloudflare-tunnel` (network `bridge`). ⚠️ Origin harus IP yang **dijangkau dari bridge** — pakai **published port** Docker (mis. `172.17.0.1:<port>` atau IP LAN `192.168.100.31:<port>`), BUKAN bind `network_mode: host` tanpa publish (diblokir firewall). Detail: `hermes/02-troubleshooting.md`.
- **Secret**: disimpan di env compose Dokploy / config service — tidak di repo ini.

## Struktur
```
.
├── README.md              ← ini (index general)
├── 9router/
│   ├── README.md          (index 9Router)
│   ├── 01-install-9router-dokploy.md
│   ├── 02-setup-codex-cli.md
│   ├── 03-cxmodel-helper.md
│   ├── 04-backup-restore.md
│   └── 05-troubleshooting.md
└── hermes/
    ├── README.md          (index Hermes)
    ├── 01-install-hermes-dokploy.md
    ├── 02-troubleshooting.md
    ├── 03-connect-9router.md
    ├── 04-setup-discord.md
    └── 05-tts.md
```
