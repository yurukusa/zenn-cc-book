---
title: "チームでスキルを共有する"
---

個人で作ったスキルをチームで共有する方法。

## リポジトリにスキルを組み込む

スキルをリポジトリの`.claude/skills/`に置くと、そのリポジトリで作業する全員がスキルを使える。

```
my-project/
  .claude/
    skills/
      deploy-helper/
        SKILL.md
        references/
          api-docs.md
  src/
  package.json
```

### コミット方法

```bash
git add .claude/skills/
git commit -m "add deploy-helper skill for team"
git push
```

これだけで、チームメンバーがこのリポジトリでClaude Codeを使うときに`deploy-helper`スキルが利用可能になる。

## グローバル vs プロジェクトスキル

| 場所 | スコープ | 用途 |
|------|----------|------|
| `~/.claude/skills/` | 全プロジェクト | 個人のワークフロー |
| `.claude/skills/` | このリポジトリだけ | チーム共有の手順 |

個人のスキルは`~/.claude/skills/`に置く。チームで共有すべきスキルは`.claude/skills/`に置いてコミットする。

## スキルのバージョン管理

スキルは通常のコードと同じようにバージョン管理される。変更履歴はgit logで追える。

```bash
git log --oneline .claude/skills/deploy-helper/
```

## チームメンバーへの共有

スキルを追加したら、READMEやCLAUDE.mdに記載しておく。

```markdown
## 利用可能なスキル

- `deploy-helper`: 「デプロイして」でプロダクションデプロイを実行
- `code-review`: 「レビューして」で自動コードレビュー
```

スキルの存在を知らなければ使われない。ドキュメントに書いておくことが重要。
