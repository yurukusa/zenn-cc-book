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

## スキルの段階的導入

チームに一度に10個のスキルを投入しても使われない。段階的に導入する。

**Week 1: 1個だけ入れる**

最も効果が明白なスキル（例: deploy-helper）を1つ追加。チームSlackで「これ使ってみて」と共有。

**Week 2: フィードバックを反映**

「こういう場合にうまく動かない」という報告が来る。SKILL.mdを修正してコミット。

**Week 3: 2個目を追加**

1個目が定着してから2個目を入れる。定着の基準は「誰かが自分で使った」。

## スキルが競合するとき

チームメンバーが`~/.claude/skills/deploy-helper/`に個人版、リポジトリに`.claude/skills/deploy-helper/`がある場合、**両方がロードされる**。

問題が起きるのは、両方が同じトリガー（例: 「デプロイして」）に反応するとき。Claude Codeはどちらを使うか不定。

**対策**: チーム共有スキルはリポジトリ版を正とし、個人版は別名にする。

```
# 個人版（名前を変えて競合を避ける）
~/.claude/skills/my-deploy/SKILL.md

# チーム版（正式名）
.claude/skills/deploy-helper/SKILL.md
```

## スキルのテンプレート

新しいスキルを作るとき、以下のテンプレートから始めると統一感が出る。

```markdown
# SKILL.md

## Description
何をするスキルか（1行）

## Triggers
- 「○○して」と言われたとき
- /skill-name で起動されたとき

## Steps
1. ...
2. ...
3. ...

## References
- references/api-docs.md（API仕様）
```

`references/`ディレクトリに外部ドキュメントを置くと、スキル実行時にClaude Codeが参照できる。SKILL.md自体を肥大化させずに詳細情報を渡す方法だ。
