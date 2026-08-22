# 9Router — Runbook Pribadi

Dokumentasi langkah-langkah yang sudah dikerjakan untuk men-deploy **9Router** (AI coding gateway) di home server dan menyambungkannya ke **Codex CLI**. Tujuannya: kalau mau pasang di device lain, tinggal baca file-file ini berurutan.

> Semua secret/token/API key di dokumen ini adalah **placeholder** (`<...>`). Nilai asli tidak ditulis di sini. Perintah untuk men-generate secret baru sudah disertakan.

## Apa itu 9Router
Gateway open-source (image `decolua/9router`) yang mengekspos **API OpenAI-compatible** di port `20128`. Menerima banyak provider/akun (mis. akun GPT Codex via OAuth) dan menyediakan satu endpoint `/v1` untuk dipakai CLI seperti Codex, Cursor, Cline, dll.

## Arsitektur setup ini
```
MacBook (Codex CLI)
   │  https://9router.lans.my.id/v1   (Authorization: Bearer <9ROUTER_API_KEY>)
   ▼
Cloudflare Edge  ──►  cloudflared (container, network: bridge)
                          │  origin: http://172.17.0.1:20128   (bridge gateway, BUKAN localhost)
                          ▼
                     Home server  ──►  Dokploy → compose service "9router"
                                          container 9router  :20128
                                          volume 9router-data → /app/data (SQLite + OAuth tokens)
```

Poin penting yang gampang salah:
- **cloudflared ada di network `bridge`**, jadi origin tunnel harus `http://172.17.0.1:20128` (gateway docker), bukan `localhost`.
- **Data persisten** ada di named volume `9router-data`. Redeploy tidak menghapusnya; hanya `docker compose down -v` / hapus volume yang menghapus.
- **Codex CLI** butuh model dengan **prefix `cx/`** dan API key dari **env var** (bukan `api_key` inline). Detail di `02-setup-codex-cli.md`.

## Daftar isi
| File | Isi |
|------|-----|
| [01-install-9router-dokploy.md](01-install-9router-dokploy.md) | Deploy 9Router di server (Dokploy API + Cloudflare tunnel + volume + secret) |
| [02-setup-codex-cli.md](02-setup-codex-cli.md) | Konfigurasi Codex CLI di client (Mac) agar lewat 9Router |
| [03-cxmodel-helper.md](03-cxmodel-helper.md) | Command `cxmodel` untuk ganti model & reasoning cepat |
| [04-backup-restore.md](04-backup-restore.md) | Backup & restore data (akun + API key) |
| [05-troubleshooting.md](05-troubleshooting.md) | Error umum + solusinya |
| [06-cxgateway.md](06-cxgateway.md) | Command `cxgateway` untuk pindah gateway (home server ⇄ lokal) |
| [07-install-9router-lokal.md](07-install-9router-lokal.md) | Jalankan 9Router lokal di MacBook via docker compose (repo `dev-tools`) |

## Instance yang sudah jalan (referensi)
- URL publik: `https://9router.lans.my.id`
- Server: `ssh home` (host `lans-hp`)
- Dokploy project: **AI-Workspace** → compose service **9router**

## Layanan terkait
- **Hermes Agent** (Nous Research) — di-deploy di server yang sama (Dokploy AI-Workspace), publik di `https://hermes.lans.my.id`. Runbook: [../hermes/README.md](../hermes/README.md). Rencananya modelnya disambungkan ke 9router ini (`provider: custom`).
