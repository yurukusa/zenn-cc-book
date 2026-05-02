---
title: "連休前に入れた Claude Code の保険——Stop hook の予算ゲートと PreCompact hook の git 保存"
emoji: "🏖"
type: "tech"
topics: ["claudecode", "hooks", "anthropic", "ai", "tips"]
published: true
---

明日からゴールデンウィークだ。家族で出かける予定があり、その間も Claude Code には簡単な作業を任せて出ようと思っている。データの整理や、長めのドキュメントの草案づくりみたいな、自分がいなくても進められそうな仕事だ。

ただ、過去の経験から言うと、Claude Code は「留守番中」 という状況を知らない。人間が連休でゆっくりしている間も、AI は同じペースで動き続ける。これが本当に怖い。実際、以前似たようなことをやって、半日で 5,000 円分くらいのトークンを溶かしたことがある。気づいたのは家から戻って API の利用画面を開いた瞬間だった。

連休前にやっておくと、帰ってきた時に自分が泣かなくて済むことが、3 つある。本記事は、その 3 つを hook の設定例まで含めて書く。

## 前提：自動圧縮は止められない

長く動かすと context compaction が走って文脈が消えるから、`autoCompactEnabled` を false にしておけば大丈夫——と書いてある記事もあるが、これは効かない。

GitHub Issue [#42817](https://github.com/anthropics/claude-code/issues/42817) で、auto-compact を無効化する全ての公式経路（`/config`、`claude config set`、`settings.json`、環境変数）が現在は機能していない、という測定付きの報告が並んでいる。コンテキスト容量の約 35% で勝手に圧縮が走るケースまで観測されている。

つまり、留守番中に圧縮は必ず起きる。問題は「圧縮が起きること」 ではなく「圧縮の前の状態が記録されていないこと」。これは hook で対処できる。

## 1 つ目：Stop hook の予算ゲート

`Stop` イベントは、Claude Code が一連の作業を終えるタイミングで発火する。ここに「当日累計のトークン消費が閾値を超えたら止める」 hook を仕込む。

```bash
#!/usr/bin/env bash
# daily-cost-guard.sh — 当日累計が閾値を超えたら警告して停止する
set -euo pipefail

DAILY_LIMIT_USD=5.00
WARN_THRESHOLD_USD=3.50
LOG_DIR="${HOME}/.claude/cost-log"
mkdir -p "${LOG_DIR}"
TODAY=$(date +%Y-%m-%d)
LOG_FILE="${LOG_DIR}/${TODAY}.jsonl"

# Claude Code の出力 JSON から usage を読み取り、当日のログに追記する
PAYLOAD=$(cat)
USAGE=$(echo "$PAYLOAD" | jq -r '.usage // empty')
[ -z "$USAGE" ] && exit 0

# 各セッションの当日コストの累計を集計する（実装は環境依存）
echo "$USAGE" | jq -c "{ts: now, usage: .}" >> "$LOG_FILE"
TODAY_USD=$(jq -s 'map(.usage.cost_usd // 0) | add' "$LOG_FILE")

if (( $(echo "$TODAY_USD > $DAILY_LIMIT_USD" | bc -l) )); then
  echo "🛑 当日 \$${TODAY_USD} が上限 \$${DAILY_LIMIT_USD} を超えました" >&2
  exit 2  # Stop hook で exit 2 = 作業を止める
elif (( $(echo "$TODAY_USD > $WARN_THRESHOLD_USD" | bc -l) )); then
  echo "⚠️  当日 \$${TODAY_USD}（警告閾値 \$${WARN_THRESHOLD_USD}）" >&2
fi

exit 0
```

`~/.claude/settings.json` への登録：

```json
{
  "hooks": {
    "Stop": [
      { "matcher": "", "hooks": [
        { "type": "command", "command": "/path/to/daily-cost-guard.sh" }
      ]}
    ]
  }
}
```

ポイントは `exit 2` で Stop hook を block すること。これで Claude Code は「作業の続きを始めない」 状態になる。家族と過ごしている間に予算が爆発する事故を防げる。

実運用では、`cost_usd` の集計のロジックは環境依存になる（Claude Code が出力する usage の形式は version で揺れる）。`/cost` コマンドの出力をパースする方が確実な場合もある。最低限「当日累計が閾値を超えたら止まる」 という挙動が成立すればよい。

## 2 つ目：PreCompact hook の git 保存

`PreCompact` イベントは、context compaction が走る直前に発火する。ここに「未コミットの変更を全部 git に一時保存する」 hook を仕込む。

```bash
#!/usr/bin/env bash
# precompact-git-checkpoint.sh — 圧縮直前で git に一時保存を作る
set -euo pipefail

git rev-parse --is-inside-work-tree &>/dev/null || exit 0

CHANGES=$(git status --porcelain 2>/dev/null | wc -l)
[ "$CHANGES" -eq 0 ] && exit 0

BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "HEAD")
TIMESTAMP=$(date -u '+%Y%m%d-%H%M%S')

# pre-commit / commit-msg hook を経由しないため --no-verify
git add -A 2>/dev/null
git commit \
  -m "checkpoint: pre-compact auto-save (${CHANGES} files, ${TIMESTAMP})" \
  --no-verify 2>/dev/null || true

echo "📸 Pre-compact 一時保存: ${CHANGES} ファイル on ${BRANCH}" >&2
echo "  復旧は: git log --oneline -5" >&2
exit 0
```

`~/.claude/settings.json` への登録：

```json
{
  "hooks": {
    "PreCompact": [
      { "matcher": "", "hooks": [
        { "type": "command", "command": "/path/to/precompact-git-checkpoint.sh" }
      ]}
    ]
  }
}
```

帰宅後、`git log --oneline -5` を見れば「最後に AI が何をやっていたか」 がわかる。圧縮で AI 自身が忘れていても、git は忘れない。

`--no-verify` を付けているのは、pre-commit / commit-msg hook の中に「テストを実行する」「lint を走らせる」 系のものがある場合、Claude Code の留守番中にそれが落ちると一時保存自体が成立しないため。一時保存の優先順位を最大にする目的の `--no-verify` で、ここでは是とする。

## 3 つ目：作業ブランチを切ってから出かける

これは hook ではなく運用ルール。

```bash
git switch -c holiday/claude-code-2026-04-29
```

任せる作業を main から切り離した枝で始める。失敗しても main に戻れる。圧縮で文脈を失った AI が「ここまでの仕事の整合性を諦める」 形で何を消すかわからないので、戻れる安全地帯を確保しておく。

帰宅後の手順：

```bash
# ブランチに何が積まれたか確認
git log --oneline holiday/claude-code-2026-04-29

# 採用するなら main にマージ
git switch main
git merge --no-ff holiday/claude-code-2026-04-29

# 採用しないなら捨てる
git branch -D holiday/claude-code-2026-04-29
```

採用と破棄を 1 コマンドで切り替えられる状態にしておけば、留守番の成果物の評価が冷静にできる。

## 連休に持って行きたい順番

家族と出かける前に、自分はこの順番で確認する。

1. 予算 hook が動いているか（`~/.claude/cost-log/$(date +%Y-%m-%d).jsonl` が増えるか軽く回して試す）
2. PreCompact hook が登録されているか（`~/.claude/settings.json` の `PreCompact` 配列を目視）
3. 任せる作業を git で枝を切って始める（失敗しても main に戻れる状態にしておく）

完璧な留守番にはならない。でも「予算が爆発」 と「進捗ゼロ」 のダブルパンチで連休が台無しになる事故は、これでだいぶ減らせる。

## 補足：Stop と PreCompact の選び方

別の `PostToolUse` hook で同じことをやる選択肢もあるが、`Stop` と `PreCompact` の方が留守番運用には合う。理由は次の 2 点。

- `PostToolUse` は 1 ツール呼び出しごとに発火するので、ログの量が膨らみやすく、性能の劣化も観測される（Issue [#34629](https://github.com/anthropics/claude-code/issues/34629) 周辺）
- `Stop` と `PreCompact` は「作業の区切り」 と「文脈消失の直前」 という、人間が介入したいタイミングそのものに 1 対 1 で対応する

ここでは留守番中の事故防止が目的なので、頻度の少ない hook の方が信頼性も高く、デバッグもしやすい。

## まとめ

人間は連休があるが、Claude Code は連休を知らない。だから連休前にこちらで仕掛けを作っておくしかない。家族の方が AI より大事だ。それを揃えてから、出かけよう。

参考にしている GitHub Issue：[#42817](https://github.com/anthropics/claude-code/issues/42817)（auto-compact 無効化が効かない）、[#34629](https://github.com/anthropics/claude-code/issues/34629)（PostToolUse hook の負荷）。
