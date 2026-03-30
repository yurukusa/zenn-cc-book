---
title: "Claude Code hookのデバッグ完全ガイド——動かない原因を5分で特定する"
emoji: "🔧"
type: "tech"
topics: ["claudecode", "ai", "hooks", "デバッグ"]
published: false
---

hookを書いた。settings.jsonに登録した。Claude Codeを再起動した。

何も起きない。ログもない。エラーもない。

hookのデバッグは「何が間違っているか分からない」のが厄介だ。Claude Codeはhookの失敗を黙殺する。**exit code 1はエラーだが、ブロックではなく許可として扱われる**。

この記事では、動かないhookを5分で特定する5ステップを示す。

## ステップ1: --doctorで基本チェック

最初にやること。診断コマンドの実行。

```bash
npx cc-safe-setup --doctor
```

jqの有無、settings.jsonの構文、hookファイルの存在、実行権限、shebang行を一括チェックする。

```
✓ jq found: jq-1.7.1
✓ settings.json: valid JSON
✓ hooks/destructive-guard.sh: exists, executable
✗ hooks/secret-guard.sh: not executable
  Fix: chmod +x ~/.claude/hooks/secret-guard.sh
```

「All checks passed」と出たのに動かない。次のステップへ。

## ステップ2: --simulateで単体テスト

hookをClaude Codeの外で直接テストする。

```bash
echo '{"tool_name":"Bash","tool_input":{"command":"rm -rf /"}}' \
  | bash ~/.claude/hooks/destructive-guard.sh
echo $?
```

exit codeを確認する。

- `2`が返る → hookは正しく動いている。問題は登録側
- `0`が返る → hookのパターンマッチが間違っている
- エラーが出る → hookスクリプト自体にバグがある

stderrも分離して確認する。

```bash
echo '{"tool_name":"Bash","tool_input":{"command":"rm -rf /"}}' \
  | bash ~/.claude/hooks/destructive-guard.sh 2>/tmp/hook-stderr
```

stderrはユーザーへのメッセージ。stdoutは許可判定JSON。混同すると動作しない。

## ステップ3: matcherの確認

settings.jsonのmatcher設定。ここが一番間違いやすい。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/guard.sh"
          }
        ]
      }
    ]
  }
}
```

matcherの値と意味:

| matcher | 対象 |
|---------|------|
| `"Bash"` | Bashツールのみ |
| `"Edit\|Write"` | EditまたはWrite |
| `"Read"` | Readツールのみ |
| `""` (空文字) | 全ツール |

よくある間違い:

- matcherを省略 → hookが発火しない
- `"Edit | Write"` → スペースが入るとマッチしない。`"Edit|Write"`が正解
- `""`（空文字）→ 全ツールで発火。Bashだけ監視したいのに空文字だと、Read/Editでも実行される

hookがcommandフィールドを前提にしていると、Readツール（commandがない）で空入力になり想定外の動作をする。

## ステップ4: exit codeの確認

hookの動作はexit codeで決まる。これがhookデバッグの核心。

### PreToolUseの場合

| exit code | 動作 |
|-----------|------|
| 0 | 許可（何もしない） |
| 2 | ブロック（ツール実行を中止） |
| 1 | エラー（許可として扱われる） |

最大の罠: **exit 1はブロックではない**。

```bash
# NG: exit 1 → エラー扱い → 許可される
if echo "$COMMAND" | grep -q 'rm -rf'; then
  echo "Blocked!" >&2
  exit 1  # ブロックしない！
fi

# OK: exit 2 → ブロック
if echo "$COMMAND" | grep -q 'rm -rf'; then
  echo "Blocked: destructive command detected" >&2
  exit 2  # これがブロック
fi
```

### PostToolUseの場合

| exit code | 動作 |
|-----------|------|
| 0 | 情報提供（正常終了） |
| 1以上 | エラー通知 |

PostToolUseにはブロック機能がない。ツールは既に実行済み。stderrに書いた内容がユーザーへの通知になる。

## ステップ5: stderr出力の活用

ユーザーにメッセージを伝えるにはstderrを使う。

```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty' 2>/dev/null)
[ -z "$COMMAND" ] && exit 0

if echo "$COMMAND" | grep -qE 'rm\s+(-[rf]+\s+)*/'; then
  echo "🛑 破壊的コマンドを検出: $COMMAND" >&2
  exit 2
fi
exit 0
```

**stdoutは許可判定JSON用。stderrがユーザーへのメッセージ。** stdoutにテキストを書くと、Claude CodeがそれをJSONとしてパースしようとして意図しない挙動になる。

## よくある罠

### 1. 改行コードの問題（Windows/WSL）

Windowsエディタでhookファイルを編集すると、改行コードがCRLF（`\r\n`）になる。

```bash
# 確認
file ~/.claude/hooks/guard.sh
# "CRLF line terminators" と出たらNG

# 修正
sed -i 's/\r$//' ~/.claude/hooks/guard.sh
```

shebang行に`\r`が含まれると`/bin/bash\r`を探しに行く。「No such file or directory」になる。

### 2. jqのバージョン

古いjq（1.5以前）だと`// empty`構文が使えない。

```bash
jq --version
# jq-1.6以上を推奨
```

### 3. Pythonフックのtimeout

WindowsでPythonフックを使うとプロセス起動が遅い。`"timeout": 10000`を明示する。

### 4. 空入力ガードの欠如

matcherが空文字だと全ツールで発火する。Readには`tool_input.command`がない。

```bash
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty' 2>/dev/null)
[ -z "$COMMAND" ] && exit 0  # この2行がないと想定外の動作をする
```

## cc-hook-testで自動テスト

手動テストが面倒なら自動テストツールを使う。

```bash
npx cc-hook-test ~/.claude/hooks/guard.sh
```

複数テストケースの自動生成、exit code検証、境界ケース（空入力、不正JSON）のテストを一括実行する。hookを書いたら手動テスト→cc-hook-testの順で検証する。

## まとめ: 5ステップチェックリスト

1. `npx cc-safe-setup --doctor` → 環境と権限の確認
2. `echo '{...}' | bash hook.sh` → 手動で単体テスト
3. matcherの値を確認 → 空文字/パイプ区切り/スペース
4. exit codeを確認 → 2がブロック。1ではない
5. stderrにメッセージ → stdoutは許可判定JSON用

この5つを順に潰せば、大半の「動かない」は5分で解決する。

---

**第2章「Safety Guards」まで無料公開中。**
🛡 `npx cc-safe-setup` — 8つの安全フックを10秒でインストール
