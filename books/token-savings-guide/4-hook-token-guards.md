---
title: "第4章 Hookでトークン浪費を自動カット"
---

CLAUDE.mdに「無駄なファイルを読むな」と書いても、Claude Codeは忘れることがある。hookは忘れない。

hookはClaude Codeの各操作の前後に自動実行されるシェルスクリプトだ。ルールを「お願い」から「強制」に変える。この章では、トークン節約に直結する9個のhookを紹介する。

## Hookの基本

hookは`settings.json`に設定する。場所は`~/.claude/settings.json`（グローバル）またはプロジェクトの`.claude/settings.json`。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{"type": "command", "command": "/path/to/your-hook.sh"}]
      }
    ]
  }
}
```

- **PreToolUse**: ツール実行の前に走る。exit 2を返すとツール実行をブロック
- **PostToolUse**: ツール実行の後に走る。結果の監視やログ記録に使う
- **PreCompact**: `/compact`の前に走る。文脈の保存に使う
- **SessionStart**: セッション開始時に走る。初期状態の設定に使う

hookのstdoutに出力した内容はClaude Codeのコンテキストに注入される。つまり「このファイルは大きすぎます。`limit`パラメータを使ってください」というメッセージをhookから出力すれば、Claude Codeはそれに従う。

本章のhookでは`$PPID`（親プロセスID）をセッション識別子として使う。Claude Codeがhookを起動する際、hookのbashプロセスから見た`$PPID`はClaude CodeのプロセスIDになるため、同じセッション中のhook呼び出しを追跡できる。

## Hook 1: 大きなファイルの読み込みガード

最もトークンを食うのは、大きなファイルの丸読みだ。

```bash
#!/bin/bash
# large-read-guard.sh — 100KB超のファイル読み込みを警告

# hookのstdinはJSON形式。tool_input内にツールの引数がある
input=$(cat)
file_path=$(echo "$input" | jq -r '.tool_input.file_path // empty')

if [ -n "$file_path" ] && [ -f "$file_path" ]; then
  size=$(stat -c%s "$file_path" 2>/dev/null || stat -f%z "$file_path" 2>/dev/null)
  threshold=${CC_LARGE_READ_KB:-100}
  threshold_bytes=$((threshold * 1024))

  if [ "$size" -gt "$threshold_bytes" ] 2>/dev/null; then
    size_kb=$((size / 1024))
    echo "⚠️ ${file_path} は ${size_kb}KB あります。"
    echo "limitパラメータで必要な部分だけ読んでください。"
    echo "例: Read file_path=${file_path} limit=100"
    exit 0  # 警告のみ（stdoutの内容がClaude Codeに伝わる）
  fi
fi
```

設定:

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Read",
      "hooks": [{"type": "command", "command": "bash /path/to/large-read-guard.sh"}]
    }]
  }
}
```

**効果**: 3,000行のファイルを丸読みする代わりに、必要な100行だけ読むようになる。1回のReadで数千トークンの節約。

:::message
以降のhookも同じ形式で`settings.json`に設定する。イベント名とmatcherは本章末尾の早見表を参照。Hook 8・9には設定例を掲載している。
:::

## Hook 2: 読み込みファイル数の上限

1セッションで100ファイル以上読むのは、ほぼ確実に非効率だ。

```bash
#!/bin/bash
# read-budget-guard.sh — セッション中の読み込みファイル数を制限

budget=${CC_READ_BUDGET:-100}
warn=${CC_READ_WARN:-50}
tracker="/tmp/cc-read-budget-$PPID"

input=$(cat)
file_path=$(echo "$input" | jq -r '.tool_input.file_path // empty')

# カウント
count=0
if [ -f "$tracker" ]; then
  if ! grep -qF "$file_path" "$tracker"; then
    echo "$file_path" >> "$tracker"
  fi
  count=$(wc -l < "$tracker")
else
  echo "$file_path" > "$tracker"
  count=1
fi

if [ "$count" -ge "$budget" ]; then
  echo "🛑 読み込みファイル数が${budget}件に達しました。"
  echo "Glob/Grepで絞り込んでから読んでください。"
  exit 2  # exit 2 = ブロック
elif [ "$count" -ge "$warn" ]; then
  echo "⚠️ ${count}/${budget}ファイル読み込み済み。残り$((budget - count))件。"
fi
```

**効果**: 「念のため全部読む」パターンを防止。GrepやGlobで必要なファイルを特定してから読む習慣がつく。

## Hook 3: トークン予算ガード

セッション全体のトークン消費を推定し、予算を超えたら警告する。

```bash
#!/bin/bash
# token-budget-guard.sh — セッションのトークン予算を管理

budget_warn=${CC_TOKEN_BUDGET:-50000}
budget_block=${CC_TOKEN_BLOCK:-100000}
tracker="/tmp/cc-token-budget-$PPID"

# ツール出力のサイズからトークン数を推定（約4文字=1トークン）
input=$(cat)
output_size=${#input}
tokens=$((output_size / 4))

total=0
if [ -f "$tracker" ]; then
  total=$(cat "$tracker")
fi
total=$((total + tokens))
echo "$total" > "$tracker"

if [ "$total" -ge "$budget_block" ]; then
  echo "🛑 推定トークン消費が${budget_block}を超えました。/compactを実行してください。"
  exit 2  # exit 2 = ブロック
elif [ "$total" -ge "$budget_warn" ]; then
  echo "⚠️ 推定トークン消費: ${total}/${budget_block}"
fi
```

**効果**: 「気づいたらコンテキストが90%」を防止。早めの/compactを促す。

## Hook 4: ツール呼び出し頻度の制限

30秒間に15回以上のツール呼び出しは、暴走のサインだ。

```bash
#!/bin/bash
# token-spike-alert.sh — ツール呼び出しの異常頻度を検出

tracker="/tmp/cc-spike-$PPID"
window=30  # 秒
threshold=15

now=$(date +%s)
echo "$now" >> "$tracker"

# window秒以内のエントリだけ残す
cutoff=$((now - window))
if [ -f "$tracker" ]; then
  awk -v cutoff="$cutoff" '$1 >= cutoff' "$tracker" > "${tracker}.tmp"
  mv "${tracker}.tmp" "$tracker"
  count=$(wc -l < "$tracker")

  if [ "$count" -ge "$threshold" ]; then
    echo "⚠️ ${window}秒間に${count}回のツール呼び出し。暴走の可能性があります。"
    echo "意図的でなければ Ctrl+C で停止してください。"
  fi
fi
```

**効果**: ループ状態（同じファイルを何度も読み直す、同じコマンドを繰り返す）の早期検出。

## Hook 5: サブエージェント数の制限

サブエージェントは便利だが、1つ起動するたびに独立したコンテキストが生まれる。

```bash
#!/bin/bash
# subagent-budget-guard.sh — 同時サブエージェント数を制限

max=${CC_MAX_SUBAGENTS:-5}
tracker="/tmp/cc-subagent-$PPID"

now=$(date +%s)
echo "$now" >> "$tracker"

# 直近5分以内に起動されたエージェントをカウント
cutoff=$((now - 300))
if [ -f "$tracker" ]; then
  count=$(awk -v cutoff="$cutoff" '$1 >= cutoff' "$tracker" | wc -l)

  if [ "$count" -gt "$max" ]; then
    echo "🛑 5分間にサブエージェント${count}個が起動されています（上限: ${max}）。"
    echo "メインのコンテキストでGlob/Grepを使ってください。"
    exit 2  # exit 2 = ブロック
  fi
fi
```

**効果**: サブエージェントの乱用防止。単純な検索にサブエージェントを使うパターンを抑制。

## Hook 6: Bashの出力サイズ制限

`ls -la`の出力は数十行で済むが、`git log`は数千行になることがある。

```bash
#!/bin/bash
# bash-output-guard.sh — Bash出力が大きすぎる場合に警告

input=$(cat)
command=$(echo "$input" | jq -r '.tool_input.command // empty')

# 出力が大きくなりがちなコマンドを検出
dangerous_patterns="git log|find / |cat |strings |xxd |hexdump"
if echo "$command" | grep -qE "$dangerous_patterns"; then
  if ! echo "$command" | grep -qE "\| head|\| tail|\| grep|--max-count|-n [0-9]"; then
    echo "⚠️ このコマンドの出力が大きくなる可能性があります。"
    echo "| head -50 や --max-count を付けてください。"
  fi
fi
```

**効果**: 無制限のコマンド出力によるコンテキスト圧迫を防止。

## Hook 7: 重複ファイル読み込みの検出

同じファイルを2回読むのは、ほぼ常に無駄だ。

```bash
#!/bin/bash
# duplicate-read-detector.sh — 同一ファイルの再読み込みを警告

tracker="/tmp/cc-read-history-$PPID"
input=$(cat)
file_path=$(echo "$input" | jq -r '.tool_input.file_path // empty')

if [ -n "$file_path" ] && [ -f "$tracker" ]; then
  if grep -qF "$file_path" "$tracker"; then
    echo "⚠️ ${file_path} は既に読み込み済みです。"
    echo "内容はコンテキストにあります。再読み込みは不要かもしれません。"
    # ブロックはしない（更新されている可能性がある）
  fi
fi

echo "$file_path" >> "$tracker"
```

**効果**: 「さっき読んだファイルをもう一度読む」パターンの可視化。

## Hook 8: /compact前の自動チェックポイント

`/compact`の前に、作業状態を自動保存する。

```bash
#!/bin/bash
# pre-compact-checkpoint.sh — /compact前にgit commit

cd "$(git rev-parse --show-toplevel 2>/dev/null)" || exit 0

if [ -n "$(git status --porcelain)" ]; then
  git add -A
  # --no-verify: チェックポイント専用のため、pre-commit hookをスキップ
  git commit -m "checkpoint: pre-compact $(date +%Y%m%d-%H%M%S)" --no-verify 2>/dev/null
  echo "✅ 圧縮前チェックポイントを保存しました。"
fi

echo "📋 コンテキストが圧縮されます。"
echo "圧縮後は前の会話の文脈が失われます。圧縮前の作業を自動で再開せず、ユーザーの指示を確認してください。"
```

設定:

```json
{
  "hooks": {
    "PreCompact": [{
      "matcher": "",
      "hooks": [{"type": "command", "command": "bash /path/to/pre-compact-checkpoint.sh"}]
    }]
  }
}
```

**効果**: 圧縮後に「何を変更していたか」をgit logで確認できる。

## Hook 9: セッション開始時の予算表示

セッション開始時に、今日の残りトークン予算を表示する。

```bash
#!/bin/bash
# session-budget-alert.sh — セッション開始時に予算状況を表示

state_file="/tmp/cc-daily-budget"
today=$(date +%Y-%m-%d)

if [ -f "$state_file" ]; then
  saved_date=$(head -1 "$state_file")
  if [ "$saved_date" = "$today" ]; then
    sessions=$(tail -1 "$state_file")
    echo "📊 本日${sessions}セッション目。トークン消費に注意してください。"
    echo "$today" > "$state_file"
    echo "$((sessions + 1))" >> "$state_file"
    exit 0
  fi
fi

echo "$today" > "$state_file"
echo "1" >> "$state_file"
echo "📊 本日1セッション目。良いスタートを。"
```

設定:

```json
{
  "hooks": {
    "SessionStart": [{
      "matcher": "",
      "hooks": [{"type": "command", "command": "bash /path/to/session-budget-alert.sh"}]
    }]
  }
}
```

## まとめ: 9個のhookの早見表

| Hook | イベント | 効果 |
|------|---------|------|
| 大きなファイルガード | PreToolUse (Read) | 100KB超の丸読みを警告 |
| 読み込み予算 | PreToolUse (Read) | セッション100ファイル上限 |
| トークン予算 | PostToolUse | 推定消費量で/compact促進 |
| スパイク検出 | PostToolUse | 30秒15回で暴走警告 |
| サブエージェント制限 | PreToolUse (Agent) | 5分5個まで |
| Bash出力制限 | PreToolUse (Bash) | 大出力コマンドに\|head推奨 |
| 重複読み込み検出 | PreToolUse (Read) | 同一ファイル再読み警告 |
| compact前チェックポイント | PreCompact | 自動git commit + 暴走防止メッセージ |
| セッション予算表示 | SessionStart | 日次セッション数表示 |

これらのhookを全部自分で書く必要はない。cc-safe-setupには650+のexample hookが用意されており、`--install-example`コマンドで個別にインストールできる。

:::message alert
**本章のhookはcc-safe-setupのexampleとは独立した実装です。** 同名のhookでも設計が異なります。例: `large-read-guard`は本章ではReadツールを対象（matcher: "Read"）、cc-safe-setupではBashツールのcatコマンドを対象。`token-budget-guard`は本章ではトークン数で管理、cc-safe-setupではドル換算。本章のhookをそのまま使う場合、cc-safe-setupのexampleとの混同に注意してください。
:::

次の章では、サブエージェントの戦略的な使い方を解説する。
