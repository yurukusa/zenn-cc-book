---
title: "付録：634+ hookカタログ"
---

cc-safe-setupは634個以上のhookを提供している。ここでは全カテゴリと代表的なhookを一覧する。

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

### データ保護（7個）
`block-database-wipe` / `secret-guard` / `hardcoded-secret-detector` / `staged-secret-scan` / `bulk-file-delete-guard` / `secret-file-read-guard` / `bash-secret-output-detector`

:::message
**シークレット流出防止（#39882）**: `secret-file-read-guard`はRead/Grepツールが`.env`、credentials、秘密鍵等を読むのをブロック。`bash-secret-output-detector`はPostToolUseでBash出力にAWSキー、JWT、接続文字列等を検出しsystemMessage警告。
:::

### Git安全（13個）
`git-config-guard` / `git-tag-guard` / `conflict-marker-guard` / `auto-stash-before-pull` / `branch-naming-convention` / `commit-quality-gate` / `commit-scope-guard` / `require-issue-ref` / `no-verify-blocker` / `git-index-lock-cleanup` / `push-requires-test-pass` / `git-checkout-safety-guard` / `git-operations-require-approval`

:::message
**CLAUDE.mdのgitルール強制（#40695）**: `git-operations-require-approval`はCLAUDE.mdの「git commit/pushは許可なく実行するな」を物理的に強制。CLAUDE.mdは助言、hookは法律。
:::

### 自動承認（11個 + PermissionRequest 7個）
`auto-approve-build` / `auto-approve-python` / `auto-approve-docker` / `auto-approve-go` / `auto-approve-cargo` / `auto-approve-make` / `auto-approve-gradle` / `auto-approve-maven` / `auto-approve-ssh` / `auto-approve-git-read` / `compound-command-approver`

**PermissionRequest**（保護ディレクトリや安全チェックの確認プロンプトを自動承認）:
`allow-git-hooks-dir` / `allow-claude-settings` / `allow-protected-dirs` / `auto-approve-compound-git` / `quoted-flag-approver` / `bash-heuristic-approver` / `edit-always-allow`

:::message
**PreToolUseではなくPermissionRequestを使う理由**: PreToolUseは内蔵の保護ディレクトリ検査の**前**に実行されるため、`permissionDecision: "allow"`を返しても上書きされる。PermissionRequestは**後**に実行されるため、プロンプトをキャッチして自動承認できる。
:::

### 品質（12個）
`syntax-check` / `diff-size-guard` / `test-deletion-guard` / `verify-before-done` / `read-before-edit` / `fact-check-gate` / `no-console-log` / `no-eval` / `no-wildcard-import` / `no-todo-ship` / `no-output-truncation` / `prefer-dedicated-tools`

:::message
**出力截断防止（#39945）**: `no-output-truncation`はビルド/テスト出力を`| tail`でパイプしてエラーを捨てるパターンをブロック。`prefer-dedicated-tools`（#39979）はBashのcat/grep/head/tailを検出し、Read/Grep/Globツールの使用を強制。トークン節約と構造化出力を確保。
:::

### セキュリティ（22個）
`env-source-guard` / `env-inherit-guard` / `prompt-injection-guard` / `no-curl-upload` / `no-port-bind` / `network-guard` / `npm-publish-guard` / `mcp-server-guard` / `staged-secret-scan` / `credential-file-cat-guard` / `compound-inject-guard` / `path-deny-bash-guard` / `sandbox-write-verify` / `context-warning-verifier` / `mcp-config-freeze` / `mcp-data-boundary` / `claudeignore-enforce-guard` / `bash-safety-auto-deny` / `env-inline-secret-guard` / `migration-verify-guard` / `skill-injection-detector` / `polyglot-rm-guard`

:::message
**多言語ファイル削除防止（#39459）**: `polyglot-rm-guard`はBash rmがdenyされた場合にPython `os.remove()`、Node `fs.unlinkSync()`、Ruby `File.delete()`、Perl `unlink`へのバイパスを検出・ブロック。目標（ファイル削除）を複数言語にまたがって防止。
:::

:::message
**OWASP MCP Top 10対応**: `mcp-config-freeze`はMCP09（過剰パーミッション）、`mcp-data-boundary`はMCP01（プロンプトインジェクション）+MCP10（不十分なログ）に対応。
:::

### デプロイ（4個）
`deploy-guard` / `no-deploy-friday` / `work-hours-guard` / `changelog-reminder`

### セッション管理（10個）
`session-resume-guard` / `compaction-transcript-guard` / `plan-mode-strict-guard` / `context-threshold-alert` / `permission-mode-drift-guard` / `cross-session-error-log` / `pre-compact-checkpoint` / `concurrent-edit-lock` / `session-end-logger` / `headless-stop-guard`

:::message
**コンテキスト窓管理**: `context-threshold-alert`はコンテキスト使用率が閾値を超えると警告。`compaction-transcript-guard`はcompaction時の重要情報ロスを防止。`session-end-logger`はSessionEnd hookでgit活動をログに記録（#40010）。`headless-stop-guard`は`-p`モードでStop hookが空結果を返すバグを回避（#38651）。
:::

### サブエージェント制御（5個）
`subagent-scope-validator` / `subagent-scope-guard` / `subagent-budget-guard` / `max-concurrent-agents` / `subagent-tool-call-limiter`

### 監視・コスト（17個）
`context-monitor` / `cost-tracker` / `token-budget-guard` / `output-length-guard` / `loop-detector` / `error-memory-guard` / `rate-limit-guard` / `resume-context-guard` / `output-explosion-detector` / `edit-error-counter` / `bash-timeout-guard` / `long-session-reminder` / `file-change-monitor` / `dotenv-watch` / `consecutive-failure-circuit-breaker` / `bg-task-cooldown-guard` / `session-health-monitor`

### ファイル保護（3個）
`file-recycle-bin` / `worktree-memory-guard` / `tmp-output-size-guard`

### プロセス管理（1個）
`plugin-process-cleanup`

:::message
**リソースリーク防止**: `plugin-process-cleanup`はSessionEndでリークしたプラグインプロセス（bun server.ts等）を自動クリーンアップ（#39137）。`tmp-output-size-guard`はSessionStartで肥大化したtask出力ファイル（95GB+事例）を検出（#39909）。
:::

### 自動承認（追加: 2個）
`heredoc-backtick-approver` / `hook-stdout-sanitizer`

:::message
`heredoc-backtick-approver`はheredocのバッククォートによる誤検知を解消（[#35183](https://github.com/anthropics/claude-code/issues/35183)起点）。
:::

### CI/CDパイプライン保護（5個）
`github-actions-secret-guard` / `ci-workflow-guard` / `gitops-drift-guard` / `dotenv-commit-guard` / `npm-supply-chain-guard`

:::message
**CI/CDセキュリティ**: `github-actions-secret-guard`はワークフロー内のハードコードされたシークレットを検出。`ci-workflow-guard`は`--no-verify`やremote script executionの追加を警告。`npm-supply-chain-guard`はnpm/yarn/pnpm installで人気パッケージのタイポスクワッティングを検出（#39421）。
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

### Bash ネットワーク制御（1個）
`bash-domain-allowlist`

:::message
**curl/wgetドメイン制限**: sandboxの`allowedDomains`がHTTPリクエストをフィルタしない問題（#40213）のワークアラウンド。`CC_ALLOWED_DOMAINS`環境変数で許可ドメインを設定。curl/wgetの宛先を検査し、リスト外をブロック。
:::

### 環境チェック（1個）
`bashrc-safety-check`

:::message
**SessionStart環境検査**: エージェントがspawnするbashシェルが`.bashrc`を読み込んでハングする問題（#40354）を検出。Angular CLI completion、nvm、conda等の非インタラクティブシェルで停止するパターンをスキャンし、ガード行追加を案内。
:::

### ファイル保護（5個 — NEW）
`write-shrink-guard` / `symlink-protect` / `file-recycle-bin` / `settings-auto-backup` / `export-overwrite-guard`

:::details write-shrink-guard
**Write時のファイルサイズ急縮小をブロック**: 31,699行のファイルが16行に切り詰められる事故（#40807）を防止。Write操作で元ファイルの10%未満のサイズになる場合にブロック。Editツールへの切り替えを推奨する。
:::

:::details symlink-protect
**symlink→通常ファイル置換の防止**: Worktreeの`symlinkDirectories`で作られたsymlinkがWrite操作で通常ファイルに置換される問題（#40857）を防止。writeをsymlink先のターゲットファイルにリダイレクトする。
:::

### セッション管理（3個 — NEW）
`session-duration-guard` / `worktree-delete-guard` / `session-handoff`

:::details session-duration-guard
**長時間セッションの品質劣化を警告**: 2時間で注意、4時間でクリティカル警告。700+時間の自律稼働経験に基づく。閾値は環境変数で調整可能。
:::

:::details worktree-delete-guard
**別セッションが使用中のworktree削除をブロック**: `git worktree remove`と`git worktree prune`をブロック（#40850）。マルチセッション環境での安全性を確保。
:::

:::details clear-command-confirm-guard
**/clearコマンドの誤実行を防止**: `/clear`を完全ブロック。`/compact`への誘導メッセージを表示（#40931）。プレフィックスマッチで`/c`+Enterが`/clear`に化ける事故を防ぐ。
:::

:::details claudemd-violation-detector
**CLAUDE.md絶対ルールのリマインド**: 20回のツール使用ごとにCLAUDE.mdからABSOLUTE/MUST NEVER/禁止マーカーを抽出し表示（#40930）。長時間セッションでの指示忘れを防ぐ。
:::

:::details subagent-context-size-guard
**薄いサブエージェントプロンプトを警告**: Agentツールのプロンプトが100文字未満の場合に警告（#40929）。サブエージェントは親のコンテキストを持たないため、十分な背景情報が必要。
:::

:::details edit-old-string-validator
**Edit並列呼び出しのカスケード失敗を防止**: old_stringがファイルに存在するか事前検証（#22264）。1件の失敗が同バッチの全Editをキャンセルする問題を元から断つ。
:::

:::details virtual-cwd-helper
**セッション中のディレクトリ切り替えを疑似実現**: `~/.claude/virtual-cwd`ファイルでCWDを管理（#3473）。コマンド実行時に仮想CWDへのcd忘れを警告。
:::

:::details cwd-drift-detector
**ディレクトリ迷子での破壊的コマンドを警告**: プロジェクトルート外でgit reset/rm -rf等を検知すると警告（#1669）。72リアクションの頻出バグに対応。
:::

:::details permission-pattern-auto-allow
**正規表現パターンでコマンドを自動許可**: 設定のexact matchの制限を回避（#819）。`~/.claude/allowed-patterns.txt`にパターンを記述。npm/git/ls等の日常コマンドを一括許可。
:::

### セッション管理・監視（5個 — v29.6.36追加）

:::details settings-mutation-detector
**settings.jsonの意図しない変更を検知**: PostToolUse hookでsettings.jsonのハッシュを監視。hookの無効化やパーミッション変更を即座に警告。
:::

:::details token-spike-alert
**トークン使用量の急増を警告**: API呼び出しごとのトークン消費を監視し、閾値を超えたら警告。コスト爆発の早期検知。
:::

:::details worktree-path-validator
**worktreeでのパス誤りを防止**: Edit/Read/Writeがmainワークスペースを指している場合にブロック（#36182）。Exploreエージェントの頻出バグに対応。
:::

:::details git-crypt-worktree-guard
**git-cryptリポジトリでのworktree作成を警告**: git-cryptの暗号化がworktreeで機能しないため、機密ファイル漏洩を防止。
:::

:::details temp-file-cleanup-stop
**セッション終了時にtmpclaude-*ファイルを自動削除**: Stopフックでワークスペースに残るテンポラリファイルを清掃（#17636）。
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
