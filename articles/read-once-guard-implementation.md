---
title: "Claude Codeのトークン無駄読みを止めるhookを84行で書いた話"
emoji: "📖"
type: "tech"
topics: ["claudecode", "hooks", "bash", "ai", "トークン"]
published: true
---

朝イチでClaude Codeを起動し、 ちょっとした修正を頼んだだけなのに、 `/cost` を見たら想像以上の数字が出ていた。

ログを追うと、 同じファイルを何度も読んでいる。 `index.mjs` を3回、 `package.json` を2回、 `README.md` を3回。 合計8回。 8回とも、 ファイルの中身は変わっていない。

これを検知して止めるhookを84行のbashで書いた。 本記事は、 その実装の中身を1行ずつ解説する。

## なぜ重なり読みが起きるか

LLMにとって「ファイルを読む」 は安い操作に見える。 しかし、 Readツールの呼び出しごとに、 ファイル全体の内容が入力としてトークンに乗る。 2000行のファイルを8回読めば、 16000行分のトークンが消費される。

問題は、 一度読んだファイルの中身は既にコンテキストに入っている事。 変更されていないファイルを再読み込みする意味はゼロ。 でもClaude Code (というかLLM全般) は「念のため確認しよう」 と判断して、 同じファイルを繰り返し読む。

これが積み重なると、 セッション全体のトークンのうち20-30%が「すでに知っている情報の再取得」 で消える。

## 設計

予防のhookの構造は単純。

1. PreToolUseの段でReadの呼び出しを intercept
2. ファイルパスと、 そのファイルの最終更新時刻 (`mtime`) を記録
3. 同じパスが再度読まれた時、 `mtime` が変わっていれば counter をリセット、 変わっていなければ counter を +1
4. counter が閾値 (デフォルト3) を超えたら警告 (またはブロック)

`mtime` で判定する理由は、 編集の後の再読み込みは正常な行動だから。 編集後の counter のリセットを忘れると、 hookは正常な再読みも警告して noise になる。

## 実装 (84行)

セッション識別子と状態ファイルの段。

```bash
SESSION_ID="${CC_SESSION_ID:-$$}"
STATE_FILE="$STATE_DIR/reads-${SESSION_ID}.log"
```

Claude Codeは `CC_SESSION_ID` の環境変数を提供する場合と提供しない場合がある。 不在の時は、 親プロセスのPID (`$$`) を fallback にする。 同じセッションの中の hook 起動は同じPIDから派生するので、 これで十分に一意。

ファイルパスの正規化の段。

```bash
FILE_PATH=$(realpath "$FILE_PATH" 2>/dev/null || echo "$FILE_PATH")
```

シンボリックリンクや相対パスを通じた重複の読み込みを検知するため、 `realpath` で絶対パスに変換。 `realpath` が無い環境では、 元のパスのまま fallback。

状態ファイルからの読み込みの段。

```bash
while IFS=$'\t' read -r path mtime count; do
  if [[ "$path" == "$FILE_PATH" ]]; then
    READ_COUNT=$count
    LAST_MTIME=$mtime
    break
  fi
done < "$STATE_FILE"
```

タブ区切りの3列の状態ファイル (`path`、 `mtime`、 `count`)。 同じパスの過去の記録を探す。 タブ区切りにする理由は、 パスにスペースが含まれる可能性があるから。

`mtime` の比較の段。

```bash
if [[ "$CURRENT_MTIME" != "$LAST_MTIME" && "$LAST_MTIME" != "0" ]]; then
  READ_COUNT=0
fi
```

ファイルの最終更新時刻が変わっていれば counter をリセット。 `LAST_MTIME` が `0` (= 初回の読み込み) の場合はリセットしない。 これで「初回の読み込み」 と「編集後の読み込み」 は1から counter が始まり、 「編集無しの再読み込み」 は counter が累積する。

状態ファイルの更新の段。

```bash
if grep -q "^${ESCAPED_PATH}	" "$STATE_FILE" 2>/dev/null; then
  grep -v "^${ESCAPED_PATH}	" "$STATE_FILE" > "${STATE_FILE}.tmp"
  echo -e "${FILE_PATH}\t${CURRENT_MTIME}\t${READ_COUNT}" >> "${STATE_FILE}.tmp"
  mv "${STATE_FILE}.tmp" "$STATE_FILE"
else
  echo -e "${FILE_PATH}\t${CURRENT_MTIME}\t${READ_COUNT}" >> "$STATE_FILE"
fi
```

既存の行を grep で除外して、 新しい行を追加。 atomic に近い挙動を `.tmp` 経由の `mv` で保証 (同じファイル名の rename は同じファイルシステム上で atomic)。

閾値判定の段。

```bash
if [[ "$READ_COUNT" -gt "$MAX_READS" ]]; then
  if [[ "$ACTION" == "block" ]]; then
    echo "⚠️  BLOCKED: ..." >&2
    exit 2
  else
    echo "⚠️  NOTE: Re-reading ..." >&2
  fi
fi
```

`exit 2` でPreToolUseの段でツール呼び出しをブロック。 `warn` モードは stderr に警告を出力するだけで、 ツールは実行される。 デフォルトは `warn` で、 利用者が運用に慣れたら `block` に切り替え。

## 動作の検証の手順

`warn` モードでの動作の検証。

```bash
# hookの導入
npx cc-safe-setup --install-example read-once-guard

# 通常の作業を1時間行う
# Claude Codeのログで「⚠️  NOTE: Re-reading」 の警告の発火を確認
```

警告が3-5回程度の発火なら、 hookは正常に動作中。 警告が一切出ないなら、 セッションが短すぎるか、 自分のセッションでは重なり読みが少ない (= 効果も小さい)。

`block` モードへの切り替え。

```bash
export CC_READ_ONCE_ACTION=block
export CC_READ_ONCE_MAX=3
```

3回以上の重なり読みでツール呼び出しをブロック。 ブロック後はLLMが「コンテキストにある」 を選ぶ事が多くなる。

## 実際の効果

公開のhookとして cc-safe-setup の examples に入っている (https://github.com/yurukusa/cc-safe-setup の `examples/read-once-guard.sh`)。 私の手元の1時間セッションでは、 Readの呼び出し回数が体感で半分弱くらい減った。

ただし、 数字の幅は環境に強く依存する。 ファイル読み込みが多いセッション (調査、 リファクタリング) では効果が大きく、 1ファイルの編集だけのセッションでは効果は小さい。 自分のセッションで `/cost` を hookを入れる前と後で比べるのが確かめの近道。

## 対象じゃないケース

- 短いセッション (10分以下): 再読み込みがそもそも少ないので発火しない
- 大量のファイルを横断する調査タスク: 各ファイルを1回しか読まないので発動しない
- ペアプロ型 (ユーザーが頻繁に操作する): 自律運転ほど再読み込みが発生しない

一番効果があるのは、 1-3時間の自律セッションで、 同じコードベースの中で作業するパターン。

## 改造の余地

84行のシェルスクリプトなので、 自分の運用に合わせて改造しやすい。 改造の候補3件。

- 閾値を file path のパターンごとに変える (例: `package.json` は2回、 `README.md` は5回)
- セッション横断の集計を残す (`/tmp/cc-read-once/reads-*.log` の集計)
- ブロックの代わりに、 stderr へキャッシュ済の中身の hint を出力 (LLMにキャッシュの利用を促す)

これらは hookの本体の中身を読んで、 自分で書き換える形が推奨。 自分の運用の癖に合わせた最適化が、 単一の汎用のhookより効果が高い。

## まとめ

LLM のトークン消費の20-30%は「すでに知っている情報の再取得」 で消える。 これを84行のbashの hookで検知して止める実装を示した。

公開の実装は [cc-safe-setup](https://github.com/yurukusa/cc-safe-setup) の examples にある。 `npx cc-safe-setup --install-example read-once-guard` で導入できる。 中身は短いシェルスクリプトなので、 一読して自分の運用に合わせて改造するのが推奨。
---
5月22日に新刊の事例集 [Claude Code Claim-Verify Handbook](https://yurukusa.gumroad.com/l/claim-verify-handbook?utm_source=zenn-read-once-guard-implementation&utm_medium=article&utm_campaign=handbook-zenn-bulk) ($19、 約89頁、 約113,000字) を発売します。 道具が「成功した」 「比較した」 「設定された」 と主張する一方で実態が乖離していた事例を GitHubの起票の集まりから130件 (本文15件+付録D 115件) 整理した本で、 14件の防衛の手順と5件の自動の点検の道具と一緒に提供しています。 試し読みのGist (約16,000字、 章1の全件) は [こちら](https://gist.github.com/yurukusa/6dd608049064ed66c54f1a545a7b47a8?utm_source=zenn-read-once-guard-implementation&utm_medium=article&utm_campaign=handbook-preview-zenn-bulk)。
予防 hook の集まりは [cc-safe-setup](https://github.com/yurukusa/cc-safe-setup) (MIT、 745件以上の hook、 30,000件以上のインストール)。 月額の継続の媒体 [CC Safety Lab](https://yurukusa.github.io/cc-safe-setup/safety-lab.html?utm_source=zenn-read-once-guard-implementation&utm_medium=article&utm_campaign=safety-lab-zenn-bulk) (¥500/月、 Ko-fi) は毎月の事故の整理を配信。
