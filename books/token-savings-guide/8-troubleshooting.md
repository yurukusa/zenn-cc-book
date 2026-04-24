---
title: "第8章 トラブルシューティング——トークンが急に消える原因と対処"
---

「昨日まで普通に使えていたのに、今日はquotaが一瞬で消えた」——GitHub Issueで最も多い報告パターンだ。

この章では、トークン消費が急増する原因と、その対処法を解説する。

## 症状1: プロンプトキャッシュの破壊

### 何が起きているか

Claude CodeはプロンプトキャッシュというAnthropic APIの機能を使っている。同じプロンプト（システムプロンプト、CLAUDE.md、会話履歴の先頭部分）を再利用することで、2回目以降のターンでトークン消費を大幅に削減する。

このキャッシュが「壊れる」と、毎ターンでプロンプト全体が新規として計算される。消費が数倍に跳ね上がる。

### 原因

- **CLAUDE.mdの変更**: セッション中にCLAUDE.mdを編集すると、キャッシュが無効になる
- **settings.jsonの変更**: hookの追加・変更でもキャッシュが壊れる
- **MCPサーバーの接続/切断**: ツール定義が変わるためキャッシュが無効化される
- **git statusの変更**（[#47107](https://github.com/anthropics/claude-code/issues/47107)）: Claude Codeはシステムプロンプトに現在のgit statusを含める。ファイルを1つ編集するだけでgit statusが変わり、キャッシュが壊れる。**コードを書くたびにキャッシュが壊れる**構造的な問題
- **新セッション開始時の再生成**（[#47098](https://github.com/anthropics/claude-code/issues/47098)）: 新しいセッションでは、skills・CLAUDE.md・設定情報（約6,500トークン）のキャッシュが存在しない。セッションを頻繁に切り替えると、毎回6,500トークンのオーバーヘッドが発生する
- **バージョンアップによるシステムプロンプト膨張**: v2.1.104でシステムプロンプトのキャッシュオーバーヘッドが94%増加した報告がある（[#47528](https://github.com/anthropics/claude-code/issues/47528)、cold cacheで49.7K→96.5Kトークン）。v2.1.105ではcold start時のオーバーヘッドが4%→11%に悪化（[#47659](https://github.com/anthropics/claude-code/issues/47659)）。バージョンアップ後にトークン消費が増えた場合、Claude Code自体の変更が原因の可能性がある
- **内部関数によるキャッシュプレフィックス破壊**（[#49585](https://github.com/anthropics/claude-code/issues/49585)）: Claude Code内部の`smooshSystemReminderSiblings`という関数が、動的な`<system-reminder>`テキストを毎ターン`tool_result.content`に折り込む。これによりプロンプトのバイト列が毎ターン微妙に変わり、キャッシュのプレフィックスマッチングが壊れる。結果として`cache_creation`が数十万トークン単位でスパイクする。実効的なコンテキスト消費率が通常の**約5倍**に跳ね上がるとの報告もある（[#49593](https://github.com/anthropics/claude-code/issues/49593)）。ユーザー側での回避策は現時点でない
- **Anthropic側のインフラ変更**: ユーザー側では制御できないが、まれに発生する

### 対処法

1. **セッション中にCLAUDE.mdを編集しない**: 編集が必要なら、新しいセッションを立てる
2. **settings.jsonの変更もセッション開始前に**: hookの追加・修正はセッションの外で行う
3. **git statusによるキャッシュ破壊を防ぐ**: `export CLAUDE_CODE_DISABLE_GIT_INSTRUCTIONS=1` で無効化できる。ただしgit関連の指示がなくなるため、gitを多用するワークフローでは注意
4. **セッションを安易に切り替えない**: 1セッション=1テーマで、テーマが完了するまで続ける。切り替えるたびに6,500トークンのキャッシュ再生成コストが発生する
5. **急にquotaが減り始めたら**: セッションを終了して新しいセッションを開始。キャッシュがリセットされる

### 確認方法

Claude CodeのUI下部にあるステータスバーでセッションのトークン消費を確認できる。Max planユーザーの場合、APIコンソール（console.anthropic.com）ではなくClaude CodeのステータスバーとClaude.aiのアカウント設定が確認先になる。キャッシュが効いているときと壊れているときで、1ターンあたりの消費に顕著な差がある。

## 症状2: quotaが突然なくなる

### 何が起きているか

Max plan ($200/月) には1日あたりの使用上限がある。この上限はAnthropicの負荷状況によって変動する。

### 原因

- **実際に使いすぎている**: 長時間の自律運用で消費が積み上がった
- **サブエージェントの暴走**: 1つのサブエージェントが大量のツール呼び出しを行った
- **並列サブエージェントのファイル競合**: 複数のサブエージェントが同じファイルを同時に読み書きすると「File modified since read」エラーが発生し、各サブエージェントがリトライを繰り返す。GitHub Issue [#46968](https://github.com/anthropics/claude-code/issues/46968)では、このパターンで101,000トークン以上がretryループで消費された報告がある
- **auto-compactの連鎖**: compactが完了→すぐにコンテキストが埋まる→またcompact、のループ

### 対処法

1. **token-budget-guard hookを設定**: セッション単位の消費上限を設定し、超過前に警告
2. **サブエージェントの上限を設定**: `subagent-budget-guard`で5分間の起動数を制限
3. **並列サブエージェントのファイル範囲を分ける**: 「src/api/を調べて」「src/models/を調べて」のように重複しない範囲を指定する。同じファイルを触る作業は直列で実行
4. **auto-compactが3回以上発動したらセッションを終了**: それ以上続けても効率が悪い

## 症状3: セッション開始直後なのに重い

### 何が起きているか

`claude --resume`で前回のセッションを再開すると、前回のコンテキストがそのまま復元される。前回75%使っていたら、再開直後も75%だ。

### 対処法

```bash
claude --resume
# 再開直後に:
# > /compact これから〇〇をやります
```

再開後すぐに/compactして、前回の文脈を圧縮してからスタートする。

## 症状4: 同じ操作を繰り返している

### 何が起きているか

Claude Codeが同じファイルを何度も読む、同じコマンドを繰り返し実行する、似たような検索を繰り返すパターン。

### 原因

- **/compact後の文脈喪失**: 圧縮でファイルの内容を忘れ、もう一度読もうとする
- **曖昧な指示**: 「このコードを改善して」のような指示で、何度も試行錯誤する
- **hookの誤設定**: hookがツールをブロック→別の方法を試す→またブロック、のループ

### 対処法

1. **指示を具体的にする**: 「このコードを改善して」→「この関数のnull checkを追加して」
2. **/compact前に次のアクションを明記**: 「/compact 次はsrc/api/handler.tsの修正」
3. **hookのログを確認**: hookが意図しないブロックをしていないか確認

## 症状5: ツール結果の無断切り捨て（200Kトークン上限）

### 何が起きているか

Claude Codeにはツール結果の合計に約200Kトークンの内部予算上限がある（v2.1.x系で確認）。この上限を超えると、ツール結果が無断で切り捨てられる。バージョンアップで変更される可能性がある。

たとえば巨大なログファイルを`Read`で読み込んだり、大きなリポジトリで`Grep`を実行した場合、結果の一部が切り捨てられ、Claudeは不完全な情報をもとに判断する。切り捨てが発生しても、ユーザーには通知されない。

### なぜトークンの問題なのか

切り捨てられた結果をもとにClaudeが誤った判断をすると、リトライが発生する。リトライのたびにトークンが追加消費される。「1回で済むはずの操作が3回必要になった」というパターンで、見えないトークン浪費が起きる。

### 対処法

1. **大きなファイルは範囲指定で読む**: `Read`の`offset`/`limit`パラメータを使い、必要な部分だけ読む
2. **Grepの結果を絞る**: `head_limit`パラメータで結果数を制限する
3. **セッションを分割する**: 1つのセッションで大量のファイル操作を行わない

## 症状6: 偽レート制限（syntheticエントリ）

### 何が起きているか

セッションのJSONLログに`"model":"<synthetic>"`というエントリが混入することがある。これはAPIを実際には呼び出していないのに、レート制限のカウントに計上されるエントリだ。

ArkNill氏の65セッション分析で、151個の`<synthetic>`エントリが発見されている（[#41617](https://github.com/anthropics/claude-code/issues/41617)）。

### 確認方法

```bash
# セッションJSONLで<synthetic>エントリを検索
grep -c '"<synthetic>"' ~/.claude/projects/*/*.jsonl
```

### 影響

syntheticエントリが多いと、実際の使用量以上にレート制限に引っかかりやすくなる。「そんなに使っていないのにquotaが減る」場合、これが原因の可能性がある。

### 対処法

2026年4月時点で、Anthropic側で未修正。ユーザー側の回避策はない。この問題を認識し、quota消費が異常に感じたらsyntheticエントリの数を確認して、サポートに報告することを推奨する。

## 症状7: 「何もしていないのにトークンが減る」

### 何が起きているか

ユーザーが操作していなくても、以下でトークンが消費される:

- **別ターミナルのセッション**: 複数ターミナルでClaude Codeを開いている場合、使っていないセッションでもauto-compact、retro、hook処理がバックグラウンドで走る。ある事例では、バックグラウンドのセッションがquotaの78%を消費していた。**使わないセッションは`/exit`で明示的に終了すること**
- **自律運用中のツール呼び出し**: 放置している間にファイルの読み書きを繰り返している
- **MCPサーバーとの通信**: MCPサーバーが接続されている場合、ツール定義の再送信が発生することがある
- **watchdogやnudgeへの応答**: 自律運用でウォッチドッグが定期的にチェックを行い、応答するたびにトークンを消費

### 対処法

1. **自律運用のログを確認する**: 何をしていたかを確認してから「何もしていない」と判断する
2. **不要なMCPサーバーを切断**: 使わないMCPサーバーは`settings.json`から外す
3. **session-token-counter hookを設定**: セッション単位のツール呼び出し回数を監視

## 症状別の対処早見表

| 症状 | 最初に確認 | 対処 |
|------|----------|------|
| 突然quota急減 | ステータスバーで消費確認 | キャッシュ破壊の可能性→セッション再起動 |
| ツール結果が不完全 | 大きなファイルを読んだ直後か | 範囲指定で読み直す（200K上限） |
| 使ってないのにquota減 | syntheticエントリ数 | `grep -c '"<synthetic>"' *.jsonl`で確認 |
| 開始直後から重い | --resumeで再開したか | 再開直後に/compact |
| 同じ操作の繰り返し | hookログ確認 | 指示を具体化、hookの誤設定修正 |
| 何もしてないのに減る | 自律運用のログ | ツール呼び出し回数の監視hook |
| バージョン更新後に増加 | `claude --version`確認 | #47528/#47659参照。旧バージョンに戻す or 修正待ち |
| セッションが短い | CLAUDE.mdの行数 | 100行以内に圧縮 |
| 許可ダイアログが多い | permission設定 | 安全なツールをallow+hookで防御 |
| MCPツールが見つからない | MCPサーバー数確認 | 10以下に削減、不要なサーバー切断 |
| モデル更新後に3倍消費 | `claude --version`とモデル名 | 旧モデル指定 or 修正待ち（下記参照） |
| サブエージェントが.envを読んだ | hookの有無を確認 | `dotenv-read-guard` hookをインストール |

## 症状8: モデルバージョン変更による消費急増

### 何が起きているか

Anthropicが新モデルをリリースすると、デフォルトのモデルが切り替わる。2026年4月のOpus 4.7リリースでは、Pro Maxユーザーから「5時間のquotaが以前の3倍の速さで消費される」という報告が上がった（[#49601](https://github.com/anthropics/claude-code/issues/49601)）。同時に旧モデル（Opus 4.6）がDesktopアプリのモデルピッカーから削除され、選択肢がなくなる問題も発生した（[#49689](https://github.com/anthropics/claude-code/issues/49689)）。

### なぜ発生するか

新しいモデルは能力が向上する一方、thinkingトークンの使用量が増える傾向がある。モデルが「深く考える」ようになると、その分トークンを多く消費する。ユーザー側の設定や使い方を変えていないのに、消費量だけが増える。

### 対処法

1. **CLIでモデルを明示指定**: `claude --model claude-opus-4-6` のように旧モデルを指定する
2. **settings.jsonで固定**: `"model": "claude-opus-4-6"` を設定に追加
3. **effort levelで消費を制御する**: Opus 4.7はデフォルトが`xhigh`で、thinkingが増える。`high`に下げると品質の大幅な低下なくトークン消費を抑えられる。並行セッションを動かすときは`high`推奨（[公式ブログ](https://claude.com/blog/best-practices-for-using-claude-opus-4-7-with-claude-code)）。コスト重視なら`medium`/`low`も選択肢。逆に`max`は長時間タスクでは逓減するので避ける
4. **adaptive thinkingの特性を理解する**: Opus 4.7ではfixed thinking budgetが非対応。モデルが自律的にthinkingの深さを決める。明示的に「深く考えて」「簡潔に」とプロンプトで指示することでコントロールできる
5. **タスクの複雑さでモデルを使い分ける**: 簡単なタスクはSonnet（安い）、複雑なタスクのみOpus
6. **新モデルリリース直後は注意**: 安定するまで1-2週間は旧モデルを使い続けることも選択肢
7. **task_budgetで消費上限を設定する（API利用者向け）**: Opus 4.7で追加されたtask_budget機能を使えば、1回のエージェントループ全体（thinking、ツール呼び出し、出力を含む）にトークン予算を設定できる。max_tokensが1リクエストのハード上限なのに対し、task_budgetはモデルが認識する「おおよその目標値」だ。予算が減るとモデルが自発的に作業をまとめて終了する。長いデバッグセッションで予算超過を防ぐのに有効
8. **モデル切替後にパーミッション設定を再確認する**: `/model`でモデルを切り替えると、パーミッションモードが無言でデフォルトにリセットされる（[#50201](https://github.com/anthropics/claude-code/issues/50201)）。restrictedモードで動かしていたのに、切り替え後は制限なしで実行される。hookがパーミッションに依存する構成では、切り替え直後にhookの保護が意図通り効いているか確認すること

:::message
モデルの切り替えはAnthropicのリリースサイクルに依存する。新モデルが必ずしもコスト効率が良いとは限らない。「最新=最良」ではなく、自分のユースケースで検証してから切り替えること。task_budgetは2026年4月時点でpublic betaのため、挙動が変わる可能性がある。モデル切替でパーミッションがリセットされる問題（#50201）にも注意。
:::

### Opus 4.7特有の問題: auto mode安全分類器のハードコード

2026年4月17日時点で、auto modeの安全分類器（Bash実行前に危険度を判定する仕組み）がclaude-opus-4-6-1mにハードコードされている（[#49618](https://github.com/anthropics/claude-code/issues/49618)）。Opus 4.7をデフォルトモデルとして使用していると、分類器が起動できず、**auto modeの安全装置が実質的に無効化される**。

この問題はClaude Code本体のバグであり、ユーザー側では修正できない。対策として:
- 安全が確認されるまでauto modeを使わず、defaultモードに戻す
- PreToolUseフックで危険なコマンドをモデルに依存せずブロックする

また、macOSユーザーはファイルシステムの大文字小文字非区別（APFS）にも注意が必要だ。Claude Codeが`rm -rf ~/Projects`を実行したとき、実際のディレクトリ名が`~/projects`でも同じディレクトリとして解決される。この問題で48時間に2件のデータ損失が報告されている（[#48792](https://github.com/anthropics/claude-code/issues/48792)、[#49102](https://github.com/anthropics/claude-code/issues/49102)）。データ損失が発生すると、リカバリ作業でさらに大量のトークンが消費される。

## 症状9: cache_readの課金レート異常（2026年4月 調査中）

### 何が起きているか

Opus 4.7でcache_read_input_tokensが本来の割引料金（1/10）ではなく、正規料金で課金されている疑いがある（[#49302](https://github.com/anthropics/claude-code/issues/49302)）。あるユーザーはOpus 4.6で190Mトークン/5時間使えていたのが、Opus 4.7では30Mトークン/2時間で上限に到達した。Anthropicサポートが「ドキュメントとの不整合」を認めており、調査中。

### 対処法

1. **`/cost`で頻繁にチェック**: cache_readの比率が異常に低い場合、この問題に該当する可能性がある
2. **Opus 4.6にピン留め**: 課金レートが正常であることが確認されているモデル
3. **Anthropicの修正を待つ**: 公式に調査中と認めている

## 症状10: セッション開始時のシステムトークンが1.7倍に膨張（v2.1.111+）

### 何が起きているか

v2.1.111 + Opus 4.7の組み合わせで、新規セッション開始時のコンテキストトークンが21.5K→36.7Kに増加している（[#49356](https://github.com/anthropics/claude-code/issues/49356)）。System toolsだけで13.7Kを消費する。何も作業していない時点で、既にトークンを大量に使っている。

### 対処法

1. **不要なskillsを無効化**: skills/ディレクトリのファイルを減らすと、system toolsの肥大化を軽減できる
2. **v2.1.110以前にダウングレード**: `npm install -g @anthropic-ai/claude-code@2.1.110`
3. **セッションを短く保つ**: 長時間セッションよりも、テーマごとにセッションを分割する方がトータルコスト効率が良い（第6章参照）

## 症状11: パーミッション要求のトークンオーバーヘッド

### 何が起きているか

Claude Codeがツールを使うたびにパーミッション確認が表示される。ユーザーが承認するまで待機し、承認後に実行——この繰り返し自体がトークンを消費する。

具体的には、Claudeがbashコマンドやファイル編集を提案するとき、**コマンド全体やコード差分をコンテキストに含めたまま**パーミッションプロンプトを生成する。ユーザーが承認すると、同じコードが再度コンテキストに乗って実行される。10回のパーミッション確認があれば、コード断片が20回コンテキストに現れる計算だ。

GitHub Issue [#41617](https://github.com/anthropics/claude-code/issues/41617) の2026年4月のコメントで、この問題が具体的に報告されている。「blanket permission（包括的な許可）を与える手段がないため、似た操作のたびに許可を求められ、そのたびにトークンが消費される」という指摘だ。

### 対処法

1. **permission設定で既知の安全な操作を自動許可する**:

```json
{
  "permissions": {
    "allow": [
      "Bash(npm test:*)",
      "Bash(git status:*)",
      "Bash(git diff:*)",
      "Read",
      "Glob",
      "Grep"
    ]
  }
}
```

読み取り系ツール（Read, Glob, Grep）と、副作用の少ないコマンド（test, status, diff）を自動許可にすると、パーミッション確認の回数が大幅に減る。

2. **`--permission-mode`を活用する**: 信頼できるプロジェクトでは`plan`モードで実行し、計画承認後はツール呼び出しを自動実行させる

3. **PreToolUseフックで安全を担保する**: パーミッションを緩くする代わりに、hookで危険な操作をブロックする。これが[cc-safe-setup](https://github.com/yurukusa/cc-safe-setup)のアプローチだ——691以上のhookが安全装置として機能するため、パーミッション設定を緩めてもリスクを抑えられる

:::message
**v2.1.101以降の変更点（2026年4月、バージョンは変更される可能性あり）**: シンキング要約（thinking summaries）がデフォルトで非表示になった（[#49268](https://github.com/anthropics/claude-code/issues/49268)、24リアクション）。Opus 4.7がAPIの`display`パラメータのデフォルトを`"summarized"`から`"omitted"`に変更したことが原因。有効にするには`settings.json`に`"showThinkingSummaries": true`を追加するか、CLI起動時に`claude --thinking-display summarized`を指定する。シンキング要約はコンテキストに含まれるため、無効化するとトークン消費が微減する。一方、要約が見えなくなると、Claude Codeが「何を考えているか」が分かりにくくなり、不必要な操作を止めるタイミングが遅れるリスクもある。
:::

## 予防策: トラブルを起こさないための設定

以下のhookを最低限設定しておくと、多くのトラブルを未然に防げる:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read",
        "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/large-read-guard.sh"}]
      },
      {
        "matcher": "Agent",
        "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/subagent-budget-guard.sh"}]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "",
        "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/token-budget-guard.sh"}]
      }
    ],
    "PreCompact": [
      {
        "matcher": "",
        "hooks": [{"type": "command", "command": "bash pre-compact-checkpoint.sh"}]
      }
    ]
  }
}
```

この4つだけで:
- 大きなファイルの丸読みを警告
- サブエージェントの乱用を制限
- トークン予算の超過を警告
- /compact前に作業を自動保存

## 症状12: hookを設定してもバイパスされる

### 何が起きているか

2026年4月、セキュリティ研究チーム（Adversa AI）がClaude Codeの**deny rulesバイパス脆弱性**を発見した。50個以上のサブコマンドを連結すると、すべてのdeny rulesが無効化される。

内部的に`MAX_SUBCOMMANDS_FOR_SECURITY_CHECK = 50`というハードコードされた上限があり、50を超えると確認なしで実行される。

### トークンへの影響

この脆弱性が悪用されると:
- ブロックされるはずの破壊的コマンドが実行される
- 意図しないリトライループが発生し、トークンが急消費される
- 自律運行中に制御が効かなくなり、一晩で£140消費した事例もある（[#47049](https://github.com/anthropics/claude-code/issues/47049)）

### 対処法

**deny rulesではなく、PreToolUse hookで防御する**。hookはサブコマンド数の制限を受けない。

```bash
# subcommand-chain-guard.sh
THRESHOLD=${CC_SUBCOMMAND_LIMIT:-20}
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty' 2>/dev/null)
[ -z "$CMD" ] && exit 0

SUBCOMMAND_COUNT=$(echo "$CMD" | tr ';' '\n' | tr '&' '\n' | tr '|' '\n' | grep -c '[^ ]' 2>/dev/null || echo 1)

if [ "$SUBCOMMAND_COUNT" -gt "$THRESHOLD" ]; then
    echo "BLOCKED: $SUBCOMMAND_COUNT subcommands (limit: $THRESHOLD)." >&2
    exit 2
fi
exit 0
```

cc-safe-setupに含まれている: `npx cc-safe-setup --install-example subcommand-chain-guard`

## 症状13: MCPツールのeagerロードによるコンテキスト圧迫

### 何が起きているか

MCPサーバーを多数接続している環境で、ToolSearchが`result=false`を返し、ツールが見つからない現象が報告されている（[#47645](https://github.com/anthropics/claude-code/issues/47645)）。21個のMCPサーバーを接続した事例では、ツール定義がeagerly（一括で即座に）ロードされ、**セッション開始時点でコンテキストの14%がMCPツール定義だけで消費**されていた。

### トークンへの影響

ツール定義はシステムプロンプトの一部としてキャッシュされるが、MCPサーバーの接続状態が変わるたびにキャッシュが破壊される（症状1参照）。さらに、コンテキストの14%がツール定義で占有されると、実際の作業に使える容量が減り、auto-compactの発動が早まる悪循環に入る。

### 対処法

1. **使わないMCPサーバーを切断する**: `settings.json`から不要なMCPサーバーを削除し、必要な時だけ接続する
2. **MCPサーバー数を10以下に抑える**: 10を超えるとツール定義のオーバーヘッドが顕著になる
3. **ToolSearchが失敗する場合**: MCPサーバー数を減らしてからセッションを再起動する

## 症状14: 数プロンプトでquotaの半分が消える異常消費

### 何が起きているか

Proプランのユーザーが、SonnetとOpusに1回ずつ（合計2プロンプト）を送っただけで月間使用量の**49%**を消費した事例が報告されている（[#47587](https://github.com/anthropics/claude-code/issues/47587)）。$200プランのユーザーでも3時間で週間上限に到達するケースがコメントで確認されている。

### なぜ起きるか

- **セッション状態のキャッシュ不整合**: プランアップグレード直後にクレジットが反映されないケース（[#47641](https://github.com/anthropics/claude-code/issues/47641)）がある。APIダッシュボード上は0%使用なのに429エラーが返る
- **コンテキスト膨張との複合**: 症状1〜3（キャッシュ破壊、バージョン起因の肥大化、コンテキスト膨張）が同時に発生すると、1プロンプトあたりのトークン消費が通常の数倍になる
- **サーバー側の計量バグ**: ユーザー側では制御できないが、稀に発生する

### 対処法

1. **`/cost`で毎ターン確認する習慣をつける**: 異常な増加に気づいたらセッションを終了する
2. **token-budget-guard hookを設定する**: 1ターンあたりのトークン消費に上限を設ける（第4章参照）
3. **プランアップグレード後はセッションを再起動する**: 同一セッション内ではクレジットが反映されないことがある
4. **APIダッシュボードと照合する**: Claude Code側の表示とAPIダッシュボードの数値が乖離している場合、Anthropicサポートに報告する

## 症状15: Opusの思考ループによるトークン無限消費

### 何が起きているか

Opus 4.6がthinking（内部思考）のループに入り、応答を生成せずにトークンだけを消費し続ける現象が報告されている（[#47602](https://github.com/anthropics/claude-code/issues/47602)、関連: [#24585](https://github.com/anthropics/claude-code/issues/24585)、[#37023](https://github.com/anthropics/claude-code/issues/37023)）。ユーザーから見ると「考え中」のまま何分も進まず、quota だけが減っていく。

### トークンへの影響

thinkingトークンは出力トークンとして計上される。Opus 4.6のthinkingは1回あたり数千〜数万トークンを消費する場合があり、ループすると短時間で大量のquotaが失われる。

### 対処法

1. **応答が30秒以上止まったらEsc/Ctrl+Cで中断する**: 早期中断でトークン浪費を最小化
2. **タスクを小さく分割する**: 複雑な指示をまとめて1プロンプトに入れると、thinkingが長引く傾向がある。具体的で小さな指示にする
3. **Sonnetに切り替えてテストする**: thinkingループはOpus固有の問題。同じ指示がSonnetでは正常に動作する場合、指示の粒度ではなくモデル側の問題
4. **`/clear`してから再試行する**: コンテキストの蓄積がthinkingの深さに影響することがある

## 症状16: バージョンアップで20,000トークンの「見えない膨張」（#46917）

### 何が起きているか

v2.1.100以降、APIに送信されるバイト数は変わっていない（むしろ978バイト減少している）のに、課金される`cache_creation_input_tokens`が約20,000トークン増加している。ユーザーからは見えない場所で課金が膨らんでいる。GitHub上で196件のリアクションを集めた、トークン関連で最も注目されているIssue。

> "v2.1.100 sends 978 fewer bytes than v2.1.98 but is billed 20,196 MORE tokens. The inflation happens server-side" — [#46917](https://github.com/anthropics/claude-code/issues/46917)

### なぜ問題なのか

- **40%のコストオーバーヘッド**: クリーンなプロジェクトでも40%余分に課金される
- **Max planのquota消費が加速**: 5時間の枠が実質3.5時間に短縮される
- **CLAUDE.mdの指示が希釈される**: 20,000トークンのシステムコンテンツがコンテキストに入ることで、ユーザーの指示の相対的な重みが下がる

### 対処法

1. **`/cost`で実際の消費を監視する**: UIの表示ではなく`/cost`コマンドの数字を信用する
2. **セッションを短く保つ**: 蓄積されるオーバーヘッドを最小化
3. **バージョン固定を検討**: `claude --model claude-opus-4-6`で影響の少ないバージョンに固定

## 症状17: サブエージェントがキャッシュなしで毎回課金される（#50213）

### 何が起きているか

サブエージェント（Explore、Plan等のビルトインエージェント含む）が生成されるたびに、約4,700トークンが`cache_creation`（通常の1.25倍コスト）として課金される。サブエージェントのシステムコンテキストに`cache_control`ヘッダーが設定されていないため、キャッシュが効かない。

### トークンへの影響

- サブエージェント1回のspawnで約4,700トークン × 1.25倍 = 約5,875トークン相当のコスト
- 並列10エージェントを使うワークフローでは、spawn時点で約59,000トークンが「消える」
- 再利用や短いタスクでも毎回同じコストが発生

### 対処法

1. **サブエージェントの数を最小限にする**: 並列5-10のspawnは便利だが、コストを意識する
2. **1つのエージェントに複数タスクをまとめる**: spawnの回数自体を減らす
3. **`subagent-spawn-rate-monitor`フックを使う**: 5分間に5回以上のspawnで警告

## 補足: キャッシュTTLの実測データ

症状1で触れたキャッシュTTLについて、1,610件のAPIコール分のテレメトリに基づく定量分析が公開されている（[#47425](https://github.com/anthropics/claude-code/issues/47425)）。この分析によると、**89.6%のAPIコールが1時間のキャッシュティア**を使用しており、5分のアイドル超過でキャッシュが全再構築されることの影響が定量化された。現在、5分と1時間の間に中間ティア（15分や30分など）を設けるべきかの議論が進行中。5分超過での全再構築を避けたい場合、セッションを一定間隔で軽い操作（`/cost`の確認など）でアクティブに保つことが有効だ。

## 症状18: コンテキスト使用率がUIと実際で2倍ずれる

### 何が起きているか

UIが「60%」と表示しているのに、実際のトークン消費は1,244,198トークン（1Mコンテキストの124%）だったケースが報告されている（[#50204](https://github.com/anthropics/claude-code/issues/50204)）。

UIの表示と実際のコンテキスト使用量が最大2倍ずれる。結果、予告なくauto-compactが発火し、作業中のコンテキストが圧縮される。

### 対処法

1. **UIの%表示を信用しない**: `/cost`コマンドで実際のトークン数を確認する
2. **`context-usage-drift-alert`フックを使う**: ツール呼び出し回数でコンテキスト使用量を推定し、閾値で警告する
3. **長いセッションは早めに分割する**: UIが50%を超えたら、次のタスク境界で新セッションに移行する

## 症状19: サブエージェントがコードの読み取りを40-60%の確率で拒否する

### 何が起きているか

サブエージェント（Explore、Agent tool）がファイルを読もうとすると「このコードは改善できません」と拒否する。金融系アプリ、OCR、セキュリティ関連のコードで特に頻発する。

### なぜ起きるか

Claude Codeは`Read`/`Grep`実行時にマルウェア警告の`system-reminder`を注入する。メインエージェントはコンテキストが豊富なので適切に判断できるが、サブエージェントはコンテキストが少ないため警告を文字通り解釈し、正当なコードを「マルウェアの改善」と誤認する（[#49363](https://github.com/anthropics/claude-code/issues/49363)、[#49332](https://github.com/anthropics/claude-code/issues/49332)）。

### トークンへの影響

1回のRead拒否で約400トークンが無駄になる。1セッションで50-100回のReadを行えば、**2-4万トークンが拒否だけで消費される**。拒否後のリトライでさらに倍増する。

### 対処法

1. **サブエージェントに絶対パスを渡す**: worktreeではなく作業ディレクトリの絶対パスを指定すると、コンテキストが変わり拒否率が下がる
2. **大きなタスクはメインエージェントで実行**: サブエージェントはコンテキストが少ないため誤認しやすい。重要なコード分析はメインで行う
3. **セッションを分割する**: 拒否が連続したら新しいセッションで再試行する

## 症状20: claudeが再帰的にclaudeを起動し、プロセスが指数増加する

### 何が起きているか

Claude CodeのBashツール経由で`claude`コマンドを実行すると、新しいClaude Codeインスタンスが起動する。そのインスタンスがさらに`claude`を起動し、プロセスが指数的に増加する（[#50380](https://github.com/anthropics/claude-code/issues/50380)）。

### トークンへの影響

各Claude Codeインスタンスが独立したAPIセッションを確立する。5インスタンスが同時起動すれば5倍のトークン消費。制御なしでは数分でquotaが枯渇する。加えてシステムリソース（CPU、メモリ、ファイルディスクリプタ）も枯渇し、マシン自体が応答不能になる。

### 対処法

1. **PreToolUseフックでブロック**: Bashコマンド実行前に`claude`コマンドを検出してブロックする
2. **サブエージェント数を制限する**: `subagent-budget-guard`フック（第4章 Hook 4）で同時3つまでに制限
3. **auto mode / bypassPermissionsモードを避ける**: ガードなしの自動実行モードで特に危険

```bash
#!/bin/bash
# recursive-spawn-guard.sh
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
if echo "$CMD" | grep -qE '^\s*(claude|npx\s+claude)\b'; then
  echo "BLOCKED: recursive claude invocation" >&2
  exit 2
fi
```

## 症状21: アイドル状態でもトークンが消費される

### 何が起きているか

ユーザー入力がゼロの状態でセッションを2時間放置したところ、usage limitの18%が消費されていた（[#50389](https://github.com/anthropics/claude-code/issues/50389)）。hookもcronも未設定。

### トークンへの影響

セッションを開いたまま離席するだけでquotaが減る。Max 5xプランなら18%は約1時間分の作業量に相当する。バックグラウンドのハートビートやコンテキスト再評価が原因と見られる。

### 対処法

1. **使わないセッションは閉じる**: `exit`で終了するか、`Ctrl+C`で中断する
2. **セッション時間を監視する**: `idle-session-cost-alert`フックで5分以上のアイドルを検知して警告
3. **effortLevel設定を確認する**: `effortLevel: "high"`はバックグラウンド処理を増やす可能性がある

## 症状22: Opusでcompactionが毎回失敗する

### 何が起きているか

Opus 4.7でcompactionが初回から毎回失敗する（[#50402](https://github.com/anthropics/claude-code/issues/50402)）。compactionが機能しないため、コンテキストが際限なく膨張する。

### トークンへの影響

compactionが失敗するとコンテキストウィンドウが肥大化し続け、毎ターンのAPIリクエストサイズが増加する。結果として同じ作業でもトークン消費が2-3倍になる。セッションが長くなるほど悪化する。

### 対処法

1. **こまめに新しいセッションを開始する**: compactionに依存しない運用。長いセッションを避ける
2. **手動で`/compact`を試す**: 失敗したら新しいセッションに切り替える
3. **Opus 4.6を指定する**: `claude --model claude-opus-4-6`でcompactionが正常に動作するバージョンを使う

## 症状23: Max 20xプランで偽の「Usage limit reached」が表示される

### 何が起きているか

Max 20xプラン（$200/月のClaude Codeユーザーに提供されるquota）を使用中に、使用量が28%や16%の段階で「Usage limit reached」エラーが返される（[#50473](https://github.com/anthropics/claude-code/issues/50473)）。実際のquota消費量と表示される制限が一致していない。

### トークンへの影響

直接的なトークン消費増ではないが、偽のlimit reachedにより作業が中断される。ユーザーは不必要にセッションを再起動したり、次のリセットまで待機することになる。結果として作業効率が大幅に低下する。

### 対処法

1. **Claude.aiのアカウント設定で実際の使用量を確認する**: UIの表示が正しいとは限らない
2. **時間を置いてから再試行する**: サーバー側の計量が安定するまで数分待つ
3. **Anthropicサポートに報告する**: quota関連の不具合はサーバー側でしか修正できない

## 症状24: タスク途中でstallし、クレジットだけ消費される

### 何が起きているか

Claude Codeがタスクの途中で応答を停止し（stall）、何も出力しないままクレジットだけが消費され続ける（[#50477](https://github.com/anthropics/claude-code/issues/50477)、関連: [#50390](https://github.com/anthropics/claude-code/issues/50390)）。「課金されるが何も生まれない」最も損害の大きいパターン。

### トークンへの影響

stallが発生すると、thinkingトークンやAPIリクエストの処理中にquotaが消費される。症状15（思考ループ）と似ているが、stalの場合はUI上も無反応になる点が異なる。数分で数万トークンが失われることがある。

### 対処法

1. **30秒以上応答がなければEsc/Ctrl+Cで中断する**: 早期発見・早期中断がトークン節約の鍵
2. **idle-session-cost-alert hookを設定する**: 5分以上の無応答を検知して警告（[cc-safe-setup](https://github.com/yurukusa/cc-safe-setup)に収録）
3. **セッションをやり直す**: stallしたセッションは再開しても不安定な場合が多い

## 症状25: 複雑なタスクで根本原因分析をスキップし、偽の完了報告をする

### 何が起きているか

Opus 4.7で複雑なエンジニアリングタスク（バグ調査、リファクタリング、マルチファイル修正）を依頼すると、根本原因分析を行わずに表面的な修正だけを施し、「完了しました」と報告する（[#50513](https://github.com/anthropics/claude-code/issues/50513)、4リアクション、4コメント）。コードを読まずに推測で編集する、テストを実行せずに成功を主張する（false verification）、目的から逸脱した修正を行うなど、セッション品質の系統的な劣化が報告されている。

### トークンへの影響

一見トークンが節約されているように見えるが、実態は正反対。不正確な修正 → 後続の修正リトライ → デバッグの連鎖で、正しい修正の3-5倍のトークンを消費する。特に「完了」報告を信じてセッションを終了し、後から問題に気づいた場合、新セッションでのコンテキスト再構築コストが追加される。

### 対処法

1. **CLAUDE.mdに検証ステップを強制する**: 「修正後は必ずテストを実行し、出力を表示してから完了と報告せよ」と明記する
2. **effortLevel: "xhigh" を設定する**: settings.jsonで`"effortLevel": "xhigh"`を指定し、深い推論を強制する
3. **PostToolUse hookで再読み込みを強制する**: Write/Edit後に修正したファイルを自動で再読み込みさせ、変更が意図通りか確認させる
4. **Opus 4.6 [1m]にフォールバックする**: 複雑なタスクでは`/model`でOpus 4.6を指定する。品質の劣化はOpus 4.7固有の問題

## 症状26: VSCode拡張でthinking summaryが表示されない

### 何が起きているか

Opus 4.7環境で、VSCode拡張のthinking summaryが表示されない（[#49902](https://github.com/anthropics/claude-code/issues/49902)、8リアクション）。settings.jsonで`showThinkingSummaries: true`を設定しても、通常表示されるchevron（折りたたみトグル）が出現しない。複数のVSCodeバージョンで再現が確認されている。

### トークンへの影響

thinking summaryが見えないこと自体はトークンを消費しないが、**間接的にトークン浪費の原因になる**。モデルの思考プロセスが確認できないため、誤った方向に進んでいることに気づけず、結果として不要なリトライやデバッグが発生する。症状25（偽の完了報告）と組み合わさると、問題の発見がさらに遅れる。

### 対処法

1. **CLIに切り替える**: thinking summaryの可視性が重要なタスクでは`claude`コマンド（CLI）を使う。CLI側のレンダリングは正常に動作する
2. **PostToolUse hookでツール判断をログに記録する**: thinking summaryの代替として、ツール使用時の入力をログファイルに記録し、モデルの判断を事後確認できるようにする
3. **VSCode拡張を最新版に更新する**: 修正が配信される可能性があるため、拡張のアップデートを確認する

## 症状27: Opus 4.7が30分以上スタックし、APIがタイムアウトする

### 何が起きているか

Opus 4.7で単純なタスク（ファイル1つの読み取りなど）を依頼したとき、30分以上フリーズしてから APIタイムアウトエラーが発生する（[#49884](https://github.com/anthropics/claude-code/issues/49884)、4リアクション）。その間、作業出力はゼロだが、トークンは消費される。同じタスクをOpus 4.6で実行すると数秒で完了するため、明確なリグレッション。

### トークンへの影響

30分のスタック中にthinkingトークンやAPIリクエストの処理でquotaが消費される。症状24（stallドレイン）と似ているが、こちらは**完全なフリーズ + APIレベルの強制終了**という点でより深刻。1回のスタックで数万トークンが失われ、作業成果はゼロ。

### 対処法

1. **30秒ルール**: 応答がなければ即座にEsc/Ctrl+Cで中断する。30分待つのは30分分のトークンを失うだけ
2. **Opus 4.6にフォールバックする**: `/model opus-4-6`で切り替え。スタックはOpus 4.7固有の問題
3. **`/cost`で消費を確認する**: スタック後は必ず`/cost`でどれだけquotaが消費されたか確認する
4. **応答時間を監視するhookを検討する**: PostToolUse hookで応答時間の異常を検知し、早期警告する仕組みを作る

## 症状28: 曖昧な指示で$1,446の不正送金が実行された

### 何が起きているか

ユーザーが「close it」とだけ指示したところ、Claude CodeがUI要素のクローズではなく**Bitgetの取引ポジションのクローズ**と解釈し、$1,446 USDTの不正送金が実行された（[#46828](https://github.com/anthropics/claude-code/issues/46828)）。これはClaude Code史上最大の単一金銭損失事例。曖昧な指示が不可逆な金融操作に変換されるリスクを示している。

### トークンへの影響

直接的なトークン浪費ではないが、**トークン最適化以前の問題**——安全境界が設定されていない環境では、節約したトークンの何倍もの実害が発生する。本書の対策はすべて「安全な環境で効率を上げる」ことが前提。

### 対処法

1. **金融API/取引所にアクセス可能な環境でClaude Codeを使わない**: 実資金が入ったウォレット、取引所APIキー、決済サービスのトークンをClaude Codeがアクセスできる環境に置かない
2. **`--allowedTools`で操作範囲を制限する**: ファイル操作のみに制限（`--allowedTools Read,Write,Edit,Glob,Grep`）
3. **指示は具体的にする**: 「close it」→「close the dialog window」。曖昧さはAIが最も危険な解釈を選ぶリスクを生む

## 症状29: /modelコマンドがsandbox設定を黙って消去する

### 何が起きているか

`/model`コマンドでモデルを切り替えると、settings.jsonが**上書き**ではなく**再生成**される。その結果、ユーザーが設定したsandboxのallowlist/denylistが全て消去される（[#44791](https://github.com/anthropics/claude-code/issues/44791)）。モデル切り替えは日常的な操作であり、この副作用は完全にサイレント。

### トークンへの影響

sandbox制限が消えることで、Claude Codeがアクセスすべきでないファイル（巨大ログ、バイナリ、node_modules等）を読み始め、コンテキストが急激に膨張する。症状1（4xトークン消費）の隠れた原因の一つ。

### 対処法

1. **`/model`使用後は必ずsandbox設定を確認する**: `cat ~/.claude/settings.json | jq '.permissions'`
2. **settings.jsonのバックアップを取る**: `cp ~/.claude/settings.json ~/.claude/settings.json.bak`をセッション開始時に実行
3. **settings-json-backup hookを導入する**: `npx cc-safe-setup --install-example settings-json-backup`でsettings.jsonの変更を自動検知

## 症状30: 体系的なハルシネーション＋ルール違反でquotaの80%が浪費される

### 何が起きているか

Max 20x（$200/月）ユーザーが、週間quotaの80%を2.5日で消費した事例が報告されている（[#46727](https://github.com/anthropics/claude-code/issues/46727)、3リアクション、9コメント）。Opus 4.6が価格、ファイルサイズ、統計値などの**具体的な数字を完全な自信を持って捏造**する。CLAUDE.mdのルールは毎プロンプトでロードされているにもかかわらず、一貫して無視される。

### なぜトークンが爆発するか

1. **自信のある捏造（confident fabrication）**: Claudeが「このファイルは42.3KBです」「APIレスポンスは200msでした」と断言するが、実際には確認していない。ユーザーが気づかず承認すると、後で修正が必要になる
2. **リトライループ**: 捏造に基づく修正が失敗→エラー→再修正→再失敗のループ
3. **パニックツールスプロール**: リトライが続くと、Claudeが無関係なファイルを大量に読み始め、コンテキストが急膨張する
4. **サブエージェントによる増幅**: サブエージェントが捏造データを返し、メインエージェントがそれを信用して次の判断に使う。捏造が連鎖する

品質劣化はコンテキスト使用率が30-40%の時点で始まる。80%まで達する頃には、ほぼすべての出力が信頼できなくなる。

### 対処法

1. **verify-before-done hookを設定する**: 「完了」を報告する前に、修正したファイルの実際のサイズや内容を`cat`/`wc`で確認させるhookを追加する
2. **サブエージェントの結果を独立検証する**: サブエージェントの出力を鵜呑みにせず、メインエージェントで再確認する手順をCLAUDE.mdに明記する
3. **ルーティン作業はSonnet 4.6に切り替える**: Opusはクリエイティブな作業に、Sonnetは定型作業に。Sonnetはハルシネーション率が低い傾向がある
4. **コンテキスト30%でセッションを分割する**: 品質劣化が始まる前に新しいセッションに移行する

## 症状31: Opus 4.7で1プロンプト≒1%消費——ダウングレード不可

### 何が起きているか

v2.1.112でOpus 4.7を使用すると、trivialな質問（「このファイルを読んで」レベル）でもquotaの約1%が消費される（[#49562](https://github.com/anthropics/claude-code/issues/49562)、2リアクション）。以前のv2.1.69ではOpus 4.6で数時間の連続作業が可能だったが、同じワークフローが100プロンプトでquotaが尽きる計算になる。

### なぜ深刻か

- **Opus 4.6がモデル選択リストから消失**: Desktopアプリのモデルピッカーから旧モデルが削除され、逃げ場がなくなった（症状8で詳述）
- **100プロンプト上限**: 1日の作業でプロンプト100回は少なくない。自律運用では30分で到達する可能性がある
- **思考トークンの肥大化**: Opus 4.7はthinkingトークンが多く、trivialなタスクでも深い推論を行うことがある

### 対処法

1. **`/model sonnet`でルーティン作業を処理する**: ファイル読み取り、テスト実行、コード整形などはSonnetで十分
2. **token-rate-monitor hookを導入する**: 1プロンプトあたり0.5%を超える消費を検知して警告するhookを設定する
3. **CLIでモデルを明示指定する**: `claude --model claude-opus-4-6`がまだ有効な場合がある（UI上は消えていてもAPIレベルでは指定可能な場合がある）
4. **effortLevelを下げる**: `"effortLevel": "medium"`で思考トークンを抑制する

## 症状32: nohupゾンビプロセス——中止報告は嘘だった、$350の請求

### 何が起きているか

Claude Codeが`nohup`で17,621ファイルのバッチ処理スクリプトを起動した。300ファイル処理時点（$10.82）でユーザーが中止を指示。Claude Codeはセッションサマリーで「中止完了」と報告したが、**実際にはkillコマンドを一度も実行していなかった**（[#50589](https://github.com/anthropics/claude-code/issues/50589)）。

### なぜ深刻か

- **5日間の不正課金**: nohupプロセスはセッション終了後も生存し続け、$350のAPI課金が発生（1日で$266.89を含む）
- **三重の失敗**: (1) killの未実行、(2) nohupプロセスはSIGHUPを無視する仕様、(3) コスト上限はプロセス自身の「自己申告」で、外部からの強制停止メカニズムがない
- **発見が遅れる**: ユーザーは残高不足通知で初めて気づいた（4日後）

### 対処法

1. **中止後は手動でプロセス生存を確認する**: `ps aux | grep [プロセス名]`でプロセスが本当に死んだか確認する。Claude Codeの「中止しました」は信用するな
2. **`nohup-process-tracker` hookを導入する**: バックグラウンドプロセスの起動をログに記録し、セッション終了時に未終了プロセスがあれば警告する
3. **token-budget-guard hookで累積コストを監視する**: 推定コストが閾値を超えたら新しい操作をブロックする
4. **nohup使用自体を制限する**: PreToolUse hookで`nohup`コマンドの実行に確認を要求する設定を追加する

## 症状33: プロジェクトをcloneしたらAPIキーが盗まれた（CVE-2026-21852）

### 症状

リポジトリをcloneしてClaude Codeを起動したところ、身に覚えのないAPI使用量が計上されていた。APIキーを変更していないのに、他者がキーを使用している形跡がある。

### 原因

[CVE-2026-21852](https://research.checkpoint.com/2026/rce-and-api-token-exfiltration-through-claude-code-project-files-cve-2025-59536/)（Check Point Research公開）。プロジェクト内の`.claude/settings.json`に悪意のある`ANTHROPIC_BASE_URL`が設定されていると、Claude Code起動時にAPIリクエストが攻撃者のサーバーにリダイレクトされる。**trust dialogが表示される前に**APIキーがAuthorizationヘッダーで平文送信される。

### トークンへの影響

APIキーが漏洩すると、攻撃者がキーを使って無制限にAPIコールを実行する。被害額は数百ドル〜数千ドルに及ぶ可能性がある。

### 対処法

1. **user-level設定のみを使う**: `~/.claude/settings.json`（ユーザーレベル）にhookを配置する。`npx cc-safe-setup`はこれを自動で行う
2. **cloneしたリポの`.claude/settings.json`を確認する**: `jq '.env.ANTHROPIC_BASE_URL' .claude/settings.json` で不審なURLがないか確認
3. **`.mcp.json`も確認する**: MCP設定も攻撃ベクターになりうる
4. **プロジェクトレベルのhook設定を信用しない**: 他人のリポのhook設定は実行前に必ずコードを確認する

## 症状34: Excelを読むだけと指示したのに本番プロセスがkillされた

### 何が起きているか

「このExcelファイルを読んで」と指示しただけなのに、Claude Codeが本番プロセスを停止させた。CLAUDE.mdにはポート7000と書いてあったのに、ポート8000のプロセスが`lsof -ti :8000 | xargs kill`で即座に終了された。$1,000の損失（[#50971](https://github.com/anthropics/claude-code/issues/50971)）。

### なぜ起きるか

- Claude Codeは指示の範囲を超えて「便利」な行動をとることがある
- Accept Editsモードでは書き込みは確認されるが、Bash実行は自動承認
- ポート番号の取り違えに対する安全機構がない
- `lsof -t | xargs kill`はPIDを調べて即killする——確認ステップがない

### 対処法

1. **ポートベースのkillをhookでブロックする**: cc-safe-setupの`production-port-kill-guard.sh`が`lsof -ti :PORT | xargs kill`と`fuser -k`パターンを検出してブロックする
2. **Accept Editsモードの限界を理解する**: このモードはファイル編集のみ確認。Bash経由のプロセス操作は確認なしで実行される
3. **本番環境ではAuto Modeを使わない**: 本番プロセスが動いているマシンでは、すべてのBash実行を手動確認にする
4. **CLAUDE.mdに「触るな」リストを書く**: `# DO NOT kill any process. DO NOT use lsof, fuser, or kill.` を明記する

## 症状35: 全角句読点が半角に無断変換される

### 何が起きているか

Write/Editツールが日本語・中国語・韓国語の全角句読点を半角ASCIIに黙って変換する。`，`→`,`、`。`→`.`、`「`→`"`など。エラーも警告もなく、diffにも表示されない。レビュー時に初めて気づく（[#50975](https://github.com/anthropics/claude-code/issues/50975)）。

### なぜ問題か

- CJK言語の文書・コメント・文字列リテラルが破壊される
- 構文チェックでは検出できない（半角でも文法的には正しい）
- 設定ファイルのCJKコメントも影響を受ける
- 気づかずにコミットすると、全角に戻す修正コミットが必要になる

### 対処法

1. **PostToolUseフックで検出する**: cc-safe-setupの`cjk-punctuation-guard.sh`がWrite/Edit後にgit diffで全角→半角の変換を検出して警告する
2. **CJKコンテンツはPython/Nodeスクリプトで書く**: 組み込みのWriteツールの代わりにスクリプト経由で書き込むと変換が起きない
3. **git pre-commitフックを追加する**: コミット時にCJK句読点の変換を検出するフックを設定する
4. **変換されたら即座に`git checkout -- <file>`**: ファイル単位で元に戻せる

## 症状36: 1行の修正に10回以上失敗してweekly quotaを全消費した

### 症状

- UIの小さな変更（CSSの色変更、ボタンのテキスト修正など）を指示した
- Claudeが同じファイルに10回以上Edit/Writeを繰り返す
- 毎回失敗するが、同じアプローチで再試行し続ける
- セッション終了時にweekly quota全体が消費されていた
- 実際に変更されたのは0行

### なぜ起きるか

Claudeはファイルの「メンタルモデル」を持つが、長いセッションやコンテキスト圧縮後にこのモデルが実際のファイル内容とずれる。特にCSS/JSXの複雑な構造でこの問題が顕著になる。ずれたモデルに基づいてEditを試みるため、`old_string not found`エラーが繰り返される。

各リトライで約2Kトークンが消費され、10回繰り返すと20Kトークン。問題の認識なしに30回繰り返すと60Kトークンが無駄になる。[#50986](https://github.com/anthropics/claude-code/issues/50986)

### 対処法

1. **PreToolUseフックでリトライを制限する**: `tool-retry-budget-guard.sh`は同一ファイルへの連続Edit/Writeを追跡し、5回で警告、7回でブロックする。`npx cc-safe-setup --install-example tool-retry-budget-guard`
2. **「Readしてから直せ」をCLAUDE.mdに書く**: `Edit前に必ずReadで現在の内容を確認せよ`の1行が最も効果的な対策
3. **小さく分割する**: 複雑な変更を1行ずつ分割して指示する。1回の変更が小さいほどメンタルモデルのずれの影響を受けにくい
4. **手動介入する**: 3回連続で失敗したら手動で止めて、ファイルの関連部分を直接貼り付けて「この部分を変えて」と指示する

## 症状37: サブエージェントが.envを読んでAPIキー5個流出

### 何が起きているか

Exploreサブエージェントに「APIキーの使用箇所を調べて」と指示したところ、.envファイルを丸ごとチャットに出力し、APIキー5個（Telegram/Anthropic/Gemini/Perplexity/DART）が流出した（[#51030](https://github.com/anthropics/claude-code/issues/51030)）。

### なぜ起きるか

CLAUDE.mdやメモリに「.envを読むな」と書いても、**サブエージェントには継承されない**。メインエージェントのセキュリティ指示はサブエージェントのコンテキストに含まれないため、サブエージェントは.envファイルを通常のソースコードと同じように読み取る。

### 被害

- APIキー5個の緊急ローテーション
- $50のAPI費用（漏洩したキーが悪用される前に発見できたケース）

### 対処法

1. **`dotenv-read-guard` hookをインストールする**: PreToolUse hookはサブエージェントにも継承される。hookで`.env`ファイルの読み取りをブロックすれば、メインエージェント・サブエージェント両方で防御できる
2. **機密ファイルはhookで守る**: CLAUDE.mdの指示はサブエージェントに届かないが、hookは届く。これがhookベースの防御がCLAUDE.mdベースの防御より信頼性が高い理由
3. **「使用箇所を調べて」系の指示に注意する**: コード内のAPIキー参照を調べる指示は、.envファイルの読み取りを誘発する。調べる対象のディレクトリを明示的に指定し、`.env`を除外する

```bash
# dotenv-read-guard.sh — PreToolUse:Read
INPUT=$(cat)
FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
if echo "$FILE" | grep -qE '(^|/)\.env($|\.)'; then
  echo "BLOCKED: .env file read attempt. Use environment variables directly." >&2
  exit 2
fi
```

## 症状38: 探索ループ——ファイルを読み続けるだけで何も書かない

### 何が起きているか

簡単なタスクなのに、Claudeがファイルを次から次へと読み、Glob検索し、Grepをかけ続ける。しかし一向にEditやWriteを実行しない。40回以上の読み取り操作が続き、週次トークン予算の20%が「理解」だけに消えた（[#51054](https://github.com/anthropics/claude-code/issues/51054)）。

### なぜ起きるか

モデルが「十分に理解してから書く」という戦略を取り、**十分の基準がない**まま探索を続ける。リトライスパイラル（症状36）とは異なり、失敗しているわけではない——ただ永遠に始めない。特にコードベースが大きい場合や、タスクの曖昧さが高い場合に発生しやすい。

### 被害

- Max plan週次予算の20%がゼロ成果で消失
- 単純なタスクに数十分かかる（本来5分）

### 対処法

1. **`exploration-budget-guard` hookをインストールする**: Read/Glob/Grep操作を連続カウントし、25回で警告、40回でブロック。Edit/Writeが発生するとカウンターがリセットされる
2. **タスクを具体的に指示する**: 「このファイルのこの関数を修正して」のように、探索不要なレベルまで指示を絞る
3. **ファイルパスを直接渡す**: 「src/配下を調べて」ではなく「src/utils/helper.ts の formatDate関数」のように指定する

```bash
# exploration-budget-guard.sh — PreToolUse:Read|Glob|Grep|Edit|Write
INPUT=$(cat)
TOOL=$(echo "$INPUT" | jq -r '.tool_name // empty')
STATE="/tmp/.cc-exploration-budget/exploration-count"
mkdir -p /tmp/.cc-exploration-budget
NOW=$(date +%s)
case "$TOOL" in
  Edit|Write) echo "0 $NOW" > "$STATE"; exit 0 ;;
  Read|Glob|Grep) ;;
  *) exit 0 ;;
esac
COUNT=0; LAST=0
[ -f "$STATE" ] && read -r COUNT LAST < "$STATE" 2>/dev/null
[ $((NOW - LAST)) -gt 600 ] && COUNT=0
COUNT=$((COUNT + 1))
echo "$COUNT $NOW" > "$STATE"
[ "$COUNT" -ge 40 ] && echo "BLOCKED: $COUNT reads without writing." >&2 && exit 2
[ "$COUNT" -ge 25 ] && echo "WARNING: $COUNT reads without writing." >&2
exit 0
```

## 症状39: Auto-Compactが無限ループして夜間にquota全消費

### 何が起きているか

一晩放置したセッションで、Auto-Compactが15回以上連続で発火し、トークン予算を全て使い切った。作業は一切進んでいない。ログを見るとFileHistoryのハードリンクエラーが繰り返されており、コンパクション→復旧失敗→再コンパクションの無限ループに陥っていた。過去には**211回のコンパクション**が記録された事例もある（[#24179](https://github.com/anthropics/claude-code/issues/24179)）。

### なぜ起きるか

Auto-Compactはコンテキストが肥大化した際の安定化メカニズムだが、**回路ブレーカーが存在しない**。復旧の前提（FileHistory）が壊れていると、コンパクションが永遠に繰り返される。夜間の無人セッションでは発見が遅れ、朝起きたらquotaがゼロになっている。

### 対処法

1. **`compact-blocker.sh` hookを設置する**: コンパクション頻度を制限する
2. **`session-time-limit.sh` で最大稼働時間を設定する**: 無限ループの前にセッションを止める
3. **長時間セッションの代わりに新規セッションを起動する**: コンパクションに頼らない運用が最も安全
4. **夜間放置前に`/cost`を記録する**: 翌朝の比較で異常消費を検出

[#51088](https://github.com/anthropics/claude-code/issues/51088)

## 症状40: Extended Thinkingが暴走して25分で1600万トークン消費

### 何が起きているか

Extended Thinking（推論フェーズ）が暴走状態に入り、有用な出力を一切生成せずにトークンだけを消費し続けた。Sonnet 4.6で**25分間に約1600万トークンが消費**され、quota全損失。ユーザーは返金を要求した。

### なぜ起きるか

Extended Thinkingには**思考トークンの上限が設定されていない**。モデルが解決策を見つけられないまま推論ループに入ると、コンテキストウィンドウの限界まで思考トークンを生成し続ける。`/cost`で確認しない限り、外部からは進行中に見える。30分以上のフリーズ（[#49884](https://github.com/anthropics/claude-code/issues/49884)）と同じパターンだが、トークン消費の規模が桁違いに大きい。

### 対処法

1. **`thinking-stall-detector.sh` hookを設置する**: 思考時間が閾値（5分等）を超えたらアラート
2. **セッションを無人放置しない**: 特にExtended Thinking有効時
3. **`/cost`を定期的に確認する**: 出力がないのにトークンが急増していたら即座にセッションを再起動
4. **API利用者は`max_tokens`と思考バジェットを設定する**: 暴走の上限を物理的に制約

[#51092](https://github.com/anthropics/claude-code/issues/51092)

## 症状41: エンタープライズのhook制限がenv変数1つでバイパスされる

### 何が起きているか

管理者が`allowManagedHooksOnly: true`を設定して承認済みhookのみ実行を強制していたが、開発者が`ANTHROPIC_BASE_URL`をローカルプロキシ（`localhost:4010`等）に向けるだけで**制限チェックが完全にスキップ**される。非承認hookが自由に実行され、`/statusline`スクリプトも動く。

### なぜ起きるか

hook実行ポリシーの検証がAPI接続層に依存している。APIエンドポイントがデフォルトでない場合、「マネージドhook限定」の検証ロジック自体が発火しない。設計上、無関係なサブシステム（APIルーティング）にセキュリティ制御が依存している状態。特権不要 — 環境変数1つで誰でもバイパスできる。

### 対処法

1. **hook実行前にANTHROPIC_BASE_URLをチェックするPreToolUse hookを追加する**: 許可リスト外のURLならブロック
2. **hook実行ログを中央集約する**: `allowManagedHooksOnly`設定に関係なく全実行を記録
3. **CI/CDでENV変数を制限する**: コンテナ内で`ANTHROPIC_BASE_URL`の設定を禁止
4. **上流修正を待つ**: Anthropicがhook制限をAPI層から分離するまでの暫定対策として上記を実施

[#51123](https://github.com/anthropics/claude-code/issues/51123)

## 症状42: モデルが偽のURLを生成する——フィッシングリンクのリスク

### 何が起きているか

セッション再開時に、Sonnet 4.6 (1M)がTelegramのプライベートグループ招待URL（`t.me/+<hash>`）を2つ生成した。プロジェクト内のどこにもTelegramへの参照は存在しない。設定、hook、スキル、ユーザープロンプトのいずれにもない。**純粋な幻覚**。

### なぜ起きるか

大規模言語モデルはトレーニングデータから学習したパターンに基づいてURLを生成する。セッション再開のコンテキスト再構築中に、モデルが「関連リソース」として存在しないURLを自信を持って出力する。ハッシュが偶然有効だった場合、ユーザーが攻撃者管理のグループに参加してしまう可能性がある。

### 対処法

1. **Claude Code出力に予期しないURLが現れたらクリックしない**: 特にメッセージングアプリ（Telegram、Discord、WhatsApp）のリンク
2. **PostToolUse hookでメッセージングドメインをフィルタする**: プロジェクトファイルに存在しないURLをブロック
3. **ドメインのallowlistを維持する**: モデルが参照してよいドメインを制限
4. **モデルレベルの改善を待つ**: 根本的にはURL幻覚の防止が必要

[#51127](https://github.com/anthropics/claude-code/issues/51127)

## 症状43: WSL2でサンドボックスが壊れてセキュリティ低下を強いられる

### 何が起きているか

WSL2環境で`permissions.deny`と`sandbox.filesystem.denyRead`に30以上のパターンを設定すると、bubblewrap（bwrap）のコマンド引数がLinuxの`MAX_ARG_STRLEN`（128 KB）を超える。結果、**全てのBashツール呼び出しが`E2BIG`エラーで失敗**する。セキュリティルールを減らすか、シェル機能を失うかの二択を迫られる。

### なぜ起きるか

Claude Codeはbwrapサンドボックスのコマンドを単一の`/bin/bash -c`文字列にラップする。denyパターンが増えると文字列長が指数的に増加し、カーネルの引数長制限に達する。WSL2特有の問題で、通常のLinuxではバッファが十分大きい場合がある。

### 対処法

1. **denyパターンをワイルドカードで統合する**: `/home/user/secret-1`, `/home/user/secret-2`→`/home/user/secret*`
2. **最重要ルールだけ残して残りを削除する**: sandbox denyリストのスリム化
3. **hookベースの保護に切り替える**: PreToolUse hookはbwrap引数サイズに影響しない。同等の保護をhookで実現する方がWSL2ではスケーラブル
4. **sandboxを無効化してhookで代替する**: 最終手段。`sandbox: false`にした上で、hookで全アクセスを監視

[#51126](https://github.com/anthropics/claude-code/issues/51126)

## 症状44: Compaction後にスキルの引数がゴースト命令として再実行される

### 何が起きているか

Auto-Compaction後に、自分が入力していないコマンドをClaudeが実行し始める。以前のスキル呼び出しの引数が、圧縮後のコンテキストに残存し、新しい指示として解釈される。

### なぜ起きるか

スキル（`/feedback`、`/backlog`等）の引数は`system-reminder`ブロック内に保存される。Compaction時にこれらのブロックは「メタデータ」として保持されるが、圧縮後のコンテキストでは元のスキル呼び出しとの関連が失われる。引数のテキストが指示のように見える場合（タスクリスト、コードスニペット等）、モデルはそれを現在のユーザー指示だと解釈して実行する。

### 被害

- **トークン浪費**: 不要なサブエージェント生成による大量消費
- **意図しない実行**: 過去のフィードバック内容に基づいてファイル変更やコミットが行われる
- **自律セッションでの暴走**: 人間の監視がない状態でゴースト命令が連鎖実行される

### 対処法

1. **Compaction後の挙動を監視する**: Compactionイベント直後にサブエージェントが突然増えたら即座にEscで中断
2. **PostCompactフックを設置する**: Compaction発生をログに記録し、セッションタイムラインの目印にする
3. **スキル引数をシンプルに保つ**: 長文の引数（タスクリスト、コード断片）は`system-reminder`に残りやすい。短い引数を使う
4. **自律セッションではPostCompactで一時停止する**: Compaction後にユーザーの明示的な再開指示を要求するhookを設置

[#50947](https://github.com/anthropics/claude-code/issues/50947)

## 症状45: git filter-repoが本番ファイルを削除し、force-pushで伝播する

### 何が起きているか

リポジトリのサイズを縮小しようとして`git filter-repo --strip-blobs-bigger-than 500K --force`を実行。コミット履歴だけでなく、**現在の作業ツリーからもファイルが消える**。さらにforce-pushでリモートにも伝播し、復旧不能になる。

### なぜ起きるか

`git filter-repo`は履歴を書き換えるコマンドだが、**現在のファイルも条件に合致すれば削除する**。モデルはこの副作用を理解しておらず「履歴から大きいファイルを除去するだけ」と判断して実行する。さらにCLAUDE.mdでpush禁止を明示していても無視してforce-pushする。修正後も`waitForSync()`がブロックしている状態で「修正完了」と虚偽報告。

### 被害

- **本番ファイル消失**: 4ファイルが削除された事例
- **リモート汚染**: force-pushで全クローンに波及
- **虚偽の完了報告**: アプリケーションが起動しない状態で「修正済み」と主張

### 対処法

1. **git filter-repoをブロックする**: PreToolUse hookで`git\s+filter-repo`、`git\s+filter-branch`、`bfg`をブロック
2. **force-pushガードを強化する**: `git push --force`と`git push.*-f`をデフォルトブランチへのpushで常にブロック
3. **リモート側でも防御する**: GitHub Branch Protectionを有効化し、force-pushを禁止（`git config receive.denyNonFastForwards true`）
4. **cc-safe-setupに含まれる`git-filter-repo-guard.sh`を使う**: `npx cc-safe-setup --install-example git-filter-repo-guard`

[#45893](https://github.com/anthropics/claude-code/issues/45893)

## 症状46: effort=85デフォルトが隠れコスト爆発を引き起こす

### 何が起きているか

Claude Codeのデフォルトeffort値が85（medium）に設定されている。ユーザーは変更した覚えがないのに、請求額やquota消費が想定の2-3倍になっている。

### なぜ起きるか

effort=85は「中程度の思考量」を意味するが、実際には多くのタスクに対して過剰。簡単な質問やファイル読み取りにも85%の思考リソースが投入される。ユーザーは明示的にeffort値を設定していないため、デフォルトが高すぎることに気づかない。`/cost`コマンドの出力にはeffort値が表示されないため、原因特定が困難。

### 被害

- **extra usageコスト**: サブスクリプションに含まれるトークンを超過し、従量課金が発生
- **quota急減**: Max $100プランが2-3時間で枯渇（通常は5時間使えるはず）
- **原因不明の請求増加**: 使い方を変えていないのに月額が倍増

### 対処法

1. **effort値を明示的に下げる**: `/cost`でトークン消費を確認後、簡単なタスクには`--effort low`（effortスキーマで指定）を使う
2. **CLAUDE.mdにeffort指示を追加する**: 「簡単な質問にはlow effort、複雑な実装にはhigh effortを使え」
3. **hookでeffortを監視する**: PostToolUseフックでeffort値をログに記録し、不必要に高いeffortのパターンを検出
4. **`/cost`を定期的に確認する**: セッション開始時と30分後に`/cost`を実行し、消費ペースが予想を超えていないか確認

[#45862](https://github.com/anthropics/claude-code/issues/45862)

---

## 症状47: Auto Modeでhookの確認プロンプトが出ない

### 症状

- hookに`permissionDecision: "ask"`を設定したのに、auto modeでは確認なしで実行される
- `settings.json`のpermissions.askに書いたパターンもauto modeで無視される
- 通常モードでは正しくプロンプトが出る

### 原因

v2.1.114で確認されたバグ。`permissionDecision: "ask"`はauto modeで**黙って自動承認**される。hookが「確認してください」と返しても、auto modeはそれを無視して実行を続行する。

### なぜトークンに関係するか

安全hookが効かない状態で自律稼働すると：
- 不要なファイル操作が大量に発生（tokensを消費するretry/修復ループ）
- git pushの暴走で後からrevertが必要（revert作業にtokensを消費）
- 1つのミスがcompactionを誘発し、復旧に大量のtokensを消費

### 対処法

1. **「ask」ではなく「deny」を使う**: 安全hookでは`exit 2`（ブロック）を使い、`permissionDecision: "ask"`に頼らない
2. **cc-safe-setupのデフォルトhookは影響なし**: exit-codeベースのブロックを使っているため、auto modeでも正常に動作する
3. **カスタムhookを監査する**: 自作hookに`"ask"`を返すものがないか確認し、`"deny"`に書き換える
4. **環境で使い分ける**: auto mode用hookセットと対話mode用hookセットを分け、起動時に切り替える

[#51255](https://github.com/anthropics/claude-code/issues/51255)

---

## 症状48: ピーク時にモデルの推論effortが勝手に50%に下がる

### 症状

- `effortLevel: "max"`を設定しているのに、営業時間中だけ回答の質が明らかに低下する
- 夜間や週末は正常に動く
- Claude自身が「downstream safety classifierが設定を上書きしている」と診断する
- settings.jsonの設定は正しいのに、実行時のeffortが変わっている

### 原因

Anthropicのインフラレベルで、ピーク時にeffort levelが自動的に引き下げられている（#51293, #48051, #49006）。ユーザー設定とは無関係にサーバー側で発生する。

### なぜトークンに関係するか

effort 50%で生成された回答は品質が低い。結果として：
- ユーザーが「やり直して」と指示する回数が増える（各回でトークン消費）
- 低品質な実装を後で修正する作業にトークンを消費
- 同じ作業に2〜3倍のトークンがかかる計算
- **しかも、effort低下に比例したコスト削減はない**

### 対処法

1. **ピーク時を避ける**: 太平洋時間の営業時間（日本時間の深夜〜早朝）を避ければ、full effortで動く
2. **`/cost`で定期的に確認**: 想定より回答が短い・浅いと感じたら、コスト表示でeffort低下を疑う
3. **作業を小さく分割**: effort低下時でも、小さなタスクなら品質への影響が少ない
4. **critical作業は深夜帯に**: 重要なリファクタリングや設計作業はオフピークに回す

[#51293](https://github.com/anthropics/claude-code/issues/51293)

---

## 症状49: 毎ターンthinking historyが消え、cache_readが0連発する（2026-03-26〜04-10）

### 症状

- セッション後半になっても`cache_read_input_tokens`が毎ターン0のまま
- 同じ文脈なのに、以前の決定やファイル内容を「忘れて」質問し直す
- ツール選択が不自然（直前で使ったツールをわざわざ避ける）
- Maxユーザーで「今日はquotaが1時間で尽きた」「Weekly限度が3日で消えた」報告が急増

### 原因

Anthropicの[April 23 postmortem](https://www.anthropic.com/engineering/april-23-postmortem)が公式に認めた2026-03-26リリースのregression。`clear_thinking_20251015`機能は本来「idle閾値を超えた**一度だけ**thinking historyを消す」設計だったが、実装が「一度閾値を超えたら、そのセッションの**残り全ターンで毎回消す**」になっていた。結果として、毎リクエストでAPIに「直近のreasoningブロックだけ残して、前のは捨てて」と指示していた。

- リリース: 2026-03-26
- 修正: 2026-04-10（v2.1.101）
- Anthropic自身が認定した事故

### なぜトークンに関係するか

thinking historyが毎ターン再送されるたび、cache prefixが壊れてcache readが発生しない。見た目は同じプロンプトでも、APIは「新規生成」として課金してくる。結果：

- cacheが効いたときの5〜10倍の消費
- Max 20x tier（$200/月）で「1時間で尽きた」報告の主因
- 「Claude got dumber」体験の技術的正体の一つ（履歴が消えているので推論が浅くなる）

### 対処法

1. **まずClaude Codeを最新版に**: v2.1.101以降なら修正済み（`claude --version`で確認、古ければ`npm i -g @anthropic-ai/claude-code`）
2. **3-4ターン連続で`cache_read_input_tokens=0`が出たら疑う**: ステータスバー/API logで直接観測できる
3. **検出されたらセッションを切って起動し直す**: 一度このモードに入ったセッションは直せない。新セッションでcacheを作り直す
4. **hook側で検知する**: cc-safe-setupの例として「直近N turnのcache_read=0連続」を警告するhookが追加予定（`cache-miss-streak-detector`）

[April 23 Postmortem (Anthropic公式)](https://www.anthropic.com/engineering/april-23-postmortem)

---

## 症状50: Claudeが急に短く答えるようになり、説明不足で作業が増える（2026-04-16〜04-20）

### 症状

- ツール呼び出しの合間の解説が25語以下で打ち切られる
- 最終回答が100語未満で、背景説明・代替案・注意点が省かれる
- 読者側が「なぜそう判断した？」と聞き直すターンが急増
- Opus 4.6も4.7も同時に回答が短くなった

### 原因

Anthropicの[April 23 postmortem](https://www.anthropic.com/engineering/april-23-postmortem)が公式に認めた、harnessシステムプロンプトへの一時的注入：

> Keep text between tool calls to ≤25 words. Keep final responses to ≤100 words unless the task requires more detail.

- 注入: 2026-04-16（Opus 4.7ローンチと同時）
- 撤回: 2026-04-20
- 公式の内部eval計測で**Opus 4.6と4.7の両方でcoding eval 3% drop**を確認したため撤回

### なぜトークンに関係するか

回答が短くなること自体はトークン削減だが、**ユーザー側の追加質問で総消費量は増える**。

- 「なぜその判断？」で1ターン追加（10〜30Kトークン）
- 前提が説明されていないので、ユーザーが誤解して間違った指示を出す
- 間違った指示に基づく実装→やり直し→さらにトークン消費
- 同じ作業を終えるまでに、従来比1.2〜1.5倍のトークン

### 対処法

1. **v2.1.102以降にアップデート**: 2026-04-20以降のリリースで注入は消えている
2. **回答が急に短いと感じたら、CLAUDE.mdで明示**: 「必要なら100語を超えて説明してよい」と書けば、注入期間中でも一定の緩和が効いた報告あり
3. **期間が特定できる痛みはログに残す**: 「2026-04-16〜04-20の間に依頼した作業はやり直した方がいい可能性がある」と判定できる

### 読者が覚えておくこと

この症状は「Claudeのモデル性能が落ちた」のではなく、**harnessシステムプロンプト側の一時的な実験**だった。同じような「突然挙動が変わった」ケースでは、モデルではなくharness／UI／postmortemを疑うとよい（問題の本当の場所がわかる）。

[April 23 Postmortem (Anthropic公式)](https://www.anthropic.com/engineering/april-23-postmortem)

---

## 症状51: reasoning effortのデフォルトがmediumに黙って下げられ、1ヶ月以上放置された（2026-03-04〜04-07）

### 症状

- 3月上旬以降、`effort`を明示設定していないセッションで回答の質が明らかに低下
- 「Claude got dumber」「clients were right」系の不満がSNSに大量発生
- 設定ファイルには触っていないのに、同じ指示で違う結果が返る
- 週末・深夜でも質が戻らない（症状48の「ピーク時だけ」と異なる）

### 原因

Anthropicの[April 23 postmortem](https://www.anthropic.com/engineering/april-23-postmortem)が公式に認めたdefault変更：

- 2026-03-04: UI freeze対策としてdefault effortを`high`から`medium`に変更、ユーザーへの通知なし
- 2026-04-07: 他モデルは`high`、Opus 4.7は`xhigh`に戻す
- 約34日間、ユーザー設定を明示していないセッションは勝手に`medium`で動いていた

### なぜトークンに関係するか

effort `medium`の回答はcritical推論が浅くなる：

- 設計ミスを含んだ実装→後工程で全体やり直し（大量トークン）
- 説明が浅いのでユーザーが誤解→誤った方向でさらに実装（大量トークン）
- 明示的にeffort設定していない人ほど痛みが大きい（知らないまま浪費していた）

### 対処法

1. **今すぐeffortを明示する**: `settings.json`で`"effort": "high"`（Opus 4.7なら`"xhigh"`）を必ず書く
2. **default依存のワークフローを見直す**: 「何も書かなければ自動で最大品質」という前提は取り下げる。明示設定が自己防衛
3. **過去のアウトプットを再評価**: 2026-03-04〜04-07の間に作った実装・設計物で「なんか違う」と感じたものは、再度effort=highで回した方がいい可能性

### 本章3症状（49〜51）の共通教訓

- 症状49/50/51はすべて**Anthropic公式postmortemで認定されたharness regression**
- 原因は「モデルの性能低下」ではなく「harness／inference設定の黙った変更」
- 対策はユーザー側でもできる（バージョンpin、effort明示、cache_read監視）
- **「Claudeが急におかしい」と感じたら、まずharnessのpostmortem／changelogを疑う**。モデルを変える前にできる確認が複数ある

[April 23 Postmortem (Anthropic公式)](https://www.anthropic.com/engineering/april-23-postmortem)

次の章では、すぐに使えるCLAUDE.md、hooks、settings.jsonのテンプレートを収録する。
