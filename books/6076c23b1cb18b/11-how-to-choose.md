---
title: "425個のhookから選ぶ方法"
---

425個もhookがあると「どれを入れればいいか」迷う。この章で解決する。

## 3つの選び方

### 方法1: --shieldで全自動

```bash
npx cc-safe-setup --shield
```

スタック検出→推奨hook自動インストール→CLAUDE.md生成。30秒。

### 方法2: --suggestでリスクから選ぶ

```bash
npx cc-safe-setup --suggest
```

プロジェクトのgit履歴とファイル構成からリスクを予測。「この問題が起きうるからこのhookを入れろ」と具体的に提案。

### 方法3: --profileで安全レベルを選ぶ

```bash
npx cc-safe-setup --profile strict    # 33個——自律セッション用
npx cc-safe-setup --profile standard  # 20個——日常開発
npx cc-safe-setup --profile minimal   # 8個——クイックタスク
```

## 用途別おすすめ

### 「とにかく安全にしたい」

```bash
npx cc-safe-setup --shield
```

### 「パーミッションプロンプトがうざい」

```bash
npx cc-safe-setup --install-example auto-approve-readonly
```

### 「チームで統一したい」

```bash
npx cc-safe-setup --team
git add .claude/ && git commit -m "chore: add hooks"
```

### 「CLAUDE.mdのルールが無視される」

```bash
npx cc-safe-setup --from-claudemd
```

### 「自然言語でルールを追加したい」

```bash
npx cc-safe-setup --guard "本番DBに触るな"
```

### 「何が起きたか確認したい」

```bash
npx cc-safe-setup --replay    # ブロック履歴
npx cc-safe-setup --analyze   # セッション分析
```

## 判断フローチャート

迷ったらこの順で考える:

```
Claude Codeを初めて使う？
  → YES → npx cc-safe-setup（8個の基本hookで始める）
  → NO ↓

自律モード（放置運転）する？
  → YES → npx cc-safe-setup --shield（最大防御）
  → NO ↓

特定の問題が起きた？
  → rm -rf事故 → --install-example rm-safety-net
  → 権限プロンプトがうるさい → --install-example auto-approve-readonly
  → CLAUDE.mdが無視される → --from-claudemd
  → テストなしでpushされた → --install-example test-before-push
  → .envがコミットされた → 基本hookに含まれる（secret-guard）

上記のどれでもない？
  → npx cc-safe-setup --suggest（プロジェクトを分析して推奨）
```

## パフォーマンスTip: ifフィールド（v2.1.85+）

hookが多いとプロセス起動のオーバーヘッドが気になる。v2.1.85の`if`フィールドで、不要な起動をゼロにできる。

```json
{
  "type": "command",
  "if": "Bash(git push *)",
  "command": "~/.claude/hooks/test-before-push.sh"
}
```

`if`で指定したパターンにマッチしない場合、hookプロセスは起動しない。`ls`でgit専用hookが起動することがなくなる。詳細は第13章「高度なhookパターン」を参照。

## 次のセッションまでに

1. `npx cc-safe-setup --shield`を実行（まだなら）
2. `npx cc-safe-setup --verify`で動作確認
3. `npx cc-safe-setup --suggest`でリスク確認
4. 気になるhookを`--install-example`で追加

425個全ての一覧は前章「hookカタログ」を参照。
