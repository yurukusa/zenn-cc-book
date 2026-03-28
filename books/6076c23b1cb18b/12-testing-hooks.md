---
title: "hookのテスト——4,820テストケースから学んだ品質保証"
---

hookは安全装置だ。テストされていない安全装置は、存在しないのと同じ。

## なぜhookをテストするのか

cc-safe-setupには443個のexample hookがある。初期は118個しかテストがなかった。残り240個は「たぶん動く」状態だった。

ある日、`response-budget-guard`がセッション中に偽陽性を出した。テスト環境ではカウンターがリセットされず、累積したツール呼び出し回数がリミットを超えたのだ。テストがあれば事前に発見できた。

## hookテストの基本パターン

hookのテストは単純だ。入力JSONをパイプで渡して、exit codeを確認する。

```bash
# ブロックすべき入力 → exit 2を期待
echo '{"tool_input":{"command":"rm -rf /"}}' | bash hook.sh
echo $?  # 2

# 許可すべき入力 → exit 0を期待
echo '{"tool_input":{"command":"ls -la"}}' | bash hook.sh
echo $?  # 0
```

## 5つのテストカテゴリ

### 1. ブロック確認（exit 2）

hookが本当にブロックするかを確認する。最も重要。

```bash
# destructive-guard: rm -rfをブロックするか
echo '{"tool_input":{"command":"rm -rf /"}}' | bash destructive-guard.sh
[ $? -eq 2 ] && echo "PASS" || echo "FAIL"

# branch-guard: mainへのpushをブロックするか
echo '{"tool_input":{"command":"git push origin main"}}' | bash branch-guard.sh
[ $? -eq 2 ] && echo "PASS" || echo "FAIL"
```

### 2. 許可確認（exit 0）

安全な操作をブロックしていないかを確認する。偽陽性は生産性を殺す。

```bash
# destructive-guard: 安全なrmは通すか
echo '{"tool_input":{"command":"rm -rf node_modules"}}' | bash destructive-guard.sh
[ $? -eq 0 ] && echo "PASS" || echo "FAIL"

# branch-guard: featureブランチへのpushは通すか
echo '{"tool_input":{"command":"git push origin feature/new-ui"}}' | bash branch-guard.sh
[ $? -eq 0 ] && echo "PASS" || echo "FAIL"
```

### 3. 空入力処理（exit 0）

hookは空入力でクラッシュしてはいけない。

```bash
echo '{}' | bash hook.sh
[ $? -eq 0 ] && echo "PASS" || echo "FAIL"
```

443個全てのhookで空入力テストを実行している。

### 4. 境界値テスト

hookのパターンマッチングの境界を突く。

```bash
# "rm -rf" を含むが実際は安全なケース
echo '{"tool_input":{"command":"echo rm -rf is dangerous"}}' | bash destructive-guard.sh
# → exit 0であるべき（echoの中は実行されない）

# 大文字小文字
echo '{"tool_input":{"command":"RM -RF /"}}' | bash destructive-guard.sh
# → hookの実装次第（grep -iを使っているか？）
```

### 5. 環境依存テスト

hookが環境変数やファイル状態に依存する場合。

```bash
# response-budget-guard: セッションカウンターが存在するとき
CC_RESPONSE_TOOL_LIMIT=1000 echo '{"tool_input":{"command":"ls"}}' | bash response-budget-guard.sh
# → 高いリミットを設定してexit 0を確認

# test-before-commit: テスト結果ファイルが存在するとき
mkdir -p /tmp/test-dir/coverage && echo '{}' > /tmp/test-dir/coverage/.last-run.json
cd /tmp/test-dir && echo '{"tool_input":{"command":"git commit -m test"}}' | bash test-before-commit.sh
# → exit 0（最近のテスト結果があるのでcommit許可）
```

## テストの自動化

`npx cc-safe-setup --validate` は構文チェックだけ行う。機能テストは別途必要。

cc-safe-setupでは `bash test.sh` で4,820テストを一括実行する:

```bash
cd /path/to/cc-safe-setup
bash test.sh
# Results: 4820/4820 passed
```

自分のカスタムhookにもテストを書くべき。テンプレート:

```bash
#!/bin/bash
PASS=0; FAIL=0

# Test 1: should block dangerous command
EXIT=0; echo '{"tool_input":{"command":"rm -rf /"}}' | bash my-hook.sh >/dev/null 2>/dev/null || EXIT=$?
[ "$EXIT" -eq 2 ] && PASS=$((PASS+1)) || FAIL=$((FAIL+1))

# Test 2: should allow safe command
EXIT=0; echo '{"tool_input":{"command":"ls"}}' | bash my-hook.sh >/dev/null 2>/dev/null || EXIT=$?
[ "$EXIT" -eq 0 ] && PASS=$((PASS+1)) || FAIL=$((FAIL+1))

echo "Results: $PASS/$((PASS+FAIL)) passed"
[ "$FAIL" -eq 0 ] || exit 1
```

## 教訓

1. **テストされていないhookは信頼できない**。443個のhookをテストなしで配布していた期間は、品質を保証できなかった
2. **偽陽性は偽陰性より有害**。ブロックし損ねるのは危険だが、安全な操作をブロックするのは作業が止まる
3. **環境依存hookは特に注意**。セッションカウンター、ファイル状態、タイムスタンプに依存するhookはテスト環境で再現が難しい
4. **hookの構文エラー（exit 2）は全ツールをロックする**。`bash -n` で構文チェックを先にやる
