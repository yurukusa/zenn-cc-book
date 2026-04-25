---
title: "第9章 設定テンプレート集——コピペで使えるCLAUDE.md・hooks・settings.json"
---

この章は、各章で解説した内容を「そのまま使える」形にまとめたテンプレート集だ。コピペして、自分の環境に合わせて調整してほしい。

## テンプレート1: トークン効率重視のCLAUDE.md

```markdown
# プロジェクト指示書

## 基本方針
- ファイル読み込みはlimitパラメータで必要な部分だけ読む
- 1セッション = 1テーマ。テーマが変わったら新しいセッション
- hookがブロックした操作はリトライしない

## コーディング規約
| 状況 | 判断 |
|------|------|
| 技術選択 | 最もポピュラーなもの |
| 不明確な要件 | 最もシンプルな解釈 |
| セキュリティ | 常に安全な方 |

## コミット
✓ fix: prevent duplicate API calls on rapid button clicks
✗ fix bug

## 禁止
hookがブロックする操作（rm -rf, git reset --hard等）はリトライしない。
```

**ポイント**: 約30行。第2章の5パターン（許可リスト、具体例、理由1行、テーブル、hook分離）をすべて適用。

## テンプレート2: トークン節約hookセット（settings.json）

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read",
        "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/large-read-guard.sh", "timeout": 5000}]
      },
      {
        "matcher": "Read",
        "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/read-budget-guard.sh", "timeout": 5000}]
      },
      {
        "matcher": "Agent",
        "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/subagent-budget-guard.sh", "timeout": 5000}]
      },
      {
        "matcher": "Read",
        "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/duplicate-read-detector.sh", "timeout": 5000}]
      },
      {
        "matcher": "Bash",
        "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/bash-output-guard.sh", "timeout": 5000}]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "",
        "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/token-budget-guard.sh", "timeout": 5000}]
      },
      {
        "matcher": "",
        "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/token-spike-alert.sh", "timeout": 5000}]
      }
    ],
    "PreCompact": [
      {
        "matcher": "",
        "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/pre-compact-checkpoint.sh", "timeout": 10000}]
      }
    ],
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/session-budget-alert.sh", "timeout": 5000}]
      }
    ]
  }
}
```

**ポイント**: 第4章で紹介した9個のhookをすべて設定。`timeout`は各hook 5秒（checkpoint のみ10秒）。

## テンプレート3: hookファイル一式

以下のファイルを`~/.claude/hooks/`に配置する。

### large-read-guard.sh

```bash
#!/bin/bash
# 100KB超のファイル読み込みを警告
input=$(cat)
file_path=$(echo "$input" | jq -r '.tool_input.file_path // empty')

if [ -n "$file_path" ] && [ -f "$file_path" ]; then
  size=$(stat -c%s "$file_path" 2>/dev/null || stat -f%z "$file_path" 2>/dev/null)
  threshold=$((${CC_LARGE_READ_KB:-100} * 1024))
  if [ "${size:-0}" -gt "$threshold" ] 2>/dev/null; then
    echo "⚠️ ${file_path} は $((size / 1024))KB。limitパラメータで必要な部分だけ読んでください。"
    exit 0  # 警告のみ（ブロックする場合はexit 2）
  fi
fi
```

### read-budget-guard.sh

```bash
#!/bin/bash
# セッション中の読み込みファイル数を制限
budget=${CC_READ_BUDGET:-100}
warn=${CC_READ_WARN:-50}
tracker="/tmp/cc-read-budget-$PPID"

input=$(cat)
file_path=$(echo "$input" | jq -r '.tool_input.file_path // empty')
[ -z "$file_path" ] && exit 0

if [ -f "$tracker" ]; then
  grep -qF "$file_path" "$tracker" || echo "$file_path" >> "$tracker"
else
  echo "$file_path" > "$tracker"
fi

count=$(wc -l < "$tracker")
if [ "$count" -ge "$budget" ]; then
  echo "🛑 読み込み${budget}件到達。Glob/Grepで絞り込んでください。"
  exit 2  # exit 2 = ブロック
elif [ "$count" -ge "$warn" ]; then
  echo "⚠️ 読み込み${count}/${budget}件。"
fi
```

### duplicate-read-detector.sh

```bash
#!/bin/bash
# 同一ファイルの再読み込みを警告
tracker="/tmp/cc-read-history-$PPID"
input=$(cat)
file_path=$(echo "$input" | jq -r '.tool_input.file_path // empty')

if [ -n "$file_path" ] && [ -f "$tracker" ]; then
  if grep -qF "$file_path" "$tracker"; then
    echo "⚠️ ${file_path} は既に読み込み済みです。再読み込みは不要かもしれません。"
  fi
fi

echo "$file_path" >> "$tracker"
```

### token-budget-guard.sh

```bash
#!/bin/bash
# トークン予算管理
budget_warn=${CC_TOKEN_BUDGET:-50000}
budget_block=${CC_TOKEN_BLOCK:-100000}
tracker="/tmp/cc-token-budget-$PPID"

input=$(cat)
tokens=$((${#input} / 4))

total=0
[ -f "$tracker" ] && total=$(cat "$tracker")
total=$((total + tokens))
echo "$total" > "$tracker"

if [ "$total" -ge "$budget_block" ]; then
  echo "🛑 推定${total}トークン消費。/compactを実行してください。"
  exit 2  # exit 2 = ブロック
elif [ "$total" -ge "$budget_warn" ]; then
  echo "⚠️ 推定トークン: ${total}/${budget_block}"
fi
```

### subagent-budget-guard.sh

```bash
#!/bin/bash
# サブエージェント数制限
max=${CC_MAX_SUBAGENTS:-5}
tracker="/tmp/cc-subagent-$PPID"
now=$(date +%s)
echo "$now" >> "$tracker"

cutoff=$((now - 300))
[ -f "$tracker" ] && count=$(awk -v c="$cutoff" '$1>=c' "$tracker" | wc -l) || count=1
if [ "$count" -gt "$max" ]; then
  echo "🛑 5分間にサブエージェント${count}個（上限${max}）。直接Grep/Globを使ってください。"
  exit 2  # exit 2 = ブロック
fi
```

### token-spike-alert.sh

```bash
#!/bin/bash
# ツール呼び出し頻度の監視
tracker="/tmp/cc-spike-$PPID"
now=$(date +%s)
echo "$now" >> "$tracker"

cutoff=$((now - 30))
[ -f "$tracker" ] && {
  awk -v c="$cutoff" '$1>=c' "$tracker" > "${tracker}.tmp"
  mv "${tracker}.tmp" "$tracker"
  count=$(wc -l < "$tracker")
  [ "$count" -ge 15 ] && echo "⚠️ 30秒間に${count}回のツール呼び出し。暴走の可能性。"
}
```

### pre-compact-checkpoint.sh

```bash
#!/bin/bash
# /compact前にgit commit
cd "$(git rev-parse --show-toplevel 2>/dev/null)" || exit 0
if [ -n "$(git status --porcelain)" ]; then
  git add -A  # .gitignoreで.envや機密ファイルを除外していること
  # --no-verify: チェックポイント専用のため、pre-commit hookをスキップ
  git commit -m "checkpoint: pre-compact $(date +%Y%m%d-%H%M%S)" --no-verify 2>/dev/null
  echo "✅ チェックポイント保存完了。"
fi
```

### bash-output-guard.sh

```bash
#!/bin/bash
# 大出力コマンドの警告
input=$(cat)
cmd=$(echo "$input" | jq -r '.tool_input.command // empty')
if echo "$cmd" | grep -qE "git log|find / |cat |strings |hexdump"; then
  echo "$cmd" | grep -qE "\| head|\| tail|\| grep|-n [0-9]" || \
    echo "⚠️ 出力が大きくなる可能性。| head -50 を検討してください。"
fi
```

### session-budget-alert.sh

```bash
#!/bin/bash
# セッション開始時の予算表示
state="/tmp/cc-daily-budget"
today=$(date +%Y-%m-%d)
if [ -f "$state" ] && [ "$(head -1 "$state")" = "$today" ]; then
  n=$(tail -1 "$state")
  echo "$today" > "$state" && echo "$((n+1))" >> "$state"
  echo "📊 本日$((n+1))セッション目。"
else
  echo "$today" > "$state" && echo "1" >> "$state"
  echo "📊 本日1セッション目。"
fi
```

## セットアップ手順

```bash
# 1. hookディレクトリを作成
mkdir -p ~/.claude/hooks

# 2. 上記のhookファイルを配置
# （各ファイルを ~/.claude/hooks/ にコピー）

# 3. 実行権限を付与
chmod +x ~/.claude/hooks/*.sh

# 4. settings.jsonにテンプレート2の内容を追加
# ~/.claude/settings.json を編集

# 5. 動作確認
claude
# > セッション開始時に「本日1セッション目」と表示されればOK
```

## cc-safe-setupを使う場合

上記のhookを一つ一つ設定する代わりに、cc-safe-setupを使えばコマンド一つでインストールできる:

```bash
npx cc-safe-setup --install-example token-budget-guard
npx cc-safe-setup --install-example large-read-guard
npx cc-safe-setup --install-example subagent-budget-guard
# ... 他のhookも同様
```

cc-safe-setupには691以上のhookが用意されている。本書で紹介した9個のhookのうち、`large-read-guard`、`read-budget-guard`、`token-budget-guard`、`token-spike-alert`、`subagent-budget-guard`、`pre-compact-checkpoint`、`session-budget-alert`の7つはcc-safe-setupに含まれている。`duplicate-read-detector`と`bash-output-guard`は本書オリジナルのため、上記テンプレート3からコピーして使用する。

## 環境変数でカスタマイズ

hookの閾値は環境変数で調整できる。`~/.claude/settings.json`の`env`セクション、または`.bashrc`/`.zshrc`に追加:

```bash
# 大きなファイルの閾値（KB）
export CC_LARGE_READ_KB=200    # デフォルト: 100

# 読み込みファイル数の上限
export CC_READ_BUDGET=150      # デフォルト: 100
export CC_READ_WARN=120        # デフォルト: 50

# トークン予算
export CC_TOKEN_BUDGET=80000   # 警告閾値。デフォルト: 50000
export CC_TOKEN_BLOCK=150000   # ブロック閾値。デフォルト: 100000

# サブエージェント上限
export CC_MAX_SUBAGENTS=3      # デフォルト: 5
```

自分の使い方に合わせて調整してほしい。最初はデフォルト値で始めて、1-2週間使ってみてから調整するのがおすすめだ。

## おわりに

## テンプレート4: .claudeignore（不要ファイル除外）

```
# .claudeignore — Claude Codeが読まないファイルを指定
# .gitignoreと同じ書式。プロジェクトルートに配置

# ビルド成果物
dist/
build/
.next/
out/

# 依存関係（大量のファイル）
node_modules/
vendor/
.venv/
__pycache__/

# ログ・一時ファイル
*.log
*.tmp
coverage/

# 大きなデータファイル
*.sql
*.csv
*.sqlite
fixtures/

# 生成されたファイル
*.min.js
*.bundle.js
*.map
```

**ポイント**: `.gitignore`で既に除外しているものは`.claudeignore`に重複して書く必要はない。Claude固有の除外（テストフィクスチャ、大きなドキュメントフォルダ、SQLダンプなど）を追加する。

---

この本は、800時間の自律運用で学んだトークン節約テクニックを体系化したものだ。

要点を3つに絞ると:

1. **読む量を減らす**（大ファイル制限、重複読み込み防止、サブエージェントの適切な使用）
2. **コンテキストを管理する**（75%ルール、手動/compact、セッション分割）
3. **hookで強制する**（ルールを「お願い」から「自動ブロック」に変える）

トークン消費は「なんとなく」で改善できるものではない。仕組みで制御する。この本のテンプレートが、その第一歩になれば幸いだ。

---

## 読了後のおすすめ

この本を読んで、トークン節約の「その先」が気になった方へ。同じ 800+時間の運用で書き溜めた他の本を紹介する。

- **[Anthropic公式ガイドにない事故防止——Claude Code 800+時間で19点→85点にした全記録（¥800）](https://zenn.dev/yurukusa/books/6076c23b1cb18b)** — トークン節約の前提になる「そもそも壊さない」を8つの hook で体系化した本。rm -rf や credential 漏洩を止める基礎編。Token Book より先に読んでも損はない
- **[Claude Code Skills 実践レシピ集——SKILL.mdの設計から運用まで（¥500）](https://zenn.dev/yurukusa/books/a1b2c3d4e5f6g7)** — CLAUDE.md よりトークン効率が良い SKILL.md の設計と 10 レシピ。Progressive Disclosure でメインコンテキストを守る続編
- **[非エンジニアがClaude Codeを800時間走らせた——失敗と学びの全記録（¥800）](https://zenn.dev/yurukusa/books/3c3c3baee85f0a19)** — $600 の赤字と 75 日の赤裸々な体験記。技術より「AI に仕事を任せるとはどういうことか」を描いたストーリー版

英語圏の読者・チームメンバー向けには Gumroad で英語版も公開している:

- **[Token Book EN ($12)](https://yurukusa.gumroad.com/l/azrdt)** — 本書とほぼ同じデータ・テクニック・hook templates を英語で再構成した英語版。海外のチームメンバーへ共有する場合に
- **[Claude Code Migration Playbook ($19)](https://yurukusa.gumroad.com/l/claude-code-migration-playbook)** — 「このまま Claude Code を使い続けるか、Cursor / Cline / 自前 stack に移行するか」を `/usage --json` 出力に対する measurable threshold で判断するための英語ガイド。Anthropic の 2026 年 3-4 月の連続 regression を時系列で記録

