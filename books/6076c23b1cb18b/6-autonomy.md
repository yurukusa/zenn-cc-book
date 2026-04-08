---
title: "第6章 Autonomy——5時間idle状態で止まっていたCCを24時間自走させるまで"
free: false
---

# 第6章 Autonomy——5時間idle状態で止まっていたCCを24時間自走させるまで

Safety Guards、Code Quality、Monitoring、Recoveryをカバーした。ここから上の3章は「CCをもっと自律的にする」仕組みだ。

## チェック15: タスクキューで「次に何をするか」を明示する

**失敗例**: CCが1つのタスクを終えた。次に何をすればよいかわからなくなって止まった。ぐらすが確認したら、5時間idle状態だった。

**根本原因**: 「次のタスク」がどこにも書いていなかった。

**対策**: 以下のようなタスクキューを `~/ops/` に配置する。

```yaml
# ~/ops/task-queue.yaml
tasks:
  - id: write-zenn-chapter-5
    status: done
    priority: high

  - id: write-zenn-chapter-6
    status: in_progress
    priority: high

  - id: qiita-dry-article
    status: pending
    priority: medium
    blocked_until: "2026-03-01"  # 1日1本ルール

  - id: devto-ci-guide
    status: pending
    priority: medium
    blocked_until: "2026-03-03"
```

CLAUDE.mdに記載する:

```markdown
## セッション開始時の手順
1. ~/ops/task-queue.yaml を確認
2. status: in_progress のタスクを再開
3. なければ status: pending かつ blocked_until が過去のタスクを開始
4. 全部完了していたら新しいタスクを自分で考えて追加する
```

タスクキューが「AIの仕事リスト」として機能するようになってから、idle停止がほぼゼロになった。

## チェック16: 「人間に聞かない」ルールを徹底する

**失敗例**: 「どのファイルを修正しますか？」「確認してから進めます」というメッセージが増えた。CCが毎回人間の承認を求めていた。自律稼働が実質止まっていた。

**根本原因**: 判断ルールがなく、不確かなことは全部人間に聞く設計だった。

**対策**: `hooks/no-ask-human.sh` を PreToolUse に追加する。

**入手方法**: `npx cc-safe-setup --install-example no-ask-human`（[cc-safe-setup](https://github.com/yurukusa/cc-safe-setup)）

このhookは、「人間の入力を要求する可能性があるコマンド」を検知してブロックする。

さらにCLAUDE.mdに判断ルールを明記する:

```markdown
## 自律判断ルール（聞かずに決める）

| 場面 | 正しい行動 |
|------|-----------|
| 技術選択（ライブラリ等） | 標準的なものを選ぶ |
| 実装の詳細 | 既存コードの規約に従う |
| 「どちらがいいか」系 | 客観的に比較し決定 |
| エラーの対処 | 自分で調査・修正（3回まで）|
| 不明な仕様 | 一般的な慣例に従う |

## 例外（これだけは聞く）
- 課金が発生する操作
- 不可逆なデータ削除
- 外部への公開（push、投稿）
- セキュリティリスクがある変更
```

「聞かずに決める範囲」を明確にしてから、CCの判断速度が上がった。

## チェック17: 永続状態ファイルでセッション間を引き継ぐ

**失敗例**: コンテキストがcompactされた後、CCが「何をしていたかわからなくなった」。同じ作業を最初からやり直した。

**根本原因**: 「今何をしているか」が会話コンテキストにしか存在しなかった。

**対策**: 以下のようなミッションファイルを `~/ops/` に配置する。

```markdown
# ~/ops/mission.md

## 現在のフォーカス
Zenn Book「Claude Code本番環境ガイド」第6章執筆中

## 完了済み
- [x] cc-health-check CLIにGitHub直リンク追加
- [x] Zenn Book 第1-5章完成
- [x] KPI定義書作成

## 次のアクション
1. 第7章 Coordination を書く
2. 第8章 Putting It Together を書く
3. content-manifest.yaml を更新する

## ブロッカー
- DRY記事: 1日1本ルールにより3/1まで待機
- Zenn Book出版: ぐらす手動（Zennダッシュボードで「本を作成」が必要）
```

コンパクト後、新セッションでmission.mdを読めば即座に状態を回復できる。

---

## Autonomy を修正した後

```
  ▸ Autonomy
    [PASS] Task queue defines next actions
    [PASS] No-ask-human prevents unnecessary interruptions
    [PASS] Persistent state enables session continuity
    100%
```

Autonomyの3チェックがそろうと、「ぐらすが1日1回チェックするだけで24時間CCが動き続ける」状態に近づく。

---

次章: Coordination——複数エージェントが協調する仕組み
