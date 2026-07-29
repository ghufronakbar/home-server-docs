# 02 — Setup Codex CLI agar Lewat 9Router

Konfigurasi OpenAI **Codex CLI** di device client (mis. MacBook) supaya semua request dirutekan ke 9Router.

## Prasyarat
- Codex CLI terpasang (`codex --version`).
- Sudah punya **API key 9Router** (`<9ROUTER_API_KEY>`, dibuat di dashboard → Endpoint/API Keys).
- Tahu URL instance (`https://9router.lans.my.id` → `<DOMAIN>`).

## Konsep penting (2 hal yang bikin gagal kalau salah)
1. **Model wajib prefix `cx/`** (mis. `cx/gpt-5.6-luna`). Tanpa prefix, 9Router kira model milik provider `openai` → `404 No active credentials for provider: openai`.
2. **API key dibaca dari ENV VAR**, bukan dari `api_key` inline. Codex custom provider mengambil key dari variabel yang disebut `env_key`. Kalau `api_key` ditulis inline di config, **diabaikan** → `401 API key required`.

---

## Langkah 1 — `~/.codex/config.toml`
Set provider proxy dan model default (perhatikan `env_key`, bukan `api_key`):
```toml
model = "cx/gpt-5.6-sol"        # model default, WAJIB prefix cx/
model_provider = "proxy"
model_reasoning_effort = "high" # minimal | low | medium | high

[model_providers.proxy]
name = "9Router Gateway"
base_url = "https://9router.lans.my.id/v1"   # ganti <DOMAIN> bila beda
env_key = "OPENAI_API_KEY"                   # nama env var yg menyimpan key
wire_api = "responses"
```

## Langkah 2 — Simpan API key di environment
Tambahkan ke `~/.zshrc` (atau `~/.bashrc`):
```bash
export OPENAI_BASE_URL="https://9router.lans.my.id/v1"
export OPENAI_API_BASE="https://9router.lans.my.id/v1"
export OPENAI_API_KEY="<9ROUTER_API_KEY>"
```
Lalu `source ~/.zshrc` (atau buka terminal baru).

Opsional (biar konsisten), samakan juga `~/.codex/auth.json`:
```json
{ "auth_mode": "apikey", "OPENAI_API_KEY": "<9ROUTER_API_KEY>" }
```

## Langkah 3 — Verifikasi
```bash
# cek endpoint & key valid
curl -s -o /dev/null -w "models -> %{http_code}\n" -H "Authorization: Bearer $OPENAI_API_KEY" "$OPENAI_BASE_URL/models"

# tes Codex end-to-end (headless)
cd ~ && codex exec --skip-git-repo-check --sandbox read-only "Reply with exactly one word: OK" < /dev/null
```
Sukses jika `models -> 200` dan codex membalas `OK` dengan `provider: proxy` + `model: cx/...`.

## Model yang tersedia
Lihat live: `curl -s -H "Authorization: Bearer $OPENAI_API_KEY" "$OPENAI_BASE_URL/models"`.
Contoh (semua prefix `cx/`): `cx/gpt-5.6-sol`, `cx/gpt-5.6-terra`, `cx/gpt-5.6-luna`, `cx/gpt-5.5`, `cx/gpt-5.4`, `cx/gpt-5.4-mini`, `cx/gpt-5.3-codex-spark` (+ varian `-review`).

## ⚠️ Jangan pakai `/model` bawaan Codex
Menu `/model` di TUI menulis nama model **tanpa** prefix `cx/` → bikin error 404. Ganti model lewat command `cxmodel` (lihat `03-cxmodel-helper.md`) atau edit `config.toml` langsung.

## Ganti model default
Edit baris `model` di `~/.codex/config.toml`, atau `cxmodel <nama>`. Perubahan berlaku di **sesi codex baru**.
