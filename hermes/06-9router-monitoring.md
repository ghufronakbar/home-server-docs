# 06 — Pemantauan 9Router oleh Agent (read-only)

Memberi agent Hermes kemampuan **membaca** usage & limit 9Router (tanpa bisa mengubah apa pun, dan tanpa memegang kredensial). Agent bisa ditanya kasual di Discord: *"cek limit tiap akun codex berapa persen?"*, *"usage 9router hari ini?"*.

## Prinsip / model keamanan
- Agent ada di **Discord publik** & memproses konten tak-tepercaya → **tidak boleh** memegang password 9Router atau akses DB (DB berisi OAuth token akun Codex).
- Maka: **host** yang memegang kredensial + baca data → tulis **snapshot JSON tersanitasi** ke data dir Hermes → agent hanya **membaca file** itu. Read-only sejati; tidak ada jalur tulis dari agent.

## Arsitektur
```
[host cron */15]
  9router-status-export.py
    ├─ docker cp DB 9router  ─────────────► usage absolut (request, token, cost-ref, per-model, per-akun, harian)
    └─ login API dashboard (password host) ─► quota LIVE per akun (used/total/%/reset)   ← data ini TIDAK ada di DB
        │
        ▼ tulis
  /home/<USER>/.hermes/9router-status.json
        │  (bind mount .hermes → /opt/data)
        ▼ baca (read-only)
  Hermes agent  ◄── skill "9router-status" menyuruh agent baca /opt/data/9router-status.json
```

## Komponen (lokasi)
| Item | Path |
|------|------|
| Exporter | `/home/<USER>/9router-status-export.py` |
| Password dashboard (host-only, chmod 600) | `/home/<USER>/.9router-admin` |
| Cron | crontab host: `*/15 * * * * /usr/bin/python3 /home/<USER>/9router-status-export.py` |
| Output JSON | `/home/<USER>/.hermes/9router-status.json` → agent lihat di `/opt/data/9router-status.json` |
| Skill | `/home/<USER>/.hermes/skills/integrations/9router-status/SKILL.md` (di volume, persisten) |

## Setup (reproduksi)
Ganti `<USER>` dan `<9ROUTER_DASHBOARD_PASSWORD>`.

**1. Password dashboard (host-side saja):**
```bash
ssh home "printf '%s' '<9ROUTER_DASHBOARD_PASSWORD>' > ~/.9router-admin && chmod 600 ~/.9router-admin"
```
> Catatan: 9Router hanya punya login admin penuh (tanpa scoped token) & ada **lockout** — jangan brute-force. Quota bersifat opsional: tanpa file password ini, exporter tetap jalan tapi hanya usage absolut (tanpa % limit).

**2. Taruh exporter** `9router-status-export.py` (isi lengkap di Lampiran) ke `/home/<USER>/`, lalu jalankan sekali:
```bash
ssh home 'python3 ~/9router-status-export.py'   # -> "wrote ... quota_available= True"
```

**3. Cron tiap 15 menit:**
```bash
ssh home "( crontab -l 2>/dev/null | grep -v 9router-status-export.py; echo '*/15 * * * * /usr/bin/python3 /home/<USER>/9router-status-export.py >/dev/null 2>&1' ) | crontab -"
```

**4. Skill** — buat `~/.hermes/skills/integrations/9router-status/SKILL.md` (isi di Lampiran). Cek terdaftar:
```bash
ssh home 'docker exec hermes hermes skills list | grep 9router'   # -> enabled
```

**5. Verifikasi end-to-end:**
```bash
ssh home 'docker exec hermes hermes -z "cek limit tiap akun codex berapa persen dan kapan reset?" --yolo'
```

## Skema JSON (`9router-status.json`)
- `generated_at`, `quota_available` (bool), `quota_error`.
- `totals`: requests, prompt/completion/total tokens, `cost_ref_usd` (referensi saja — Codex subscription), success/errors, first/last request.
- `by_model`, `by_account` (usage absolut), `daily`.
- `api_keys`: name, key (masked), active, createdAt.
- `codex_accounts[]`: name, email, active, priority, **`quota`**: `{used, total, remaining, used_percent, reset_at, limit_reached, plan}`.

## Cara pakai
Tanya bot kasual di Discord (skill terpicu otomatis) — **tidak perlu instruksi khusus**:
- "berapa usage 9router hari ini?"
- "cek limit tiap akun codex berapa persen, kapan reset?"
- "list API key 9router"

Data snapshot ≤15 menit (agent menyebut waktunya). Untuk aksi **menulis/manage**, skill sudah diinstruksikan menolak & mengarahkan ke dashboard/operator.

## Interval & alternatif on-demand
- Cron `*/15` dipilih karena bagian quota memanggil **API live** (login + fetch per akun) — bukan panggilan LLM, jadi tidak makan kuota chat, tapi tetap API call.
- Ingin **real-time on-demand** (tanpa cron)? Perlu **layanan kecil di host** yang memegang password dan dipicu agent via `curl 127.0.0.1:PORT` (kredensial tetap di host). Jangan menaruh password/DB di dalam agent — itu membuka risiko injection. (Belum dipasang; cron sudah cukup.)

## Troubleshooting
| Gejala | Fix |
|--------|-----|
| `quota_available: false` di JSON | Cek `quota_error`. Password salah/lockout → betulkan `~/.9router-admin`. Absolut usage tetap ada. |
| Agent bilang file tidak ada | Exporter belum jalan / cron belum aktif. Jalankan manual (Setup #2), cek `crontab -l`. |
| Usage 0 / stale | Exporter membaca DB via `docker cp`; pastikan container `9router` jalan. |
| Login lockout | Berhenti; tunggu reset. Jangan menebak password. |

## Lampiran A — exporter (`9router-status-export.py`)
Baca DB (usage) + login API dashboard (quota live), tulis JSON. Password dibaca dari `~/.9router-admin` (host-only); jika tak ada, quota di-skip dengan aman.

```python
#!/usr/bin/env python3
import sqlite3, json, os, subprocess, datetime, tempfile, urllib.request, urllib.error, http.cookiejar
from collections import defaultdict
OUT = "/home/<USER>/.hermes/9router-status.json"
BASE = "http://localhost:20128"
PW_FILE = "/home/<USER>/.9router-admin"
TMP = tempfile.mkdtemp(prefix="9r_")
def cp(src):
    subprocess.run(["docker","cp",f"9router:/app/data/db/{src}",f"{TMP}/{src}"], check=False,
                   stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
for f in ("data.sqlite","data.sqlite-wal","data.sqlite-shm"): cp(f)
c = sqlite3.connect(f"{TMP}/data.sqlite"); c.row_factory = sqlite3.Row
def num(v):
    try: return float(v)
    except: return 0.0
rows = c.execute("select * from usageHistory").fetchall()
def toks(r):
    t=num(r["tokens"]); return t if t else num(r["promptTokens"])+num(r["completionTokens"])
def agg(key):
    d=defaultdict(lambda:[0,0,0.0])
    for r in rows:
        k=r[key] or "?"; d[k][0]+=1; d[k][1]+=int(toks(r)); d[k][2]+=num(r["cost"])
    return {k:{"requests":v[0],"tokens":v[1],"cost":round(v[2],4)} for k,v in sorted(d.items(),key=lambda x:-x[1][1])}
conns = {r["id"]:r for r in c.execute("select * from providerConnections")}
per_conn = {}
for k,v in agg("connectionId").items():
    cn=conns.get(k); per_conn[(cn["email"] or cn["name"]) if cn else str(k)] = v
def fetch_quota():
    if not os.path.exists(PW_FILE): return {}, "no password file"
    pw=open(PW_FILE).read().strip()
    if not pw: return {}, "empty password file"
    cj=http.cookiejar.CookieJar(); op=urllib.request.build_opener(urllib.request.HTTPCookieProcessor(cj))
    try:
        op.open(urllib.request.Request(BASE+"/api/auth/login", data=json.dumps({"password":pw}).encode(),
                headers={"Content-Type":"application/json"}), timeout=15)
    except Exception as e:
        return {}, f"login failed: {e}"
    out={}
    for cid in conns:
        try:
            r=op.open(BASE+f"/api/usage/{cid}", timeout=30); q=json.loads(r.read().decode())
            s=(q.get("quotas") or {}).get("session") or {}
            used=s.get("used"); total=s.get("total")
            pct=round(used/total*100,1) if isinstance(used,(int,float)) and isinstance(total,(int,float)) and total else None
            out[cid]={"plan":q.get("plan"),"limit_reached":q.get("limitReached"),"used":used,"total":total,
                      "remaining":s.get("remaining"),"used_percent":pct,"reset_at":s.get("resetAt"),"unlimited":s.get("unlimited")}
        except Exception as e:
            out[cid]={"error":str(e)[:120]}
    return out, None
quota, qerr = fetch_quota()
def mask(k):
    k=str(k or ""); return (k[:12]+"..."+k[-4:]) if len(k)>18 else "***"
keys=[{"name":r["name"],"key":mask(r["key"]),"active":bool(int(r["isActive"] or 0)),"createdAt":r["createdAt"]}
      for r in c.execute("select * from apiKeys")]
accounts=[]
for r in conns.values():
    e={"name":r["name"],"email":r["email"],"provider":r["provider"],"active":bool(int(r["isActive"] or 0)),"priority":r["priority"]}
    if r["id"] in quota: e["quota"]=quota[r["id"]]
    accounts.append(e)
daily={}
for r in c.execute("select * from usageDaily"):
    try: daily[r["dateKey"]]=json.loads(r["data"])
    except: pass
tsv=[str(r["timestamp"]) for r in rows if r["timestamp"]]
out={"generated_at":datetime.datetime.now().isoformat(timespec="seconds"),
     "note":"READ-ONLY snapshot. Cost is reference-only (Codex subscription). 'quota' = live Codex session limit.",
     "quota_available":qerr is None,"quota_error":qerr,
     "totals":{"requests":len(rows),"prompt_tokens":int(sum(num(r["promptTokens"]) for r in rows)),
               "completion_tokens":int(sum(num(r["completionTokens"]) for r in rows)),
               "total_tokens":int(sum(toks(r) for r in rows)),"cost_ref_usd":round(sum(num(r["cost"]) for r in rows),4),
               "success":sum(1 for r in rows if (r["status"] or "")=="ok"),
               "errors":sum(1 for r in rows if (r["status"] or "")!="ok"),
               "first_request":min(tsv) if tsv else None,"last_request":max(tsv) if tsv else None},
     "by_model":agg("model"),"by_account":per_conn,"daily":daily,"api_keys":keys,"codex_accounts":accounts}
with open(OUT,"w") as f: json.dump(out,f,indent=2,ensure_ascii=False)
os.chmod(OUT,0o644); subprocess.run(["rm","-rf",TMP])
print("wrote",OUT,"; quota_available=",qerr is None, qerr or "")
```

## Lampiran B — SKILL.md
Lokasi: `~/.hermes/skills/integrations/9router-status/SKILL.md`. Frontmatter (`name`, `description`, `tags`) + instruksi: kapan dipakai, baca `/opt/data/9router-status.json`, cara laporkan limit (`used_percent`, `reset_at`, flag `limit_reached`), dan aturan **read-only** (tolak permintaan menulis, arahkan ke dashboard/operator). Versi terpasang: v1.1.
