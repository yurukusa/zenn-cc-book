---
title: "205個のhookから選ぶ方法"
---

205個もhookがあると「どれを入れればいいか」迷う。この章で解決する。

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

## 次のセッションまでに

1. `npx cc-safe-setup --shield`を実行（まだなら）
2. `npx cc-safe-setup --verify`で動作確認
3. `npx cc-safe-setup --suggest`でリスク確認
4. 気になるhookを`--install-example`で追加

160個全ての一覧は前章「hookカタログ」を参照。
