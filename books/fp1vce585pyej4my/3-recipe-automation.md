---
title: "レシピ1-3: ワークフロー自動化スキル"
---

この章では、ワークフロー自動化型のスキルを3つ作る。

## レシピ1: 記事投稿スキル

特定のプラットフォーム（Zenn、Qiita、dev.to）への記事投稿手順をスキル化する。

### なぜスキルにするのか

記事投稿は毎回同じ手順:
1. ドラフトファイルを読む
2. プラットフォームのAPIに合わせてフォーマットする
3. 投稿する
4. 結果を確認する

これを毎回説明するのは無駄。スキルにすれば「投稿して」の一言で動く。

### SKILL.md

```markdown
---
name: post-article
description: Publish articles to Zenn/Qiita/dev.to
trigger: When user says "投稿して", "記事を公開", or "post article"
tools:
  - Bash
  - Read
  - Write
---

# 記事投稿

1. `~/content-manifest.yaml` からターゲット記事を特定
2. ドラフトファイルを読み込む
3. プラットフォーム別に投稿
   - Zenn: GitHub pushで自動反映
   - Qiita: API経由（`references/qiita-api.md`参照）
   - dev.to: API経由（`references/devto-api.md`参照）
4. 投稿後にスクリーンショットで確認

投稿前チェック:
- scheduled_dateが今日以前か確認
- 重複投稿がないかAPI/ブラウザで確認
- ファクトチェック済みか確認
```

### references/qiita-api.md

```markdown
# Qiita API

## 記事投稿
POST https://qiita.com/api/v2/items
Headers: Authorization: Bearer $QIITA_TOKEN
Body:
{
  "title": "記事タイトル",
  "body": "マークダウン本文",
  "tags": [{"name": "ClaudeCode"}],
  "private": false
}

## 記事更新
PATCH https://qiita.com/api/v2/items/:id
同じBodyフォーマット。GETで既存データを取得してから更新すること。
GETなしのPUTは絶対禁止（過去に2回事故）。
```

### ポイント

- triggerに日本語と英語の両方を入れる（ユーザーがどちらで指示するか分からない）
- 「GETなしのPUTは禁止」のような**教訓**はSKILL.md本文に書く（常に見えるべき制約）
- API仕様の詳細はreferences/に分離（毎回読む必要はない）

## レシピ2: リリースチェックリスト

新バージョンをリリースするときのチェックリストをスキル化する。

### SKILL.md

```markdown
---
name: release-checklist
description: Version release workflow with safety checks
trigger: When user mentions "リリース", "release", "publish", or "バージョンアップ"
tools:
  - Bash
  - Read
---

# リリースチェックリスト

実行前:
- [ ] 全テスト通過: `npm test`
- [ ] lint通過: `npm run lint`（あれば）
- [ ] CHANGELOG更新済み
- [ ] package.jsonのversion更新済み

実行:
- [ ] `git tag v{version}`
- [ ] `git push origin v{version}`
- [ ] `npm publish`（npmパッケージの場合）

実行後:
- [ ] GitHubリリースノート作成
- [ ] 関連ドキュメント更新

注意: テスト全パスを確認せずにリリースしない。
```

### ポイント

- チェックリスト形式はClaudeが1項目ずつ実行してくれる
- references/は不要（シンプルなワークフローなので2層で十分）
- 「テスト全パスを確認せずにリリースしない」のような**ガードレール**を本文に書く

## レシピ3: 環境診断スキル

開発環境の状態を診断するスキル。

### SKILL.md

```markdown
---
name: env-check
description: Development environment diagnostic
trigger: When user asks "環境チェック", "env check", or "何か壊れてない？"
tools:
  - Bash
---

# 環境診断

以下を順番にチェック:

1. **認証**: `gh auth status`, `npm whoami`
2. **ディスク**: `df -h /` — 90%超えたら警告
3. **Git**: `git status` — uncommitted changesの有無
4. **テスト**: `npm test` — 全パスか
5. **依存関係**: `npm outdated` — セキュリティ更新の有無

全チェック完了後、結果をサマリーで報告。
問題があれば修正案を提示。
```

### ポイント

- triggerに「何か壊れてない？」のような自然な表現を入れる
- 診断結果は**サマリーで報告**と明記（長い出力をそのまま出すな、という指示）
- 各チェック項目に**判断基準**を書く（「90%超えたら警告」）
