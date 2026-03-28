---
title: "付録：507+ hookカタログ"
---

cc-safe-setup v29.6.27は507個以上のhookを提供している。ここでは全カテゴリと代表的なhookを一覧する。

## インストール

```bash
# 基本（8個のコアhook）
npx cc-safe-setup

# 最大安全（スタック検出+全推奨hook）
npx cc-safe-setup --shield

# プロファイル選択
npx cc-safe-setup --profile strict    # 33個
npx cc-safe-setup --profile standard  # 20個
npx cc-safe-setup --profile minimal   # 8個

# チーム共有（プロジェクトにコミット）
npx cc-safe-setup --team

# 個別インストール
npx cc-safe-setup --install-example <hook名>

# 一覧表示
npx cc-safe-setup --examples
```

## カテゴリ別一覧

### 破壊的コマンド防止（12個）
`destructive-guard` / `branch-guard` / `no-sudo-guard` / `scope-guard` / `protect-dotfiles` / `symlink-guard` / `uncommitted-work-guard` / `no-install-global` / `protect-claudemd` / `case-sensitive-guard` / `worktree-guard` / `strict-allowlist`

### データ保護（5個）
`block-database-wipe` / `secret-guard` / `hardcoded-secret-detector` / `staged-secret-scan` / `bulk-file-delete-guard`

### Git安全（11個）
`git-config-guard` / `git-tag-guard` / `conflict-marker-guard` / `auto-stash-before-pull` / `branch-naming-convention` / `commit-quality-gate` / `commit-scope-guard` / `require-issue-ref` / `no-verify-blocker` / `git-index-lock-cleanup` / `push-requires-test-pass`

### 自動承認（11個 + PermissionRequest 7個）
`auto-approve-build` / `auto-approve-python` / `auto-approve-docker` / `auto-approve-go` / `auto-approve-cargo` / `auto-approve-make` / `auto-approve-gradle` / `auto-approve-maven` / `auto-approve-ssh` / `auto-approve-git-read` / `compound-command-approver`

**PermissionRequest**（保護ディレクトリや安全チェックの確認プロンプトを自動承認）:
`allow-git-hooks-dir` / `allow-claude-settings` / `allow-protected-dirs` / `auto-approve-compound-git` / `quoted-flag-approver` / `bash-heuristic-approver` / `edit-always-allow`

:::message
**PreToolUseではなくPermissionRequestを使う理由**: PreToolUseは内蔵の保護ディレクトリ検査の**前**に実行されるため、`permissionDecision: "allow"`を返しても上書きされる。PermissionRequestは**後**に実行されるため、プロンプトをキャッチして自動承認できる。
:::

### 品質（10個）
`syntax-check` / `diff-size-guard` / `test-deletion-guard` / `verify-before-done` / `read-before-edit` / `fact-check-gate` / `no-console-log` / `no-eval` / `no-wildcard-import` / `no-todo-ship`

### セキュリティ（10個）
`env-source-guard` / `env-inherit-guard` / `prompt-injection-guard` / `no-curl-upload` / `no-port-bind` / `network-guard` / `npm-publish-guard` / `mcp-server-guard` / `staged-secret-scan` / `credential-file-cat-guard`

### デプロイ（4個）
`deploy-guard` / `no-deploy-friday` / `work-hours-guard` / `changelog-reminder`

### 監視・コスト（13個）
`context-monitor` / `cost-tracker` / `token-budget-guard` / `output-length-guard` / `loop-detector` / `error-memory-guard` / `rate-limit-guard` / `resume-context-guard` / `output-explosion-detector` / `edit-error-counter` / `bash-timeout-guard` / `long-session-reminder` / `file-change-monitor`

### ユーティリティ（17個）
`comment-strip` / `cd-git-allow` / `api-error-alert` / `session-handoff` / `compact-reminder` / `revert-helper` / `tmp-cleanup` / `hook-debug-wrapper` / `notify-waiting` / `auto-checkpoint` / `context-snapshot` / `lockfile-guard` / `auto-answer-question` / `fish-shell-wrapper` / `plan-repo-sync` / `parallel-session-guard` / `edit-retry-loop-guard`

### 多言語サポート
- **Python**: `destructive_guard.py` / `secret_guard.py`
- **Go**: `destructive_guard.go`
- **TypeScript**: `destructive-guard.ts`

## Webツール

| ツール | URL |
|--------|-----|
| Cheat Sheet | https://yurukusa.github.io/cc-safe-setup/hooks-cheatsheet.html |
| Hook Builder | https://yurukusa.github.io/cc-safe-setup/builder.html |
| FAQ | https://yurukusa.github.io/cc-safe-setup/faq.html |
| Playground | https://yurukusa.github.io/cc-hook-registry/playground.html |
| Registry | https://yurukusa.github.io/cc-hook-registry/ |

## CLI全コマンド（56個）

基本: `--status` / `--verify` / `--dry-run` / `--uninstall` / `--examples` / `--install-example` / `--full`

分析: `--audit` / `--scan` / `--learn` / `--stats` / `--analyze` / `--benchmark` / `--dashboard` / `--watch` / `--suggest` / `--replay` / `--health` / `--why`

生成: `--create` / `--generate-ci` / `--report` / `--shield` / `--quickfix` / `--guard` / `--from-claudemd`

管理: `--export` / `--import` / `--diff` / `--compare` / `--lint` / `--doctor` / `--share` / `--issues` / `--migrate` / `--migrate-from` / `--profile` / `--team` / `--diff-hooks`
