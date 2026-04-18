---
title: "Claude Code Stopフック活用ガイド——セッション終了時に自動実行する仕組み"
emoji: "🛑"
type: "tech"
topics: ["claudecode", "ai", "hooks", "自動化"]
published: false
---

セッション終了後、`/tmp`に500個のファイルが残っていた。手動で消すのは論外。

Claude Codeにはフック機構がある。コマンド実行前後やセッション終了時に、任意のスクリプトを自動で走らせる仕組みだ。このうち**Stopフック**を使えば、「終わり方」を自動化できる。

## Stopフックとは

Claude Codeが終了するタイミングで発火するフック。具体的には以下。

- `/stop`コマンドの実行
- 応答完了後の自然終了
- `Ctrl+C`での中断

セッション終了の「直前」に走る。後片付け、記録、警告を差し込める。

## exit codeの意味

Stopフックのexit codeは2種類。

| exit code | 意味 | 動作 |
|-----------|------|------|
| `0` | 情報提供 | stdoutの内容を表示し、終了を続行 |
| `2` | 終了ブロック | stdoutの内容を表示し、終了を阻止 |

`exit 0`はログ出力や通知に使う。終了は止めない。
`exit 2`は「本当に終了していいのか？」の確認。未保存の変更がある場合など、終了を防ぎたい場面で有効。

## 実用例①: 一時ファイルの自動削除

冒頭の問題を解決する。セッション中に生成された一時ファイルを終了時に自動で消す。

```bash:~/.claude/hooks/temp-file-cleanup.sh
#!/bin/bash
# Claude Codeが生成した一時ファイルを削除
COUNT=$(find /tmp -name "claude-*" -user "$USER" 2>/dev/null | wc -l)
if [ "$COUNT" -gt 0 ]; then
    find /tmp -name "claude-*" -user "$USER" -delete 2>/dev/null
    echo "一時ファイル ${COUNT}件を削除した"
fi
exit 0
```

`exit 0`なので終了はブロックしない。削除結果だけ表示して終わる。

## 実用例②: セッション成果の自動記録

セッションの成果を終了時に自動記録する。

```bash:~/.claude/hooks/session-summary.sh
#!/bin/bash
LOG_DIR="$HOME/ops/proof-log"
TODAY=$(date +%Y-%m-%d)
LOG_FILE="$LOG_DIR/$TODAY.md"
mkdir -p "$LOG_DIR"
# 直近のgitコミットを記録
COMMITS=$(git log --oneline --since="8 hours ago" 2>/dev/null | head -10)
if [ -n "$COMMITS" ]; then
    {
        echo ""
        echo "## Session $(date +%H:%M)"
        echo "$COMMITS"
    } >> "$LOG_FILE"
    echo "セッション記録を $LOG_FILE に追記した"
fi
exit 0
```

proof-logに追記する形式。日次の事実ログになる。

## 実用例③: 未コミット変更の警告

作業途中の終了を警告し、ブロックする。

```bash:~/.claude/hooks/uncommitted-changes-alert.sh
#!/bin/bash

# gitリポジトリ外なら何もしない
git rev-parse --git-dir > /dev/null 2>&1 || exit 0

CHANGES=$(git status --porcelain 2>/dev/null)

if [ -n "$CHANGES" ]; then
    FILE_COUNT=$(echo "$CHANGES" | wc -l)
    echo "未コミットの変更が ${FILE_COUNT}件ある"
    echo ""
    echo "$CHANGES" | head -10
    [ "$FILE_COUNT" -gt 10 ] && echo "...他 $((FILE_COUNT - 10))件"
    echo ""
    echo "コミットしてから終了することを推奨する"
    exit 2  # 終了をブロック
fi

exit 0
```

**`exit 2`で終了をブロック**する。未コミット変更がなければ`exit 0`で素通り。

## 実用例④: APIエラー検知アラート

APIエラーによる異常終了を検知して通知する。

```bash:~/.claude/hooks/api-error-alert.sh
#!/bin/bash

# 直近のログからAPIエラーを検出
LOG_DIR="$HOME/.claude/logs"
LATEST_LOG=$(ls -t "$LOG_DIR"/*.log 2>/dev/null | head -1)

[ -z "$LATEST_LOG" ] && exit 0

if grep -q "API.*error\|rate.*limit\|503\|529" "$LATEST_LOG" 2>/dev/null; then
    echo "APIエラーを検出した"
    grep "API.*error\|rate.*limit\|503\|529" "$LATEST_LOG" | tail -3
    echo ""
    echo "レート制限またはサーバーエラーの可能性がある"
fi

exit 0
```

通知だけなので`exit 0`。エラーの事実を伝え、終了は許可する。

## settings.jsonへの設定方法

`~/.claude/settings.json`に登録する。

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/temp-file-cleanup.sh"
          }
        ]
      },
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/uncommitted-changes-alert.sh"
          }
        ]
      }
    ]
  }
}
```

ポイントは3つ。

1. `matcher`は空文字列。Stopフックにマッチ対象はない
2. 複数フックを配列で並べられる。上から順に実行
3. プロジェクト固有の設定は`.claude/settings.json`（プロジェクトルート）に書く

## 注意点

Stopフックは素早く終了すべきだ。ユーザーは「終了」を押している。数秒以上待たされると体験が悪い。重い処理はバックグラウンドで。

```bash
# 重い処理はバックグラウンドに逃がす
heavy_task &
echo "バックグラウンドで処理を開始した"
exit 0
```

タイムアウトの設定も推奨する。

```json
{
  "type": "command",
  "command": "timeout 5 bash ~/.claude/hooks/session-summary.sh"
}
```

5秒で強制終了。フックのハングを防ぐ。

## まとめ

Stopフックで「終わり方」を制御する。一時ファイルの掃除、成果の記録、未コミット警告。毎回やっていた手作業が自動化される。

**終わり方を整えると、次の開始がスムーズになる。**

---

**第2章「Safety Guards」まで無料公開中。**
🛡 `npx cc-safe-setup` — 8つの安全フックを10秒でインストール
---
:::message
**Claude Codeの自律運用をもっと深く学びたい方へ**
トークン消費の突然の急増、ファイル消失、無限ループ——700+時間の自律稼働で起きた失敗と対策をまとめました:
[Claude Codeを本番品質にする——実践ガイド](https://zenn.dev/yurukusa/books/6076c23b1cb18b)（¥800・第3章まで無料）
:::
:::message
**トークン消費が気になる方へ**
[Token Checkup（無料）](https://yurukusa.github.io/cc-safe-setup/token-checkup.html)で30秒診断。どこでトークンが消えているか可視化できる。
もっと深く知りたい方は → [Claude Codeのトークン消費を半分にする（¥2,500）](https://zenn.dev/yurukusa/books/token-savings-guide)
:::
