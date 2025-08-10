---
title: "コマンド1回で“今”を1行記録：任意のティッカー（仮想通貨の価格）と為替を手動取得するシェル" 
authors:
  - "JapanNomu"
published: 2025-08-10T14:00:00+09:00 
topics:
  - "仮想通貨" 
  - "為替" 
  - "自動取得スクリプト" 
  - "作業効率向上"
---

# コマンド1回で“今”を1行記録：任意のティッカー（仮想通貨の価格）と為替を手動取得するシェル

## 経緯
毎日それぞれの画面を開いて個別に数値を調べるのが手間だったため、最小の操作で“今の値”を**1行でサクッと残す**小さなシェルを作りました。

## この例で使う対象
- トークン：**MOVEUSDT／WALUSDT**（MEXCのUSDT建てティッカー）  
- 為替：**USD/JPY**（外部APIから取得）

※他のトークンにしたい場合は、後述の `SYMBOLS=(...)` を書き換えるだけ。

## 出力イメージ（1行）
```
2025-08-10T15:32 USDJPY=\146.12 MOVEUSDT=$0.1427 WALUSDT=$0.4206
```

## 何を出力するか
- タイムスタンプ（JST、分まで：`YYYY-MM-DDTHH:MM`）
- **USD/JPY：小数第2位**（値の前に `\`）
- **各トークン価格：小数第4位**（値の前に `$`）
- 標準出力に今回分の**1行**を表示、同じ1行を `/home/<ユーザー名>/cript/price_log.txt` の**先頭**へ追記

---

## 使い方（最短）

### 1) 依存パッケージをインストールする
```bash
sudo apt-get update && sudo apt-get install -y jq curl
```

### 2) 保存先を作る
```bash
mkdir -p /home/<ユーザー名>/cript
```

### 3) スクリプト全文（cat EOFで配置）
> `$USER` を使っているので、そのまま貼って動きます。必要なら `/home/$USER` を `/home/<あなたのユーザー名>` に置き換えてください。

```bash
cat <<'EOF' > /home/$USER/cript/price_snap.sh
#!/usr/bin/env bash
set -euo pipefail
export LC_ALL=C

# ===== 設定 =====
OUT="/home/$USER/cript/price_log.txt"
SYMBOLS=("MOVEUSDT" "WALUSDT")   # ← ここに MEXC のティッカーを追加/削除/並べ替え（例: "BTCUSDT"）
MAX_LOOKBACK_DAYS=10
FETCH_USDTUSD="${FETCH_USDTUSD:-0}"  # 0=1.0固定, 1=厳密取得（必要時のみ）

mkdir -p "$(dirname "$OUT")"
TS=$(TZ=Asia/Tokyo date "+%Y-%m-%dT%H:%M")

# ===== ユーティリティ =====
jq_get() { jq -r "$1 // empty"; }

# 為替ソース
fx_from_exchangerate_host() {  # $1: "latest" or "YYYY-MM-DD"
  local url
  if [[ "$1" == "latest" ]]; then
    url="https://api.exchangerate.host/latest?base=USD&symbols=JPY"
  else
    url="https://api.exchangerate.host/$1?base=USD&symbols=JPY"
  fi
  curl -s "$url" | jq_get '.rates.JPY'
}
fx_from_frankfurter() {        # $1: "latest" or "YYYY-MM-DD"
  local url
  if [[ "$1" == "latest" ]]; then
    url="https://api.frankfurter.app/latest?from=USD&to=JPY"
  else
    url="https://api.frankfurter.app/$1?from=USD&to=JPY"
  fi
  curl -s "$url" | jq_get '.rates.JPY'
}
fx_from_erapi_latest() {       # 追加の保険（最新のみ）
  curl -s "https://open.er-api.com/v6/latest/USD" | jq_get '.rates.JPY'
}

# ===== USD/JPY 取得：最新 → 直近営業日へ遡及 =====
get_usdjpy() {
  local rate=""
  rate="$(fx_from_exchangerate_host latest)"; [[ -n "$rate" ]] && { echo "$rate"; return 0; }
  rate="$(fx_from_frankfurter       latest)"; [[ -n "$rate" ]] && { echo "$rate"; return 0; }
  rate="$(fx_from_erapi_latest)";           [[ -n "$rate" ]] && { echo "$rate"; return 0; }

  for ((i=1; i<=MAX_LOOKBACK_DAYS; i++)); do
    local d
    d=$(TZ=Asia/Tokyo date -d "-$i day" "+%Y-%m-%d")
    rate="$(fx_from_exchangerate_host "$d")"; [[ -n "$rate" ]] && { echo "$rate"; return 0; }
    rate="$(fx_from_frankfurter       "$d")"; [[ -n "$rate" ]] && { echo "$rate"; return 0; }
  done
  echo ""
}

USDJPY="$(get_usdjpy)"; [[ -z "$USDJPY" ]] && USDJPY="NA"

# ===== USDT→USD（必要時のみ。既定は1.0） =====
if [[ "$FETCH_USDTUSD" == "1" ]]; then
  USDTUSD=$(curl -s "https://api.coingecko.com/api/v3/simple/price?ids=tether&vs_currencies=usd"     | jq -r '.tether.usd // empty' || true)
  [[ -z "${USDTUSD:-}" ]] && USDTUSD="1.0"
else
  USDTUSD="1.0"
fi

# ===== 出力フォーマット =====
# USDJPY：小数第2位、先頭にバックスラッシュ（環境によっては¥表示）
if [[ "$USDJPY" == "NA" ]]; then
  fx_part="USDJPY=NA"
else
  fx_val="$(awk -v r="$USDJPY" 'BEGIN{printf "%.2f", r}')"
  fx_part="USDJPY=\${fx_val}"
fi

# 各トークン：小数第4位、先頭に$
coin_parts=()
for SYM in "${SYMBOLS[@]}"; do
  RAW=$(curl -s "https://api.mexc.com/api/v3/ticker/price?symbol=${SYM}" || true)
  P_USDT=$(echo "${RAW}" | jq -r '.price // empty')
  if [[ -z "$P_USDT" ]]; then
    coin_parts+=( "${SYM}=NA" )
  else
    P_OUT=$(awk -v p="$P_USDT" -v u="$USDTUSD" 'BEGIN{printf "%.4f", p*u}')
    coin_parts+=( "${SYM}=\$${P_OUT}" )
  fi
done

line="${TS} ${fx_part} ${coin_parts[*]}"

# ===== ログに先頭追記（新しいものが上） =====
TMP=$(mktemp)
printf "%s\n" "$line" > "$TMP"
if [[ -f "$OUT" ]]; then cat "$OUT" >> "$TMP"; fi
mv "$TMP" "$OUT"

# ===== 標準出力：今回分の1行 =====
printf "%s\n" "$line"
EOF
```

### トークンの追加・変更
- 取得対象はスクリプト先頭の `SYMBOLS` 配列で決めます。  
- **MEXCのUSDT建てティッカー**をここに**追加・削除・並べ替え**すれば、そのまま**1行出力に反映**されます。  
- 例：`SYMBOLS=("MOVEUSDT" "WALUSDT" "BTCUSDT")`  

### 4) 実行権限を付与
```bash
chmod +x /home/<ユーザー名>/cript/price_snap.sh
```

### 5) 実行する
```bash
/home/<ユーザー名>/cript/price_snap.sh
```

### 6) 実行結果の標準出力サンプル
```
2025-08-10T15:32 USDJPY=\146.12 MOVEUSDT=$0.1427 WALUSDT=$0.4206
```

### 7) 生成されたログファイルの中身サンプル（先頭が最新）
```
2025-08-10T15:32 USDJPY=\146.12 MOVEUSDT=$0.1427 WALUSDT=$0.4206
2025-08-10T13:10 USDJPY=\145.98 MOVEUSDT=$0.1401 WALUSDT=$0.4180
2025-08-09T19:05 USDJPY=\146.10 MOVEUSDT=$0.1399 WALUSDT=$0.4175
```

---

## 今後の改善余地
- `SYMBOLS` を引数や環境変数で切り替え可能にする  
- `balances.txt` を読み込んで評価額（USD/JPY）を同じ1行に合算  
- **定期実行（cron）**：例）毎日09:00に1行取得  
  ```bash
  crontab -e
  # 例: 毎日9時に実行（ユーザー名は適宜）
  0 9 * * * /home/<ユーザー名>/cript/price_snap.sh >> /home/<ユーザー名>/cript/cron_stdout.log 2>&1
  ```
- **直接Excelへ出力（.xlsx）**：  
  - 例1）ログをCSVに併記→`libreoffice --headless --convert-to xlsx` で変換  
  - 例2）小さなPython補助で `xlsxwriter` を使って1行ずつ追記  
- CSVへ同時出力、月次／年次の自動分割  
- API障害時のリトライ／タイムアウト強化、失敗タグの明確化  
- ログから簡易グラフを出す補助スクリプト（別コマンド）

---

## まとめ
このスクリプト1つで毎日の作業がかなり楽になりました。  
今後は自動化、Excelへの取り込みなども行えるようにして、全自動で毎日最新の情報が整理された状態で出力されるようにしようと思います。  
AIと共同で作っていくと、一人で作るよりも調べたり考えたりする時間が削減されて、記事完成までの時間が大幅に短縮されました。今回はchatGPT 5と5 Thinkingを使いましたが、まだ物足りないところも多いのでより良いchatGPT 6を待ちたいと思います。

