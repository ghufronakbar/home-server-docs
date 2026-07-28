# 05 — Troubleshooting

## Error dari Codex CLI

### `404 Not Found: No active credentials for provider: openai`
**Sebab:** nama model tidak ber-prefix `cx/` (mis. `gpt-5.6-luna`), jadi 9Router merutekan ke provider `openai` yang tidak terhubung.
**Fix:** pakai `cx/...` di `~/.codex/config.toml` (mis. `model = "cx/gpt-5.6-luna"`), atau `cxmodel gpt-5.6-luna`. Ini sering terjadi setelah memakai menu `/model` bawaan Codex — **jangan pakai `/model`**, pakai `cxmodel`.

### `401 Unauthorized: API key required for remote API access`
**Sebab:** Codex tidak mengirim API key. Biasanya karena key ditulis sebagai `api_key` inline di provider (diabaikan Codex) atau env var kosong.
**Fix:** gunakan `env_key = "OPENAI_API_KEY"` di block `[model_providers.proxy]`, dan pastikan `export OPENAI_API_KEY=...` ada di `~/.zshrc` lalu `source ~/.zshrc`. Cek: `echo ${OPENAI_API_KEY:0:8}`.

### `Missing environment variable: OPENAI_API_KEY`
**Sebab:** `env_key` menunjuk var yang belum di-export di shell.
**Fix:** `export OPENAI_API_KEY=...` di `~/.zshrc`, buka terminal baru / `source ~/.zshrc`.

### `MCP client for 'github' failed... GITHUB_PAT_TOKEN not set`
**Tidak terkait 9Router** — itu plugin/MCP github di Codex. Set `export GITHUB_PAT_TOKEN=...` atau nonaktifkan plugin. Abaikan kalau tidak dipakai.

---

## Error di sisi server / tunnel

### URL publik tidak bisa diakses (502/timeout), padahal container `Up`
**Sebab umum:** origin Cloudflare Tunnel salah. cloudflared di network `bridge` tidak bisa `localhost`.
**Fix:** set origin tunnel ke `http://172.17.0.1:20128` (gateway docker bridge). Uji dari server:
```bash
ssh home 'curl -s -o /dev/null -w "%{http_code}\n" http://172.17.0.1:20128/'   # harus 307
```

### Container tidak listen di 20128
```bash
ssh home 'docker logs 9router | tail -30; ss -tlnp | grep 20128'
```
Pastikan env `PORT=20128` & `HOSTNAME=0.0.0.0`, dan port `20128:20128` ter-publish.

### Data hilang setelah redeploy
Seharusnya tidak terjadi kalau volume `9router-data` dipakai. Cek mount:
```bash
ssh home 'docker inspect 9router --format "{{range .Mounts}}{{.Name}} -> {{.Destination}}{{end}}"'
```
Harus `9router-data -> /app/data`. Kalau kosong/anonymous volume, perbaiki compose (bagian `volumes:`) dan restore dari backup (`04-backup-restore.md`).

### Lupa password login dashboard
Password = env `INITIAL_PASSWORD`. Kalau di DB sudah ada hash, mengubah `INITIAL_PASSWORD` saja mungkin tidak reset. Cara pasti: reset via dashboard bila masih bisa login; atau (last resort) restore/backup DB. Simpan `INITIAL_PASSWORD` di password manager.

---

## Referensi cek cepat (server)
```bash
ssh home '
docker ps --filter name=9router --format "{{.Names}} {{.Status}} {{.Ports}}" | grep -v cloudflare
curl -s -o /dev/null -w "gateway 20128 -> %{http_code}\n" http://172.17.0.1:20128/
'
# publik + API key (ganti <DOMAIN> & <KEY>)
ssh home 'curl -s -o /dev/null -w "no-key %{http_code}\n" https://<DOMAIN>/v1/models
curl -s -o /dev/null -w "with-key %{http_code}\n" -H "Authorization: Bearer <KEY>" https://<DOMAIN>/v1/models'
```

## Manajemen via Dokploy API
```bash
# list project / cari composeId
ssh home "curl -s -H 'x-api-key: <DOKPLOY_API_TOKEN>' http://localhost:3000/api/project.all"
# redeploy
ssh home "curl -s -X POST -H 'x-api-key: <DOKPLOY_API_TOKEN>' -H 'Content-Type: application/json' \
  -d '{\"composeId\":\"<COMPOSE_ID>\"}' http://localhost:3000/api/compose.deploy"
```
