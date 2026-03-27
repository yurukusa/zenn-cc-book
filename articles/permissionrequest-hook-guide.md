---
title: "PermissionRequestフック——Claude Code hookの第3のトリガーを使いこなす"
emoji: "🔐"
type: "tech"
topics: ["claudecode", "security", "ai", "automation"]
published: false
---

## Claude Code hooksの3つのトリガー

Claude Code hooksには**3種類のトリガー**がある。

| トリガー | 発火タイミング | 主な用途 |
|---|---|---|
| **PreToolUse** | ツール実行の直前（許可済み） | 入力の検証・変換・ブロック |
| **PostToolUse** | ツール実行の直後 | 結果のログ・後処理 |
| **PermissionRequest** | 許可ダイアログの表示直前 | 自動許可・自動拒否・ログ |

PreToolUseとPostToolUseは多くの記事で紹介されているが、**PermissionRequest**は情報が少ない。この記事ではPermissionRequestの仕組みと実践的な使い方を解説する。

## PermissionRequestとは何か

Claude Codeは特定の操作を実行する前に、ユーザーに許可を求めるダイアログを表示する。たとえば：

- 未許可のBashコマンド
- `.git/`や`.claude/`などの保護ディレクトリへの書き込み
- 外部ネットワークへのアクセス

**PermissionRequestフックは、この許可ダイアログが表示される「前」に発火する。** フックの出力によって、許可ダイアログを表示せずに自動で判断できる。

## PreToolUseとの決定的な違い

ここが最も混同しやすいポイントだ。

> PreToolUse → 許可チェック → **PermissionRequest**（未許可時のみ） → ツール実行

**PreToolUse**は許可済みの操作に対して発火する（`allow`リスト内のコマンド、Read・Glob等の組み込みツール）。**PermissionRequest**は未許可の操作で許可ダイアログが出る直前に発火する。

重要な帰結：**保護ディレクトリへの書き込みをPreToolUseで`allow`しても許可ダイアログは消えない**。保護ディレクトリのチェックはPreToolUseの後に行われるからだ。PermissionRequestフックを使う必要がある。

## 出力フォーマット

PermissionRequestフックは、JSON形式で判断を返す。

`permissionDecision`に`"allow"`か`"deny"`を指定する。何も出力せず`exit 0`すれば通常の許可ダイアログに委ねられる。

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "permissionDecision": "allow",
    "permissionDecisionReason": "理由をここに書く"
  }
}
```

`"allow"`を`"deny"`に変えれば自動拒否になる。

## 実装例1: force-pushの自動拒否

`git push --force`は事故の元だ。PermissionRequestフックで自動的にブロックできる。

```bash
#!/bin/bash
# deny-force-push.sh — PermissionRequest hook (Matcher: Bash)
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty' 2>/dev/null)
[ -z "$COMMAND" ] && exit 0

if echo "$COMMAND" | grep -qE 'git\s+push\s+.*--force'; then
  jq -n '{"hookSpecificOutput":{"hookEventName":"PermissionRequest","permissionDecision":"deny","permissionDecisionReason":"force-push is blocked by hook policy"}}'
  exit 0
fi
exit 0
```

`settings.json`への登録は、`PermissionRequest`をキーにしてmatcherとcommandを指定する：

```json
{
  "hooks": {
    "PermissionRequest": [{
      "matcher": "Bash",
      "hooks": [{ "type": "command", "command": "~/.claude/hooks/deny-force-push.sh" }]
    }]
  }
}
```

## 実装例2: テストコマンドの自動許可

テスト実行は安全な操作だ。毎回の許可ダイアログは邪魔になる。

```bash
#!/bin/bash
# allow-test-commands.sh — PermissionRequest hook (Matcher: Bash)
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty' 2>/dev/null)
[ -z "$COMMAND" ] && exit 0

BASE=$(echo "$COMMAND" | awk '{print $1}')
case "$BASE" in
  npm|npx|yarn|pnpm)
    SUBCMD=$(echo "$COMMAND" | awk '{print $2}')
    case "$SUBCMD" in
      test|vitest|jest|mocha|pytest)
        jq -n '{"hookSpecificOutput":{"hookEventName":"PermissionRequest","permissionDecision":"allow","permissionDecisionReason":"Test command auto-approved"}}'
        exit 0 ;;
    esac ;;
esac
exit 0
```

## 実装例3: 許可リクエストのログ記録

どの操作に許可が求められたかを記録しておくと、セキュリティ監査に使える。判断を返さないので他のフックと併用できる。

```bash
#!/bin/bash
# log-permission-requests.sh — PermissionRequest hook
INPUT=$(cat)
TOOL=$(echo "$INPUT" | jq -r '.tool_name // empty' 2>/dev/null)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // "N/A"' 2>/dev/null)
echo "$(date -Iseconds) PermissionRequest tool=$TOOL command=$COMMAND" \
  >> ~/.claude/permission-requests.log
exit 0
```

## 実践的なユースケース: classifierダウン時の救済

Auto Modeの安全性classifier（Sonnet）がダウンすると、`cat`や`ls`まで全ブロックされる。PermissionRequestフックで読み取り専用コマンドだけ自動許可すれば回避できる：

```bash
npx cc-safe-setup --install-example classifier-fallback-allow
```

## 注意点

- **発火条件**: `allow`リストに含まれる操作ではPermissionRequestは発火しない。許可ダイアログが出る場面でのみ発火する
- **優先順位**: 複数フックがある場合、最初に`allow`/`deny`を返したフックの判断が採用される
- **セキュリティ**: `allow`を返すフックは許可ダイアログを迂回する。信頼できる環境（ローカル、CI/CD、Docker）でのみ使うこと。共有環境では`deny`やログ記録に限定するのが安全だ

## まとめ

| やりたいこと | 使うトリガー |
|---|---|
| 許可済み操作の前処理 | PreToolUse |
| 許可ダイアログの自動化 | **PermissionRequest** |
| 保護ディレクトリの許可バイパス | **PermissionRequest** |
| 実行結果の後処理 | PostToolUse |

PermissionRequestは「許可フローの自動化」という、PreToolUseでは届かない領域をカバーする。日常的に使うコマンドの自動許可から、危険な操作の自動拒否、監査ログまで、活用の幅は広い。

---

**cc-safe-setupでPermissionRequestフックを試す：**

```bash
# 実装例をインストール
npx cc-safe-setup --install-example allow-git-hooks-dir

# hookの設定状況を診断
npx cc-health-check
```

cc-safe-setupには358個のフック実装例と4,572件のテストが含まれている。PermissionRequestフックのパターンも複数収録されているので、自分の環境に合ったものを選んで使ってほしい。

Zenn Bookではhooksの体系的な解説を公開している：
https://zenn.dev/yurukusa/books/6076c23b1cb18b
