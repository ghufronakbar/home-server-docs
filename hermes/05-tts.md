# 05 — Text-to-Speech (TTS) di Hermes

Mengaktifkan TTS agar bot bisa mengirim voice (mis. balasan bahasa Jepang). Provider default Hermes = **Edge TTS** (gratis, tanpa API key), tapi paket `edge-tts` tidak ikut di image → harus dipasang.

## Masalah & konsep kunci
- Image Hermes memakai **venv tersegel** di `/opt/hermes/.venv` (tanpa `pip`). Install ke sana **gagal** untuk user bot dan **hilang saat redeploy**.
- Hermes punya **durable lazy-install target**: `HERMES_LAZY_INSTALL_TARGET=/opt/data/lazy-packages` (sudah di-set di image). Isinya ada di **volume data → persisten**, dan `hermes_bootstrap.activate_durable_lazy_target()` menambahkannya ke `sys.path` saat boot.
- ✅ **Pasang `edge-tts` ke lazy target itu**, bukan ke venv. Installer: `uv` (tersedia di image).

---

## Langkah 1 — Pasang edge-tts (persisten)
```bash
ssh home 'docker exec -u hermes -e HOME=/tmp -e UV_CACHE_DIR=/tmp/uvcache hermes \
  uv pip install --target /opt/data/lazy-packages --python /opt/hermes/.venv/bin/python edge-tts'
```
Verifikasi:
```bash
ssh home 'docker exec -e PYTHONPATH=/opt/data/lazy-packages hermes /opt/hermes/.venv/bin/python -c "import edge_tts; print(edge_tts.__version__)"'
```

## Langkah 2 — Set provider & voice
```bash
ssh home 'docker exec hermes sh -c "hermes config set tts.provider edge && hermes config set tts.edge.voice ja-JP-NanamiNeural"'
```
Config keys (`~/.hermes/config.yaml`):
```yaml
tts:
  provider: edge
  edge:
    voice: ja-JP-NanamiNeural   # wanita Jepang; pria = ja-JP-KeitaNeural
    # speed: 1.0                 # opsional
```
Voice lain: `en-US-AriaNeural` (Inggris), format umum `<lang>-<Name>Neural`. Daftar penuh: `edge-tts --list-voices`.

## Langkah 3 — ffmpeg (untuk kirim audio)
Sudah ada di image (`/usr/bin/ffmpeg`). Diperlukan untuk transcode audio (mis. Opus/voice-bubble). Tidak perlu install.

## Langkah 4 — Restart gateway
```bash
ssh home 'docker restart hermes'
```
Proses gateway yang berjalan memuat `edge-tts` (via lazy target) + config baru saat start.

## Langkah 5 — Verifikasi end-to-end
Tes sintesis langsung (bukti edge-tts + jaringan ke Microsoft):
```bash
ssh home 'docker exec -e PYTHONPATH=/opt/data/lazy-packages hermes /opt/hermes/.venv/bin/python -c "
import asyncio, edge_tts, os
async def m():
    c = edge_tts.Communicate(\"こんにちは、テストです\", \"ja-JP-NanamiNeural\")
    await c.save(\"/tmp/t.mp3\")
asyncio.run(m()); print(\"mp3 bytes:\", os.path.getsize(\"/tmp/t.mp3\"))
"'
```
Lalu di Discord: minta bot menjadikan teks sebagai voice (di dalam thread). Bot memanggil tool `text_to_speech` dan mengirim audio.

---

## Persistensi & redeploy
- Karena `edge-tts` ada di `/opt/data/lazy-packages` (volume), **tetap ada setelah redeploy Dokploy**.
- Kalau volume di-reset atau restore dari backup bersih → ulangi **Langkah 1**.

## Alternatif provider (kualitas lebih tinggi / suara konsisten)
Butuh API key; set lewat env/config lalu ganti provider:
```bash
# ElevenLabs
hermes config set tts.provider elevenlabs      # + ELEVENLABS_API_KEY di env gateway
# OpenAI
hermes config set tts.provider openai          # + OPENAI_API_KEY / VOICE_TOOLS_OPENAI_KEY
```
Untuk Jepang, Edge TTS umumnya sudah cukup baik dan gratis.

## Troubleshooting
| Gejala | Sebab / Fix |
|--------|-------------|
| `No module named 'edge_tts'` di log | Belum terpasang di lazy target, atau install ke venv (salah). Ulangi Langkah 1 (target `/opt/data/lazy-packages`). |
| `No module named pip` saat install | Venv tersegel — pakai `uv` (Langkah 1), bukan `python -m pip`. |
| Voice keluar aksen Inggris | `tts.edge.voice` masih default `en-US-AriaNeural`. Set ke `ja-JP-*` (Langkah 2) + restart. |
| Audio tidak terkirim / format aneh | Cek `ffmpeg` ada (`docker exec hermes which ffmpeg`). |
| Hilang setelah redeploy | Terpasang di venv, bukan lazy target. Pasang ulang ke `/opt/data/lazy-packages`. |
