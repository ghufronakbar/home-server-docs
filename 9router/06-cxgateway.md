# 06 — `cxgateway`: Pindah Gateway (Home Server ⇄ Lokal)

Command zsh untuk memindahkan Codex CLI antar instance 9Router — **home server** (`https://9router.lans.my.id`) atau **lokal** (docker di MacBook, `http://localhost:20128`). Sekali jalan, empat tempat ikut berubah sekaligus.

## Kenapa perlu
Base URL 9Router tersimpan di **4 tempat**, dan API key tiap instance **berbeda**:

| Tempat | Isi |
|---|---|
| `~/.codex/config.toml` → `[model_providers.proxy] base_url` | base URL untuk Codex CLI |
| `~/.codex/auth.json` → `OPENAI_API_KEY` | key untuk Codex CLI |
| env `OPENAI_BASE_URL` / `OPENAI_API_BASE` | dipakai `curl`, `cxmodel`, CLI lain |
| env `OPENAI_API_KEY` | key yang dibaca lewat `env_key` di config.toml |

Ganti manual satu-satu gampang kelewat → error `401 API key required` atau `404 No active credentials`.

## Prasyarat
- zsh, `fzf`, `curl` (macOS: `brew install fzf`).
- Instance lokal jalan: `docker compose up -d 9router` di repo `dev-tools`.
- Punya API key masing-masing instance (dashboard → Endpoint / API Keys).

## Cara pakai
```bash
cxgateway              # menu fzf: profil aktif ● , lainnya ○
cxgateway local        # langsung pindah ke 9Router lokal
cxgateway home         # langsung pindah ke 9Router home server
cxgateway -l           # profil aktif + base_url + health check
cxgateway -a staging https://host/v1 sk-xxx    # tambah/ubah profil
```
Shell tempat command dijalankan **langsung berubah**. Sesi `codex` **baru** ikut setting terbaru.

Contoh output `cxgateway local`:
```
✅ gateway -> local  (9Router — Lokal (docker dev-tools))
   base_url : http://localhost:20128/v1
   api_key  : sk-10853f2b7…f986
   health   : ✅ 200 (OK)
```

## Struktur profil
Kredensial **tidak** ditaruh di `~/.zshrc`, tapi di `~/.config/cxgateway/` (dir `700`, file `600`):
```
~/.config/cxgateway/
├── cxgateway.zsh   # fungsinya
├── active          # nama profil aktif, mis. "local"
├── home.env        # CXGW_LABEL / CXGW_BASE_URL / CXGW_API_KEY
└── local.env
```

## Pasang
1. Simpan fungsi di `~/.config/cxgateway/cxgateway.zsh` (isi lengkap ada di file tsb di mesin ini).
2. Buat profil:
   ```bash
   mkdir -p ~/.config/cxgateway && chmod 700 ~/.config/cxgateway
   cat > ~/.config/cxgateway/local.env <<'EOF'
   CXGW_LABEL="9Router — Lokal (docker dev-tools)"
   CXGW_BASE_URL="http://localhost:20128/v1"
   CXGW_API_KEY="<9ROUTER_LOCAL_API_KEY>"
   EOF
   chmod 600 ~/.config/cxgateway/*.env
   echo -n local > ~/.config/cxgateway/active
   ```
3. Di `~/.zshrc`, **hapus** export `OPENAI_BASE_URL` / `OPENAI_API_BASE` / `OPENAI_API_KEY`, ganti jadi:
   ```zsh
   source "$HOME/.config/cxgateway/cxgateway.zsh"
   ```
   Baris terakhir file itu (`_cxgw_load`) yang meng-export env dari profil aktif tiap shell start.
4. `source ~/.zshrc`.

## Cara kerja singkat
- `base_url` di `config.toml` diganti pakai `awk` yang **hanya** menyentuh baris di dalam blok `[model_providers.proxy]` — section lain (projects, plugins, mcp_servers) tidak tersentuh.
- `auth.json` ditulis ulang penuh (isinya cuma 2 field) + `chmod 600`.
- Health check = `GET $BASE_URL/models`; `200` OK, `401/403` key ditolak, `000` gateway tidak terjangkau.

## Hubungan dengan `cxmodel`
Saling melengkapi, tidak bentrok:
- `cxgateway` → ganti **instance** (base_url + API key).
- `cxmodel` → ganti **model & reasoning** di instance yang aktif.

`cxmodel` mengambil daftar model live dari `$OPENAI_BASE_URL/models`, jadi setelah `cxgateway` pindah, `cxmodel` otomatis menampilkan model milik instance yang baru.

## Troubleshooting
| Gejala | Sebab |
|---|---|
| `health: ❌ tidak terjangkau` di profil `local` | container mati → `docker compose up -d 9router` |
| `health: ⚠️ 530` di profil `home` | Cloudflare Tunnel putus (error 1033) / server mati — lihat `05-troubleshooting.md` |
| `health: 🔑 401` | key salah untuk instance itu — key home ≠ key lokal |
| `404 No active credentials for provider` | belum ada akun tersambung di **Providers** instance tsb |
| Sesi codex lama masih pakai gateway lama | wajar — buka sesi `codex` baru |

## Catatan implementasi (dua jebakan zsh)
Saat membangun daftar menu, **jangan** `source` file profil di dalam loop:
```zsh
for p in ...; do
  local CXGW_LABEL CXGW_BASE_URL CXGW_API_KEY   # ❌
  source "$CXGW_DIR/$p.env"
done
```
Di zsh, `local NAME` pada variabel yang **sudah ada isinya di scope yang sama** akan
**mencetak** `NAME=value`, bukan mendeklarasi ulang diam-diam. Akibatnya iterasi ke-2
menumpahkan nilai iterasi ke-1 ke dalam daftar pilihan — termasuk `CXGW_API_KEY`.
Solusi: ambil `base_url` pakai `sed`, jangan pernah baca key untuk keperluan tampilan.

Kedua, marker profil **wajib karakter non-spasi untuk semua baris**:
```zsh
[[ $p == $cur ]] && mark='●' || mark=' '   # ❌ baris non-aktif jadi diawali spasi
[[ $p == $cur ]] && mark='●' || mark='○'   # ✅
```
Kalau marker-nya spasi, `awk` menggabung whitespace di depan sehingga `$1` jadi nama
profil dan `$2` jadi URL — memilih profil non-aktif dari menu akan gagal.
