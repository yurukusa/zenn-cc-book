---
title: "第1章 自分のスコアを確認する"
free: true
---

# 第1章 自分のスコアを確認する

まず現在地を知る。診断なしに治療はできない。

## 2通りの診断方法

### 方法1: ブラウザ版（チェックリスト）

[https://yurukusa.github.io/cc-health-check/](https://yurukusa.github.io/cc-health-check/)

インストール不要。20問のチェックリストに答えるだけでスコアが出る。

### 方法2: CLI版（ローカルファイルを直接スキャン）

```bash
npx cc-health-check
```

`~/.claude/settings.json`と`CLAUDE.md`を自動スキャン。より正確に現状を反映する。

## スコアの読み方

| スコア | グレード | 意味 |
|--------|----------|------|
| 80-100 | Production Ready | 長時間自律稼働に耐えられる設定 |
| 60-79  | Getting There | 基本的な保護はある。重要なものがまだ欠けている |
| 35-59  | Needs Work | 複数の重要な項目が未設定。危険な状態で動かしている |
| 0-34   | Critical | ほぼ無防備。すぐに修正が必要 |

## 6つのディメンション

スコアは6つのカテゴリから計算される。

### Safety Guards（最重要）
- 危険なコマンドをブロックするhookがあるか
- APIキーがCLAUDE.mdに直書きされていないか
- main/masterへの直pushが防止されているか
- エラーが存在する状態で外部APIを叩かないか

**ここが低いと**: ファイル消失、APIコスト暴走、本番事故

### Code Quality
- ファイル編集後に構文チェックが走るか
- エラーコードが自動検知されるか
- 完了の定義（DoD）があるか
- AIが自分のアウトプットを検証するか

**ここが低いと**: 壊れたコードが本番に入る、エラーに気づかない

### Monitoring
- コンテキスト残量が監視されているか
- 活動ログが記録されているか
- 日次サマリーが生成されているか

**ここが低いと**: 何が起きているかわからない、気づいた時には手遅れ

### Recovery
- 変更前にgitバックアップが作られるか
- ウォッチドッグがアイドル/フリーズを検知するか
- ループ検知の仕組みがあるか

**ここが低いと**: 失敗からの復旧ができない

### Autonomy
- タスクキューからAIが自律実行できるか
- 不要な質問が抑制されているか
- セッション再起動後も作業継続できるか

**ここが低いと**: 常に人間が必要で、自律稼働できない

### Coordination
- 意思決定のログが残るか
- 複数AIインスタンスが連携できるか
- 教訓が蓄積されるか

**ここが低いと**: 同じ失敗を繰り返す

## よくある初回スコア

初めて診断すると、多くの人が50〜70点になる。

これは正常だ。Claude Codeのデフォルト設定は「対話セッション」向けであり、「自律稼働」向けではない。

私の初回スコアも63点だった。

```
  Score: 63/100 — Getting There

  Top fixes:
    → Add a PreToolUse hook that blocks destructive commands.
      ↳ https://github.com/yurukusa/claude-code-hooks/blob/main/hooks/branch-guard.sh

    → Add an error-tracker that prevents publishing when errors exist.
      ↳ https://github.com/yurukusa/cc-safe-setup/blob/main/examples/error-memory-guard.sh
```

## 今すぐできる3つのクイックウィン

スコアが何点であれ、この3つは今すぐ効果がある:

### 1. 安全hookを入れる（30秒）
```bash
npx cc-safe-setup
```
rm -rf / のブロック、mainへのpush防止、シークレット漏洩防止が即座に有効になる。

### 2. 権限プロンプトを80%減らす（10秒）
```bash
npx cc-safe-setup --install-example auto-approve-readonly
```
cat、ls、git logなどの読み取り専用コマンドを自動承認。

### 3. hookが動くか確認する（10秒）
```bash
npx cc-safe-setup --doctor
```
13項目の自動診断。全て✓なら安心。

---

次章からは、各カテゴリの修正方法を具体的に解説する。

## 次のステップ

まず自分のスコアを確認してから次の章へ進んでほしい。何が低いかがわかれば、読むべき章が絞られる。

---

次章: Safety Guards——最初に直すべき4つのチェック
