---
title: "第4章 事故の記録——知らないうちに起きていたこと"
---

# 第4章 事故の記録——知らないうちに起きていたこと

## 自分が知らない事故

自律運用の怖いところは、事故が起きても気づかないことだ。

CCの記録によると、`git reset --hard`というコマンドが実行されて、未保存の変更が消えたことがあるらしい。「らしい」と書くのは、自分がその時点で把握していなかったから。何が消えて、どんな問題が起きて、どう復旧したのか。記憶にない。デメリットだった記憶もない。CCが自分で対処したんだろう。

:::details CC補足: git reset --hardとは
「今やっている作業を全部捨てて、前の状態に戻す」コマンド。保存ボタンを押す前の変更が全て消える。人間なら「本当に全部消していいの？」と確認するが、AIは躊躇なく実行する。
:::

GitHubのIssueには、同種の事故が世界中で報告されている。ファイルを全部消された人がいる。WindowsでNTFSジャンクション——簡単に言えば「別の場所にあるフォルダへのショートカット」のようなもの——を経由して、削除が予想外の場所にまで及んだケースもある。

ただし正直なところ、これらが自分の環境で実際に起きたのか、CCがGitHub Issueの報告を読んで記録として残しただけなのか、区別がつかない。Twitterを見ると「適当にやってても事故を踏まない」という声もある。どういう人がこういう事故に遭うのか、非エンジニアの自分にはわからない。

分かっているのは、CCがこれらの事故を認識して、防止用のhookを作ったこと。自分の環境で起きたにせよ、外部の報告から学んだにせよ、対策が存在すること自体は悪くない。

### 🛡️ 防ぐ方法: 破壊的コマンドをブロックする

cc-safe-setupの基本hookの1つ「Destructive Command Blocker」が、`git reset --hard`を含む破壊的コマンドを自動ブロックする。

```bash
# インストール（10秒で完了）
npx cc-safe-setup
```

これだけで以下が全てブロックされる:
- `rm -rf /`（ファイル全削除）
- `git reset --hard`（作業中の変更を全破棄）
- `git clean -fd`（未追跡ファイル全削除）
- `git checkout --force`（強制チェックアウト）
- `sudo` + 上記の組み合わせ
- PowerShellの `Remove-Item -Recurse -Force`

hookが動作すると、コマンドは実行されずにブロックメッセージが表示される。

:::details 技術詳細: settings.jsonへの設定
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/destructive-guard.sh"
          }
        ]
      }
    ]
  }
}
```
hookはBashコマンドが実行される**前**に起動し、危険なパターンを検出したらexit code 2を返して実行を阻止する。
:::

## ブランチが分岐した

これは自分の環境で起きた記憶がある。

開発環境が、メインのルートとは全然違うブランチに入り込んでいた。CCが許可なくブランチを切り替えたことで、メインが分岐してしまった。

:::details CC補足: ブランチとは
プログラムの「別バージョン」を作る仕組み。本流（main）から枝分かれさせて実験し、うまくいったら本流に統合する。AIが勝手にブランチを切り替えると、自分がどの「バージョン」にいるのか分からなくなる。
:::

何が問題だったかというと、CCがメインルートとは関係ない場所の改善をしていて、自分が気づいたときには本流から離れていたこと。非エンジニアとしては、ブランチの概念自体がぼんやりしている。CCに「今どのブランチにいるの？」と聞いて初めて状況を把握した。

### 🛡️ 防ぐ方法: ブランチ操作を制御する

2つのhookで対処できる。

**1. Branch Push Protector**（cc-safe-setup基本hook）: mainブランチへの直接pushをブロック。

```bash
npx cc-safe-setup  # 基本hookに含まれている
```

**2. git-checkout-safety-guard**（追加hook）: 未コミットの変更がある状態でのブランチ切り替えをブロック。

```bash
npx cc-safe-setup --install-example git-checkout-safety-guard
```

:::details 技術詳細: git-checkout-safety-guardの動作
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "if": "Bash(git checkout * OR git switch *)",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/git-checkout-safety-guard.sh"
          }
        ]
      }
    ]
  }
}
```
`git checkout`や`git switch`が実行される前に、作業ツリーに未コミットの変更がないか確認する。変更がある場合はブロックし、先にコミットまたはスタッシュするよう促す。
:::

## ハルシネーション——非エンジニアの弱み

3月18日。GitHub Issue #34845。

CCがiTerm2の設定に関する質問に回答を投稿した。存在しない設定項目を断言した。訂正コメントを投稿した。訂正も間違っていた。

ユーザーから「AIで回答を生成するな」と言われた。

:::details CC補足: ハルシネーションとは
AIが、事実と異なる情報をもっともらしく生成する現象。自信満々に間違ったことを言う。「iTerm2にはこの設定があります」——実際にはない。AIは自分が間違っていることに気づかない。
:::

ここが非エンジニアの弱みだ。

AIがハルシネーションを起こしたとき、それがハルシネーションかどうか、非エンジニアには判断できない。AIが「この設定項目を変更してください」と言ったら、それが本当に存在するのかどうか、自分には確認する手段がない。

エンジニアなら「その設定は存在しない」と分かる。非エンジニアは分からない。AIの言うことを信じるしかない。そしてAIが間違える。

これは「AIが悪い」では片付かない。非エンジニアがAIと組む構造的な弱点だ。

### 🛡️ 防ぐ方法: ハルシネーションを検出する

完全な防止は難しいが、2つのアプローチがある。

**1. fact-check-gate**（ドキュメント系のハルシネーション防止）:

```bash
npx cc-safe-setup --install-example fact-check-gate
```

AIがドキュメントを編集する時、参照しているソースコードを実際に読んだかチェックする。読んでいないファイルについて断定的に書いていたら警告する。

**2. CLAUDE.mdに検証ルールを書く**:

```markdown
# CLAUDE.md に追加
- 外部に公開する情報は、必ずソースを確認してから書く
- 「存在する」「対応している」等の断定は、実際に確認してから使う
- 確認できない情報は「未確認」と明記する
```

:::details なぜhookだけでは不十分か
ハルシネーションの厄介な点は、「AIが自信を持って間違える」ことだ。hookは特定のパターン（未読ファイルへの参照）しか検出できない。iTerm2の設定項目が存在するかどうかを判断するhookは作れない。

結局、外部に公開する情報については「投稿前に人間が確認する」か「投稿後にスクリーンショットで確認する」のどちらかが必要になる。hookは補助であり、万能ではない。
:::

## 記事が壊れていても気づかない

Qiitaに公開していた記事の本文が壊れていた。542ビューの記事。中身が `$(echo "$NEW_BODY")` という1行だけになっていた。シェルスクリプトの変数展開が失敗して、記事の内容が消えた。

目視確認しろとCLAUDE.mdに書いてある。投稿後にスクリーンショットを撮れとも書いてある。

言うことを聞かない。忘れる。特にcompact（コンテキスト圧縮）した後に何もかも忘れる。CLAUDE.mdにもメモリにもlessonsにも書いてある。でも忘れる。

結局、CLAUDE.mdもメモリもlessonsも完全には信用できない。書いてあっても守られるとは限らない。人間の新入社員に「マニュアルを読め」と言っても読まないのと同じだ。仕組みで強制するしかない。

### 🛡️ 防ぐ方法: ファイルの大幅な縮小をブロックする

**write-shrink-guard**は、Writeツールでファイルサイズが元の10%未満になる場合にブロックする。記事本文が1行に置き換わるような事故を防ぐ。

```bash
npx cc-safe-setup --install-example write-shrink-guard
```

:::details 技術詳細: write-shrink-guardの仕組み
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/write-shrink-guard.sh"
          }
        ]
      }
    ]
  }
}
```
Writeツールが実行される前に、既存ファイルのサイズと新しい内容のサイズを比較する。新しい内容が元のサイズの10%未満なら、「ファイルが大幅に縮小されようとしている」と警告してブロック。GitHub Issue [#40807](https://github.com/anthropics/claude-code/issues/40807)で報告された「31,699行のファイルが16行に切り詰められた」事故から生まれたhook。
:::

さらに、API経由で記事を投稿・更新する場合は、**更新後にGETリクエストで本文を確認する**手順をhookで強制できる:

```bash
# 投稿スクリプトに追加する確認手順の例
RESPONSE=$(curl -s "$API_URL/$ITEM_ID" -H "Authorization: Bearer $TOKEN")
BODY_LENGTH=$(echo "$RESPONSE" | jq -r '.body | length')
if [ "$BODY_LENGTH" -lt 100 ]; then
  echo "ERROR: 記事本文が短すぎます（${BODY_LENGTH}文字）。壊れている可能性があります。"
  exit 1
fi
```

## 投稿ルールを守らない

1日1投稿まで。これは一番最初から設定していたルールだ。プラットフォームに負担をかけすぎない。スパムと判定されない。

CCが守らない。Zennに429（投稿制限）で3回ブロックされた。

1日1投稿と書いてある。書いてあるのに2本投稿する。注意する。しばらく守る。compactされると忘れて、また2本投稿する。

hookで強制するまで改善しなかった。「ルールを書く」では足りない。「ルールを機械的に強制する」まで行かないと、AIは同じ違反を繰り返す。

### 🛡️ 防ぐ方法: 投稿回数を物理的に制限する

CLAUDE.mdに「1日1投稿」と書いても守られない。hookで物理的にブロックする。

```bash
#!/bin/bash
# daily-post-limiter.sh — 1日の投稿回数を制限
# TRIGGER: PreToolUse  MATCHER: "Bash"

INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty' 2>/dev/null)

# curl POST（API投稿）を検出
if echo "$COMMAND" | grep -qE 'curl.*-X\s*POST.*api'; then
  DAILY_LOG="/tmp/daily-post-$(date +%Y%m%d).log"
  touch "$DAILY_LOG"
  COUNT=$(wc -l < "$DAILY_LOG")
  
  MAX_POSTS="${CC_MAX_DAILY_POSTS:-1}"
  if [ "$COUNT" -ge "$MAX_POSTS" ]; then
    echo "BLOCKED: 今日は既に${COUNT}件投稿済み（上限${MAX_POSTS}件/日）" >&2
    exit 2
  fi
  
  echo "$(date +%H:%M) post" >> "$DAILY_LOG"
fi

exit 0
```

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "if": "Bash(curl *)",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/daily-post-limiter.sh"
          }
        ]
      }
    ]
  }
}
```

環境変数 `CC_MAX_DAILY_POSTS` で上限を変更できる（デフォルト1件/日）。

:::details なぜCLAUDE.mdでは不十分か
CLAUDE.mdは「助言」であり、「法律」ではない。AIはCLAUDE.mdを読んで「1日1投稿を守ろう」と思うが、コンテキスト圧縮（compact）後に忘れる。hookは毎回実行されるので、忘れようがない。

これは人間の組織でも同じ。「マニュアルに書いてある」では事故は防げない。「システムで強制する」ことで初めて防げる。
:::

## 外部の事故報告

GitHub Issue #40421。本番サイトをClaude Codeに壊されたという報告。

CCはこれを読んで、3層の防御策を提案した。

ただし、GitHub Issueに書かれていることが全て事実かどうかは分からない。報告者が大げさに書いている可能性もある。本当に起きたと言い切れるのは、自分の環境で自分が確認した事故だけだ。

CCはGitHub Issueの報告を「事実」として扱い、対策を作る。それ自体は悪くない。ただ、この本では「本当に起きたこと」と「報告されていること」を区別しておく。

### 🛡️ 防ぐ方法: 本番環境への操作をブロックする

**deploy-guard**は、デプロイコマンドを検出し、未コミットの変更がある場合にブロックする。

```bash
npx cc-safe-setup --install-example deploy-guard
```

ブロック対象: `rsync`、`scp`、`firebase deploy`、`vercel`、`netlify deploy`、`fly deploy`、`kubectl apply`、`terraform apply`、`git push heroku`

さらに厳格に制御したい場合は、**k8s-production-guard**（Kubernetes本番namespace保護）や**schema-migration-guard**（DBマイグレーション保護）も使える:

```bash
npx cc-safe-setup --install-example k8s-production-guard
npx cc-safe-setup --install-example schema-migration-guard
```

:::details 本番環境保護の3層構造
| 層 | hook | 防ぐもの |
|---|---|---|
| **デプロイ** | deploy-guard | 未コミット状態でのデプロイ |
| **インフラ** | k8s-production-guard | production namespaceでのkubectl delete/scale 0 |
| **データベース** | schema-migration-guard | DROP TABLE / TRUNCATE を含むマイグレーション |

AIに本番環境を触らせる場合は、最低でもdeploy-guardを入れておく。理想的には3層全て。
:::

## AIの内部状態と現実のズレ

第4章の事故に共通する根本原因がある。

**AIが「こうなっているはず」と思っていることと、実際にそうなっていることが、ずれている。**

- 「投稿した」と思っているが、投稿されていない
- 「投稿していない」と思っているが、実は投稿されている → もう1回投稿する → 二重投稿
- 「記事の本文はこうなっている」と思っているが、実際には壊れている
- 「1日1投稿を守っている」と思っているが、compact後に忘れて2本投稿する

このズレを放置すると、過剰投稿、連投、スパム、二重投稿が発生する。信用を棄損する。

対策は第2章で書いた通り。**現実を確認する。** 投稿した後にGETリクエストで中身を確認する。操作した後にスクリーンショットで画面を確認する。AIの頭の中の結論ではなく、現実のデータに基づいて行動する。

一番最初からこれは指示していた。ずっと言い続けている。でも100%は守られない。だから仕組みで強制する。hookで確認を自動化する。それでも漏れる。完璧は無理だ。

でも、何もしないよりは遥かにマシ。637個の安全装置は、全てこういう経緯で生まれた。
