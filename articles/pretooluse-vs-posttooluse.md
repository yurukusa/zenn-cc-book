---
title: "PreToolUseとPostToolUse、どっちを使うべきか——Claude Code hookの使い分けガイド"
emoji: "🔀"
type: "tech"
topics: ["claudecode", "security", "ai", "automation"]
published: true
---

Claude Codeのhookには**PreToolUse**と**PostToolUse**の2種類がある。どちらもツール実行に介入するが、タイミングが違う。タイミングが違うと、できることが根本的に変わる。

「とりあえずPostToolUseでチェックすればいい」と思っていると、**ブロックしたかった操作がすでに実行された後**だった——という事態になる。

## 2つのhookの実行タイミング

```
ユーザーの指示
  ↓
Claude がツール呼び出しを決定
  ↓
┌─────────────────────┐
│  PreToolUse 実行     │  ← ツール実行「前」
│  exit 2 → ブロック   │
└─────────────────────┘
  ↓（exit 0 なら通過）
┌─────────────────────┐
│  ツール本体の実行     │  ← rm, git push, Write 等
└─────────────────────┘
  ↓
┌─────────────────────┐
│  PostToolUse 実行    │  ← ツール実行「後」
│  結果の検証・記録     │
└─────────────────────┘
  ↓
Claude が結果を解釈して次のステップへ
```

**PreToolUse**はツールが動く前に走る。`exit 2`を返せば実行そのものを止められる。
**PostToolUse**はツールが動いた後に走る。実行結果を検査できるが、実行を止めることはできない。

## PreToolUse：実行前に止める

PreToolUseの役割は**門番**だ。危険な操作を実行させない。

### 使うべき場面

- **破壊的コマンドのブロック**: `rm -rf /`、`git reset --hard`、`chmod 777`
- **ブランチ保護**: main/masterへの直接push、force-push
- **秘密情報の保護**: `.env`や秘密鍵の`git add`
- **入力値のバリデーション**: 実行前にコマンドの引数を検査

### 実装例：破壊的コマンドのブロック

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "bash ~/.claude/hooks/destructive-guard.sh"
      }]
    }]
  }
}
```

```bash
#!/bin/bash
# destructive-guard.sh
# stdin から tool_input を読み取る
input=$(cat)
command=$(echo "$input" | jq -r '.tool_input.command // empty')

# 破壊的パターンを検出
if echo "$command" | grep -qE 'rm\s+-[a-zA-Z]*r[a-zA-Z]*f|git\s+reset\s+--hard|git\s+push\s+.*--force'; then
  echo '{"decision":"block","reason":"破壊的コマンドを検出。手動で実行してください。"}'
  exit 2
fi
```

PreToolUseが`exit 2`を返すと、**ツールは実行されない**。`rm -rf /`は走らない。ファイルは消えない。

## PostToolUse：実行後に検査する

PostToolUseの役割は**検査官**だ。実行結果を確認し、問題があればClaudeにフィードバックする。

### 使うべき場面

- **構文チェック**: ファイル編集後にlint/compileで構文エラーを検出
- **ログ記録**: どのツールが何を実行したかを記録
- **出力の検証**: 実行結果が期待通りかチェック
- **メトリクス収集**: コンテキスト使用量、実行回数の監視
- **自動フォーマット**: 編集後のコードをフォーマッタで整形

### 実装例：編集後の構文チェック

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "bash ~/.claude/hooks/syntax-check.sh"
      }]
    }]
  }
}
```

```bash
#!/bin/bash
# syntax-check.sh
input=$(cat)
file_path=$(echo "$input" | jq -r '.tool_input.file_path // empty')

case "$file_path" in
  *.py) python -m py_compile "$file_path" 2>&1 ;;
  *.js) node --check "$file_path" 2>&1 ;;
  *.sh) bash -n "$file_path" 2>&1 ;;
esac
```

構文エラーが出力されると、Claudeはそれを読んで**自動的に修正を試みる**。

## 判断テーブル

| やりたいこと | 使うhook | 理由 |
|---|---|---|
| `rm -rf` をブロック | **PreToolUse** | 実行後では手遅れ |
| main への push を禁止 | **PreToolUse** | push 後に取り消すのは困難 |
| `.env` の commit を防止 | **PreToolUse** | git 履歴に残ると完全削除が面倒 |
| 編集後の構文チェック | **PostToolUse** | 編集結果がないと検査できない |
| ツール実行のログ記録 | **PostToolUse** | 実行結果を含めて記録したい |
| コンテキスト使用量の監視 | **PostToolUse** | 実行後の状態を計測する |
| 特定ディレクトリへの書き込み禁止 | **PreToolUse** | 書き込み後では元に戻せない可能性 |
| コードフォーマットの自動適用 | **PostToolUse** | 書き込まれたコードを整形する |

判断基準はシンプルだ。

**「実行されたら取り返しがつかないか？」**

YESならPreToolUse。NOならPostToolUse。

## よくある間違い：PostToolUseでブロックしようとする

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "bash ~/.claude/hooks/block-dangerous.sh"
      }]
    }]
  }
}
```

これは**意味がない**。PostToolUseが走る時点で、コマンドはすでに実行されている。`rm -rf`でファイルが消えた後に「危険なコマンドでした」と言われても遅い。

PostToolUseで`exit 2`を返しても、**実行済みの操作は巻き戻らない**。

:::message alert
PostToolUseはブロック用ではない。ブロックが必要ならPreToolUseを使う。これを間違えると「hookを入れたのに事故が起きた」という最悪の結果になる。
:::

## 両方を組み合わせる

実戦では両方を使う。PreToolUseで危険を防ぎ、PostToolUseで品質を担保する。

```
PreToolUse（門番）        PostToolUse（検査官）
├─ destructive-guard     ├─ syntax-check
├─ branch-guard          ├─ context-monitor
├─ secret-guard          ├─ execution-log
└─ path-guard            └─ auto-format
```

[cc-safe-setup](https://github.com/yurukusa/cc-safe-setup)は349のhook実装例と4,549のテストから、この組み合わせをワンコマンドで適用する。

## 始め方

```bash
npx cc-safe-setup       # 8つの安全hookをインストール
npx cc-health-check     # 設定の抜け漏れを診断
```

PreToolUseとPostToolUseの使い分けを自分で考える必要はない。

---

📘 hookの設計思想から実戦パターンまで体系的に学ぶなら [Claude Code 実戦ガイド](https://zenn.dev/yurukusa/books/6076c23b1cb18b)
