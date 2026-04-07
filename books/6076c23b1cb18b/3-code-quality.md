---
title: "第3章 Code Quality——構文エラーを自動で防ぐ"
free: true
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

ここまでの3章で、Safety Guards（ファイル保護）とCode Quality（構文チェック）をカバーした。これだけでもスコアは42点分上がる。

次の第4章では、**トークンがどこで消えているかを可視化する**。

GitHub Issue [#40524](https://github.com/anthropics/claude-code/issues/40524)（220リアクション）で「Max Planの5時間分が15分で消えた」という報告が相次いでいる。原因の多くはプロンプトキャッシュの無効化で、キャッシュ読み取り率が89%→4%に暴落すると消費量が10〜20倍になる。第4章で、この暴走を検知するhookとトークン消費を可視化するロガーの実装を解説する。

---

## ここから先の章で何が変わるか

ここまでの3章で、最低限の安全はカバーした。`cc-health-check`を走らせれば、スコアは19点→42点前後になっているはずだ。

だが85点に到達するには、あと4つの領域が要る。

| 領域 | 何が解決するか | 章 |
|------|--------------|-----|
| **Monitoring** | 「いつの間にかトークンが消えていた」を可視化する | 第4章 |
| **Recovery** | ファイルが消えても5分で復旧する仕組みを作る | 第5章 |
| **Autonomy** | 「寝ている間に30ファイル変わっていた」を安全に制御する | 第6章 |
| **Coordination** | 複数エージェントがルールを守って協調する | 第7章 |

第10章には、700時間の運用で実際に遭遇した「やらかし」10パターンをまとめた。キャッシュ破壊で$8消えた話、サブエージェントがCLAUDE.mdを無視した話、git pushが本番に飛んだ話――全て実体験だ。

第1章で`cc-health-check`を実行した人は、ここでもう一度スコアを確認してみてほしい。42点を超えていれば、ここまでの3章でカバーした安全策が機能している。超えていなければ、第4章以降で扱うMonitoring・Recovery・Autonomyが足りていないということだ。

19点→85点の全工程は第8章にまとまっている。どこから読んでもいい。
