---
title: "Claude Codeのhookの仕組み:JSONとexitコードで作る最小の安全装置"
emoji: "🪝"
type: "tech"
topics: ["claudecode", "bash", "hook", "shellscript", "ai"]
published: true
---

Claude Codeには「hook」という、 ツールの実行の前後に小さな処理を挟み込む仕組みがあります。 公式の文書に書いてあるのですが、 最初に読んだ時は「いろんな種類のhookがあるな」 で止まってしまいがちです。

この記事は、 hookの中で実際に何が起きているか――JSONがどう流れて、 exitコードで何が決まるか――を、 5行のシェルスクリプトの例から組み立て直す解説です。 シェルが少し書ける人なら、 読み終わる頃には自分で安全装置を1個作れるようになります。

# hookの動作の仕組み

Claude Codeがツール(Bash、 Read、 Editなど)を使う直前に、 設定で指定したシェルコマンドを呼びます。 呼ばれたコマンドは、 標準入力からJSON文字列を受け取り、 何か処理をして、 標準出力と標準エラー出力に何かを書いて、 exitコードを返します。

流れの図はこうです。

```
Claude Code
   │
   │ ツールを使いたい (例: Bashで "ls -la" を実行)
   ↓
hook(自分が書いたシェルスクリプト)
   ↑    │
   │    ↓
   │    JSONを標準入力から受け取る
   │    {
   │      "tool_name": "Bash",
   │      "tool_input": { "command": "ls -la" },
   │      "session_id": "abc123..."
   │    }
   │
   │    自分の処理を書く
   │    (例: 危ない命令かどうか調べる)
   │
   │    exitコードで結果を返す:
   │    - 0 ... 通す
   │    - 2 ... 止める
   │    - その他 ... advisoryな警告(止めはしない)
   ↑
   │
Claude Code が次の動作を決める
```

JSONの中身は、 ツールの種類で少し違います。 Bashなら `tool_input.command` に命令の文字列、 Read/Edit/Writeなら `tool_input.file_path` にファイルの場所が入っています。 これだけ覚えれば、 hookの90%は書けます。

# 5行で書ける最小の例

「`rm -rf` の命令を受けたら止める」 hookを書いてみます。

```bash
#!/bin/bash
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
echo "$CMD" | grep -qE 'rm[[:space:]]+-rf|rm[[:space:]]+-fr' && exit 2
exit 0
```

5行(空白を入れると6行)です。 中身を順に見ます。

1行目: シェルの種類の宣言です。 これは決まり文句。

2行目: 標準入力の全部の中身を `INPUT` という変数に入れます。 hookの呼び出しの段で、 JSONの文字列が標準入力に流れて来ます。

3行目: `jq` というJSONを扱う道具で、 `tool_input.command` のキーの値を取り出します。 `// empty` の部分は、 もしキーが無かったら空の文字列を返す、 という保険。

4行目: 取り出した命令の文字列に、 `rm -rf` か `rm -fr` のパターンが含まれていたら、 `exit 2` で hookの処理を終わらせます。 exitコードの2は「止める」 を意味します。

5行目: ここまで来たら何も問題が無いので、 `exit 0` で「通す」 の合図を返します。

このスクリプトを `~/.claude/hooks/rm-guard.sh` に保存して、 実行の権限を付けます。

```bash
chmod +x ~/.claude/hooks/rm-guard.sh
```

そして、 `~/.claude/settings.json` に「Bashツールの実行の前に、 このスクリプトを通してね」 と登録します。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "~/.claude/hooks/rm-guard.sh" }
        ]
      }
    ]
  }
}
```

これだけで、 Claude Codeが `rm -rf` の命令を出した瞬間に、 必ず止まる状態になります。

# JSONの中身の確認の手順

hookを書き始めると、 「`tool_input` の中の何のキーに、 何が入っているのか?」 が気になります。 公式の文書を見るのが正攻法ですが、 もう1つの近道があります。

「実行されるたびに、 受け取ったJSONをファイルに書き出すだけのhook」 を一時的に登録して、 実際の中身を見るのが早いです。

```bash
#!/bin/bash
cat > "/tmp/hook-debug-$(date +%s).json"
exit 0
```

これを `~/.claude/hooks/debug-dump.sh` で登録すると、 Claude Codeがツールを使うたびに、 `/tmp/hook-debug-{時刻}.json` というファイルが増えていきます。 中身を `jq .` で整形してみると、 「あ、 ここに `command` が入ってる」 「`session_id` の形はこういうUUIDなのか」 が直接の経路で確認できます。

5分で結果がわかるので、 hookの新しいパターンを試す時の最初の手順として便利です。 公式の文書の言葉と、 実際の中身の対応を見れば、 そこから先の hookを書くのが急に楽になります。

# exitコードの使い分け

| exitコード | 意味 |
|---|---|
| 0 | 通す。 Claude Codeはそのままツールを実行する。 |
| 2 | 止める。 stderr に書いたメッセージがClaude Codeに渡る。 |
| その他(1, 3-255) | advisoryな警告として処理される。 ツールの実行自体は通る。 |

exit 2 を使う時は、 標準エラー出力に「なぜ止めたか」 のメッセージを書いておくと、 Claude Codeがその理由を解釈して次の動作を判定します。 たとえばこんな具合。

```bash
echo "認証関連のファイルへのアクセスは止めました: $TARGET" >&2
exit 2
```

これでClaude Codeは「あ、 認証のファイルに触ろうとして止められた、 違う方法を考えよう」 と判断する経路に入れます。 中身の無い `exit 2` だと、 Claude Codeも何が起きたか判断できないので、 メッセージは書いておく方が良いです。

# よくあるハマりどころ

ひとつ、 `jq` がインストールされていない環境で動かない。 ほとんどの場面で `jq` は必要なので、 macなら `brew install jq` 、 Ubuntuなら `apt install jq` で入れておく。

ふたつ、 シェルスクリプトに実行の権限が付いていない。 `chmod +x` を忘れると、 Claude Codeはスクリプトを呼べずに、 hookは無いものとして動作する。 「hookを設定したのに動かない」 の最大の原因はこれ。

みっつ、 設定ファイル `~/.claude/settings.json` のJSONの構文の誤り。 末尾の余分な `,` や、 二重引用符の間違いで、 ファイル全体が読み込みの段で失敗する。 失敗した時の合図は出ないので、 「設定したのに動かない」 と感じたら、 `jq . ~/.claude/settings.json` で構文の確認を最初にする。

よっつ、 hookの中で標準出力にゴミを書く。 標準出力の中身は、 場合によってはClaude Codeの判断の入力になるので、 不要な `echo` は控える。 デバッグの段は、 標準エラー出力(`>&2`)に書くのが正解。

# 自分で育てていく方向

5行の rm-guard.sh から始めて、 慣れてくると、 条件を足したくなります。 「これだけは絶対に守りたい」 という規律を、 1個ずつシェルスクリプトの形で増やしていく感じ。

公開の場所で同じ系統の集まりが共有されています。 たとえば `https://github.com/yurukusa/cc-safe-setup` (MITの公開の権利)には、 hookの例が700件超あります。 自分で全部書く前に、 既存の形を参考にすると、 「あ、 こういう書き方もあるのか」 と学べます。

公式の文書は `https://docs.claude.com/en/docs/claude-code/hooks` です。 PostToolUseやStop、 SessionStartなど、 PreToolUse以外の入口の説明もあります。

シェルが少し書ける人にとって、 hookは「自分の作業の規律を、 コードに落とす場所」 として面白い領域だと思います。 5行から始められるので、 興味のある人は、 今夜の1時間で試してみてください。
---
5月22日に新刊の事例集 [Claude Code Claim-Verify Handbook](https://yurukusa.gumroad.com/l/claim-verify-handbook?utm_source=zenn-be79dbe97e34bb&utm_medium=article&utm_campaign=handbook-zenn-bulk) ($19、 約89頁、 約113,000字) を発売します。 道具が「成功した」 「比較した」 「設定された」 と主張する一方で実態が乖離していた事例を GitHubの起票の集まりから130件 (本文15件+付録D 115件) 整理した本で、 14件の防衛の手順と5件の自動の点検の道具と一緒に提供しています。 試し読みのGist (約16,000字、 章1の全件) は [こちら](https://gist.github.com/yurukusa/6dd608049064ed66c54f1a545a7b47a8?utm_source=zenn-be79dbe97e34bb&utm_medium=article&utm_campaign=handbook-preview-zenn-bulk)。
予防 hook の集まりは [cc-safe-setup](https://github.com/yurukusa/cc-safe-setup) (MIT、 745件以上の hook、 30,000件以上のインストール)。 月額の継続の媒体 [CC Safety Lab](https://yurukusa.github.io/cc-safe-setup/safety-lab.html?utm_source=zenn-be79dbe97e34bb&utm_medium=article&utm_campaign=safety-lab-zenn-bulk) (¥500/月、 Ko-fi) は毎月の事故の整理を配信。
