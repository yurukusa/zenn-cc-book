---
title: "第2章 Safety Guards——最初に直すべき4つのチェック"
free: false
---

# 第2章 Safety Guards——最初に直すべき4つのチェック

Safety Guardsが低いと、何をしても危ない。ここを最初に直す。

## チェック1: 危険なコマンドをブロックする

**失敗例**: CCが「古いファイルを削除してクリーンな状態にします」と言って `rm -rf ./src/` を実行した。

**対策**: `hooks/branch-guard.sh` を PreToolUse に追加する。

```json
// ~/.claude/settings.json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/branch-guard.sh"
          }
        ]
      }
    ]
  }
}
```

**[branch-guard.sh を入手](https://github.com/yurukusa/claude-code-hooks/blob/main/hooks/branch-guard.sh)**

このhookは以下をブロックする:
- `rm -rf` （再帰的削除）
- `git reset --hard`
- `git push --force` / `git push origin main`
- `git clean -fd`

`exit 2` を返すことでCCにツール実行をキャンセルさせる。

## チェック2: APIキーをCLAUDE.mdに書かない

**失敗例**: CLAUDE.mdに `API_KEY=sk-xxxx` を書いたら、スクリーンショットで流出しかけた。

**対策**: `~/.credentials` か `~/.env` に保存する。

```bash
# ~/.credentials
OPENAI_API_KEY=sk-xxxx
GITHUB_TOKEN=ghp_xxxx
```

CLAUDE.mdには「認証情報は ~/.credentials を参照」とだけ書く。

**[CLAUDE-autonomous.md テンプレートを参照](https://github.com/yurukusa/claude-code-hooks/blob/main/templates/CLAUDE-autonomous.md)**

## チェック3: main/masterへの直pushを防ぐ

**失敗例**: CCが「変更を反映します」と言ってmainに直push。翌日まで気づかなかった。

**対策**: branch-guard.shが既にこれもカバーしている。

加えてCLAUDE.mdに明示的に書く:

```markdown
## Git Rules
- 新しいブランチで作業する: `git checkout -b feature/task-name`
- mainへの直pushは禁止
- PRを作ってからmerge
```

## チェック4: エラーがある状態で外部APIを叩かない

**失敗例**: 構文エラーがあるコードのままnpm publishを実行。壊れたパッケージが公開された。

**対策**: `hooks/error-gate.sh` を PreToolUse に追加する。

**[error-gate.sh を入手](https://github.com/yurukusa/claude-code-hooks/blob/main/hooks/error-gate.sh)**

このhookは `~/.claude/error-log.jsonl` にUnresolvedエラーが存在する場合、以下をブロック:
- `git push`
- `npm publish`
- `curl -X POST/PUT/DELETE`
- `gh pr create`

## Safety Guards を直した後の状態

これら4つを設定すると、Safety Guardsが75〜100%になる。

```
  ▸ Safety Guards
    [PASS] PreToolUse hook blocks dangerous commands
    [PASS] API keys stored in dedicated files
    [PASS] Setup prevents pushing to main/master
    [PASS] Error-aware gate blocks external calls when errors exist
```

「直すのが面倒だな」と思ったかもしれない。でも **1回の事故の復旧コストは、1時間のhook設定コストより高い**。

実は10秒で終わる方法がある:

```bash
npx cc-safe-setup
```

8つの安全フック（destructive-guard, branch-guard, secret-guard, syntax-check, context-monitor, comment-strip, cd-git-allow, api-error-alert）が自動でインストールされる。上の4チェックのうち3つが即座にPASSになる。

インストール後、フックが正しく動作しているか確認するには:

```bash
npx cc-safe-setup --verify
```

各フックにテスト入力を送り、ブロック/許可が期待通りか検証する。CIパイプラインにも組み込める（失敗時exit 1）。

### プロジェクト固有のガードを追加する

8つの基本フックに加えて、プロジェクトに特化したガードが必要なことがある。cc-safe-setupには358個のexample hookが同梱されている:

```bash
# 一覧を表示
npx cc-safe-setup --examples

# 特定のexampleをワンコマンドでインストール
npx cc-safe-setup --install-example block-database-wipe
```

よく使われるexample:
- `block-database-wipe` — Laravel/Django/Rails/Prismaの破壊的DBコマンドをブロック
- `protect-dotfiles` — ~/.bashrc, ~/.aws/, ~/.ssh/の変更をブロック
- `scope-guard` — プロジェクト外のファイル操作をブロック
- `auto-checkpoint` — 編集後に自動コミット（context compaction対策）
- `git-config-guard` — git config --globalの無断変更をブロック

---

次章: Code Quality——構文エラーを自動で防ぐ
