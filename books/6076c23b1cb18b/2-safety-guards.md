---
title: "第2章 Safety Guards——最初に直すべき4つのチェック"
free: true
---

# 第2章 Safety Guards——最初に直すべき4つのチェック

Safety Guardsが低いと、何をしても危ない。ここを最初に直す。

## チェック1: 危険なコマンドをブロックする

**失敗例**: CCが「古いファイルを削除してクリーンな状態にします」と言って `rm -rf ./src/` を実行した。

**対策**: 10秒で解決する方法がある。

```bash
npx cc-safe-setup
```

これだけで`destructive-guard`（rm -rf, git reset --hard をブロック）と`branch-guard`（mainへのpush防止）が自動インストールされる。

手動で設定したい場合はsettings.jsonに直接書く:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/destructive-guard.sh"
          }
        ]
      }
    ]
  }
}
```

destructive-guardは以下をブロックする:
- `rm -rf /` `rm -rf ~/` `rm -rf ../`（再帰的削除）
- `git reset --hard`
- `git clean -fd`
- `chmod -R 777 /`

`exit 2` を返すことでClaude Codeにツール実行をキャンセルさせる。モデルはこれをバイパスできない。

## チェック2: APIキーをCLAUDE.mdに書かない

**失敗例**: CLAUDE.mdに `API_KEY=sk-xxxx` を書いたら、スクリーンショットで流出しかけた。

**対策**: `~/.credentials` か `~/.env` に保存する。

```bash
# ~/.credentials
OPENAI_API_KEY=sk-xxxx
GITHUB_TOKEN=ghp_xxxx
```

CLAUDE.mdには「認証情報は ~/.credentials を参照」とだけ書く。cc-safe-setupの`secret-guard`フックが`.env`や鍵ファイルのgit add/commitを自動ブロックする。

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

**対策**: cc-safe-setupの`syntax-check`フックが編集後の構文エラーを自動検知する。さらに`npm-publish-guard`や`no-push-without-tests`で、テスト未実行のままpush/publishすることをブロックできる。

```bash
npx cc-safe-setup --install-example npm-publish-guard
npx cc-safe-setup --install-example no-push-without-tests
```

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

8つの基本フックに加えて、プロジェクトに特化したガードが必要なことがある。cc-safe-setupには592個のexample hookが同梱されている:

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

**次章以降のハイライト:**
- **第3章**: 構文エラーを自動検出し、壊れたコードがコミットされる前にブロックする
- **第5章**: セッション劣化の検知と自動リカバリ——長時間運用の生命線
- **第9章**: 592+ hookの全カタログ——カテゴリ別に解説付き
- **第12章**: 9,014テストケースから学んだhookのテスト戦略

次章: Code Quality——構文エラーを自動で防ぐ
