---
title: "上級パターン——スキルの最適化"
---

基本を押さえたら、さらに効率を上げるパターン。

## パターン1: 条件分岐付きtrigger

同じスキルで複数のワークフローを切り替える。

```yaml
---
name: project-manager
trigger: |
  When user mentions:
  - "新規プロジェクト" or "init" → run setup workflow
  - "リリース" or "release" → run release workflow
  - "クリーンアップ" or "cleanup" → run cleanup workflow
---
```

SKILL.md本文で条件に応じた手順を分岐させる:

```markdown
## ワークフロー別手順

### 新規プロジェクト
1. ディレクトリ作成
2. git init
3. テンプレート展開

### リリース
1. テスト実行
2. バージョン更新
3. タグ作成とpush

### クリーンアップ
1. 未使用ファイル削除
2. 依存関係整理
3. git gc
```

## パターン2: スキル間の連携

複数のスキルを組み合わせて使う。

例: リリーススキルがコードレビュースキルを呼び出す。

```markdown
# リリース手順

1. まず「レビューして」でコードレビュースキルを実行
2. レビュー結果に問題がなければリリースに進む
3. テスト実行 → バージョン更新 → publish
```

スキル間の連携は、SKILL.md本文で「別のスキルのトリガーワード」を使うだけ。Claude Codeが自動的に対応するスキルを発火させる。

## パターン3: 動的references

references/の内容をスクリプトで動的に生成する。

```markdown
# 環境診断

現在のプロジェクトの状態は `references/current-state.md` を参照。

> このファイルは `npm run check-state > references/current-state.md` で更新できる。
```

静的なドキュメントではなく、コマンドの出力をreferences/に保存しておく。スキル発火時に最新の状態を参照できる。

## パターン4: フロントマターの最適化

フロントマターは常にコンテキストに含まれるため、最小限にする。

**悪い例:**
```yaml
---
name: deploy-helper
description: This skill helps with deploying applications to production environments including staging and development, handling rollbacks, health checks, and notification to team members via Slack and email
trigger: When the user wants to deploy, release, ship, publish, push to production, or mentions anything related to deployment workflows
tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---
```

**良い例:**
```yaml
---
name: deploy-helper
description: Production deployment with rollback
trigger: deploy, release, ship, リリース, デプロイ
tools:
  - Bash
  - Read
---
```

descriptionは1行。triggerはキーワードだけ。toolsは実際に使うものだけ。

## パターン5: スキルの無効化

一時的にスキルを使わせたくない場合。

```bash
# スキルを無効化（ファイル名を変更）
mv ~/.claude/skills/deploy-helper/SKILL.md ~/.claude/skills/deploy-helper/SKILL.md.disabled

# 再有効化
mv ~/.claude/skills/deploy-helper/SKILL.md.disabled ~/.claude/skills/deploy-helper/SKILL.md
```

SKILL.mdが存在しないスキルは読み込まれない。

## まとめ

1. 条件分岐で1つのスキルに複数ワークフローを入れる
2. スキル間連携はトリガーワードで実現する
3. references/は動的に生成できる
4. フロントマターは最小限にする
5. 無効化はファイル名変更で行う
