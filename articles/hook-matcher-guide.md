---
title: "Claude Code hookのmatcher完全ガイド——どのツールに発火させるか"
emoji: "🎯"
type: "tech"
topics: ["claudecode", "ai", "hooks", "設定"]
published: false
---

## hookが全ツールで発火する

hookを書いた。`settings.json`に登録した。動く。

だが遅い。

Bashを実行するたびに発火。Editでも発火。Readでも、Globでも、Grepでも。全ツール呼び出しのたびにスクリプトが走る。

原因は`matcher`を知らなかったこと。

## matcherとは

hookエントリに指定する文字列。**どのツールに対して発火するかを制御する**。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/my-hook.sh"
          }
        ]
      }
    ]
  }
}
```

`matcher`の値が`"Bash"`なら、Bashツール呼び出し時だけ発火する。Editには発火しない。Readにも発火しない。

hookに渡されるJSONの`tool_name`フィールドとmatcherが照合される。一致したときだけhookが実行される。

## 基本パターン

### 空文字 `""` — 全ツールに発火

```json
{
  "matcher": "",
  "hooks": [
    { "type": "command", "command": "~/.claude/hooks/context-monitor.sh" }
  ]
}
```

通知やログ記録に使う。全ツール呼び出しで発火するため、重い処理には向かない。

### `"Bash"` — Bashツールのみ

```json
{
  "matcher": "Bash",
  "hooks": [
    { "type": "command", "command": "~/.claude/hooks/destructive-guard.sh" }
  ]
}
```

最も使用頻度が高い。`rm -rf`や`git push --force`の阻止に。

### その他のツール

| matcher | 対象 |
|---------|------|
| `"Edit"` | ファイル部分編集 |
| `"Write"` | ファイル全体書き込み |
| `"Read"` | ファイル読み取り |
| `"Glob"` | ファイル検索 |
| `"Grep"` | コンテンツ検索 |

## 複数ツールの指定

パイプ`|`で区切る。

```json
{
  "matcher": "Edit|Write",
  "hooks": [
    { "type": "command", "command": "~/.claude/hooks/syntax-check.sh" }
  ]
}
```

`"Edit|Write"`はEditまたはWriteのどちらかが呼ばれたら発火する。正規表現の`|`（OR）と同じ考え方。

## 実用パターン集

### 破壊的コマンドの防止

```json
{
  "matcher": "Bash",
  "hooks": [
    { "type": "command", "command": "~/.claude/hooks/destructive-guard.sh" }
  ]
}
```

Bashだけに絞る。EditやReadには不要。

### ファイル変更後の構文チェック

```json
{
  "PostToolUse": [
    {
      "matcher": "Edit|Write",
      "hooks": [
        { "type": "command", "command": "~/.claude/hooks/syntax-check.sh" }
      ]
    }
  ]
}
```

**PostToolUseで`Edit|Write`**。ファイル変更後に自動で構文検証が走る。

### 機密ファイルへの書き込み防止

```json
{
  "matcher": "Write",
  "hooks": [
    { "type": "command", "command": "~/.claude/hooks/write-secret-guard.sh" }
  ]
}
```

Writeだけに絞る。Editでの部分編集とは分けて管理。

## 用途別の早見表

| 用途 | matcher | イベント |
|------|---------|----------|
| 破壊的コマンド防止 | `"Bash"` | PreToolUse |
| ブランチ保護 | `"Bash"` | PreToolUse |
| 構文チェック | `"Edit\|Write"` | PostToolUse |
| 機密ファイル保護 | `"Write"` | PreToolUse |
| コマンド自動承認 | `"Bash"` | PreToolUse |
| ファイル変更監視 | `"Edit\|Write"` | PostToolUse |
| 全操作ログ | `""` | PostToolUse |

## パフォーマンスへの影響

matcherなし（空文字`""`）は全ツールで発火する。

Claude Codeは1タスクで数十回ツールを呼ぶ。全呼び出しでシェルスクリプトが起動すれば、体感速度は確実に落ちる。

対策は単純。**必要なツールだけに絞る**。

破壊的コマンドを止めたいなら`"Bash"`。構文チェックなら`"Edit|Write"`。全ツールに発火させる理由がないなら、空文字は避ける。

## よくある間違い

### ツール名の大文字小文字

matcherは大文字小文字を区別する。`"bash"`ではなく`"Bash"`。`"edit"`ではなく`"Edit"`。

正しいツール名一覧:
- `Bash`
- `Edit`
- `Write`
- `Read`
- `Glob`
- `Grep`

### 存在しないツール名

matcherに存在しないツール名を書いてもエラーにならない。だが一度も発火しない。タイポに注意。

### 空文字とmatcher省略の違い

`"matcher": ""`は「全ツールに発火」。matcherキー自体を省略した場合も同じ挙動。意図的に全ツール対象にするなら、空文字を明示するほうが読みやすい。

## まとめ

- 特定ツールだけ → `"Bash"`、`"Edit"`など
- 複数ツール → `"Edit|Write"`でパイプ区切り
- 全ツール → `""`（空文字）

**必要なツールだけに絞る。それだけでhookは速くなる。**

---

**第2章「Safety Guards」まで無料公開中。**
🛡 `npx cc-safe-setup` — 8つの安全フックを10秒でインストール
---
:::message
**Claude Codeの自律運用をもっと深く学びたい方へ**
トークン消費の突然の急増、ファイル消失、無限ループ——700+時間の自律稼働で起きた失敗と対策をまとめました:
[Claude Codeを本番品質にする——実践ガイド](https://zenn.dev/yurukusa/books/6076c23b1cb18b)（¥800・第3章まで無料）
:::
