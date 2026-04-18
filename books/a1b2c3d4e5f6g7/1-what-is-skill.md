---
title: "Skillsとは何か——CLAUDE.mdとの違い"
free: true
---

Claude CodeにはAIの挙動を制御する仕組みが3つある。

1. **CLAUDE.md** — プロジェクトルール。「テストを書け」「mainにpushするな」
2. **Hooks** — 実行時の安全装置。危険なコマンドをブロックする
3. **Skills** — 専門知識の注入。「この手順でデプロイしろ」「このAPIはこう使え」

CLAUDE.mdは「何をするか」、Hooksは「何をしてはいけないか」、Skillsは「どうやってやるか」。

## Skillsの仕組み

Skillsは`~/.claude/skills/`ディレクトリに置く。各スキルは以下の構造を持つ。

```
~/.claude/skills/
  my-skill/
    SKILL.md          ← メインファイル（常に読み込まれるフロントマター）
    references/       ← 必要時のみ読み込まれる詳細情報
      api-docs.md
      error-patterns.md
```

### フロントマター

SKILL.mdの先頭にあるYAMLブロックがフロントマター。ここに書いた内容は**常に**Claude Codeのコンテキストに含まれる。

```yaml
---
name: deploy-helper
description: Production deployment workflow
trigger: When the user mentions deploying, releasing, or shipping
tools:
  - Bash
  - Read
---
```

重要なのは`trigger`。これがスキルの発火条件になる。ユーザーが「デプロイして」と言ったら、このスキルが読み込まれる。

### SKILL.md本文

フロントマターの下に書く本文は、スキルが発火したときだけ読み込まれる。手順、コード例、注意事項をここに書く。

### references/

さらに詳細な情報（API仕様、エラーパターン、長い設定例）は`references/`ディレクトリに置く。SKILL.mdの本文から`references/api-docs.md`を参照すると、必要なときだけ読み込まれる。

## なぜ3層に分けるのか

Claude Codeには**コンテキストウィンドウの制限**がある。全てのスキルの全内容を常に読み込んでおくと、コンテキストが圧迫されてパフォーマンスが落ちる。

Progressive Disclosureの3層構造:
- **第1層（フロントマター）**: 常に読み込む → 短く、トリガー明示。数百バイト
- **第2層（SKILL.md本文）**: スキル発火時に読み込む → 核心の手順のみ。数キロバイト
- **第3層（references/）**: 必要時のみ → 詳細・API仕様・エラーパターン

この構造を守るだけで、スキルのトークン消費を40%以上削減できた実績がある。

## CLAUDE.mdとの使い分け

| | CLAUDE.md | Skills |
|---|---|---|
| 読み込みタイミング | 常に | トリガー発火時 |
| 用途 | ルール・制約 | 手順・知識 |
| 例 | 「テストを書け」 | 「こうやってテストを書け」 |
| 長さ | 短い方がいい | 3層で管理 |

ルールはCLAUDE.md、手順はSkills。迷ったら「それはルールか手順か」で判断する。
:::message
**トークン消費も気になる？**
[Token Checkup（無料）](https://yurukusa.github.io/cc-safe-setup/token-checkup.html)で30秒診断 → [トークン消費を半分にする実践ガイド（¥2,500）](https://zenn.dev/yurukusa/books/token-savings-guide)
:::
