---
title: "レシピ4-6: 外部サービス連携スキル（GitHub/CDP/API）"
---

外部サービスと連携するスキルは、GitHub CLI、ブラウザ操作（CDP）、API呼び出しなど、Claude Codeの外側にある情報源やツールとの橋渡しを自動化する。

## レシピ4: GitHub Issue管理スキル

GitHub Issueの確認、回答、クローズをスキル化する。

### SKILL.md

```markdown
---
name: issue-manager
description: GitHub Issue triage and response
trigger: When user mentions "Issue確認", "issue check", or "Issueに回答"
tools:
  - Bash
---

# Issue管理ワークフロー

## 確認
gh search issues --repo {repo} --state open --sort reactions --limit 10

## トリアージ基準
- 10+ reactions: 高優先度
- hookで解決可能: 回答候補
- 感謝/報告: フォローアップ不要

## 回答ルール
- 回答の目的は「人助け」。自分のツールの宣伝ではない
- 根本原因を診断してから回答する
- 回答後は `~/bin/gh-comment` を使う（自動購読解除のため）

API詳細は `references/gh-api.md` 参照。
```

### ポイント

- 「回答の目的は人助け」のようなポリシーはSKILL.md本文に書く
- gh CLIコマンドの詳細はreferences/に分離

## レシピ5: ブラウザ操作スキル

CDP（Chrome DevTools Protocol）を使ったブラウザ操作をスキル化する。

### SKILL.md

```markdown
---
name: browser-helper
description: Browser automation via CDP
trigger: When user asks to check a webpage, take screenshot, or interact with browser
tools:
  - Bash
---

# ブラウザ操作

## 基本操作
- スクリーンショット: `cdp-bridge screenshot /tmp/screenshot.png`
- ページ遷移: `cdp-bridge navigate "https://example.com"`
- クリック: `cdp-bridge click x y`（物理クリック優先）
- テキスト入力: `cdp-bridge type "text"`

## 重要ルール
- **物理クリックが第一選択**。JSの`.click()`はReact/SPAで保存トリガーが発火しない
- 操作後は必ずスクリーンショットで結果を確認
- `cdp-bridge` コマンドに統一。生WebSocket CDP構築は禁止

クリックが効かない場合の対処は `references/cdp-troubleshoot.md` 参照。
```

### ポイント

- 「物理クリックが第一選択」のような**過去の教訓**はSKILL.md本文に書く
- トラブルシューティングはreferences/に分離（普段は不要）

## レシピ6: API呼び出しスキル

外部APIの呼び出しパターンをスキル化する。

### SKILL.md

```markdown
---
name: api-client
description: External API call patterns
trigger: When user asks to call an API, fetch data, or make HTTP requests
tools:
  - Bash
---

# API呼び出し

## パターン
1. GET: `curl -s "url" | jq '.field'`
2. POST: `curl -s -X POST -H "Content-Type: application/json" -d '{"key":"value"}' "url"`
3. 認証付き: `-H "Authorization: Bearer $TOKEN"`

## 安全ルール
- 認証トークンは環境変数から取得（`~/.credentials/` 参照）
- コードにハードコードしない
- POST/PUT/DELETEは実行前に確認を取る

## レスポンス処理
- JSON: `jq` でパース
- エラー: HTTPステータスコードで分岐

プラットフォーム固有のAPI仕様は `references/` に追加していく。
```

### ポイント

- 汎用パターンをSKILL.md本文に、特定APIの仕様はreferences/に分離
- 「認証トークンをハードコードしない」のようなセキュリティルールは本文に書く
