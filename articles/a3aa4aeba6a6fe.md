---
title: "Claude Codeに「聞くな、自分で決めろ」と教えたら、5時間のidle停止がゼロになった"
emoji: "🤖"
type: "tech"
topics: ["ai", "automation", "devtools", "claudecode"]
published: true
---

Claude Codeが止まっていた。5時間。

「どのファイルを修正しますか？」「確認してから進めます」——こういうメッセージを残して、ずっと待っている。誰も見ていない夜中に。

自律運行を目指してClaude Codeを走らせていた。108時間の無人稼働。そのうち、この「止まって聞いてくる」問題で失われた時間がどれだけあったか。数えたくない。

3つの仕組みで解決した。タスクキュー、判断ルール、永続状態ファイル。

この記事では、その3つを具体的に書く。

:::message
この記事は2026年3月時点の情報に基づいている。
Claude Code Hooksについては[公式ドキュメント](https://code.claude.com/docs/en/hooks)を参照。
:::

## 問題1: 「次に何をするか」がわからなくて止まる

1つのタスクが終わった。CCは次に何をすればいいかわからない。idle状態のまま5時間が経過した。

原因は単純。「次のタスク」がどこにも書いていなかった。

## 対策: task-queue.yaml

\`~/ops/task-queue.yaml\`にタスクの一覧を置く。

\`\`\`yaml
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
    blocked_until: "2026-03-01"

  - id: devto-ci-guide
    status: pending
    priority: medium
    blocked_until: "2026-03-03"
\`\`\`

CLAUDE.mdにセッション開始手順を書く。

\`\`\`markdown
## セッション開始時の手順
1. ~/ops/task-queue.yaml を確認
2. status: in_progress のタスクを再開
3. なければ status: pending かつ blocked_until が過去のタスクを開始
4. 全部完了していたら新しいタスクを自分で考えて追加する
\`\`\`

これを入れた瞬間、idle停止がほぼゼロになった。「仕事リスト」があるだけで、AIは自分で動き続ける。

:::details CC補足: なぜYAMLか
JSONでもTOMLでも構わないが、YAMLはコメントが書ける。\`blocked_until\`のような理由付きの制約を1行で説明できるのが利点。Claude Codeはどの形式でも読める。
:::

## 問題2: 全部聞いてくる

「どのライブラリを使いますか？」「この設計でよいですか？」

自律運行なのに毎回聞いてくる。1回聞かれるたびに、次にぐらす（自分）が返事するまでCCは止まる。就寝中なら8時間。

## 対策: 「聞かずに決める範囲」を明確にする

CLAUDE.mdに判断ルールを明記した。

\`\`\`markdown
## 自律判断ルール（聞かずに決める）

| 場面 | 正しい行動 |
|------|-----------|
| 技術選択（ライブラリ等） | 標準的なものを選ぶ |
| 実装の詳細 | 既存コードの規約に従う |
| 「どちらがいいか」系 | 客観的に比較し決定 |
| エラーの対処 | 自分で調査・修正（3回まで） |
| 不明な仕様 | 一般的な慣例に従う |

## 例外（これだけは聞く）
- 課金が発生する操作
- 不可逆なデータ削除
- 外部への公開（push、投稿）
- セキュリティリスクがある変更
\`\`\`

さらに、[no-ask-human.sh](https://github.com/yurukusa/cc-safe-setup/blob/main/hooks/no-ask-human.sh)をPreToolUseフックとして追加した。人間の入力を要求するパターンを検知してブロックする。

「聞くな。自分で決めろ。ただし金と安全に関わることだけは聞け。」

このルールを入れてから、CCの判断速度が目に見えて上がった。

## 問題3: compactされると「何をしていたか」忘れる

Claude Codeにはコンテキストウィンドウの上限がある。長いセッションだとcompact（圧縮）が入る。圧縮後、CCが「何をしていたかわからなくなった」。同じ作業を最初からやり直した。

## 対策: mission.md

\`~/ops/mission.md\`に「今やっていること」を書く。

\`\`\`markdown
# ~/ops/mission.md

## 現在のフォーカス
Zenn Book「Claude Code本番品質ガイド」第6章執筆中

## 完了済み
- [x] cc-health-check CLIにGitHub直リンク追加
- [x] Zenn Book 第1-5章完成

## 次のアクション
1. 第7章 Coordination を書く
2. 第8章 Putting It Together を書く

## ブロッカー
- DRY記事: 1日1本ルールにより3/1まで待機
\`\`\`

CLAUDE.mdに「セッション開始時にmission.mdを読む」と書いておけば、compact後でも新セッションでも即座に状態を回復できる。

:::details CC補足: mission.mdの更新タイミング
意味のある区切り（タスク完了、方針変更、ブロッカー発生）で更新する。毎分更新する必要はない。逆に「完了したらチェックを入れる」を忘れると、次のセッションで同じ作業をやり直す原因になる。
:::

## 3つを組み合わせた結果

| 仕組み | 解決する問題 |
|--------|------------|
| task-queue.yaml | 「次に何をするか」がわからない |
| 自律判断ルール + no-ask-human.sh | 全部聞いてきて止まる |
| mission.md | compact後に「何をしていたか」忘れる |

この3つを入れた後、「自分が1日1回チェックするだけで24時間CCが動き続ける」状態になった。

実際に108時間以上の無人稼働を達成した。その記録は[Qiitaの記事](https://qiita.com/yurukusa/items/db14ed08ca4bf02f4f36)に書いた。

---


Book全体では、Safety Guards（事故防止）、Code Quality（構文エラー自動検出）、Monitoring（行動監視）、Recovery（障害復旧）、Coordination（複数エージェント協調）を含む全9章で、Claude Codeの自律運行に必要な仕組みをカバーしている。


---

:::message
**Claude Codeの安全対策、まだですか？**
`npx cc-safe-setup` — ワンコマンドで `rm -rf` 誤爆・秘密鍵コミット・force pushを防止。9,200+テスト済み。
→ [GitHub](https://github.com/yurukusa/cc-safe-setup)
:::
---
:::message
**Claude Codeの自律運用をもっと深く学びたい方へ**
トークン消費の突然の急増、ファイル消失、無限ループ——700+時間の自律稼働で起きた失敗と対策をまとめました:
[Claude Codeを本番品質にする——実践ガイド](https://zenn.dev/yurukusa/books/6076c23b1cb18b)（¥800・第3章まで無料）

「AIに任せて大丈夫なのか」という不安を持ったまま800時間使い続けた記録は[非エンジニアがClaude Codeを800時間走らせた——失敗と学びの全記録](https://zenn.dev/yurukusa/books/3c3c3baee85f0a19)（¥800・第2章まで無料）に書いた。
:::
