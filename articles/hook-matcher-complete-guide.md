---
title: "Claude Codeのhookマッチャー完全ガイド——どのツールにどのhookが発火するか"
type: "tech"
emoji: "🎯"
topics: ["claudecode", "hooks", "設定", "AI"]
published: true
---

## hookのマッチャーとは

Claude Codeのhookには`matcher`フィールドがある。これは「どのツールが呼ばれたときにhookを発火させるか」を制御する。

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/my-hook.sh"}]
    }]
  }
}
```

この例では`Bash`ツールが呼ばれたときだけhookが発火する。`Read`や`Edit`では発火しない。

## 使えるマッチャー一覧

Claude Codeで使用可能な主要ツール名（＝マッチャー値）:

| マッチャー | ツール | よくあるhook用途 |
|-----------|--------|------------------|
| `Bash` | シェルコマンド実行 | 危険コマンドのブロック、自動承認 |
| `Read` | ファイル読み取り | 読み取り回数の制限、機密ファイルの保護 |
| `Edit` | ファイル編集 | 構文チェック、差分サイズ制限 |
| `Write` | ファイル新規作成 | シークレット漏洩防止、パス制限 |
| `Glob` | ファイルパターン検索 | （あまり使わない） |
| `Grep` | テキスト検索 | （あまり使わない） |
| `Agent` | サブエージェント起動 | 並列数の制限、スコープ制限 |
| `AskUserQuestion` | ユーザーへの質問 | 自動回答、通知 |

### マッチャーを省略すると

`matcher`フィールドを省略すると、**全てのツール呼び出し**でhookが発火する。

```json
{
  "hooks": {
    "PreToolUse": [{
      "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/rate-limiter.sh"}]
    }]
  }
}
```

これは`tool-call-rate-limiter`（全ツールの呼び出し頻度を制限するhook）のように、ツール種別に関係なく発火させたい場合に使う。

### 複数のマッチャー

同じhookを複数のツールに適用したい場合は、エントリを複数書く:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit",
        "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/syntax-check.sh"}]
      },
      {
        "matcher": "Write",
        "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/syntax-check.sh"}]
      }
    ]
  }
}
```

## イベントタイプ × マッチャーの組み合わせ

| イベント | 使えるマッチャー | 説明 |
|---------|-----------------|------|
| `PreToolUse` | 全ツール名 | ツール実行**前**。exit 2でブロック可能 |
| `PostToolUse` | 全ツール名 | ツール実行**後**。結果の検証 |
| `Notification` | `Stop`, `compact` 等 | システムイベント通知 |
| `UserPromptSubmit` | なし（常に発火） | ユーザー入力時 |
| `SessionStart` | なし（常に発火） | セッション開始時 |
| `PermissionRequest` | 全ツール名 | パーミッション確認時 |

### PreToolUse vs PermissionRequest

```
ユーザー入力 → PreToolUse → 内蔵パーミッションチェック → PermissionRequest → ツール実行
```

- **PreToolUse**: 内蔵チェックの**前**。`permissionDecision: "allow"`を返しても、内蔵チェックで再度ブロックされる可能性がある
- **PermissionRequest**: 内蔵チェックの**後**。ここで`allow`を返すと最終的に許可される

:::message
**実用的な違い**: 保護ディレクトリ（`~/.ssh/`等）への操作を自動承認したい場合は**PermissionRequest**を使う。PreToolUseの`allow`は内蔵チェックに上書きされる。
:::

## v2.1.85 `if`フィールド

v2.1.85から、マッチャーに加えて`if`フィールドでさらに細かい条件を指定できる:

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "if": "tool_input.command matches 'git push'",
      "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/push-guard.sh"}]
    }]
  }
}
```

`if`フィールドを使うと、条件に合致しない場合はhookプロセスが**起動すらしない**。パフォーマンスが向上する。

## よくあるマッチャー設定パターン

### パターン1: 安全ガード（PreToolUse + Bash）

```json
{"matcher": "Bash", "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/destructive-guard.sh"}]}
```

### パターン2: 構文チェック（PostToolUse + Edit|Write）

```json
{"matcher": "Edit", "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/syntax-check.sh"}]}
```

### パターン3: サブエージェント制限（PreToolUse + Agent）

```json
{"matcher": "Agent", "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/subagent-budget.sh"}]}
```

### パターン4: 自動回答（PreToolUse + AskUserQuestion）

```json
{"matcher": "AskUserQuestion", "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/auto-answer.sh"}]}
```

### パターン5: 全ツール監視（マッチャー省略）

```json
{"hooks": [{"type": "command", "command": "bash ~/.claude/hooks/rate-limiter.sh"}]}
```

## ワンコマンドで全設定

```bash
npx cc-safe-setup
```

634個のexample hookが含まれており、上記のマッチャーパターンは全てカバーされている。13,931テストで検証済み。

---

📌 関連記事:
- [hookのifフィールドで不要なプロセス起動をゼロにする](https://qiita.com/yurukusa/items/7079866e9dc239fcdd57)
- [ハーネスエンジニアリングを700時間の自律運用で実践している話](https://qiita.com/yurukusa/items/fb73a4122b583197071b)
---
5月22日に新刊の事例集 [Claude Code Claim-Verify Handbook](https://yurukusa.gumroad.com/l/claim-verify-handbook?utm_source=zenn-hook-matcher-complete-guide&utm_medium=article&utm_campaign=handbook-zenn-bulk) ($19、 約89頁、 約113,000字) を発売します。 道具が「成功した」 「比較した」 「設定された」 と主張する一方で実態が乖離していた事例を GitHubの起票の集まりから130件 (本文15件+付録D 115件) 整理した本で、 14件の防衛の手順と5件の自動の点検の道具と一緒に提供しています。 試し読みのGist (約16,000字、 章1の全件) は [こちら](https://gist.github.com/yurukusa/6dd608049064ed66c54f1a545a7b47a8?utm_source=zenn-hook-matcher-complete-guide&utm_medium=article&utm_campaign=handbook-preview-zenn-bulk)。
予防 hook の集まりは [cc-safe-setup](https://github.com/yurukusa/cc-safe-setup) (MIT、 745件以上の hook、 30,000件以上のインストール)。 月額の継続の媒体 [CC Safety Lab](https://yurukusa.github.io/cc-safe-setup/safety-lab.html?utm_source=zenn-hook-matcher-complete-guide&utm_medium=article&utm_campaign=safety-lab-zenn-bulk) (¥500/月、 Ko-fi) は毎月の事故の整理を配信。
