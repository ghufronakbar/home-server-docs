# 08 — Provider Antigravity (`ag/*`) mati total: 429 / 403

Catatan insiden **2026-09-23** di instance `https://9router.lans.my.id`. Semua model `ag/*` gagal,
provider lain (`cx/*`) normal. Akar masalahnya **versi 9Router yang ketinggalan**, bukan quota,
bukan IP, bukan scope OAuth.

Ringkas: **9Router `v0.5.40` → `v0.5.86`, selesai.** Detail di bawah supaya kalau kejadian lagi
tidak perlu menebak dari nol.

## Gejala
- Provider Antigravity ter-add, 2 koneksi OAuth aktif, tapi **semua model `ag/*` gagal**.
- "Available Models" di dashboard: `The operation was aborted due to timeout`.
- Quota Tracker menunjukkan **0/1000 (100% sisa)** untuk semua model, tapi request tetap ditolak.
- Akun #1 (`ancika…@gmail.com`) → `[403]: HTTP 403`; akun #2 (`strix…@gmail.com`) → `[429] RESOURCE_EXHAUSTED`.
- Karena dua-duanya gagal, failover habis → `all 2 accounts locked`, terasa "semua model mati".

```
[ProjectId] onboardUser failed after 5 attempts: onboardUser done but no project_id in response
✗ ERROR 429 · antigravity/gemini-3.1-pro-low · 20364ms
  URL: https://cloudcode-pa.googleapis.com/v1internal:generateContent
  [429]: {"error":{"code":429,"message":"Resource has been exhausted (e.g. check quota).",
          "status":"RESOURCE_EXHAUSTED"}}
[AUTH] antigravity | all 2 accounts locked for gemini-3.1-pro-low | lastError=[403]: HTTP 403
```

## Akar masalah: endpoint chat Antigravity pindah host

Google memindahkan trafik chat Antigravity ke host baru. 9Router sudah menyesuaikan di rilis
baru; instance kita masih menembak host lama, jadi dibalas `RESOURCE_EXHAUSTED` / `403`.

| | v0.5.40 (yang jalan) | v0.5.86 (latest) |
|---|---|---|
| `apiEndpoint` (chat) | `https://cloudcode-pa.googleapis.com` | `https://daily-cloudcode-pa.googleapis.com` |
| `loadCodeAssistEndpoint` | `cloudcode-pa…` | `cloudcode-pa…` (tetap) |
| `onboardUserEndpoint` | `cloudcode-pa…` | `cloudcode-pa…` (tetap) |
| `scopes` | 5 scope | 5 scope (**identik**) |

Host di v0.5.40 **persis** host yang muncul di log error. Hanya endpoint chat yang pindah;
`loadCodeAssist` & `onboardUser` tetap di host lama di kedua versi.

> **429 di sini bukan berarti quota habis.** Quota Tracker yang menampilkan 0/1000 itu akurat —
> quota memang belum kepakai. Jadi menunggu reset quota (5 jam / 7 hari) **tidak menolong**.

## Cara verifikasi sendiri

⚠️ Deploy kita pakai **image Docker** (`decolua/9router`), bukan `npm i -g 9router`. Layout npm
package ≠ layout image (npm: `app/.next-cli-build/…`, image: `/app/open-sse/…`), jadi
**verifikasi harus di image**, dan fix-nya juga `docker pull`, bukan `npm i -g`.

```bash
# versi yang sedang jalan + endpoint yang dipakai
ssh home 'docker exec 9router sh -lc "
  grep -m1 version /app/package.json
  grep -n apiEndpoint /app/open-sse/providers/registry/antigravity.js"'

# versi terbaru di registry npm (nomor versinya sama dengan tag image)
npm view 9router version

# intip image terbaru TANPA menyentuh container yang jalan
ssh home 'docker pull decolua/9router:latest'
ssh home 'docker run --rm --entrypoint sh decolua/9router:latest -lc "
  grep -m1 version /app/package.json
  grep -n cloudcode-pa /app/open-sse/providers/registry/antigravity.js"'
```

Hasil saat insiden: container `0.5.40` → `apiEndpoint: "https://cloudcode-pa.googleapis.com"`,
image baru `0.5.86` → `apiEndpoint: "https://daily-cloudcode-pa.googleapis.com"`. Cukup untuk
memutuskan upgrade.

## Perbaikan: upgrade image

Prosedur ini generik — pakai juga untuk upgrade 9Router rutin.

```bash
# 1) backup volume dulu (SQLite + OAuth token)
ssh home 'mkdir -p ~/backup && docker run --rm -v 9router-data:/data -v ~/backup:/out alpine \
  tar czf /out/9router-data-pre-<VERSI>.tar.gz -C /data .'

# 2) cari direktori compose milik Dokploy (id-nya bisa berubah, jangan dihafal)
ssh home 'docker inspect 9router \
  --format "{{index .Config.Labels \"com.docker.compose.project.working_dir\"}}"'
# → /etc/dokploy/compose/<compose-id>/code

# 3) pull + recreate service 9router saja
ssh home 'docker pull decolua/9router:latest'
ssh home 'cd /etc/dokploy/compose/<compose-id>/code && docker compose up -d 9router'

# 4) verifikasi
ssh home 'docker exec 9router sh -lc "grep -m1 version /app/package.json"'
ssh home 'curl -s -o /dev/null -w "gateway -> %{http_code}\n" http://172.17.0.1:20128/'   # 307
```

Data aman: `9router-data` adalah named volume, recreate container tidak menghapusnya.
Backup insiden ini: `~/backup/9router-data-pre-0.5.86.tar.gz` (3.0 MB) di home server.

Alternatif lewat Dokploy API (`compose.deploy`) ada di [05-troubleshooting.md](05-troubleshooting.md).

## Hasil setelah upgrade

Semua diuji lewat gateway publik dengan API key (`cxgateway` → `home.env`):

| Uji | Hasil |
|---|---|
| `GET /v1/models` | 200 — 123 model, **20 di antaranya `ag/*`** |
| `ag/gemini-3-flash` × 5 | 200 semua, `finish_reason: stop` |
| `ag/gemini-3.1-pro-low` (korban 429 di log asli) | 200, `content: "OK"` |
| `ag/gemini-3.8-flash` | 200, `content: "OK"` |
| `ag/claude-sonnet-4-6` | 200, balasan normal |
| Log sejak restart | **nol** `ERROR` / `403` / `429` / `locked` / `ProjectId` |

Contoh uji cepat:
```bash
set -a; . ~/.config/cxgateway/home.env; set +a
curl -s -m 120 -X POST "$CXGW_BASE_URL/chat/completions" \
  -H "Authorization: Bearer ${CXGW_API_KEY}" -H "Content-Type: application/json" \
  -d '{"model":"ag/gemini-3-flash","messages":[{"role":"user","content":"Balas: OK"}],"max_tokens":300,"stream":false}'
```

> Kalau pakai `max_tokens` kecil (mis. 20), balasan bisa kosong dengan `finish_reason: "length"` —
> itu **bukan error**. Model `ag/*` memakai reasoning token dulu (`reasoning_tokens`), jadi beri
> minimal ~300.

## Koreksi terhadap dugaan awal

Tiga dugaan yang ternyata **salah**, dicatat supaya tidak diulang:

1. **"403 akun #1 karena scope OAuth kurang (`cclog`, `experimentsandconfigs`) — perlu hapus koneksi & OAuth ulang."**
   Salah. Scope di v0.5.40 dan v0.5.86 **identik** (5 scope, dua itu sudah ada di keduanya).
   Setelah upgrade, **akun #1 justru yang melayani semua request uji dan 200 terus, tanpa
   re-OAuth**. Jadi 403 juga gejala endpoint lama, bukan scope, bukan ban.
2. **"`gemini-3.8-flash` model invalid / belum dikenal."** Salah — terdaftar di `/v1/models`
   dan balas 200. Silang merah di dashboard itu penanda "model terakhir yang gagal dites",
   bukan penanda model invalid.
3. **"Fix-nya `npm i -g 9router@latest`."** Tidak berlaku untuk instance ini — deploy-nya
   image Docker via Dokploy, jadi `docker pull` + `docker compose up -d 9router`.

Yang **benar** dari dugaan awal: endpoint pindah host, 429 bukan quota habis, dan
`onboardUser`/`ProjectId` gagal itu non-fatal (sejak upgrade tidak muncul lagi).

## Yang sudah dieliminasi sebagai penyebab

| Dugaan | Kenapa gugur |
|---|---|
| IP server diblokir / region | `cx/*` jalan normal dari server yang sama, di log yang sama |
| Quota akun habis | Quota Tracker 0/1000, dan setelah upgrade akun yang sama langsung 200 |
| Data/koneksi OAuth korup | Volume tidak disentuh saat upgrade, koneksi lama langsung jalan |

## Risiko yang perlu disadari

Banner risiko di dashboard itu valid: memakai Antigravity lewat proxy/router **bukan pemakaian
resmi**, akun bisa kena pembatasan atau ban. Insiden ini ternyata bukan ban — tapi bukan berarti
risikonya tidak ada. Pertimbangkan sebelum menambah akun baru ke pool.

## Pelajaran

`decolua/9router:latest` **tidak auto-update**. Container insiden ini `Up 4 weeks` di `v0.5.40`
sementara upstream sudah `v0.5.86` — 46 rilis tertinggal. Provider yang tiba-tiba mati total
sementara provider lain sehat → **cek selisih versi dulu**, sebelum mengutak-atik akun atau quota.
