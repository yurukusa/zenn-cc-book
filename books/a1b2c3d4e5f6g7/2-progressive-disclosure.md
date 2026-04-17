---
title: "Progressive Disclosureの実装——SKILL.mdを3層に分離する"
---

前章で3層構造の概念を説明した。この章では、実際に手を動かしてスキルを3層に分離する。

## Before: 1ファイルに全部書いた状態

よくある失敗パターン。SKILL.mdに全部書く。

```markdown
---
name: deploy-helper
description: Deployment workflow
trigger: When user mentions deploy
---

# Deploy手順

1. テストを実行する
2. ビルドする
3. デプロイする

## API仕様

POST /api/deploy
Headers: Authorization: Bearer <token>
Body: { "version": "1.2.3", "environment": "production" }
Response: { "status": "ok", "deploy_id": "abc123" }

## エラーパターン

- 403: トークン期限切れ → `/api/auth/refresh`で更新
- 409: 既にデプロイ中 → 5分待って再試行
- 500: サーバーエラー → ログを確認
- 502: ゲートウェイエラー → CDNキャッシュをパージ

## 設定ファイル

```yaml
deploy:
  target: production
  region: ap-northeast-1
  timeout: 300
  rollback: true
  health_check: /api/health
  notifications:
    slack: "#deploys"
    email: ops@company.com
```
```

このSKILL.mdは約2,000バイト。これがスキル未使用時もコンテキストに含まれる。

## After: 3層に分離した状態

```
deploy-helper/
  SKILL.md          ← 400バイト
  references/
    api-docs.md     ← 800バイト（必要時のみ）
    error-patterns.md ← 500バイト（必要時のみ）
    config-example.md ← 300バイト（必要時のみ）
```

### SKILL.md（第1層 + 第2層）

```markdown
---
name: deploy-helper
description: Production deployment workflow with rollback
trigger: When the user mentions deploying, releasing, or shipping to production
tools:
  - Bash
  - Read
---

# Deploy手順

1. `npm test` でテスト実行。失敗したらデプロイしない
2. `npm run build` でビルド
3. `./scripts/deploy.sh production` でデプロイ
4. ヘルスチェック: `curl -f https://api.example.com/health`
5. 失敗したら `./scripts/rollback.sh` で戻す

API仕様の詳細は `references/api-docs.md` を参照。
エラーが出たら `references/error-patterns.md` を確認。
```

### references/api-docs.md（第3層）

```markdown
# Deploy API

POST /api/deploy
Headers: Authorization: Bearer <token>
Body: { "version": "1.2.3", "environment": "production" }
Response: { "status": "ok", "deploy_id": "abc123" }
```

### references/error-patterns.md（第3層）

```markdown
# エラー対応表

| Code | 原因 | 対処 |
|------|------|------|
| 403 | トークン期限切れ | `/api/auth/refresh`で更新 |
| 409 | 既にデプロイ中 | 5分待って再試行 |
| 500 | サーバーエラー | ログ確認 |
| 502 | ゲートウェイ | CDNキャッシュパージ |
```

## 効果

- **フロントマターの常時消費**: 2,000バイト → 400バイト（80%削減。全体のトークン消費削減は約40%）
- **情報の欠落**: なし（references/に全て保持）
- **必要なとき**: 「エラーが出た」→ references/error-patterns.md が読み込まれる

## 分離の判断基準

何を第2層に残し、何を第3層に移すか。

**第2層（SKILL.md本文）に残すもの:**
- 手順の概要（番号付きリスト）
- 最も使う1-2個のコマンド
- 判断基準（「テスト失敗ならデプロイしない」）

**第3層（references/）に移すもの:**
- API仕様の詳細
- エラーコードの一覧
- 設定ファイルのテンプレート
- 滅多に使わないオプション

判断に迷ったら、「この情報がないと手順を始められないか？」と考える。始められるなら第3層。

:::message
**トークン消費の体系的な削減**: Progressive DisclosureはCLAUDE.md最適化の1つだが、他にもhookによる自動制御、コンテキスト管理、ワークフロー設計で大幅に削減できる。800時間の実測データと設定テンプレートは[Token Book](https://zenn.dev/yurukusa/books/token-savings-guide)（¥2,500）に収録。無料の[Token Checkup](https://yurukusa.github.io/cc-safe-setup/token-checkup.html)で先に自分の消費パターンを診断できる。
:::
