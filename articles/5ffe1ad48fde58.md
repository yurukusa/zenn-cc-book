---
title: "Agent Teamsのアーキテクチャ：5エージェント並列運用の実践"
emoji: "🏭"
type: "tech"
topics: ["tech", "claudecode", "agentteams"]
published: true
---

---
title: "Claude Code Agent Teamsで5人チームを起動してゲームスタジオを回した記録"
emoji: "🏭"
type: "tech"
topics: ["claudecode", "ai", "gamedev", "agentteams", "godot"]
published: false
---

## やったこと

Claude Code Agent Teamsの機能を使って、5人のAIエージェントチームを1コマンドで起動し、ゲーム開発・出荷・コンテンツ制作・マーケティング計測を並列で回した。

チーム名は「factory」。メンバーは以下の5名:

| 名前 | 役割 | 担当 |
|------|------|------|
| builder | 開発者 | ゲームのコード改善・新機能実装 |
| designer | デザイナー | スクリーンショット撮影・UI素材・カバー画像 |
| researcher | 調査員 | 市場調査・次のゲーム企画・配信プラットフォーム調査 |
| grower | 成長係 | 全プラットフォームの指標収集・分析・記事ドラフト |
| shipper | 出荷係 | itch.ioページ更新・CrazyGames投稿・記事公開 |

これに加えて team-lead（リーダー）が全体を統括する。合計6エージェント。

## チーム構成ファイル

Agent Teamsは `~/.claude/teams/{チーム名}/config.json` で管理される。各メンバーにはプロンプト（役割の詳細指示）、モデル、色が設定される。

```json
{
  "name": "factory",
  "description": "AIゲームファクトリー＋コンテンツ事業体",
  "members": [
    {
      "name": "builder",
      "agentType": "general-purpose",
      "model": "claude-opus-4-6",
      "color": "blue"
    },
    {
      "name": "designer",
      "agentType": "general-purpose",
      "model": "claude-opus-4-6",
      "color": "green"
    }
    // ... 他3名も同様
  ]
}
```

## タスク分配の仕組み

タスクは `~/.claude/tasks/{チーム名}/` 配下にJSONファイルとして保存される。各タスクにはID、subject、description、owner、status、blockedByがある。

今回のセッションでは17タスクを作成した。タスク間の依存関係も設定できる:

```
#1 [builder] ゲームフィール改善
#2 [designer] スクリーンショット撮影 → #6に必要
#3 [researcher] CrazyGames調査
#4 [researcher] 次のゲーム企画
#5 [grower] 全PF指標収集 → #12に必要
#6 [shipper] itch.ioページ更新 (blocked by #2)
```

`blockedBy` を設定すると、依存タスクが完了するまでそのタスクは着手不可になる。これにより、designerのスクショが完成してからshipperがページ更新する、という順序が自動的に守られる。

## 並列稼働の実態

6エージェントが同時に動く。各自が独立したタスクを持ち、完了したら次のタスクを拾う。

実際に並列で進んでいた作業:
- **builder**: Spell Cascade v0.8.3のヒットストップ・パーティクル改善を実装
- **designer**: ゲームプレイスクリーンショット5枚 + GIF + カバー画像を生成
- **researcher**: CrazyGamesのSDK要件調査 + 次のゲーム「Merge Alchemist」の企画
- **grower**: itch.io / dev.to / Qiita / Zenn / Gumroadの指標を一括収集・分析
- **shipper**: itch.ioページの更新計画を策定

1人が1つのタスクを終えると、TaskListを確認して次の未着手タスクを自分でclaimする。

## メンバー間連携

メンバー同士はSendMessage（DM）でやりとりする。

実際に起きた連携:
1. **designer → shipper**: 「スクショ完成、パスはここ」→ shipperがページ更新に着手
2. **researcher → builder**: 「CrazyGamesはGodot HTML5対応。SDK統合が必要」→ builderが技術要件を把握
3. **grower → team-lead**: 「Qiitaが最強PF。Gumroad全滅。4 Hooksが伸びてる」→ リーダーが戦略判断
4. **researcher → team-lead**: 「次のゲームはMerge Alchemist（錬金マージ）を推奨」→ builderに実装タスク作成

## 実際の成果（1セッション）

### 完了タスク（9/17）
- Spell Cascade v0.8.3 ゲームフィール改善
- スクリーンショット5枚 + GIF + カバー画像
- CrazyGames提出要件の完全調査
- 次のゲーム企画（10→3→1で「Merge Alchemist」選定）
- 全5プラットフォームの指標収集・ファネル分析
- Merge Alchemistの市場調査・デザインパターン分析
- Merge Alchemist UIアセットデザイン
- note記事ドラフト完成
- この記事のドラフト

### 進行中タスク（5/17）
- itch.ioページ更新計画
- Merge Alchemist MVP開発
- Merge Alchemistブランディング
- 4記事のカバー画像作成
- Zenn記事（この記事）

### 数字で見る成果
- **タスク完了率**: 53%（9/17）
- **並列稼働数**: 最大6エージェント同時
- **指標収集**: 5プラットフォーム、30+記事、合計4,000+ビュー
- **新ゲーム企画**: 10案→3案→1案選定完了

## 発見

### 1. タスク依存関係が並列稼働の鍵

`blockedBy` がないと、designerのスクショが完成する前にshipperがページ更新を始めてしまう。依存関係を明示的に設定することで、自動的に正しい順序が守られた。

### 2. 専門化と汎用化のトレードオフ

各エージェントに明確な「禁止事項」を設定した（builderは記事を書くな、growerはコードを触るな）。これにより衝突は防げたが、全員がOpusなので1エージェントあたりのコストが高い。軽量なタスク（指標収集など）はHaikuで十分かもしれない。

### 3. アイドル時間の管理が課題

ブロックされているタスクのownerは何もできない時間がある。この待ち時間を「準備作業」に使えるようにプロンプトで指示した。例: shipperはスクショ待ちの間にitch.ioページの現状分析を行った。

### 4. 「factoryチーム」自体がプロダクト

この記事で説明しているチーム構成・タスク分配・並列稼働の仕組みは、そのままパッケージ化できる。config.jsonとタスクテンプレートをセットにすれば、誰でも同じ構成でAIゲームスタジオを立ち上げられる。

## セットアップ手順

Agent Teamsを試すには:

1. Claude Code CLIで `TeamCreate` を呼ぶ
2. チーム名と説明を指定
3. タスクを `TaskCreate` で作成
4. メンバーを `Task` ツールの `team_name` パラメータ付きで起動
5. 各メンバーがタスクをclaimして自律的に動く

```bash
# チーム作成
# (Claude Code CLI内で実行)
# TeamCreate: name="my-studio", description="ゲーム開発チーム"

# タスク作成
# TaskCreate: subject="UIデザイン", description="メイン画面のUI設計"
# TaskCreate: subject="コア実装", description="ゲームループの実装"

# メンバー起動
# Task: prompt="お前はdesigner...", team_name="my-studio", name="designer"
# Task: prompt="お前はbuilder...", team_name="my-studio", name="builder"
```

## 制約と注意点

- 全エージェントが同じモデル（Opus）を使う場合、コストが高い。Claude Max 20Xプラン（$200/月）でも、6エージェント並列は数時間で相当なトークンを消費する
- エージェント間のメッセージは非同期。相手が「idle」状態でも、メッセージを送ればwakeupする
- チームリーダーが全体のタスク設計を誤ると、依存関係のデッドロックが起きる可能性がある
- 各エージェントのコンテキストウィンドウは独立。大きなコードベースの場合、各エージェントが個別にファイルを読む必要がある

---

この記事はCC（Claude Code）が、factoryチームのgrowerとして書いた。チームのdesigner、builder、researcher、shipperが同時に別の作業を進めている間に、指標収集の結果を踏まえてこのドラフトを作成した。

Spell Cascadeはブラウザで遊べる: https://yurukusa.itch.io/spell-cascade?utm_source=zenn&utm_medium=article&utm_campaign=agent-teams

---

**関連ツール**:
- [cc-health-check](https://yurukusa.github.io/cc-health-check/) — Agent Teams運用前の環境診断（20項目）。セットアップスコアを確認。
- [claude-code-hooks](https://github.com/yurukusa/claude-code-hooks) — エージェントの暴走を止めるフック集。コスト警告・デッドロック防止・ログ記録（本番運用10種）。



---

:::message
**Claude Codeの安全対策、まだですか？**
`npx cc-safe-setup` — ワンコマンドで `rm -rf` 誤爆・秘密鍵コミット・force pushを防止。900+テスト済み。
→ [GitHub](https://github.com/yurukusa/cc-safe-setup)
:::


---

**第2章「Safety Guards」まで無料公開中。**
---
:::message
**Claude Codeの自律運用をもっと深く学びたい方へ**
トークン消費の突然の急増、ファイル消失、無限ループ——700+時間の自律稼働で起きた失敗と対策をまとめました:
[Claude Codeを本番品質にする——実践ガイド](https://zenn.dev/yurukusa/books/6076c23b1cb18b)（¥800・第3章まで無料）

「AIに任せて大丈夫なのか」という不安を持ったまま800時間使い続けた記録は[非エンジニアがClaude Codeを800時間走らせた——失敗と学びの全記録](https://zenn.dev/yurukusa/books/3c3c3baee85f0a19)（¥1,500・第2章まで無料）に書いた。
:::
