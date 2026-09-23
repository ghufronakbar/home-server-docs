# 03 — `cxmodel`: Ganti Model & Reasoning Codex Cepat

Command shell (zsh) untuk mengganti **model** dan **reasoning effort** Codex tanpa edit manual `config.toml`, sekaligus **selalu menambah prefix `cx/`** yang benar. Menggantikan `/model` bawaan Codex (yang menghapus prefix dan bikin error 404).

## Prasyarat
- zsh, `fzf`, `python3`, `curl` (macOS: `brew install fzf`).
- Sudah set `OPENAI_API_KEY` & `OPENAI_BASE_URL` (lihat `02-setup-codex-cli.md`).
- `sed -i ''` di bawah adalah sintaks **BSD/macOS**. Di Linux ganti jadi `sed -i` (tanpa `''`).

## Cara pakai
```bash
cxmodel            # menu tree: pilih Model / Reasoning (picker fzf), loop sampai "Keluar"
cxmodel gpt-5.5    # set model langsung → cx/gpt-5.5 (prefix otomatis)
cxmodel -r xhigh   # set reasoning langsung (auto|minimal|low|medium|high|xhigh|max)
cxmodel -l         # tampilkan model & reasoning aktif
cxmodels           # refresh daftar model dari gateway aktif (alias: cxmodel -u)
```

Model dari provider mana pun bisa dipilih, bukan cuma `cx/*`:
`dahono/claude-opus-5`, `ag/auto`, `ollama/…`, dst.
Perubahan berlaku di **sesi codex baru**.

## Pasang: tempel ke `~/.zshrc`
```zsh
# === 9Router / Codex switcher (tree) — ganti model & reasoning ===
# cxmodel            -> menu tree: pilih Model / Reasoning (picker fzf)
# cxmodel gpt-5.5    -> set model langsung (prefix cx/ otomatis)
# cxmodel -r xhigh   -> set reasoning langsung (auto|minimal|low|medium|high|xhigh|max)
# cxmodel -l         -> tampilkan model & reasoning aktif
_cx_set_model() {
  local cfg="$HOME/.codex/config.toml" choice="$1" live cur; local -a models
  live=$(curl -s -m 6 -H "Authorization: Bearer ${OPENAI_API_KEY}" "${OPENAI_BASE_URL:-https://9router.lans.my.id/v1}/models" 2>/dev/null \
        | python3 -c 'import sys,json;print("\n".join(m["id"] for m in json.load(sys.stdin).get("data",[])))' 2>/dev/null)
  if [[ -n "$live" ]]; then models=(${(f)live}); else
    models=(cx/gpt-5.6-sol cx/gpt-5.6-terra cx/gpt-5.6-luna cx/gpt-5.5 cx/gpt-5.4 cx/gpt-5.4-mini cx/gpt-5.3-codex-spark)
  fi
  cur=$(sed -nE 's/^model = "(.*)"/\1/p' "$cfg" | head -1)
  [[ -z "$choice" ]] && choice=$(print -l -- "${models[@]}" | fzf --height=40% --reverse --prompt="model (aktif: ${cur}) > ")
  [[ -z "$choice" ]] && return 1
  [[ "$choice" != cx/* ]] && choice="cx/$choice"
  sed -i '' -E "s|^model = .*|model = \"${choice}\"|" "$cfg"
  echo "✅ model -> ${choice}"
}
_cx_set_effort() {
  local cfg="$HOME/.codex/config.toml" choice="$1" cur
  local -a efforts=(auto minimal low medium high xhigh max)
  cur=$(sed -nE 's/^model_reasoning_effort = "(.*)"/\1/p' "$cfg" | head -1)
  [[ -z "$choice" ]] && choice=$(print -l -- "${efforts[@]}" | fzf --height=30% --reverse --prompt="reasoning (aktif: ${cur}) > ")
  [[ -z "$choice" ]] && return 1
  if [[ " ${efforts[*]} " != *" ${choice} "* ]]; then echo "❌ effort tidak valid: ${choice} (pilih: ${efforts[*]})"; return 1; fi
  if grep -qE '^model_reasoning_effort = ' "$cfg"; then
    sed -i '' -E "s|^model_reasoning_effort = .*|model_reasoning_effort = \"${choice}\"|" "$cfg"
  else
    awk -v v="$choice" '{print} /^model_provider = /{print "model_reasoning_effort = \"" v "\""}' "$cfg" > "$cfg.tmp" && mv "$cfg.tmp" "$cfg"
  fi
  echo "✅ reasoning -> ${choice}"
}
cxmodel() {
  local cfg="$HOME/.codex/config.toml"
  [[ -f "$cfg" ]] || { echo "config.toml tidak ada: $cfg"; return 1; }
  case "$1" in
    -l|--list)
      echo "Model aktif    : $(sed -nE 's/^model = "(.*)"/\1/p' "$cfg" | head -1)"
      echo "Reasoning aktif: $(sed -nE 's/^model_reasoning_effort = "(.*)"/\1/p' "$cfg" | head -1)"
      return 0 ;;
    -r|--reasoning) _cx_set_effort "$2"; return $? ;;
    -m|--model)     _cx_set_model "$2"; return $? ;;
    ?*)             _cx_set_model "$1"; return $? ;;
  esac
  while true; do
    local m e sel
    m=$(sed -nE 's/^model = "(.*)"/\1/p' "$cfg" | head -1)
    e=$(sed -nE 's/^model_reasoning_effort = "(.*)"/\1/p' "$cfg" | head -1)
    sel=$(printf '%s\n' "Model      : ${m}" "Reasoning  : ${e}" "Keluar" \
          | fzf --height=30% --reverse --prompt="cxmodel > ")
    case "$sel" in
      Model*)     _cx_set_model ;;
      Reasoning*) _cx_set_effort ;;
      *) echo "(mulai sesi codex baru untuk memakai perubahan)"; break ;;
    esac
  done
}
```
Setelah tempel: `source ~/.zshrc`.

## `cxmodels` — refresh daftar model
Jalankan tiap habis menambah provider/model baru di dashboard 9Router. Hasilnya di-cache
**per gateway** di `~/.config/cxgateway/models.<gateway>.txt`, dan perubahannya dilaporkan:
```
➕ 3 model baru:
   cx/gpt-5.5
   dahono/claude-opus-5
   dahono/glm-5.3
✅ home: 94 model tersimpan di ~/.config/cxgateway/models.home.txt
   ag         9
   cx         14
   dahono     64
   ollama     7
```
Cache dipisah per gateway karena home server dan instance lokal punya provider berbeda.

## Cara kerja singkat
- Daftar model diambil **live** dari `$OPENAI_BASE_URL/models` tiap kali picker dibuka,
  dan hasilnya sekalian menyegarkan cache. Kalau gateway mati, picker jatuh ke cache
  (peringatan ditampilkan); kalau cache juga kosong, dipakai daftar minimal.
- Hanya baris `model` / `model_reasoning_effort` yang diubah; `model_provider` & lainnya tidak tersentuh.
- Prefix `cx/` **hanya** ditambahkan kalau nama model belum punya prefix provider sama sekali
  (tidak mengandung `/`). Jadi `cxmodel gpt-5.5` → `cx/gpt-5.5`, sedangkan
  `dahono/glm-5.3` dibiarkan apa adanya. Versi lama memaksa `cx/` ke apa pun yang bukan
  `cx/*`, sehingga model provider lain rusak jadi `cx/dahono/glm-5.3`.
- Effort divalidasi terhadap `auto|minimal|low|medium|high|xhigh|max` — mengikuti pilihan yang tersedia di dashboard 9Router.
- ⚠️ Codex CLI (v0.147.0) **tidak** memvalidasi `model_reasoning_effort` secara lokal: nilai apa pun (bahkan `NGAWUR`) diterima dan diteruskan apa adanya ke gateway. Jadi daftar di atas murni pagar dari sisi kita; yang menentukan sah/tidaknya adalah 9Router + model tujuan.
