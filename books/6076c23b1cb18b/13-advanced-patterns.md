---
title: "高度なhookパターン——チェックポイント保護、フック連鎖、条件分岐"
---

基本的なhookを設定したら、次はより高度なパターンに進む。

## チェックポイント保護

GitHub Issue [#38841](https://github.com/anthropics/claude-code/issues/38841)で報告された問題: Claudeがhookのチェックポイントファイルを直接操作してバイパスした。

hookがファイルベースの状態管理（チェックポイント、カウンター等）を使っている場合、Claudeはそのファイルを書き換えてhookを無効化できる。

対策: **checkpoint-tamper-guard**

```bash
#!/bin/bash
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty' 2>/dev/null)
FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty' 2>/dev/null)

PROTECTED=(".claude/checkpoints" "session-call-count" "hook-state")

for pattern in "${PROTECTED[@]}"; do
    if [[ "$CMD" == *"$pattern"* ]] && echo "$CMD" | grep -qE "(echo|cat|tee|cp|mv|rm|chmod)"; then
        echo "BLOCKED: Cannot manipulate hook state" >&2
        exit 2
    fi
    if [[ "$FILE" == *"$pattern"* ]]; then
        echo "BLOCKED: Cannot edit hook state file" >&2
        exit 2
    fi
done
exit 0
```

原則: **hookの状態ファイルとhookの強制ロジックは、モデルが到達できない閉じたシステムを形成すべき。**

## 環境変数による条件分岐

同じhookを開発環境と本番環境で異なる動作にする。

```bash
#!/bin/bash
CMD=$(cat | jq -r '.tool_input.command // empty' 2>/dev/null)
[ -z "$CMD" ] && exit 0

# 開発モードでは警告のみ、本番モードではブロック
if echo "$CMD" | grep -qE 'git push.*main'; then
    if [ "${CC_ENV:-development}" = "production" ]; then
        echo "BLOCKED: Push to main in production mode" >&2
        exit 2
    else
        echo "WARNING: Push to main (allowed in dev mode)" >&2
        exit 0
    fi
fi
exit 0
```

使い方:
```bash
# 開発時
CC_ENV=development claude

# 本番（自律運用時）
CC_ENV=production claude
```

## フック連鎖

1つのmatcherに複数のhookを登録すると、**全てが順番に実行される**。1つでもexit 2を返すとブロック。

```json
{
  "PreToolUse": [{
    "matcher": "Bash",
    "hooks": [
      { "type": "command", "command": "~/.claude/hooks/destructive-guard.sh" },
      { "type": "command", "command": "~/.claude/hooks/branch-guard.sh" },
      { "type": "command", "command": "~/.claude/hooks/secret-guard.sh" }
    ]
  }]
}
```

実行順序: destructive-guard → branch-guard → secret-guard。最初にexit 2が出た時点で後続はスキップされる。

**設計原則**: 最も頻繁にブロックするhookを先に配置する。不要なhook実行を減らせる。

## matcher設計

```
"Bash"       → Bashコマンドのみ（最も使用頻度高い）
"Edit|Write" → ファイル変更のみ
"Read"       → ファイル読み取りのみ
""           → 全ツール（オーバーヘッド大。注意）
```

**空文字列matcherの罠**: matcher `""` は全ツールにマッチする。hookに構文エラーがあると、`exit 2`が返り、**全ツール**がブロックされる。3時間のロックアウトを経験した（Chapter 5参照）。

matcher `""` を使う場合は:
1. hookに `bash -n` 構文チェックを通す
2. `npx cc-safe-setup --validate` で検証
3. 開発中は `"Bash"` に限定してテスト

## 自動承認パターン

読み取り専用コマンドを自動承認して、権限プロンプトを80%削減する。

```bash
#!/bin/bash
INPUT=$(cat)
TOOL=$(echo "$INPUT" | jq -r '.tool_name // empty' 2>/dev/null)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty' 2>/dev/null)

# Bashの読み取り専用コマンド
if [ "$TOOL" = "Bash" ]; then
    SAFE_CMDS="ls|cat|head|tail|grep|find|wc|diff|git status|git log|git diff|git branch|pwd|echo|date|which|type|file"
    FIRST_CMD=$(echo "$CMD" | sed 's/;.*//' | sed 's/|.*//' | sed 's/&&.*//' | awk '{print $1}')
    if echo "$FIRST_CMD" | grep -qxE "$SAFE_CMDS"; then
        echo '{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"allow","permissionDecisionReason":"Read-only command"}}'
        exit 0
    fi
fi

exit 0
```

## --simulateで事前確認

hookを本番に入れる前に、動作を確認する。

```bash
npx cc-safe-setup --simulate "rm -rf /"
# ✗ BLOCK — destructive-guard

npx cc-safe-setup --simulate "git push origin main"
# ✗ BLOCK — branch-guard

npx cc-safe-setup --simulate "git status"
# ✓ APPROVE — auto-approve-readonly

npx cc-safe-setup --simulate "npm test"
# ✓ PASS — no hook matches
```

## まとめ

| パターン | 用途 | 難易度 |
|---------|------|--------|
| チェックポイント保護 | hookバイパス防止 | 中 |
| 環境変数分岐 | 開発/本番切り替え | 低 |
| フック連鎖 | 多段チェック | 低 |
| 空matcherの制限 | 全ツール監視 | 高（リスクあり） |
| 自動承認 | 権限プロンプト削減 | 中 |
| --simulate | 事前検証 | 低 |

hookの設計は「何をブロックするか」だけでなく、「何を許可するか」「どう検証するか」も含む。テストと事前検証を組み合わせて、安全かつ快適な自律運用環境を作ろう。
