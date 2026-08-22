# 07 — Install 9Router Lokal di MacBook (docker compose)

Instance 9Router kedua yang jalan **lokal di MacBook**, terpisah dari home server. Berguna saat home server mati / tunnel putus, atau saat mau kerja offline-ish tanpa lewat Cloudflare.

Bedanya dengan `01-install-9router-dokploy.md`: tanpa Dokploy, tanpa Cloudflare Tunnel, tanpa domain — cukup `docker compose` dan diakses lewat `http://localhost:20128`.

## Prasyarat
- Docker Desktop jalan (image `decolua/9router:latest` punya varian **arm64**, native di Apple Silicon).
- Repo `dev-tools` (berisi `docker-compose.yml` untuk service dev lokal).

## Langkah 1 — Secret di `.env`
Di root repo `dev-tools`:
```bash
cat > .env <<EOF
NINEROUTER_JWT_SECRET=$(openssl rand -hex 32)
NINEROUTER_API_KEY_SECRET=$(openssl rand -hex 32)
NINEROUTER_MACHINE_ID_SALT=$(openssl rand -hex 16)
NINEROUTER_INITIAL_PASSWORD=$(openssl rand -base64 12 | tr -d '/+=' | cut -c1-16)
NINEROUTER_PORT=20128
NINEROUTER_REQUIRE_API_KEY=true
EOF
chmod 600 .env
printf '.env\n.DS_Store\n' > .gitignore     # jangan sampai ke-commit
```
`NINEROUTER_INITIAL_PASSWORD` = password login dashboard (login 9Router **hanya password**, tanpa username).

> Catatan: `MACHINE_ID_SALT` ternyata **tidak dibaca** oleh build image saat ini (tidak ada referensinya di `/app`). Dibiarkan ada supaya seragam dengan compose home server; tidak berefek apa-apa.

## Langkah 2 — Service di `docker-compose.yml`
Pakai `profiles` supaya 9Router **tidak ikut nyala** waktu `docker compose up -d` untuk mysql/redis/minio dkk:
```yaml
  9router:
    image: decolua/9router:latest
    container_name: 9router
    profiles: [ "9router" ]
    restart: unless-stopped
    ports:
      - "127.0.0.1:${NINEROUTER_PORT:-20128}:20128"   # 127.0.0.1 = tidak terekspos ke LAN
    volumes:
      - 9router_local_data:/app/data
    environment:
      DATA_DIR: /app/data
      PORT: "20128"
      HOSTNAME: "0.0.0.0"
      NODE_ENV: production
      JWT_SECRET: "${NINEROUTER_JWT_SECRET}"
      INITIAL_PASSWORD: "${NINEROUTER_INITIAL_PASSWORD}"
      API_KEY_SECRET: "${NINEROUTER_API_KEY_SECRET}"
      MACHINE_ID_SALT: "${NINEROUTER_MACHINE_ID_SALT}"
      AUTH_COOKIE_SECURE: "false"
      REQUIRE_API_KEY: "${NINEROUTER_REQUIRE_API_KEY:-true}"
      BASE_URL: "http://localhost:${NINEROUTER_PORT:-20128}"
      NEXT_PUBLIC_BASE_URL: "http://localhost:${NINEROUTER_PORT:-20128}"
      CLOUD_URL: "https://9router.com"

volumes:
  9router_local_data:
    name: 9router_local_data
```

## Langkah 3 — Switch on/off
Karena service-nya punya `profiles`, sebut namanya eksplisit agar **hanya** dia yang jalan:
```bash
docker compose up -d 9router     # ON  (service lain tidak ikut)
docker compose stop 9router      # OFF
docker compose start 9router     # ON lagi
docker compose rm -sf 9router    # hapus container; data tetap di volume
```
Bisa juga lewat toggle di Docker Desktop setelah containernya pernah dibuat.

## Langkah 4 — Verifikasi
```bash
docker ps --filter name=9router --format "{{.Names}} {{.Status}} {{.Ports}}"
curl -s -o /dev/null -w "/           -> %{http_code}\n" http://localhost:20128/
curl -s -o /dev/null -w "/v1/models  -> %{http_code}\n" http://localhost:20128/v1/models
```
Sukses jika container `Up`, `/` → **307** (redirect ke `/login`), `/v1/models` tanpa key → **401**.

## Langkah 5 — Dashboard & API key
1. Buka `http://localhost:20128` → login pakai `NINEROUTER_INITIAL_PASSWORD`.
2. **API key sudah ada otomatis**: 9Router menyemai satu **"Default Key"** saat boot pertama (dibuat barengan migrasi DB `applied #1 initial`).
   - Key-nya **unik per install**: image tidak punya `/etc/machine-id`, jadi `machineIdSync()` jatuh ke `crypto.randomUUID()` yang lalu dipersist ke `/app/data/machine-id`. Bukan konstanta yang sama untuk semua orang.
   - Boleh dipakai langsung, atau hapus & bikin baru kalau mau.
3. **Providers** → connect akun Codex (OAuth). Tanpa ini, request kena `404 No active credentials for provider`.

## Langkah 6 — Arahkan Codex CLI ke sini
Pakai `cxgateway` (lihat [06-cxgateway.md](06-cxgateway.md)):
```bash
cxgateway local
```
Cek log gateway untuk memastikan request beneran lewat sini:
```bash
docker logs --tail 20 9router
# [04:25:28] 🔵 ▶ POST cx/gpt-5.6-sol → codex/gpt-5.6-sol · ACC:<akun>@icloud.com
```

## Backup
Volume `9router_local_data` isinya `/app/data` (SQLite + OAuth token + machine-id):
```bash
docker run --rm -v 9router_local_data:/data -v "$PWD":/out alpine \
  tar czf /out/9router-local-backup.tar.gz -C /data .
```
Restore & detail lain: [04-backup-restore.md](04-backup-restore.md).
