---
title: "付録：592+ hookカタログ"
---

cc-safe-setup v29.6.32は592個以上のhookを提供している。ここでは全カテゴリと代表的なhookを一覧する。

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

### 破壊的コマンド防止（13個）
`destructive-guard` / `branch-guard` / `no-sudo-guard` / `scope-guard` / `protect-dotfiles` / `symlink-guard` / `uncommitted-work-guard` / `no-install-global` / `protect-claudemd` / `case-sensitive-guard` / `worktree-guard` / `strict-allowlist` / `shell-wrapper-guard`

### データ保護（5個）
`block-database-wipe` / `secret-guard` / `hardcoded-secret-detector` / `staged-secret-scan` / `bulk-file-delete-guard`

### Git安全（12個）
`git-config-guard` / `git-tag-guard` / `conflict-marker-guard` / `auto-stash-before-pull` / `branch-naming-convention` / `commit-quality-gate` / `commit-scope-guard` / `require-issue-ref` / `no-verify-blocker` / `git-index-lock-cleanup` / `push-requires-test-pass` / `git-checkout-safety-guard`

### 自動承認（11個 + PermissionRequest 7個）
`auto-approve-build` / `auto-approve-python` / `auto-approve-docker` / `auto-approve-go` / `auto-approve-cargo` / `auto-approve-make` / `auto-approve-gradle` / `auto-approve-maven` / `auto-approve-ssh` / `auto-approve-git-read` / `compound-command-approver`

**PermissionRequest**（保護ディレクトリや安全チェックの確認プロンプトを自動承認）:
`allow-git-hooks-dir` / `allow-claude-settings` / `allow-protected-dirs` / `auto-approve-compound-git` / `quoted-flag-approver` / `bash-heuristic-approver` / `edit-always-allow`

:::message
**PreToolUseではなくPermissionRequestを使う理由**: PreToolUseは内蔵の保護ディレクトリ検査の**前**に実行されるため、`permissionDecision: "allow"`を返しても上書きされる。PermissionRequestは**後**に実行されるため、プロンプトをキャッチして自動承認できる。
:::

### 品質（10個）
`syntax-check` / `diff-size-guard` / `test-deletion-guard` / `verify-before-done` / `read-before-edit` / `fact-check-gate` / `no-console-log` / `no-eval` / `no-wildcard-import` / `no-todo-ship`

### セキュリティ（21個）
`env-source-guard` / `env-inherit-guard` / `prompt-injection-guard` / `no-curl-upload` / `no-port-bind` / `network-guard` / `npm-publish-guard` / `mcp-server-guard` / `staged-secret-scan` / `credential-file-cat-guard` / `compound-inject-guard` / `path-deny-bash-guard` / `sandbox-write-verify` / `context-warning-verifier` / `mcp-config-freeze` / `mcp-data-boundary` / `claudeignore-enforce-guard` / `bash-safety-auto-deny` / `env-inline-secret-guard` / `migration-verify-guard` / `skill-injection-detector`

:::message
**OWASP MCP Top 10対応**: `mcp-config-freeze`はMCP09（過剰パーミッション）、`mcp-data-boundary`はMCP01（プロンプトインジェクション）+MCP10（不十分なログ）に対応。
:::

### デプロイ（4個）
`deploy-guard` / `no-deploy-friday` / `work-hours-guard` / `changelog-reminder`

### セッション管理（8個）
`session-resume-guard` / `compaction-transcript-guard` / `plan-mode-strict-guard` / `context-threshold-alert` / `permission-mode-drift-guard` / `cross-session-error-log` / `pre-compact-checkpoint` / `concurrent-edit-lock`

:::message
**コンテキスト窓管理**: `context-threshold-alert`はコンテキスト使用率が閾値を超えると警告。`compaction-transcript-guard`はcompaction時の重要情報ロスを防止。Anthropic公式ベストプラクティスでも「コンテキスト窓は最重要リソース」と明言されている。
:::

### サブエージェント制御（5個）
`subagent-scope-validator` / `subagent-scope-guard` / `subagent-budget-guard` / `max-concurrent-agents` / `subagent-tool-call-limiter`

### 監視・コスト（16個）
`context-monitor` / `cost-tracker` / `token-budget-guard` / `output-length-guard` / `loop-detector` / `error-memory-guard` / `rate-limit-guard` / `resume-context-guard` / `output-explosion-detector` / `edit-error-counter` / `bash-timeout-guard` / `long-session-reminder` / `file-change-monitor` / `dotenv-watch` / `consecutive-failure-circuit-breaker` / `bg-task-cooldown-guard`

### ファイル保護（2個）
`file-recycle-bin` / `worktree-memory-guard`

### 自動承認（追加: 2個）
`heredoc-backtick-approver` / `hook-stdout-sanitizer`

:::message
`heredoc-backtick-approver`はheredocのバッククォートによる誤検知を解消（[#35183](https://github.com/anthropics/claude-code/issues/35183)起点）。
:::

### CI/CDパイプライン保護（4個）
`github-actions-secret-guard` / `ci-workflow-guard` / `gitops-drift-guard` / `dotenv-commit-guard`

:::message
**CI/CDセキュリティ**: `github-actions-secret-guard`はワークフロー内のハードコードされたシークレットを検出。`ci-workflow-guard`は`--no-verify`やremote script executionの追加を警告。GitOps環境では`gitops-drift-guard`がmain上のinfra file直接編集を防止。
:::

### クラウド/インフラ（3個）
`k8s-production-guard` / `schema-migration-guard` / `network-exfil-guard`

:::message
**本番環境保護**: `k8s-production-guard`はproduction namespace上の`kubectl delete`/`scale --replicas=0`をブロック。`schema-migration-guard`はDROP TABLE/TRUNCATEを含むマイグレーションを警告。
:::

### MCP安全性（2個）
`mcp-server-allowlist` / `mcp-tool-audit-log`

:::message
**OWASP MCP Top 10対応**: `mcp-server-allowlist`は未許可MCPサーバーのツール呼び出しをブロック。`mcp-tool-audit-log`は全MCPツール呼び出しをログに記録（MCP09: Insufficient Logging対策）。
:::

### ロールベース制御（1個）
`role-tool-guard`

:::message
**エージェントチーム向け**: PMはEdit/Write/Bash不可、ArchitectはBash不可、Reviewerは読み取りのみ、Developerは全許可。`.claude/current-role.txt`でロールを切り替え。
:::

### セッションリカバリ（3個）
`session-resume-env-fix` / `pre-compact-knowledge-save` / `headless-empty-result-guard`

### プロセス強制（4個）
`spec-file-scope-guard` / `read-all-files-enforcer` / `self-modify-bypass-guard` / `permission-entry-validator`

### プラットフォーム互換（2個）
`windows-path-guard` / `no-git-amend`

### ファイル安全（4個）
`file-age-guard` / `binary-upload-guard` / `cwd-project-boundary-guard` / `file-change-undo-tracker`

### コンテキスト管理（1個）
`compact-blocker`

:::message
**auto-compaction完全制御**: PreCompactフックでexit 2を返し、自動圧縮を無効化。手動でコンテキストを管理するパワーユーザー向け。条件付きブロック（フラグファイルで切り替え）も可能。
:::

### コマンド修正（1個）
`git-show-flag-sanitizer`

:::message
**自動バグ修正**: Claude Codeが`git show --no-stat`（無効フラグ）を実行する既知バグをPreToolUseフックで自動修正。コンテキスト浪費を防止。
:::

### WebFetch制御（1個）
`webfetch-domain-allow`

:::message
**ドメインallowlist**: sandbox modeで`WebFetch(domain:*)`が動作しない問題のワークアラウンド。環境変数`CC_WEBFETCH_ALLOW_DOMAINS`でドメインリストを設定可能。
:::

### ユーティリティ（20個）
`comment-strip` / `cd-git-allow` / `api-error-alert` / `session-handoff` / `compact-reminder` / `revert-helper` / `tmp-cleanup` / `hook-debug-wrapper` / `notify-waiting` / `auto-checkpoint` / `context-snapshot` / `lockfile-guard` / `auto-answer-question` / `fish-shell-wrapper` / `plan-repo-sync` / `parallel-session-guard` / `edit-retry-loop-guard` / `plan-mode-enforcer` / `direnv-auto-reload` / `api-retry-limiter`

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
