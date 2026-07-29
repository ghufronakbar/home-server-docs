# 02 — Troubleshooting Hermes

## 502 di `https://<DOMAIN>` (Cloudflare host error)
**Sebab:** cloudflared (container di network `bridge`) tidak bisa menjangkau dashboard.

Diagnosa dari perspektif cloudflared:
```bash
ssh home 'docker run --rm --network bridge alpine/curl -s -m 5 -o /dev/null -w "%{http_code}\n" http://172.17.0.1:9119/'
```
- `302` → jangkauan OK; masalah di config tunnel (origin/host salah). Pastikan origin `172.17.0.1:9119` (atau IP LAN `192.168.x.x:9119`), type HTTP.
- `000` → dashboard tidak terjangkau. Penyebab tersering:

### Penyebab utama: dashboard pakai `network_mode: host` (bukan published port)
Bind `network_mode: host` **tidak** membuat aturan iptables, jadi firewall host memblokir trafik container→host di port 9119. Bandingkan: 9router pakai `-p 20128` (published) → Docker menambah aturan → tembus dari bridge.

**Fix:** jalankan dashboard sebagai published port. Di compose, service `dashboard`:
```yaml
    # HAPUS: network_mode: host
    ports:
      - "9119:9119"
    command: ["dashboard", "--host", "0.0.0.0", "--port", "9119", "--no-open"]
```
Redeploy. Setelah publish, `docker ps` menampilkan `0.0.0.0:9119->9119/tcp` dan jangkauan dari bridge jadi `302`.

> Kenapa tidak buka firewall saja? `sudo` di server ini butuh password, dan mengubah firewall = setting keamanan sistem. Publish port lewat Docker mencapai hasil sama tanpa sudo.

---

## Dashboard crash-loop (restarts naik terus, exit 1)
**Log kunci:** `Refusing to bind dashboard to 0.0.0.0 — the auth gate engages on non-loopback binds, but no auth providers are registered.`

**Sebab:** sejak hardening Juni 2026, bind ke alamat non-loopback WAJIB punya auth provider; `--insecure` sudah no-op.

**Fix:** set `dashboard.basic_auth` (lihat `01` Langkah 5), lalu `docker restart hermes-dashboard`. Verifikasi log: `HERMES_DASHBOARD_READY port=9119`.

Alternatif: bind `--host 127.0.0.1` (tanpa auth) dan akses via SSH tunnel `ssh -L 9119:localhost:9119 home` — tapi ini tidak bisa dijangkau cloudflared.

---

## Cek cepat status
```bash
ssh home '
docker ps --filter name=hermes --format "{{.Names}} | {{.Status}} | {{.Ports}}"
docker run --rm --network bridge alpine/curl -s -m 5 -o /dev/null -w "reachable(bridge) -> %{http_code}\n" http://172.17.0.1:9119/
curl -s -m 15 -o /dev/null -w "public -> %{http_code}\n" -L https://<DOMAIN>/
docker logs hermes-dashboard 2>&1 | tail -8'
```

## Gateway: "No messaging platforms enabled" / "No env user allowlists"
Normal saat belum ada platform yang dikonfigurasi. Aktifkan platform (mis. Discord) + allowlist lewat dashboard atau `hermes config set`, lalu container `hermes` akan mengelolanya.

## Reset / lihat konfigurasi
```bash
ssh home 'docker exec hermes hermes config get | head -50'   # lihat config aktif
ssh home 'grep -nA3 basic_auth /home/lanstheprodigy/.hermes/config.yaml'   # cek auth dashboard
```

## Manajemen Dokploy (API)
```bash
ssh home "curl -s -H 'x-api-key: <DOKPLOY_API_TOKEN>' http://localhost:3000/api/project.all"   # cari composeId
ssh home "curl -s -X POST -H 'x-api-key: <DOKPLOY_API_TOKEN>' -H 'Content-Type: application/json' \
  -d '{\"composeId\":\"<COMPOSE_ID>\"}' http://localhost:3000/api/compose.deploy"              # redeploy
```
