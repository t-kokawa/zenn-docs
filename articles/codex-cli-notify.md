---
title: "Codex CLIの完了通知をWindowsの通知センターに送る"
emoji: "🔔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Codex", "AI駆動開発", "AIエージェント", "AI", "開発生産性"]
published: false
---
私は株式会社Digeonでインターンをしている大学生です。

この記事は、Codex CLIの完了通知をWindowsの通知センターに送るようにした事について書いています。

# 前提

WSLでCodex CLIを使用しています。

Windows側ではトースト通知のためにPowerShellから`BurntToast`モジュールを使えるようにしてあります。

また、通知スクリプト内では`jq`や`curl`などのコマンドを使用しています。

# 課題：Codex CLIの作業完了に気づかない

Codex CLIは、タスクの実行が完了すると、ターミナルに完了のメッセージを表示します。

しかし、ターミナルを見ていないと、完了のメッセージに気づかないことがあります。

タスクの実行が完了しているのに、ターミナルを見ていないために、次のタスクを投げるのが遅れてしまうと、作業効率が下がってしまいます。

# 解決策：notify機能

Codex CLIには、タスクの完了をフックにして、任意のコマンドを実行できるnotify機能があります。

`~/.codex/config.toml`に以下のように設定すると、

```toml
notify = [
  "bash",
  "/home/t-kokawa/.codex/my_notify/main.sh"
]
```

タスクの完了時に`main.sh`が実行されるようになります。

`main.sh`に通知を送る処理を書いておくと、タスクの完了を通知することができます。

# 通知スクリプト

`main.sh`は、Codex CLIから渡されたJSON形式の引数をパースして、Windowsの通知センターに通知を送るスクリプトです。

WSL上で動くシェルスクリプトからWindows側のPowerShellを呼び出して、Windowsの通知センターにトースト通知を送っています。

また、ntfyを使用して、スマホにも通知を送るようにしています。

ただし、ntfyに送った通知内容は外部サービスを経由します。公開トピックを使う場合は、通知本文に機密情報や詳細な作業内容を含めすぎないよう注意してください。

## main.sh

```sh
#!/usr/bin/env bash
set -euo pipefail

DIR="/home/t-kokawa/.codex/my_notify"

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

## config.sh

```sh
#!/usr/bin/env bash

# ===== Channel switches =====
ENABLE_WINDOWS_TOAST=1
ENABLE_NTFY=1

# ===== ntfy settings =====
NTFY_URL="https://ntfy.sh/codex-notify-kokawa"

# ===== Other settings =====
MAX_BODY_LENGTH=240
```

# 開発用スクリプト

通知スクリプトの開発を効率的に行えるようにするスクリプトを作成して、開発を進めました。

以下の`sample.sh`と`mock.sh`は、notifyに渡される入力を確認したり、通知スクリプトの動作確認をしたりするための補助スクリプトです。通知を使うだけであれば必須ではありませんが、開発中の試行錯誤がかなり楽になります。

## sample.sh

`sample.sh`は、Codex CLIから渡されるJSON形式の引数のサンプルを`sample.log`に出力するスクリプトです。

サンプル取得時は`~/.codex/config.toml`のnotifyの設定を`sample.sh`が呼び出されるように変更します。

そうすることで、タスクの実行が完了するたびに、JSON形式の引数のサンプルが`sample.log`に出力されるようになります。

```sh
#!/usr/bin/env bash
set -u

DIR="/home/t-kokawa/.codex/my_notify"
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

`mock.sh`は、`sample.log`に出力されたJSON形式の引数のうち最新のものを`main.sh`に渡して、通知スクリプトの動作をテストするスクリプトです。

```sh
#!/usr/bin/env bash
set -euo pipefail

DIR="/home/t-kokawa/.codex/my_notify"
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

これで、効率的にCodex CLIをまわすことができるようになりました。

みなさんも、Codex CLIのnotify機能を使って、タスクの完了を通知してみてはいかがでしょうか。
