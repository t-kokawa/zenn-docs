---
title: "Codex CLIの完了通知をWindowsの通知センター等に送る"
emoji: "🔔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Codex", "AI駆動開発", "AIエージェント", "AI", "開発生産性"]
published: false
---
私は株式会社Digeonでインターンをしている大学生です。

この記事では、WSLで動かしているCodex CLIの作業完了を、Windowsの通知センターにトースト通知で届ける方法を書きます。

あわせて、スマホにも通知できるように `ntfy` をつなぐ方法も載せます。

![トースト通知](https://storage.googleapis.com/zenn-user-upload/0459d907a48f-20260312.png)

# やること

1. Codex CLI の `notify` 機能で、タスク完了時に任意のスクリプトを呼ぶ
2. WSL 上のシェルスクリプトから Windows 側の PowerShell を呼ぶ
3. Windows の通知センターにトースト通知を出す / `ntfy` に送る

通知スクリプトは Bash で書きます。`ntfy` にも送るようにすることで、席を離れているときでも気づきやすくなります。

# 必要環境

この記事の構成は以下の環境を前提にしています。

- Codex CLI を WSL 上で使っている
- WSL 側で `bash` `jq` `curl` `iconv` `base64` を使える
- Windows 側で PowerShell から `BurntToast` モジュールを利用できる

Windows 側の通知だけ使うなら、最低限必要なのは `BurntToast` です。

# 課題：Codex CLIの作業完了に気づかない

Codex CLI はタスクの実行が完了すると、ターミナルに完了メッセージを表示します。

ただ、別の作業をしていたり、ブラウザやエディタを見ていたりすると、その表示に気づかないことがあります。

完了に気づくのが遅れると、次の指示を投げるまでの待ち時間が積み重なって、思ったよりテンポが落ちます。

# 解決策：notify機能でWindows通知を出す

Codex CLI には、タスク完了をフックにして任意のコマンドを実行できる `notify` 機能があります。

`~/.codex/config.toml` に以下のように設定すると、タスクの完了時に `main.sh` が実行されます。

```toml
notify = [
  "bash",
  "/home/your-user/.codex/my_notify/main.sh"
]
```

`/home/your-user/` の部分は、自分の WSL 上のホームディレクトリに置き換えてください。

この `main.sh` の中で Windows 側へ通知を飛ばせば、Codex CLI の完了を Windows の通知センターで拾えるようになります。

# notifyで渡される入力

`notify` で実行されるコマンドには Codex CLI から JSON 文字列が引数で渡されます。

たとえば、この記事で使っているスクリプトでは次のような値を参照しています。

```json
{
  "type": "agent-turn-complete",
  "cwd": "/home/your-user/work/example-project",
  "last-assistant-message": "実装とテストが終わりました。変更点を確認してください。"
}
```

このうち、最低限見ているのは以下の3つです。

- `type`: どのタイミングの通知かを判定するために使う
- `cwd`: どのディレクトリで動いていたかを通知タイトルに入れるために使う
- `last-assistant-message`: 通知本文に出すメッセージとして使う

# 通知スクリプト

`main.sh` は、Codex CLI から渡された JSON をパースし、Windows の通知センターにトースト通知を送るスクリプトです。

流れは次の通りです。

1. 引数として渡された JSON を読む
2. `type` や `last-assistant-message` を取り出す
3. WSL から Windows の `powershell.exe` を呼ぶ
4. `BurntToast` で Windows の通知センターにトースト通知を出す / `ntfy` に送る

ただし、`ntfy` は外部サービスを経由するため、後述の注意点を読んだうえで有効化してください。

## main.sh

```sh
#!/usr/bin/env bash
set -euo pipefail

DIR="/home/your-user/.codex/my_notify"

CONFIG="$DIR/config.sh"
if [[ -f "$CONFIG" ]]; then
  # shellcheck disable=SC1090
  source "$CONFIG"
fi

raw="${1:-}"
[[ -z "$raw" ]] && exit 0

json="$(printf '%s' "$raw" | tr -d '\\')"

type="$(jq -r '.type // empty' <<<"$json")"
cwd="$(jq -r '.cwd // empty' <<<"$json")"
last_assistant_message="$(jq -r '."last-assistant-message" // empty' <<<"$json")"

max_len="${MAX_BODY_LENGTH:-240}"
body="$(printf '%s' "$last_assistant_message" | tr '\r\n' ' ' | head -c "$max_len")"
cwd_basename="$(basename "$cwd")"
title="Codex ($cwd_basename)"

route() {
  [[ "${ENABLE_WINDOWS_TOAST:-1}" == "1" ]] && send_windows_toast "$title" "$body" || true
  [[ "${ENABLE_NTFY:-0}" == "1" ]] && send_ntfy "$title" "$body" || true
}

send_windows_toast() {
  local title="$1"
  local body="$2"

  read -r -d '' ps_script <<-EOF || true
		\$ErrorActionPreference = 'Stop'
		Import-Module BurntToast -ErrorAction Stop

		\$title = @"
		$title
		"@

		\$body = @"
    $body
		"@

		New-BurntToastNotification -Text @(\$title, \$body) | Out-Null
	EOF

  ps_script="${ps_script/__BODY__/$body}"

  local encoded
  encoded="$(
    printf '%s' "$ps_script" \
      | iconv -f UTF-8 -t UTF-16LE \
      | base64 -w 0
  )"

  local powershell_exe="powershell.exe"
  command -v pwsh.exe && ps_exe="pwsh.exe"

  "$powershell_exe" -NoProfile -ExecutionPolicy Bypass -EncodedCommand "$encoded"
}

send_ntfy() {
  local title="$1"
  local body="$2"

  curl \
    -H "Title: $title" \
    -d "$body" \
    "$NTFY_URL" \
    > /dev/null 2>&1 || true
}

if [[ "$type" == "agent-turn-complete" ]]; then
  route
fi

exit 0
```

`DIR` は自分の環境に合わせて変更してください。

## config.sh

```sh
#!/usr/bin/env bash

# ===== Channel switches =====
ENABLE_WINDOWS_TOAST=1
ENABLE_NTFY=1

# ===== ntfy settings =====
NTFY_URL="https://ntfy.sh/your-private-topic"

# ===== Other settings =====
MAX_BODY_LENGTH=240
```

`ENABLE_NTFY=0` のままなら、Windows 通知だけが有効です。まずはこの状態で動かすのが分かりやすいと思います。

`NTFY_URL` を設定する場合は、自分専用のトピック名に置き換えてください。記事中のようなサンプル値をそのまま使わない方が安全です。

# ntfyを使うときの注意点

`ntfy` は便利ですが、Windows の通知センターに閉じておらず、通知内容が外部サービスを通る構成になります。

特に公開トピックを使う場合は、次の点に注意した方がよいです。

- 通知本文に機密情報や顧客名、具体的な作業内容を入れすぎない
- 推測されやすいトピック名を避ける
- 必要なければ `ENABLE_NTFY=0` のままにする

# 開発用スクリプト

通知スクリプトの開発を楽にするために、入力確認用と動作確認用の補助スクリプトも用意しました。

`notify` にどんな JSON が渡ってくるのかを確認したり、通知処理だけを繰り返し試したりできるので、試行錯誤がかなり楽になります。

## sample.sh

`sample.sh` は、Codex CLI から渡される引数や環境変数を `sample.log` に記録するスクリプトです。

サンプル取得時は `~/.codex/config.toml` の `notify` を一時的に `sample.sh` に差し替えます。そうすると、タスク完了のたびに notify の入力内容を確認できます。

```sh
#!/usr/bin/env bash
set -u

DIR="/home/your-user/.codex/my_notify"
LOG="$DIR/sample.log"
mkdir -p "$DIR"

{
  echo "===== $(date -Is) ====="
  echo "PWD: $(pwd)"
  echo "ARGV($#):"
  i=0
  for a in "$@"; do
    printf '  [%d]=%q\n' "$i" "$a"
    i=$((i+1))
  done

  echo "--- ENV ---"
  env | LC_ALL=C sort

  echo "--- STDIN ---"
  cat || true
  echo
} >> "$LOG" 2>&1

exit 0
```

## mock.sh

`mock.sh` は、`sample.log` に保存した最新の引数を取り出して `main.sh` に渡し、通知スクリプトだけを単体でテストするためのスクリプトです。

```sh
#!/usr/bin/env bash
set -euo pipefail

DIR="/home/your-user/.codex/my_notify"
LOG="$DIR/sample.log"
NOTIFY="$DIR/main.sh"

# Ensure required files exist
if [[ ! -f "$LOG" ]]; then
  echo "sample.log not found: $LOG" >&2
  exit 1
fi

if [[ ! -x "$NOTIFY" ]]; then
  echo "notify.sh not found or not executable: $NOTIFY" >&2
  exit 1
fi

# Extract the last ARGV[0] line from the blocks that start with "ARGV(1):"
raw="$(
  awk '
    /^ARGV\(1\):/ { capture=1; next }
    capture && /^[[:space:]]+\[0\]=/ {
      sub(/^[[:space:]]+\[0\]=/, "")
      print
      capture=0
    }
  ' "$LOG" | tail -n 1
)"

# Fail if nothing was extracted
if [[ -z "${raw:-}" ]]; then
  echo "No ARGV[0] entry found in $LOG" >&2
  exit 1
fi

# Call notify.sh with the exact same single argument
exec "$NOTIFY" "$raw"
```

# おわりに

Codex CLI の `notify` を使うと、WSL で動かしていても Windows の通知センターに自然につなげられます。

これで、作業完了に気づくまでの無駄な待ち時間がかなり減りました。

みなさんもぜひ試してみてください。
