---
title: "第7章 Coordination——複数エージェントが協調する仕組み"
free: false
---

# 第7章 Coordination——複数エージェントが協調する仕組み

1つのCCでできることには限界がある。調査、実装、レビューを並列で動かすには、エージェント間の協調が必要だ。

## チェック18: 意思決定を記録・警告する

**失敗例**: CCが「本番サーバーに直接pushします」と言って実行した。問題が起きた後にログを見たら、「なぜそうしたか」の記録が何もなかった。

**根本原因**: 重要な決定がどこにも記録されていなかった。

**対策**: `hooks/decision-warn.sh` を PreToolUse に追加する。

**[decision-warn.sh を入手](https://github.com/yurukusa/claude-code-hooks/blob/main/hooks/decision-warn.sh)**

このhookは、「取り返しのつかない操作」を検知した時に警告ログを出力する:

```bash
# 警告が記録される例
[DECISION] Bash: git push origin main
  → Reason: deployment to production
  → Logged to: ~/.claude/decision-log.jsonl
```

decision-log.jsonlを読めば「いつ、何を、なぜやったか」が全部残る。何か起きた時の根本原因分析に使える。

## チェック19: 並列タスクを安全に分配する

**失敗例**: 調査と実装を同時にやらせようとした。2つのCCセッションが同じファイルを書き換えて、どちらかの変更が消えた。

**根本原因**: 並列実行時のファイル競合を考慮していなかった。

**対策**: サブエージェント使用時のルールをCLAUDE.mdに明記する。

```markdown
## 並列タスクのルール

### 並列OKなタスク
- 調査・探索（ファイルを読むだけ）
- 異なるファイルへの書き込み
- 外部API呼び出し（独立したもの）

### 並列NGなタスク（順番に実行）
- 同じファイルへの書き込み
- DBマイグレーション
- デプロイ操作

### 並列実行時の命名規則
- サブエージェントには作業範囲を限定して渡す
- 「このファイルだけ触っていい」を明示する
- 完了したら main に報告させる
```

この分類を意識するだけで、並列実行時のファイル競合がゼロになった。

## チェック20: 学習ログで同じミスを繰り返さない

**失敗例**: 先週のセッションで「Twitter予約投稿のReact select stateが更新されない」問題に2時間かけた。今週また同じ問題にハマって、また2時間かけた。

**根本原因**: 「ミスから学ぶ」仕組みがなかった。

**対策**: `tasks/lessons.md` を `~/ops/` に配置して、修正を受けるたびに更新する。

```markdown
# ~/ops/lessons.md

## 2026-02-28 Twitter予約投稿のReact select
**症状**: select値を変更してもReact stateに反映されない
**原因**: ReactがvalueをStateで管理しているため、DOMを直接操作しても無効
**解決**: 今日のツイートが0本なら即時投稿に切り替える。React UIを直接操作しようとするな
**再発防止**: 予約投稿は翌日分を前日に設定しておく

## 2026-02-27 Zenn API PUTで全文消失
**症状**: PUTしたら記事の本文が空になった
**原因**: GETなしでPUTしたため、既存フィールドが全部消えた
**解決**: 常にGET→加工→PUTの3ステップ
**再発防止**: ZennはPUT前のGETが必須。このルールをhooksに書いておく
```

lessons.mdに蓄積されるほど、CCは同じミスを繰り返さなくなる。

**[CLAUDE-autonomous.md テンプレート](https://github.com/yurukusa/claude-code-hooks/blob/main/templates/CLAUDE-autonomous.md)** に「セッション開始時にlessons.mdを読む」ルールが含まれている。

---

## Coordination を修正した後

```
  ▸ Coordination
    [PASS] Decision audit trail records important actions
    [PASS] Parallel task rules prevent file conflicts
    [PASS] Lessons log prevents repeat mistakes
    100%
```

Coordinationは「CCが賢くなる仕組み」だ。ミスが記録され、学習され、次のセッションに引き継がれる。

---

次章: Putting It Together——全部つなぎ合わせる
