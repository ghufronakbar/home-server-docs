# 01 — Install 9Router di Server (Dokploy + Cloudflare Tunnel)

Runbook deploy 9Router sebagai service **Docker Compose** yang dikelola Dokploy, diekspos ke internet lewat Cloudflare Tunnel.

## Prasyarat
- Server Linux dengan **Docker** + **Dokploy** terpasang.
- **Cloudflare Tunnel** (token-managed / `cloudflared`) sudah jalan di server.
- Sebuah **Dokploy API Token** (Dokploy → Settings/Profile → API/Swagger → generate). Simpan sebagai `<DOKPLOY_API_TOKEN>`.
- Akses SSH ke server (di sini `ssh home`).
- Domain untuk instance, mis. `9router.lans.my.id` → `<DOMAIN>`.

Semua perintah `curl` di bawah dijalankan **di server** (via `ssh home '...'`) menargetkan Dokploy di `http://localhost:3000`.

---

## Langkah 1 — Generate secret
Jalankan di mesin mana pun, catat hasilnya (dipakai di compose):
```bash
echo "JWT_SECRET=$(openssl rand -hex 32)"
echo "API_KEY_SECRET=$(openssl rand -hex 32)"
echo "MACHINE_ID_SALT=$(openssl rand -hex 16)"
echo "INITIAL_PASSWORD=$(openssl rand -base64 12 | tr -d '/+=' | cut -c1-16)"
```
- `INITIAL_PASSWORD` = password login dashboard (login 9Router **hanya password**, tanpa username).

## Langkah 2 — Temukan project & environment di Dokploy
```bash
ssh home "curl -s -H 'x-api-key: <DOKPLOY_API_TOKEN>' http://localhost:3000/api/project.all"
```
Catat `projectId` dan `environmentId` (environment `production` yang `isDefault:true`) dari project tujuan (mis. **AI-Workspace**).

## Langkah 3 — Buat service Compose (kosong)
```bash
ssh home "curl -s -X POST -H 'x-api-key: <DOKPLOY_API_TOKEN>' -H 'Content-Type: application/json' \
  -d '{\"name\":\"9router\",\"description\":\"9Router AI coding gateway\",\"environmentId\":\"<ENVIRONMENT_ID>\",\"composeType\":\"docker-compose\"}' \
  http://localhost:3000/api/compose.create"
```
Catat `composeId` dari respons.

## Langkah 4 — Siapkan file compose
Buat `docker-compose.yml` (ganti `<...>` dengan secret Langkah 1 dan `<DOMAIN>`):
```yaml
services:
  9router:
    image: decolua/9router:latest
    container_name: 9router
    restart: always
    ports:
      - "20128:20128"
    volumes:
      - 9router-data:/app/data
    environment:
      DATA_DIR: /app/data
      PORT: "20128"
      HOSTNAME: "0.0.0.0"
      NODE_ENV: production
      JWT_SECRET: "<JWT_SECRET>"
      INITIAL_PASSWORD: "<INITIAL_PASSWORD>"
      API_KEY_SECRET: "<API_KEY_SECRET>"
      MACHINE_ID_SALT: "<MACHINE_ID_SALT>"
      AUTH_COOKIE_SECURE: "false"
      REQUIRE_API_KEY: "true"          # true = endpoint /v1 wajib API key (disarankan utk publik)
      BASE_URL: "https://<DOMAIN>"
      NEXT_PUBLIC_BASE_URL: "https://<DOMAIN>"
      CLOUD_URL: "https://9router.com"

volumes:
  9router-data:
    name: 9router-data
```
Catatan env:
- `DATA_DIR=/app/data` + volume `9router-data` → **wajib**, ini yang bikin data persisten.
- `REQUIRE_API_KEY`: set `true` untuk endpoint publik. Untuk setup awal boleh `false` dulu, lalu ubah ke `true` setelah bikin API key.
- `AUTH_COOKIE_SECURE=false` menghindari login-loop di belakang proxy http→origin.

## Langkah 5 — Kirim compose ke Dokploy (sourceType raw) & deploy
Bangun payload JSON dengan aman lalu kirim:
```bash
# di mesin dgn file docker-compose.yml:
python3 - <<'PY'
import json
compose = open("docker-compose.yml").read()
payload = {"composeId":"<COMPOSE_ID>","sourceType":"raw","composeFile":compose,
           "composeType":"docker-compose","composePath":"./docker-compose.yml","autoDeploy":False}
open("update.json","w").write(json.dumps(payload))
PY
scp update.json home:/tmp/9router-update.json

ssh home "curl -s -X POST -H 'x-api-key: <DOKPLOY_API_TOKEN>' -H 'Content-Type: application/json' \
  -d @/tmp/9router-update.json http://localhost:3000/api/compose.update >/dev/null && echo update-OK"

ssh home "curl -s -X POST -H 'x-api-key: <DOKPLOY_API_TOKEN>' -H 'Content-Type: application/json' \
  -d '{\"composeId\":\"<COMPOSE_ID>\"}' http://localhost:3000/api/compose.deploy"
```
> Alternatif tanpa Dokploy: taruh `docker-compose.yml` + secret di `.env` lalu `docker compose up -d`. 9Router juga jalan dgn `docker run -d -p 20128:20128 -v 9router-data:/app/data -e DATA_DIR=/app/data decolua/9router:latest`.

## Langkah 6 — Arahkan Cloudflare Tunnel
Di Cloudflare Zero Trust → Networks → Tunnels → tunnel Anda → **Public Hostname**:
- Subdomain/host: `<DOMAIN>`
- Service (origin): **`http://172.17.0.1:20128`**  ← gateway docker bridge (karena cloudflared di network `bridge`).
  - Kalau cloudflared pakai `network_mode: host`, boleh `http://localhost:20128`.

## Langkah 7 — Verifikasi
```bash
ssh home '
docker ps --filter name=9router --format "{{.Names}} {{.Status}} {{.Ports}}" | grep -v cloudflare
curl -s -o /dev/null -w "local  20128 -> %{http_code}\n" http://172.17.0.1:20128/
curl -s -o /dev/null -w "public       -> %{http_code}\n" -L https://<DOMAIN>/
# dgn REQUIRE_API_KEY=true: tanpa key harus 401, dgn key harus 200
curl -s -o /dev/null -w "no-key  /v1/models -> %{http_code}\n" https://<DOMAIN>/v1/models
curl -s -o /dev/null -w "with-key /v1/models -> %{http_code}\n" -H "Authorization: Bearer <9ROUTER_API_KEY>" https://<DOMAIN>/v1/models'
```
Sukses jika: container `Up`, local 307, public 307→200 (redirect ke `/login`), no-key 401, with-key 200.

## Langkah 8 — Konfigurasi via dashboard
1. Buka `https://<DOMAIN>` → login dengan `INITIAL_PASSWORD`.
2. **Providers** → hubungkan akun (mis. Codex OAuth, bisa >1 akun untuk load-balance).
3. **Endpoint / API Keys** → buat **API key** (`<9ROUTER_API_KEY>`), dipakai client di `02-setup-codex-cli.md`.
4. Pastikan **Require API Key** aktif (kalau belum di-set via env).

## Update ke versi baru
```bash
# backup dulu (lihat 04-backup-restore.md), lalu:
ssh home "curl -s -X POST -H 'x-api-key: <DOKPLOY_API_TOKEN>' -H 'Content-Type: application/json' \
  -d '{\"composeId\":\"<COMPOSE_ID>\"}' http://localhost:3000/api/compose.deploy"
```
Dokploy pull image `:latest` terbaru & recreate container; volume `9router-data` tetap aman.
