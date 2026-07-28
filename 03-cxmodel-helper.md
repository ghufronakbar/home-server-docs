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
cxmodel -r high    # set reasoning langsung (minimal|low|medium|high)
cxmodel -l         # tampilkan model & reasoning aktif
```
Perubahan berlaku di **sesi codex baru**.

## Pasang: tempel ke `~/.zshrc`
```zsh
# === 9Router / Codex switcher (tree) — ganti model & reasoning ===
# cxmodel            -> menu tree: pilih Model / Reasoning (picker fzf)
# cxmodel gpt-5.5    -> set model langsung (prefix cx/ otomatis)
# cxmodel -r high    -> set reasoning langsung (minimal|low|medium|high)
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
  local cfg="$HOME/.codex/config.toml" choice="$1" cur; local -a efforts=(minimal low medium high)
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

## Cara kerja singkat
- Daftar model diambil **live** dari `$OPENAI_BASE_URL/models` (fallback ke list statis bila offline).
- Hanya baris `model` / `model_reasoning_effort` yang diubah; `model_provider` & lainnya tidak tersentuh.
- Effort divalidasi terhadap `minimal|low|medium|high`.
