---
title: "終章 Putting It Together——全部つなぎ合わせる"
free: false
---

# 終章 Putting It Together——全部つなぎ合わせる

20のチェックをカバーした。最後に「全部つなぎ合わせる」ための話をする。

## 全チェック一覧

| # | カテゴリ | チェック | Hook/Template |
|---|---------|---------|---------------|
| 1 | Safety Guards | 危険コマンドをPreToolUseでブロック | branch-guard.sh |
| 2 | Safety Guards | APIキーを専用ファイルに保管する | CLAUDE-autonomous.md |
| 3 | Safety Guards | main/masterへの直pushを防ぐ | branch-guard.sh |
| 4 | Safety Guards | エラーがある状態で外部APIを叩かない | error-memory-guard.sh |
| 5 | Code Quality | ファイル編集後に構文チェックを走らせる | syntax-check.sh |
| 6 | Code Quality | bashの出力からエラーを自動検知する | activity-logger.sh |
| 7 | Code Quality | 完了の定義（DoD）を持つ | dod-checklists.md |
| 8 | Code Quality | AIが自分のアウトプットを検証する | CLAUDE-autonomous.md |
| 9 | Monitoring | コンテキスト残量を監視する | context-monitor.sh |
| 10 | Monitoring | 活動ログを記録する | activity-logger.sh |
| 11 | Monitoring | 日次サマリーを自動生成する | proof-log-session.sh |
| 12 | Recovery | 作業前に安全ブランチを作る | branch-guard.sh |
| 13 | Recovery | watchdogで自律稼働を監視する | cc-solo-watchdog |
| 14 | Recovery | ループ検知で無限ループを止める | CLAUDE-autonomous.md |
| 15 | Autonomy | タスクキューで「次に何をするか」を明示する | task-queue.yaml |
| 16 | Autonomy | 「人間に聞かない」ルールを徹底する | no-ask-human.sh |
| 17 | Autonomy | 永続状態ファイルでセッション間を引き継ぐ | mission.md |
| 18 | Coordination | 意思決定を記録・警告する | decision-warn.sh |
| 19 | Coordination | 並列タスクを安全に分配する | CLAUDE-autonomous.md |
| 20 | Coordination | 学習ログで同じミスを繰り返さない | lessons.md |

## 最速セットアップ手順

### 推奨: ワンコマンド（30秒）

```bash
npx cc-safe-setup --shield
```

これだけで8個のコアhook + プロジェクト検出 + 推奨hookが全てインストールされる。500個のhookから最適な組み合わせを自動選択する。

### 手動でやりたい場合（30分）

仕組みを理解したい、またはカスタマイズしたい場合は手動セットアップもできる。

#### Step 1: hookスクリプトをインストール（10分）

```bash
# ワンコマンドで8つの安全hookをインストール
npx cc-safe-setup

# 追加のexample hookをインストール（500種から選択可能）
npx cc-safe-setup --install-example block-database-wipe
npx cc-safe-setup --install-example auto-checkpoint
```

#### Step 2: settings.jsonにhookを登録（5分）

```json
// ~/.claude/settings.json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {"type": "command", "command": "bash ~/.claude/hooks/branch-guard.sh"},
          {"type": "command", "command": "bash ~/.claude/hooks/error-memory-guard.sh"},
          {"type": "command", "command": "bash ~/.claude/hooks/no-ask-human.sh"}
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {"type": "command", "command": "bash ~/.claude/hooks/syntax-check.sh"},
          {"type": "command", "command": "bash ~/.claude/hooks/activity-logger.sh"}
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {"type": "command", "command": "bash ~/.claude/hooks/activity-logger.sh"},
          {"type": "command", "command": "bash ~/.claude/hooks/context-monitor.sh"},
          {"type": "command", "command": "bash ~/.claude/hooks/decision-warn.sh"}
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {"type": "command", "command": "bash ~/.claude/hooks/proof-log-session.sh"}
        ]
      }
    ]
  }
}
```

#### Step 3: templatesをセットアップ（10分）

```bash
# CLAUDE.mdに自律運用ルールを追加
# 以下のルールを自分のCLAUDE.mdに追記する:
# - 人間に聞かないルール
# - ループ検知ルール（同一エラー×3回=停止）
# - セッション開始時手順（前回の状態確認→継続）
# テンプレートはcc-safe-setupリポを参照:
# https://github.com/yurukusa/cc-safe-setup
```

#### Step 4: CLAUDE.mdに基本ルールを追記（5分）

CLAUDE-autonomous.mdの内容を自分のCLAUDE.mdにコピーする。「人間に聞かないルール」「ループ検知ルール」「セッション開始時手順」を含める。

#### Step 5: watchdogを起動

```bash
# 別ターミナルで起動
cc-solo-watchdog &
```

## スコアがどう変わるか

このBookの手順を全て実施した場合のスコア変化:

```
Before（デフォルト設定）:
  Safety Guards:  0%
  Code Quality:   0%
  Monitoring:     0%
  Recovery:       0%
  Autonomy:       0%
  Coordination:   0%
  Total:          0/100

After（本書の手順全実施）:
  Safety Guards:  100% (+21pts)
  Code Quality:   100% (+21pts)
  Monitoring:     100% (+16pts)
  Recovery:       100% (+16pts)
  Autonomy:       100% (+16pts)
  Coordination:   100% (+10pts)
  Total:          100/100
```

## 700+時間稼働して学んだこと

最終的に一番大切だったのは「仕組みで安全を担保する」ことだった。

ルールをCLAUDE.mdに書いても守られないことがある。でもhookでブロックすれば物理的に止まる。監視ツールを入れれば問題を知ることができる。ログがあれば原因を追える。

「AIを信頼する」のではなく、「信頼できる仕組みを作る」。それが本番品質への道だった。

---

**cc-health-checkで現在のスコアを診断する**:

```bash
npx cc-health-check
```

または: [https://yurukusa.github.io/cc-health-check/](https://yurukusa.github.io/cc-health-check/)

**10秒で8つの安全フックをインストール**:

```bash
npx cc-safe-setup
```

**500個のexample hookからプロジェクトに合ったものを追加**:

```bash
npx cc-safe-setup --examples        # 一覧（5カテゴリ）
npx cc-safe-setup --install-example block-database-wipe  # 個別インストール
```

**全フック・レシピを入手する**:

[https://github.com/yurukusa/cc-safe-setup](https://github.com/yurukusa/cc-safe-setup)

---

*このBookは、Claude Codeが700+時間自律稼働した実体験から生まれました。スコア0から100への道のりは、実際に動いているシステムの記録です。*
