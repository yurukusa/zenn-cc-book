---
title: "第3章 Code Quality——構文エラーを自動で防ぐ"
free: false
---

# 第3章 Code Quality——構文エラーを自動で防ぐ

Safety Guardsでファイルを守ったら、次はコードの質を守る。

## チェック5: ファイル編集後に構文チェックを走らせる

**失敗例**: Pythonファイルを編集した直後にテストを実行。Syntax Errorで失敗。CCは「エラーを直します」と言ってまた構文エラーを作った。このループが10回続いた。

**根本原因**: PostToolUseで構文チェックが走っていなかった。

**対策**: `hooks/syntax-check.sh` を PostToolUse に追加する。

```json
// ~/.claude/settings.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/syntax-check.sh"
          }
        ]
      }
    ]
  }
}
```

**入手方法**: `npx cc-safe-setup`（[cc-safe-setup](https://github.com/yurukusa/cc-safe-setup)）

対応言語:
- Python: `python3 -m py_compile`
- Shell: `bash -n`
- JSON: `python3 -m json.tool`
- YAML: `python3 -c "import yaml; yaml.safe_load(...)"`
- JavaScript/TypeScript: `node --check`

このhookを入れると、CCは構文エラーを即座に検知して修正できる。エラーループが激減した。

## チェック6: bashの出力からエラーを自動検知する

**失敗例**: `npm install` がエラーで失敗しているのに、CCはその後の作業を続けた。エラーを見落としていた。

**対策**: `hooks/activity-logger.sh` が bash 出力を記録して、エラーパターンを検知する。

**入手方法**: `npx cc-safe-setup --install-example activity-logger`（[cc-safe-setup](https://github.com/yurukusa/cc-safe-setup)）

activity-loggerは全ての bash 実行を JSONL ログに記録する:

```json
{"ts":"2026-02-28T12:00:00Z","tool":"Bash","file":"npm install","exit_code":1,"error_pattern":"ENOENT"}
```

エラーが記録されると、error-memory-guard.shが次の外部アクションをブロックする。「エラーを記録する仕組み」と「エラーがある時にブロックする仕組み」がセットで機能する。

## チェック7: 完了の定義（DoD）を持つ

**失敗例**: CCが「記事を投稿しました！」と報告。確認したら投稿はされていたが、誤字がある・タグが間違っている・URLが死んでいる状態だった。

**根本原因**: 「投稿ボタンを押す = 完了」という定義で動いていた。

**対策**: `templates/dod-checklists.md` を `~/.claude/` に配置する。

テンプレートは[cc-safe-setup](https://github.com/yurukusa/cc-safe-setup)を参照

DoDチェックリストの例:

```markdown
## 記事投稿のDoD
- [ ] 投稿後にURLが開けることを確認した
- [ ] タイトルとタグが正しいことを確認した
- [ ] スクリーンショットで公開状態を目視確認した
- [ ] content-manifest.yamlに記録した
```

CLAUDE.mdにも明記する:

```markdown
## タスク完了時
~/.claude/dod-checklists.md のDoDを満たすまで「完了」と報告するな。
```

## チェック8: AIが自分のアウトプットを検証する

**失敗例**: 「GitHubにpushしました」と報告したが、実際にはpushに失敗していた。CCは確認していなかった。

**対策**: CLAUDE.mdにVerification stepを明記する。

```markdown
## 検証ルール
- git push後: `git log --oneline -1 origin/main` で確認
- 記事投稿後: URLにアクセスして公開状態を目視確認
- ファイル作成後: `ls -la` で存在確認
- APIレスポンス後: レスポンスのHTTPステータスを確認
```

このパターンはCLAUDE.mdの自律運用テンプレートに含まれている（[cc-safe-setup](https://github.com/yurukusa/cc-safe-setup)参照）。

---

## Code Quality を修正した後

```
  ▸ Code Quality
    [PASS] Syntax checks run after every file edit
    [PASS] Error detection and tracking from command output
    [PASS] Definition of Done checklist exists
    [PASS] AI verifies its own output
    100%
```

ここまでで8チェックをカバーした（42点分）。

構文チェックは `npx cc-safe-setup` で自動設定される。PostToolUseフックとしてsyntax-check.shがインストールされ、Edit/Write後にPython/Shell/JSON/YAML/JSの構文を自動検証する。

---

次章: Monitoring——AIが「何をしているか」を知る
