---
title: "レシピ7-10: ドキュメント・品質管理スキル"
---

## レシピ7: コードレビュースキル

変更されたコードの品質チェックをスキル化する。

### SKILL.md

```markdown
---
name: code-review
description: Automated code review checklist
trigger: When user asks to review code, check quality, or "レビューして"
tools:
  - Bash
  - Read
  - Grep
---

# コードレビュー

変更されたファイルに対して以下をチェック:

1. **構文チェック**: 言語に応じた構文検証
2. **セキュリティ**: ハードコードされた秘密情報、SQL injection、XSS
3. **テスト**: 変更に対応するテストがあるか
4. **ドキュメント**: 公開APIの変更にドキュメント更新があるか

`git diff --stat` で変更ファイルを特定し、1ファイルずつ確認する。

判断基準は `references/review-checklist.md` 参照。
```

## レシピ8: ドキュメント生成スキル

コードからドキュメントを自動生成するスキル。

### SKILL.md

```markdown
---
name: doc-gen
description: Generate documentation from code
trigger: When user asks to document, generate docs, or "ドキュメント書いて"
tools:
  - Read
  - Write
  - Grep
---

# ドキュメント生成

1. 対象ファイルを読む
2. 公開関数/クラスを抽出
3. 使用例をコードから推測
4. README.mdまたはAPI.mdに書き出す

ルール:
- 存在しない関数をドキュメントに書かない（ファクトチェック必須）
- 使用例はテストコードから引用する（推測で書かない）
- 生成後にソースコードと照合する
```

### ポイント

- 「存在しない関数を書かない」はClaude Codeのハルシネーション対策
- テストコードからの引用を指定することで事実に即したドキュメントになる

## レシピ9: 依存関係チェックスキル

プロジェクトの依存パッケージの健全性とセキュリティを定期的にチェックするスキル。「npm outdated」を手動で打つ代わりに、「依存関係チェックして」の一言で網羅的に確認できる。

### SKILL.md

```markdown
---
name: deps-check
description: Check dependency health and security
trigger: When user mentions "依存関係", "dependencies", "outdated", or "脆弱性"
tools:
  - Bash
---

# 依存関係チェック

1. `npm outdated` — 古いパッケージ一覧
2. `npm audit` — セキュリティ脆弱性
3. `npm ls --depth=0` — インストール済みパッケージ

報告形式:
- 🔴 セキュリティ脆弱性（即対応）
- 🟡 メジャーバージョン遅れ（計画的に更新）
- 🟢 マイナー/パッチ遅れ（余裕があれば）
```

### ポイント

- 報告形式を3段階で指定することで、Claude Codeが優先度を自動判断する
- npm以外（pip、cargo、gem）にも同じパターンを適用できる。コマンド部分だけ差し替えればよい
- `references/`に許容リスト（既知の脆弱性で対応不要なもの）を置くと、ノイズが減る

## レシピ10: パフォーマンス計測スキル

ビルド時間やファイルサイズの劣化を検知するスキル。前回の計測値をreferences/に保存しておき、変化があれば自動で報告する。

### SKILL.md

```markdown
---
name: perf-check
description: Performance measurement and analysis
trigger: When user asks about performance, speed, or "遅い"
tools:
  - Bash
  - Read
---

# パフォーマンス計測

1. **ビルド時間**: `time npm run build`
2. **テスト時間**: `time npm test`
3. **起動時間**: `time node index.js --help`（該当する場合）
4. **ファイルサイズ**: `du -sh dist/`

前回の計測結果と比較し、劣化があれば報告する。
前回の結果は `references/perf-baseline.md` に保存。
```

### ポイント

- `references/perf-baseline.md`に前回値を保存する設計がProgressive Disclosureの実例。ベースライン値はスキル起動時にだけ必要
- 計測結果は「前回比+15%」のように相対値で報告させると、劣化の判断が容易になる
- CIに組み込む場合は、計測結果をJSONで出力させてパイプラインで処理する

## スキル設計の共通パターン

ここまで10個のレシピを見て、共通パターンが見える:

1. **triggerは日本語と英語の両方を入れる**
2. **制約・教訓はSKILL.md本文に書く**（常に見えるべき情報）
3. **詳細・仕様はreferences/に分離する**（必要時のみ）
4. **判断基準を明示する**（「何を良しとするか」をClaude Codeに伝える）
5. **出力形式を指定する**（「サマリーで報告」「チェックリスト形式」）
