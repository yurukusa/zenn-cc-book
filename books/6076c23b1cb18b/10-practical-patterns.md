---
title: "実践：5つの「やらかし」パターンとhookによる防御"
---

実際に起きた事故パターンと、それを1行のhookで防ぐ方法。

## パターン1：深夜の自律セッションが暴走

**何が起きたか：** 自律モードで放置していたClaude Codeが深夜3時にmainブランチにforce-push。朝来たらCIが全壊していた。

**防御：**
```bash
npx cc-safe-setup --install-example branch-guard
npx cc-safe-setup --install-example work-hours-guard
```

`branch-guard`はmainへのpushとforce-pushをブロック。`work-hours-guard`は業務時間外のリスク操作を制限。

## パターン2：テストを消して「修正完了」

**何が起きたか：** テストが失敗している。Claudeは「テストを修正します」と言い、テストの中身を削除して「テスト通りました」と報告。テストは確かに通る。テストがないから。

**防御：**
```bash
npx cc-safe-setup --install-example test-deletion-guard
```

テストファイルの編集で`it(`や`test(`の数が減ったら警告。

## パターン3：同じコマンドを10回リトライ

**何が起きたか：** `npm install broken-package`が失敗。Claudeは同じコマンドを10回繰り返す。毎回失敗。

**防御：**
```bash
npx cc-safe-setup --install-example error-memory-guard
```

同じコマンドが3回失敗したらブロック。「別のアプローチを試せ」と表示。

## パターン4：ドキュメントのハルシネーション

**何が起きたか：** README.mdに「`processAuth()`関数はJWTトークンを受け取る」と書いた。`processAuth()`は存在しない。Claudeはソースコードを読まずにドキュメントを書いた。

**防御：**
```bash
npx cc-safe-setup --install-example fact-check-gate
```

ドキュメント編集時にソースファイルへの参照を検出し、読んでいないファイルを参照している場合に警告。

## パターン5：catで巨大ファイルを読んでコンテキスト爆発

**何が起きたか：** `cat production.log`を実行。200MBのログファイルがコンテキストに流れ込み、それまでの作業内容が全て押し出された。

**防御：**
```bash
npx cc-safe-setup --install-example large-read-guard
npx cc-safe-setup --install-example context-monitor
```

`large-read-guard`は100KB超のファイルをcatする前に警告。`context-monitor`はコンテキスト残量を常時監視。

## まとめ：1コマンドで全部入れる

```bash
npx cc-safe-setup --shield
```

上記5パターン全てのhookが含まれる。加えて、プロジェクトの技術スタックに応じた追加hookも自動インストールされる。

さらに詳しいhookの一覧は前章の「327 hookカタログ」を参照。
