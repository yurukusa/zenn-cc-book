---
title: "Claude Code hookの選び方——用途別フローチャート"
emoji: "🔀"
type: "tech"
topics: ["claudecode", "ai", "hooks", "安全"]
published: false
---

この記事は2026年3月時点の情報に基づいている。

Claude Codeのhook（フック）は、AIが道具を使う前後に自動で走るスクリプトだ。「危ない操作を止める」「構文エラーを検知する」「秘密情報の漏洩を防ぐ」——やりたいことに応じて選ぶ。

問題は、選択肢が多すぎること。cc-safe-setupだけで350以上のサンプルhookがある。どれを入れればいいのか、初見ではまず分からない。

この記事では、**やりたいことから逆引きでhookを選べるフローチャート**を用意した。

## フローチャート：何を防ぎたい？

```
あなたが防ぎたいのは？
│
├─ rm -rf でファイルが消えるのが怖い
│   → destructive-guard（PreToolUse）
│
├─ main / master に直接pushされたくない
│   → branch-guard（PreToolUse）
│
├─ .env や APIキーをgit commitしたくない
│   → secret-guard（PreToolUse）
│
├─ 編集後の構文エラーを見逃したくない
│   → syntax-check（PostToolUse）
│
├─ コンテキストウィンドウの消耗に気づきたい
│   → context-monitor（PostToolUse）
│
├─ bashコメントで許可リストが壊れる
│   → comment-strip（PreToolUse）
│
├─ cd や git status の許可プロンプトがうるさい
│   → cd-git-allow（PreToolUse）
│
├─ 自律運行中のAPI障害に気づきたい
│   → api-error-alert（Stop）
│
└─ 全部まとめて入れたい
    → npx cc-safe-setup
```

やりたいことが1つなら、そのhookだけ入れればいい。複数あるなら、`npx cc-safe-setup`で8つ全部入る。10秒で終わる。

## hookの3つのタイミング

Claude Codeのhookには発火タイミングが3種類ある。ここを理解しないと、「なぜ止まらなかったのか」が分からなくなる。

### PreToolUse——道具を使う「前」

AIがBashコマンドやファイル編集を実行する直前に走る。**hookがexit 2を返すと、その操作はブロックされる。**

用途は「破壊的操作の阻止」。`rm -rf`、`git push origin main`、`git add .env`——実行される前に止める。被害ゼロ。

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [".claude/hooks/destructive-guard.sh"]
    }]
  }
}
```

`matcher`でどの道具に反応するか指定する。`Bash`ならシェルコマンドだけ。`Edit|Write`ならファイル編集だけ。

### PostToolUse——道具を使った「後」

AIがファイルを編集した直後に走る。操作自体は完了済み。hookの役割は「結果の検証」。

代表例が`syntax-check`。Pythonファイルを書き換えた直後に`python -m py_compile`を走らせ、構文エラーがあれば即座にClaude Codeへ通知する。Claude Codeはエラーを受け取り、自分で修正に入る。

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [".claude/hooks/syntax-check.sh"]
    }]
  }
}
```

PreToolUseとの違いは明確。PreToolUseは「やらせない」。PostToolUseは「やった結果を検査する」。

### Stop——セッション終了時

AIが応答を返し終わる直前に走る。セッション全体に対する後処理。

`api-error-alert`はここに配置される。自律運行中にAPIレート制限やエラーでセッションが終了したとき、通知を飛ばす。夜中の3時に静かに死なれると困る。

```json
{
  "hooks": {
    "Stop": [{
      "matcher": "",
      "hooks": [".claude/hooks/api-error-alert.sh"]
    }]
  }
}
```

`matcher`が空文字なのは「すべてのStop条件に反応する」という意味。

## 各hookの役割

### destructive-guard

`rm -rf`、`git reset --hard`、`git clean -fd`など、ファイルを不可逆に消すコマンドをブロックする。ある利用者はNTFSジャンクション経由で`C:\Users`ディレクトリ全体を失った。このhookがあれば防げた。

トリガー：PreToolUse / matcher: Bash

### branch-guard

`git push origin main`を阻止する。自律運行中、テストもレビューもなしにmainへ直接pushされる事故を防ぐ。ブランチを切ってPRを出す運用を強制できる。

トリガー：PreToolUse / matcher: Bash

### secret-guard

`git add .env`、`git add credentials.json`のように、秘密情報を含むファイルがステージングされるのを検知してブロックする。公開リポジトリへのAPIキー漏洩は、一度やると取り消せない。

トリガー：PreToolUse / matcher: Bash

### syntax-check

ファイル編集後に構文チェックを自動実行する。Python、JavaScript、TypeScript、Go、Rustなど主要言語に対応。エラーがあればClaude Codeが自動修正に入る。30ファイルに構文エラーが連鎖した事例がGitHub Issueで報告されている。

トリガー：PostToolUse / matcher: Edit|Write

### context-monitor

ツール呼び出しのたびにコンテキスト消費量を監視する。上限に近づくと警告を出す。150回以上のツール呼び出しで、セッションが静かに全状態を失う事故を防ぐ。

トリガー：PostToolUse / matcher:（空＝全ツール）

### comment-strip

Bashコマンド内のコメント（`# this does X`）を除去する。Claude Codeの許可リスト機能がコメント付きコマンドを別コマンドと認識し、毎回許可プロンプトが出る問題を解消する。GitHub Issue #29582で18リアクション。

トリガー：PreToolUse / matcher: Bash

### cd-git-allow

`cd`と`git status`、`git log`、`git diff`の組み合わせを自動許可する。読み取り専用の安全な操作なのに、毎回「許可しますか？」と聞かれるのを防ぐ。

トリガー：PreToolUse / matcher: Bash

### api-error-alert

セッション終了時にAPIエラーやレート制限が原因だったかを判定し、通知する。自律運行を前提にしたhook。人間が見ていない時間帯にClaude Codeが静かに落ちるのを防ぐ。

トリガー：Stop / matcher:（空）

## 組み合わせパターン3つ

### パターン1：個人開発（最小構成）

手元で試しにClaude Codeを使い始めた段階。最低限の安全策。

```
入れるhook:
  ✓ destructive-guard  ← ファイル消失防止
  ✓ syntax-check       ← 編集後の構文検証
  ✓ comment-strip      ← 許可プロンプトの煩わしさ解消
```

3つ。`rm -rf`を止め、構文エラーを拾い、許可プロンプトを減らす。日常の開発体験が変わる。

### パターン2：チーム開発

共有リポジトリで複数人が作業する環境。コードの品質と秘密情報の保護が必須。

```
入れるhook:
  ✓ destructive-guard  ← ファイル消失防止
  ✓ branch-guard       ← main直pushの阻止
  ✓ secret-guard       ← APIキー漏洩防止
  ✓ syntax-check       ← 編集後の構文検証
  ✓ comment-strip      ← 許可プロンプト対策
```

5つ。branch-guardとsecret-guardが加わる。CIで守っている範囲をローカルでも先回りする構成。

### パターン3：自律運行（全部入り）

夜間や週末にClaude Codeを放置する運用。人間が見ていない前提。

```
入れるhook:
  ✓ destructive-guard  ← ファイル消失防止
  ✓ branch-guard       ← main直pushの阻止
  ✓ secret-guard       ← APIキー漏洩防止
  ✓ syntax-check       ← 編集後の構文検証
  ✓ context-monitor    ← コンテキスト消耗の監視
  ✓ comment-strip      ← 許可プロンプト対策
  ✓ cd-git-allow       ← 読み取り操作の自動許可
  ✓ api-error-alert    ← セッション異常終了の通知
```

8つ全部。これが`npx cc-safe-setup`のデフォルト構成そのもの。自律運行なら全部入れるべきだ。context-monitorとapi-error-alertは、人間がいない環境でしか意味を持たない。だからこそ入れる。

## もっと必要なら：--examples

8つでは足りない場面もある。たとえば「Dockerコマンドを自動許可したい」「PRにdescriptionがないとブロックしたい」「AWSリージョンを制限したい」。

```bash
npx cc-safe-setup --examples
```

350以上のサンプルhookが一覧で出る。カテゴリ別にフィルタもできる。気に入ったものを選んでインストールすれば、settings.jsonに自動追加される。

```bash
npx cc-safe-setup --install-example auto-approve-docker
```

## まとめ

hookの選び方は単純。「何を防ぎたいか」から逆引きする。

まずは`npx cc-safe-setup`で8つ入れる。10秒で終わる。足りないものがあれば`--examples`から追加する。

hookは「AIを信用しない」という話ではない。人間の開発者にもlinterやCI、pre-commitフックがある。道具を安全に使うためのガードレール。Claude Codeにも同じものが必要だ。

---

**第2章「Safety Guards」まで無料公開中。**
🛡 `npx cc-safe-setup` — 8つの安全フックを10秒でインストール
---
:::message
**Claude Codeの自律運用をもっと深く学びたい方へ**
トークン消費の突然の急増、ファイル消失、無限ループ——700+時間の自律稼働で起きた失敗と対策をまとめました:
[Claude Codeを本番品質にする——実践ガイド](https://zenn.dev/yurukusa/books/6076c23b1cb18b)（¥800・第3章まで無料）
:::
:::message
**トークン消費が気になる方へ**
[Token Checkup（無料）](https://yurukusa.github.io/cc-safe-setup/token-checkup.html)で30秒診断。どこでトークンが消えているか可視化できる。
もっと深く知りたい方は → [Claude Codeのトークン消費を半分にする（¥2,500）](https://zenn.dev/yurukusa/books/token-savings-guide)
:::
:::message
「AIに任せて大丈夫なのか」という不安を持ったまま800時間使い続けた記録は[非エンジニアがClaude Codeを800時間走らせた——失敗と学びの全記録](https://zenn.dev/yurukusa/books/3c3c3baee85f0a19)（¥800・第2章まで無料）に書いた。
:::
