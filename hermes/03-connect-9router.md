# 03 — Sambungkan Hermes ke 9router

Arahkan model Hermes ke instance **9router** (OpenAI-compatible) sebagai provider `custom`, sehingga Hermes memakai akun/model yang sudah dihubungkan di 9router (mis. Codex `cx/*`).

## Prasyarat
- Hermes sudah jalan (lihat `01-install-hermes-dokploy.md`).
- 9router aktif di `https://9router.lans.my.id/v1` (`<9ROUTER_URL>`), dan sebuah **API key 9router** (`<9ROUTER_API_KEY>`) — boleh key yang sama dengan Codex, atau buat key khusus Hermes di dashboard 9router.
- Model target ber-prefix `cx/` (mis. `cx/gpt-5.6-sol`). Lihat daftar: `curl -H "Authorization: Bearer <9ROUTER_API_KEY>" <9ROUTER_URL>/models`.

## Konsep
- Hermes provider `custom` memakai OpenAI SDK → memanggil **`/v1/chat/completions`** (bukan `/responses`). 9router mendukung ini.
- ⚠️ **Gotcha User-Agent (403):** 9router/Cloudflare mengembalikan **`403 "Your request was blocked"`** untuk request ber-`User-Agent: OpenAI/Python...` (default OpenAI SDK). Header `X-Stainless-*` tidak masalah — hanya UA. **Fix:** override `model.default_headers.User-Agent` ke UA biasa (mis. `curl/8.7.1`).

---

## Langkah 1 — (opsional) verifikasi endpoint dulu
```bash
# harus 200 dengan UA biasa:
ssh home 'curl -s -m 40 -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer <9ROUTER_API_KEY>" -H "Content-Type: application/json" \
  -d "{\"model\":\"cx/gpt-5.6-sol\",\"messages\":[{\"role\":\"user\",\"content\":\"hi\"}]}" <9ROUTER_URL>/chat/completions'
```

## Langkah 2 — Set provider di Hermes
```bash
ssh home 'docker exec hermes sh -c "
hermes config set model.provider custom &&
hermes config set model.base_url <9ROUTER_URL> &&
hermes config set model.default cx/gpt-5.6-sol &&
hermes config set model.api_key <9ROUTER_API_KEY>
"'
```

## Langkah 3 — Override User-Agent (WAJIB, kalau tidak → 403)
```bash
ssh home 'docker exec hermes hermes config set model.default_headers.User-Agent "curl/8.7.1"'
```
Hasil di `config.yaml`:
```yaml
model:
  provider: custom
  base_url: https://9router.lans.my.id/v1
  default: cx/gpt-5.6-sol
  api_key: <9ROUTER_API_KEY>
  default_headers:
    User-Agent: "curl/8.7.1"
```

## Langkah 4 — Verifikasi end-to-end
```bash
ssh home 'docker exec hermes hermes -z "Reply with exactly one word: OK" --yolo'
```
Sukses jika membalas `OK`. Kalau `HTTP 403: Your request was blocked` → Langkah 3 belum diterapkan.

## Langkah 5 — Terapkan ke gateway berjalan
```bash
ssh home 'docker restart hermes'
```
(Sesi platform seperti Discord akan memakai config baru.)

## Ganti model
```bash
ssh home 'docker exec hermes hermes config set model.default cx/gpt-5.5'   # atau cx/* lain
```
Atau lewat dashboard → menu Models. Daftar model valid = keluaran `<9ROUTER_URL>/models` (semua prefix `cx/`).

## Troubleshooting cepat
| Gejala | Sebab / Fix |
|--------|-------------|
| `403 Your request was blocked` | UA OpenAI SDK diblokir → set `model.default_headers.User-Agent` (Langkah 3). |
| `404 No active credentials for provider: openai` | Model tanpa prefix `cx/`. Pakai `cx/<model>`. |
| `401` | API key salah/kadaluarsa, atau 9router `REQUIRE_API_KEY` aktif tapi key tidak terkirim. |
| Jawaban lambat/timeout | Model besar; naikkan timeout atau pilih model lebih kecil (mis. `cx/gpt-5.4-mini`). |
