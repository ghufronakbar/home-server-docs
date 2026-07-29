# 01 — Install Hermes di Server (Dokploy + Cloudflare)

Deploy Hermes sebagai compose service Dokploy, dashboard diamankan basic-auth, diekspos lewat Cloudflare Tunnel.

## Prasyarat
- Server Linux dengan Docker + Dokploy + Cloudflare Tunnel (`cloudflared`) sudah jalan.
- Dokploy API token (`<DOKPLOY_API_TOKEN>`) dan `environmentId` project tujuan. Cara ambil: lihat `../9router/01-install-9router-dokploy.md` (Langkah 2).
- SSH ke server (`ssh home`). User host: `<USER>` (mis. `lanstheprodigy`), uid/gid mis. `1000`.
- Domain, mis. `hermes.lans.my.id` → `<DOMAIN>`.

Cek uid/gid & IP: `ssh home 'id; ip -4 addr show docker0 | grep inet'` (docker0 biasanya `172.17.0.1`; catat juga IP LAN `192.168.x.x`).

---

## Langkah 1 — Siapkan folder data
```bash
ssh home 'mkdir -p ~/.hermes && chown 1000:1000 ~/.hermes'
```
Data Hermes (config.yaml, state.db, sesi, dll) persisten di sini via bind mount.

## Langkah 2 — Buat compose service di Dokploy
```bash
ssh home "curl -s -X POST -H 'x-api-key: <DOKPLOY_API_TOKEN>' -H 'Content-Type: application/json' \
  -d '{\"name\":\"hermes\",\"description\":\"Hermes Agent\",\"environmentId\":\"<ENVIRONMENT_ID>\",\"composeType\":\"docker-compose\"}' \
  http://localhost:3000/api/compose.create"
```
Catat `composeId`.

## Langkah 3 — File compose
`docker-compose.yml` (ganti `<USER>`):
```yaml
services:
  gateway:
    image: nousresearch/hermes-agent:latest
    container_name: hermes
    restart: unless-stopped
    network_mode: host
    volumes:
      - /home/<USER>/.hermes:/opt/data
    environment:
      - HERMES_UID=1000
      - HERMES_GID=1000
    command: ["gateway", "run"]

  dashboard:
    image: nousresearch/hermes-agent:latest
    container_name: hermes-dashboard
    restart: unless-stopped
    depends_on:
      - gateway
    ports:
      - "9119:9119"          # WAJIB publish (bukan host-net) agar cloudflared bisa menjangkau
    volumes:
      - /home/<USER>/.hermes:/opt/data
    environment:
      - HERMES_UID=1000
      - HERMES_GID=1000
    command: ["dashboard", "--host", "0.0.0.0", "--port", "9119", "--no-open"]
```
Kenapa dashboard **publish port** (bukan `network_mode: host`): lihat `02-troubleshooting.md` (§ 502). Singkatnya: host-net bind tidak dijangkau cloudflared karena firewall; publish port memicu Docker menambah aturan izin sendiri.

## Langkah 4 — Kirim compose (raw) & deploy pertama
```bash
python3 -c "import json;c=open('docker-compose.yml').read();open('u.json','w').write(json.dumps({'composeId':'<COMPOSE_ID>','sourceType':'raw','composeFile':c,'composeType':'docker-compose','composePath':'./docker-compose.yml','autoDeploy':False}))"
scp u.json home:/tmp/u.json
ssh home "curl -s -X POST -H 'x-api-key: <DOKPLOY_API_TOKEN>' -H 'Content-Type: application/json' -d @/tmp/u.json http://localhost:3000/api/compose.update >/dev/null"
ssh home "curl -s -X POST -H 'x-api-key: <DOKPLOY_API_TOKEN>' -H 'Content-Type: application/json' -d '{\"composeId\":\"<COMPOSE_ID>\"}' http://localhost:3000/api/compose.deploy"
```
Pull image pertama besar (±5–10 menit). Pantau: `ssh home 'docker ps --filter name=hermes'`.

> Pada deploy pertama, **dashboard akan crash-loop** dengan pesan *"Refusing to bind dashboard ... no auth providers registered"*. Itu normal — lanjut Langkah 5 untuk set auth, lalu restart.

## Langkah 5 — Set auth dashboard (WAJIB untuk bind non-loopback)
Generate password kuat, hash di dalam image (password via env var, tidak ditaruh di string):
```bash
PW='<DASHBOARD_PASSWORD>'   # ganti dgn password kuat Anda
HASH=$(ssh home "docker exec -e PW='$PW' -w /opt/hermes hermes python -c 'import os; from plugins.dashboard_auth.basic import hash_password; print(hash_password(os.environ[\"PW\"]))'")
echo "$HASH"   # bentuk: scrypt$....
```
Tulis ke config.yaml lewat CLI (aman dari ekspansi `$` karena via env var):
```bash
ssh home "printf '%s' '$HASH' > /tmp/hh"
ssh home 'H="$(cat /tmp/hh)"; docker exec -e H="$H" hermes sh -c "hermes config set dashboard.basic_auth.username admin && hermes config set dashboard.basic_auth.password_hash \"\$H\""; rm -f /tmp/hh'
```
Restart dashboard agar mengikat ulang:
```bash
ssh home 'docker restart hermes-dashboard'
```
Verifikasi bind sukses: log berisi `HERMES_DASHBOARD_READY port=9119` dan root → `/login`.
> Alternatif auth: OAuth Nous Portal via `docker exec hermes hermes dashboard register`.

## Langkah 6 — Verifikasi jangkauan (seperti cloudflared)
```bash
ssh home 'docker run --rm --network bridge alpine/curl -s -m 5 -o /dev/null -w "172.17.0.1:9119 -> %{http_code}\n" http://172.17.0.1:9119/'
```
Harus `302` (redirect ke /login). Kalau `000`, port belum ter-publish / container down (lihat `02`).

## Langkah 7 — Cloudflare Tunnel + Access
1. **Public Hostname** (Zero Trust → Networks → Tunnels → tunnel → Public Hostname → Add):
   - Host: `<DOMAIN>` · Type: `HTTP` · URL: **`172.17.0.1:9119`** (atau IP LAN `192.168.x.x:9119` — keduanya jalan setelah port di-publish).
2. **Cloudflare Access** (disarankan; Zero Trust → Access → Applications → Self-hosted):
   - Domain: `<DOMAIN>` · Policy Allow → Emails: email Anda.
   - Tanpa Access, satu-satunya gerbang adalah login Hermes (password Langkah 5) di URL publik — jaga baik-baik.

## Langkah 8 — Verifikasi publik & login
```bash
ssh home 'curl -s -m 20 -o /dev/null -w "%{http_code}\n" -L https://<DOMAIN>/'   # 200
```
Buka `https://<DOMAIN>` → (Access, jika ada) → login Hermes `admin` + `<DASHBOARD_PASSWORD>`.

## Update / manajemen
```bash
# redeploy (image :latest terbaru; data di /home/<USER>/.hermes tetap aman)
ssh home "curl -s -X POST -H 'x-api-key: <DOKPLOY_API_TOKEN>' -H 'Content-Type: application/json' \
  -d '{\"composeId\":\"<COMPOSE_ID>\"}' http://localhost:3000/api/compose.deploy"
# ubah config
ssh home 'docker exec hermes hermes config set <key> <value>'
```
Backup data: `tar czf hermes-backup.tgz -C /home/<USER>/.hermes .` (berisi API key & sesi — perlakukan sebagai rahasia).
