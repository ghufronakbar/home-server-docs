# 04 — Backup & Restore Data 9Router

Semua data 9Router (SQLite `data.sqlite` + token OAuth akun di `auth/` + `machine-id`) ada di **named volume `9router-data`** yang di-mount ke `/app/data`. Kehilangan volume = harus connect ulang semua akun + bikin ulang API key. Backup rutin = wajib sebelum update besar.

## Apa yang aman & tidak
- ✅ **Aman** (data tetap): `docker compose up -d`, redeploy Dokploy, `docker restart`, `docker rm` container (volume terpisah dari container).
- ❌ **Menghapus data**: `docker compose down -v`, `docker volume rm 9router-data`, hapus service **beserta volume** dari Dokploy.

## Backup (satu perintah)
```bash
ssh home 'mkdir -p ~/9router-backups && \
  docker run --rm -v 9router-data:/data:ro -v ~/9router-backups:/backup alpine \
  sh -c "tar czf /backup/9router-data-\$(date +%Y%m%d-%H%M%S).tgz -C /data ."'
```
Cek hasil:
```bash
ssh home 'ls -lh ~/9router-backups/ && tar tzf $(ls -t ~/9router-backups/*.tgz | head -1) | grep -E "auth/|db/data.sqlite|machine-id"'
```

### Tarik backup ke lokal (opsional)
```bash
scp home:'~/9router-backups/*.tgz' ./
```

## Restore
```bash
# 1) hentikan service (via Dokploy stop, atau):
ssh home 'docker stop 9router'

# 2) kembalikan isi volume dari backup (ganti nama file):
ssh home 'docker run --rm -v 9router-data:/data -v ~/9router-backups:/backup alpine \
  sh -c "rm -rf /data/* && tar xzf /backup/9router-data-YYYYMMDD-HHMMSS.tgz -C /data"'

# 3) start lagi:
ssh home 'docker start 9router'   # atau redeploy dari Dokploy
```

## Restore ke device/instance BARU
1. Deploy 9Router baru (lihat `01-install-9router-dokploy.md`) — biarkan idle dulu.
2. Restore isi volume `9router-data` dari `.tgz` (langkah di atas) ke server baru.
3. Penting: agar OAuth token tetap valid, secret harus **sama** dengan instance lama — set env `JWT_SECRET`, `API_KEY_SECRET`, `MACHINE_ID_SALT`, `INITIAL_PASSWORD` sesuai nilai lama. Kalau beda, kemungkinan perlu login ulang akun & regenerate API key.

## Jadwal disarankan
Backup sebelum tiap `compose.deploy`/update image, dan simpan minimal beberapa versi terakhir.
