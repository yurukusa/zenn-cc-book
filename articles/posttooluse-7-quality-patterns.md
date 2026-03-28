---
title: PostToolUseフックで品質を自動チェックする——7つの実践パターン
type: tech
emoji: 🔍
topics: [claudecode, hooks, 品質管理, AI]
published: false
---

## はじめに

Claude Codeのhook（フック）は、AIが特定の操作をしたタイミングで自動的にスクリプトを実行する仕組みだ。中でも**PostToolUse**は、AIがツールを使い終わった直後に発火するフックで、品質チェックの自動化に最適なトリガーになる。

この記事では、PostToolUseフックを使った7つの実践パターンを紹介する。すべて`npx cc-safe-setup`で導入できるhook群（443個・5,890テストで検証済み）から抜粋したものだ。

## PostToolUseとは

Claude Codeには4つのフックタイミングがある。

| タイミング | 発火条件 |
|---|---|
| PreToolUse | ツール実行**前** |
| PostToolUse | ツール実行**後** |
| UserPromptSubmit | ユーザーがプロンプトを送信した時 |
| Stop | AIがレスポンスを完了した時 |

PostToolUseは「AIが何かをした直後」に走る。つまり、AIの出力に対するバリデーションを自動化できる。

フックの設定は`.claude/settings.json`に記述する。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "python -m py_compile \"$CLAUDE_FILE_PATH\"",
        "timeout": 10000
      }
    ]
  }
}
```

`matcher`でどのツールに反応するかを指定し、`command`で実行するスクリプトを書く。フックが失敗（非ゼロ終了）すると、AIにエラーが返され、修正を促す。

## パターン1: JSON構文チェック

AIが設定ファイルやpackage.jsonを編集した後、JSONとして正しいかを自動チェックする。

```bash
#!/bin/bash
# PostToolUse: Write|Edit で .json ファイルに反応
FILE="$CLAUDE_FILE_PATH"
if [[ "$FILE" == *.json ]]; then
  python3 -c "import json; json.load(open('$FILE'))" 2>&1
  if [ $? -ne 0 ]; then
    echo "ERROR: JSONの構文が壊れている: $FILE"
    exit 1
  fi
fi
```

**なぜ必要か:** AIはJSONのカンマ抜けやブラケットの閉じ忘れをよくやる。人間が気づくのは次にそのファイルを使おうとした時。hookがあれば、壊した瞬間に直させられる。

## パターン2: マークダウンリンク検証

記事ドラフトを編集した後、リンクの書式が正しいかをチェックする。

```bash
#!/bin/bash
# PostToolUse: Write|Edit で .md ファイルに反応
FILE="$CLAUDE_FILE_PATH"
if [[ "$FILE" == *.md ]]; then
  # [text](url) の形式で url が空のリンクを検出
  BROKEN=$(grep -n '\[.*\]()' "$FILE" 2>/dev/null)
  if [ -n "$BROKEN" ]; then
    echo "ERROR: 空のリンクを検出:"
    echo "$BROKEN"
    exit 1
  fi
  # [text]( ) のようにスペースだけのリンクも検出
  SPACE_ONLY=$(grep -n '\[.*\](\s\+)' "$FILE" 2>/dev/null)
  if [ -n "$SPACE_ONLY" ]; then
    echo "ERROR: URLがスペースのみのリンクを検出:"
    echo "$SPACE_ONLY"
    exit 1
  fi
fi
```

**なぜ必要か:** AIは「リンクは後で埋める」つもりで空リンクを残すことがある。そのまま公開されると読者体験を損なう。

## パターン3: 連続エラー検出

AIが同じファイルで何度もエラーを出している場合、根本的なアプローチの変更を促す。

```bash
#!/bin/bash
# PostToolUse: Bash に反応
ERROR_LOG="/tmp/claude-error-count"
EXIT_CODE="$(echo "$INPUT" | jq -r '.tool_result.exit_code // 0')"

if [ "$EXIT_CODE" -ne 0 ]; then
  COUNT=$(cat "$ERROR_LOG" 2>/dev/null || echo 0)
  COUNT=$((COUNT + 1))
  echo "$COUNT" > "$ERROR_LOG"
  if [ "$COUNT" -ge 3 ]; then
    echo "WARNING: 連続${COUNT}回のエラー。アプローチを見直してください。"
    echo "0" > "$ERROR_LOG"
    exit 1
  fi
else
  echo "0" > "$ERROR_LOG"
fi
```

**なぜ必要か:** AIは同じ失敗を繰り返す傾向がある。3回失敗したら「やり方を変えろ」と伝えることで、無限ループを防ぐ。

## パターン4: 日次使用量追跡

AIがツールを使うたびに使用回数をカウントし、1日の使用量を可視化する。

```bash
#!/bin/bash
# PostToolUse: 全ツールに反応（matcher: "*"）
TODAY=$(date +%Y-%m-%d)
LOG_FILE="/tmp/claude-usage-${TODAY}.log"

TOOL_NAME="$CLAUDE_TOOL_NAME"
echo "$(date +%H:%M:%S) $TOOL_NAME" >> "$LOG_FILE"

COUNT=$(wc -l < "$LOG_FILE")
if [ "$COUNT" -ge 500 ]; then
  echo "INFO: 本日のツール使用回数が${COUNT}回を超えました"
fi
```

**なぜ必要か:** AIの作業量を把握するため。1日500回以上のツール使用は、何かが非効率な可能性を示唆する。

## パターン5: コミットメッセージ品質

`git commit`を実行した後、コミットメッセージが最低限の品質基準を満たしているか確認する。

```bash
#!/bin/bash
# PostToolUse: Bash で git commit を含むコマンドに反応
COMMAND="$CLAUDE_COMMAND"
if [[ "$COMMAND" == *"git commit"* ]]; then
  LAST_MSG=$(git log -1 --pretty=%B 2>/dev/null)
  MSG_LEN=${#LAST_MSG}
  if [ "$MSG_LEN" -lt 10 ]; then
    echo "ERROR: コミットメッセージが短すぎる（${MSG_LEN}文字）。最低10文字。"
    exit 1
  fi
  if [[ "$LAST_MSG" == "fix" || "$LAST_MSG" == "update" || "$LAST_MSG" == "wip" ]]; then
    echo "ERROR: 意味のないコミットメッセージ: '$LAST_MSG'"
    exit 1
  fi
fi
```

**なぜ必要か:** AIは「fix」「update」のような無意味なコミットメッセージを書くことがある。後から履歴を追うとき、何が起きたかわからなくなる。

## パターン6: 出力のクレデンシャルスキャン

AIが書いたファイルに認証情報やAPIキーが含まれていないかスキャンする。

```bash
#!/bin/bash
# PostToolUse: Write|Edit に反応
FILE="$CLAUDE_FILE_PATH"
if [ -f "$FILE" ]; then
  # 一般的なシークレットパターンを検出
  SECRETS=$(grep -inE '(api[_-]?key|secret[_-]?key|password|token)\s*[:=]\s*["\x27][A-Za-z0-9]' "$FILE" 2>/dev/null | grep -v '\.env\.example' | grep -v '#.*example')
  if [ -n "$SECRETS" ]; then
    echo "ERROR: 認証情報の可能性がある文字列を検出:"
    echo "$SECRETS"
    echo "環境変数に移してください。"
    exit 1
  fi
fi
```

**なぜ必要か:** AIはサンプルコードにダミーのAPIキーを書くつもりで、実際のキーを埋め込むことがある。一度GitHubにpushされたキーは、履歴から完全に消すのが極めて困難だ。

## パターン7: テスト結果検証

テストコマンドの実行後、失敗したテストがあれば即座にフィードバックする。

```bash
#!/bin/bash
# PostToolUse: Bash で test/pytest/jest を含むコマンドに反応
COMMAND="$CLAUDE_COMMAND"
EXIT_CODE="$(echo "$INPUT" | jq -r '.tool_result.exit_code // 0')"

if [[ "$COMMAND" == *"test"* || "$COMMAND" == *"pytest"* || "$COMMAND" == *"jest"* ]]; then
  if [ "$EXIT_CODE" -ne 0 ]; then
    echo "ERROR: テストが失敗しました。失敗したテストを修正してから次に進んでください。"
    echo "全テストが通るまでコミットしないでください。"
    exit 1
  fi
fi
```

**なぜ必要か:** AIは「テスト失敗」を無視して次のタスクに進むことがある。このhookがあれば、テストが通るまで次に進めない。

## すべてまとめて導入する

7つのパターンを個別に設定する必要はない。

```bash
npx cc-safe-setup
```

これだけで、443個のhookが一括で導入される。上記の7パターンはその一部だ。すべて5,890のテストで動作が検証されている。

## hookを書く際の注意点

1. **タイムアウトを設定する。** hookが無限ループすると、AIの作業が止まる。10秒以内に終わるように設計する
2. **exit 1で失敗を伝える。** hookが`exit 0`で終わると「問題なし」とみなされる。問題がある場合は必ず`exit 1`で終了する
3. **stderr/stdoutにメッセージを出す。** AIはフックの出力を読んで対応を判断する。「何が悪いか」「どう直すか」を明示的に書く
4. **matcherは絞る。** `"*"`（全ツール）で反応させると、すべての操作にオーバーヘッドがかかる。必要なツールだけに反応させる

## おわりに

PostToolUseフックは「AIの仕事を信頼するが、検証する」ための仕組みだ。信頼しないのではなく、ミスが起きたときに即座に気づける環境を作る。

hookの設計と実践例をさらに詳しく知りたい方は、以下の記事を参照してほしい。

https://qiita.com/yurukusa/items/fb73a4122b583197071b

Zenn Bookでは、Claude Codeの環境設計を体系的に解説している。

https://zenn.dev/yurukusa/books/6076c23b1cb18b
