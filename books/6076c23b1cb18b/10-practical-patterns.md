---
title: "実践：10の「やらかし」パターン——トークン暴走からファイル消失まで"
---

実際に起きた事故パターンと、それを1行のhookで防ぐ方法。

## パターン1：深夜の自律セッションが暴走

**何が起きたか：** 自律モードで放置していたClaude Codeが深夜3時にmainブランチにforce-push。朝来たらCIが全壊していた。

**防御：**
```bash
npx cc-safe-setup --install-example branch-guard
npx cc-safe-setup --install-example work-hours-guard
```

`branch-guard`はmainへのpushとforce-pushをブロック。`work-hours-guard`は業務時間外のリスク操作を制限。

## パターン2：テストを消して「修正完了」

**何が起きたか：** テストが失敗している。Claudeは「テストを修正します」と言い、テストの中身を削除して「テスト通りました」と報告。テストは確かに通る。テストがないから。

**防御：**
```bash
npx cc-safe-setup --install-example test-deletion-guard
```

テストファイルの編集で`it(`や`test(`の数が減ったら警告。

## パターン3：同じコマンドを10回リトライ

**何が起きたか：** `npm install broken-package`が失敗。Claudeは同じコマンドを10回繰り返す。毎回失敗。

**防御：**
```bash
npx cc-safe-setup --install-example error-memory-guard
```

同じコマンドが3回失敗したらブロック。「別のアプローチを試せ」と表示。

## パターン4：ドキュメントのハルシネーション

**何が起きたか：** README.mdに「`processAuth()`関数はJWTトークンを受け取る」と書いた。`processAuth()`は存在しない。Claudeはソースコードを読まずにドキュメントを書いた。

**防御：**
```bash
npx cc-safe-setup --install-example fact-check-gate
```

ドキュメント編集時にソースファイルへの参照を検出し、読んでいないファイルを参照している場合に警告。

## パターン5：catで巨大ファイルを読んでコンテキスト爆発

**何が起きたか：** `cat production.log`を実行。200MBのログファイルがコンテキストに流れ込み、それまでの作業内容が全て押し出された。

**防御：**
```bash
npx cc-safe-setup --install-example large-read-guard
npx cc-safe-setup --install-example context-monitor
```

`large-read-guard`は100KB超のファイルをcatする前に警告。`context-monitor`はコンテキスト残量を常時監視。

## パターン6：.envファイルをコミットしてAPIキーが漏洩

**何が起きたか：** Claude Codeが`git add .`を実行し、`.env`ファイルがコミットに含まれた。GitHub Issue [#2142](https://github.com/anthropics/claude-code/issues/2142)で報告された事例では、CLAUDE.mdのセキュリティ指示を無視してAPIキーがバージョン管理に露出した。自分の環境ではGitHubのpush protectionがダミーキーを検出して拒否してくれたが、本番のキーだったら漏洩していた。

**防御：**
```bash
npx cc-safe-setup  # secret-guardが含まれる
```

`secret-guard`は`git add .env`、`git add credentials.json`、`.pem`/`.key`ファイルの追加を検出してブロック。`git add .`や`git add -A`の場合も、ワーキングディレクトリに`.env`が存在すればブロックする。

## パターン7：構文エラーが30ファイルに連鎖

**何が起きたか：** Claude Codeがファイルを編集した際に構文エラーを入れ、そのまま次のファイルに進んだ。GitHubのIssueで報告されたケースでは、30ファイル以上に構文エラーが波及していた。syntax-checkを入れる前の自分の環境でも、PythonとJSONの構文エラーが数ファイルに残ったまま次のタスクに進んでいた。

**防御：**
```bash
npx cc-safe-setup  # syntax-checkが含まれる
```

PostToolUseフックとして、Edit/Write実行後に自動で構文チェックを走らせる。Python、Shell、JSON、YAML、JavaScriptに対応。エラーが見つかったらClaudeに通知され、即座に修正を試みる。

## パターン8：出力トークン上限でセッションが死ぬ

**何が起きたか：** 複雑なタスクを実行中、突然「Claude's response exceeded the 32000 output token maximum」エラーが出てセッションが中断。GitHub Issue [#24055](https://github.com/anthropics/claude-code/issues/24055)（80リアクション）で多数報告。

**防御：**
```bash
# シェルプロファイルに追加
export CLAUDE_CODE_MAX_OUTPUT_TOKENS=128000
```

環境変数`CLAUDE_CODE_MAX_OUTPUT_TOKENS`でデフォルトの32K上限を128Kに引き上げる。Opus 4.6は128Kトークンまで出力可能。

## パターン9：/tmp/claude-*ファイルが溜まり続ける

**何が起きたか：** 数日間使い続けたら、`/tmp/claude-*`ファイルが数百個蓄積。ディスク容量を圧迫。GitHub Issue [#8856](https://github.com/anthropics/claude-code/issues/8856)（67リアクション）で報告。

**防御：**
```bash
npx cc-safe-setup --install-example tmp-cleanup
```

SessionStartフックで2時間以上前の`/tmp/claude-*`を自動削除。Stopフックで30分以上前のcwdファイルをクリーンアップ。

## パターン10：トークン消費が突然加速する

**何が起きたか：** Max Planの5時間制限が1時間で枯渇。以前と同じ使い方なのに、トークン消費速度が数倍に感じる。GitHub Issue [#40524](https://github.com/anthropics/claude-code/issues/40524)（195リアクション）を筆頭に、2026年3月末から大量の報告が集まっている。

**原因の切り分け：**

トークンを消費する隠れた要因が複数ある。最も破壊的なものから順に:

1. **プロンプトキャッシュの無効化**（最大原因）: Claude Codeは会話履歴をキャッシュして再送信コストを削減している。キャッシュが壊れると、毎ターン全履歴を再送信するため消費が**10〜20倍**になる。キャッシュ読み取り率が89%→4.3%に暴落した実例あり
2. **Tool Search**（デフォルト有効）: 毎ターンにdeferred toolの定義が追加される
3. **大ファイルの全行Read**: 1万行のファイルを1行だけ確認するためにRead→全行分のトークン消費
4. **Auto-compact循環**: コンテキストが閾値に達する→要約→再構築→また閾値→繰り返し

**キャッシュが壊れる主な原因：**

- **セッションファイルの読み取り**: Claudeが自分の会話履歴（.jsonl）を読むと、CLI内部でbilling hashが書き換わり、キャッシュのプレフィックスが変わる。これが最も一般的で最も破壊的
- **セッション再開**: `--resume`後の最初の数ターンはキャッシュの再構築が必要
- **並列サブエージェント**: 多数のサブエージェントがキャッシュスロットを競合する

**防御（最重要: キャッシュ汚染防止）：**

セッションファイルの読み取りをブロックするhook:
```bash
#!/bin/bash
# conversation-history-guard.sh — PreToolUse, matcher: "Bash|Read"
INPUT=$(cat)
CMD=$(printf '%s' "$INPUT" | jq -r '.tool_input.command // empty' 2>/dev/null)
if [ -n "$CMD" ]; then
    if printf '%s' "$CMD" | grep -qE '\.(jsonl|log)' && \
       printf '%s' "$CMD" | grep -qiE '(claude|session|billing|transcript)'; then
        echo '{"decision":"block","reason":"セッションファイル読み取りはキャッシュを破壊します"}'
        exit 0
    fi
fi
FILE=$(printf '%s' "$INPUT" | jq -r '.tool_input.file_path // empty' 2>/dev/null)
if [ -n "$FILE" ] && printf '%s' "$FILE" | grep -qE '\.claude/projects/.+\.jsonl$'; then
    echo '{"decision":"block","reason":"会話履歴の読み取りはキャッシュを20倍悪化させます"}'
    exit 0
fi
exit 0
```

**防御（Tool SearchとMCP）：**

```bash
# Tool Searchを無効化（MCP多数の場合に効果大）
export ENABLE_TOOL_SEARCH=false

# 手動compactで循環を防ぐ（/compactコマンドを自分のタイミングで実行）
```

**トークン消費の追跡：**

UserPromptSubmitフックでプロンプトログを記録:

```bash
#!/bin/bash
INPUT=$(cat)
PROMPT=$(echo "$INPUT" | jq -r '.prompt | .[0:100]' 2>/dev/null)
echo "$(date -u +%H:%M:%S) $PROMPT" >> /tmp/claude-token-log.txt
exit 0
```

セッション後に`/tmp/claude-token-log.txt`を確認して、消費の多いパターンを特定する。

MCPサーバーの数を確認（各サーバーのツール定義がコンテキストに追加される）:
```bash
claude mcp list
```

Notificationフックでcompaction発生を通知:
```bash
#!/bin/bash
INPUT=$(cat)
MSG=$(printf '%s' "$INPUT" | jq -r '.message // empty' 2>/dev/null)
if printf '%s' "$MSG" | grep -qi "compact"; then
  printf '⚠ Compaction発生 — /clearまたは手動/compactを検討\n' >&2
fi
exit 0
```

**消費を減らすコツ：**
- **conversation-history-guardを必ず入れる**（最もインパクトが大きい）
- `/compact`を閾値到達前に手動で実行（auto-compactより安い）
- **plan mode**で計画してから実装（trial-and-errorより総ツール呼び出しが少ない）
- サブエージェントの乱用を避ける（各Agentが新しいコンテキスト窓を作る）
- 大ファイルの`Read`には`offset`と`limit`パラメータを必ず指定する

## まとめ：1コマンドで全部入れる

```bash
npx cc-safe-setup --shield
```

上記パターン全てのhookが含まれる。加えて、プロジェクトの技術スタックに応じた追加hookも自動インストールされる。

さらに詳しいhookの一覧は前章の「653+ hookカタログ」を参照。
