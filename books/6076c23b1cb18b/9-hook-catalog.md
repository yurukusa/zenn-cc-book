---
title: "付録：315 hookカタログ"
---

cc-safe-setup v28.3.1は315個のhookを提供している。ここでは全カテゴリと代表的なhookを一覧する。

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

### データ保護（3個）
`block-database-wipe` / `secret-guard` / `hardcoded-secret-detector`

### Git安全（8個）
`git-config-guard` / `git-tag-guard` / `conflict-marker-guard` / `auto-stash-before-pull` / `branch-naming-convention` / `commit-quality-gate` / `commit-scope-guard` / `require-issue-ref`

### 自動承認（11個）
`auto-approve-build` / `auto-approve-python` / `auto-approve-docker` / `auto-approve-go` / `auto-approve-cargo` / `auto-approve-make` / `auto-approve-gradle` / `auto-approve-maven` / `auto-approve-ssh` / `auto-approve-git-read` / `compound-command-approver`

### 品質（10個）
`syntax-check` / `diff-size-guard` / `test-deletion-guard` / `verify-before-done` / `read-before-edit` / `fact-check-gate` / `no-console-log` / `no-eval` / `no-wildcard-import` / `no-todo-ship`

### セキュリティ（6個）
`env-source-guard` / `prompt-injection-guard` / `no-curl-upload` / `no-port-bind` / `network-guard` / `npm-publish-guard`

### デプロイ（4個）
`deploy-guard` / `no-deploy-friday` / `work-hours-guard` / `changelog-reminder`

### 監視・コスト（7個）
`context-monitor` / `cost-tracker` / `token-budget-guard` / `output-length-guard` / `loop-detector` / `error-memory-guard` / `rate-limit-guard`

### ユーティリティ（12個）
`comment-strip` / `cd-git-allow` / `api-error-alert` / `session-handoff` / `compact-reminder` / `revert-helper` / `tmp-cleanup` / `hook-debug-wrapper` / `notify-waiting` / `auto-checkpoint` / `context-snapshot` / `lockfile-guard`

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

### セキュリティ（4個）
`prompt-injection-guard` / `hardcoded-secret-detector` / `typosquat-guard` / `stale-env-guard`

### デプロイ（4個）
`deploy-guard` / `no-deploy-friday` / `work-hours-guard` / `changelog-reminder`

## CLI全コマンド（40個）

基本: `--status` / `--verify` / `--dry-run` / `--uninstall` / `--examples` / `--install-example` / `--full`

分析: `--audit` / `--scan` / `--learn` / `--stats` / `--analyze` / `--benchmark` / `--dashboard` / `--watch` / `--suggest` / `--replay` / `--health` / `--why`

生成: `--create` / `--generate-ci` / `--report` / `--shield` / `--quickfix` / `--guard` / `--from-claudemd`

管理: `--export` / `--import` / `--diff` / `--compare` / `--lint` / `--doctor` / `--share` / `--issues` / `--migrate` / `--migrate-from` / `--profile` / `--team` / `--diff-hooks`
