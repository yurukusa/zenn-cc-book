---
title: "第5章 Recovery——ファイル消失から5分で復旧する仕組み"
free: false
---

# 第5章 Recovery——ファイル消失から5分で復旧する仕組み

AIは必ず失敗する。問題は「失敗するかどうか」ではなく「失敗した後に回復できるかどうか」だ。

## チェック12: 作業前に安全ブランチを作る

**失敗例**: CCが「リファクタリングします」と言って大量のファイルを書き換えた。動かなくなった。`git log` を見たら、元のコードはどこにもなかった。コミットしないまま作業していた。

**根本原因**: 作業前のバックアップが習慣化されていなかった。

**対策**: `hooks/branch-guard.sh` の安全ブランチ機能を使う（チェック1と同じhook）。

**入手方法**: `npx cc-safe-setup` で自動インストール（[cc-safe-setup](https://github.com/yurukusa/cc-safe-setup)）

またはCLAUDE.mdに明記する:

```markdown
## 安全ブランチルール
破壊的変更の前（rm, 大規模リファクタ, DBマイグレーション等）は必ず:
git checkout -b backup/before-changes-$(date +%Y%m%d-%H%M%S)
```

このルールを入れてから、「元に戻せない」事故がゼロになった。

## チェック13: watchdogで自律稼働を監視する

**失敗例**: CCが深夜に作業していた。朝起きたら「idle状態で止まっていた」。watchdogがなかったので気づかなかった。8時間の無駄。

**対策**: `cc-solo-watchdog` を常時起動する。

**入手方法**: [cc-safe-setupリポ](https://github.com/yurukusa/cc-safe-setup)を参照

watchdogがやること:

```
cc-solo-watchdog が監視するもの:
1. CCがidle（停止）になったらnudge（「次のタスクへ」を送信）
2. CRITICAL（コンテキスト93%）になったら自動で /compact を送信
3. 長時間エラーループを検知したら emergency 停止
```

自律稼働には必須。watchdogなしで「24時間稼働」は不可能だった。

## チェック14: ループ検知で無限ループを止める

**失敗例**: CCが同じエラーを直そうとして、同じ間違いを10回繰り返した。activity-logを見たら、全く同じbashコマンドが16回実行されていた。$3.00無駄に消えた。

**根本原因**: 「同じことを3回やったら止まる」というルールがなかった。

**対策**: CLAUDE.mdにループ検知ルールを明記する。

```markdown
## ループ検知ルール
同じコマンド/アクションを3回実行して失敗したら:
1. 止まる（同じことを繰り返すな）
2. ~/ops/pending_for_human.md に「詰まっている理由」を書く
3. 別のタスクに切り替える

損切りライン: 同一エラー × 3回 = 人間待ちに回す
```

このパターンはCLAUDE.mdの自律運用テンプレートに含まれている（[cc-safe-setup](https://github.com/yurukusa/cc-safe-setup)参照）。

ループ検知ルールを入れてから、「同じコマンドを延々と繰り返す」事故が激減した。

---

## Recovery を修正した後

```
  ▸ Recovery
    [PASS] Git backup branch created before destructive changes
    [PASS] Watchdog monitors and nudges idle CC
    [PASS] Loop detection prevents infinite retry
    100%
```

Recoveryの3チェックは「失敗前提の設計」だ。CCは失敗する。でも失敗しても戻れる仕組みを作れば、失敗コストはゼロに近づく。

### 補足: auto-checkpoint — context compaction対策

長いセッションで「context compaction」が発動すると、未コミットの編集が巻き戻されることがある（[#34674](https://github.com/anthropics/claude-code/issues/34674)）。

`auto-checkpoint`はEdit/Writeの後に自動で軽量コミットを作る。compactionが起きても`git log`から復旧できる。

```bash
npx cc-safe-setup --install-example auto-checkpoint
```

同様に、`auto-snapshot`はgit外の場所にファイルコピーを保存する。両方使うと二重の保険になる。

```bash
npx cc-safe-setup --install-example auto-snapshot
```

### 補足: edit-retry-loop-guard — Edit toolのリトライループ検出

bashコマンドのループだけでなく、Edit toolでも同じファイルを何度も修正→失敗→再修正するパターンがある（[#35576](https://github.com/anthropics/claude-code/issues/35576)）。

`edit-retry-loop-guard`はEdit/Writeのファイルパスを記録し、同一ファイルへの短時間の連続編集を検出する:

```bash
npx cc-safe-setup --install-example edit-retry-loop-guard
```

5回以上の連続Edit/Writeを検出すると警告を出す。チェック14のCLAUDE.mdルールとの組み合わせで、bashループ（hook）+ Editループ（hook）+ 行動ルール（CLAUDE.md）の三重防御になる。

### 補足: pre-compact-transcript-backup — compaction失敗時のデータ喪失防止

compactionには致命的な落とし穴がある：compaction処理はまずtranscriptの内容を消去し、**その後に**APIでcompaction要約を生成する。もしAPIコールが失敗したら（rate limit、タイムアウト等）、元のtranscriptは既に消えている。要約もない。数千メッセージが空文字列になった壊れたJSONLだけが残る（[#40352](https://github.com/anthropics/claude-code/issues/40352)）。

`pre-compact-transcript-backup`はPreCompactフックで、compaction開始**前**にJSONL全体をバックアップする：

```bash
npx cc-safe-setup --install-example pre-compact-transcript-backup
```

バックアップは`~/.claude/compact-backups/`に直近3世代保存される。compaction失敗でtranscriptが壊れたら：

```bash
# バックアップ一覧を確認
ls ~/.claude/compact-backups/<session-id>/

# 最新のバックアップからtranscriptを復元
cp ~/.claude/compact-backups/<session-id>/latest.jsonl <元のtranscriptパス>
```

auto-checkpointがコード編集を守り、pre-compact-transcript-backupが会話全体を守る。両方入れると、compactionに関連する2種類のデータ喪失を防げる。

## 事故発生時の復旧手順

予防の仕組みを入れたら、次は「事故が起きた時にどう使うか」を知っておく。以下は実際にファイルが消えた・壊れた時の復旧フローだ。

### パターン1: リファクタで壊れた → backup branchから復旧

```bash
# 1. 壊れたことを確認（テストが通らない、ファイルが消えた等）
git status

# 2. backup branchの一覧を確認
git branch | grep backup/

# 3. 壊れたファイルだけを取り出す
git checkout backup/before-changes-20260407-143025 -- path/to/broken-file.py

# 4. 全体を巻き戻すなら
git checkout backup/before-changes-20260407-143025
```

所要時間: 約1分。

### パターン2: コミットせずに上書きされた → auto-checkpointから復旧

auto-checkpointはEdit/Write後に自動コミットを作る。CCが勝手にファイルを書き換えた場合でも、直前の状態がgit logに残っている。

```bash
# 1. auto-checkpointのコミット一覧を確認
git log --oneline | grep "auto-checkpoint"

# 2. 特定ファイルの変更履歴を確認
git log --oneline -- path/to/file.py

# 3. 特定コミットからファイルを取り出す
git checkout <commit-hash> -- path/to/file.py
```

所要時間: 約2分。

### パターン3: gitごと壊れた → auto-snapshotから復旧

auto-snapshotはgit外の場所にファイルコピーを保存する。gitの状態に関係なく復旧できる。

```bash
# 1. スナップショットの場所を確認
ls ~/.claude/snapshots/

# 2. 対象ファイルの最新スナップショットを探す
find ~/.claude/snapshots/ -name "file.py" -ls

# 3. コピーで復元
cp ~/.claude/snapshots/<timestamp>/path/to/file.py ./path/to/file.py
```

所要時間: 約2分。

### パターン4: compaction失敗でセッションが壊れた → transcript復元

pre-compact-transcript-backupがcompaction前の完全なtranscriptを保存している。

```bash
# 1. バックアップを確認
ls ~/.claude/compact-backups/<session-id>/

# 2. 復元
cp ~/.claude/compact-backups/<session-id>/latest.jsonl <元のtranscriptパス>
```

所要時間: 約30秒。

### 5分復旧の判断フロー

```
ファイルが消えた/壊れた
  ├─ backup branchがある → パターン1（1分）
  ├─ auto-checkpointがある → パターン2（2分）
  ├─ auto-snapshotがある → パターン3（2分）
  └─ 全部ない → git reflog を確認
       └─ git reflog show HEAD | head -20
          └─ 目的のコミットがあれば git checkout <hash> -- <file>
```

予防ツールが入っていれば、どのパターンでも5分以内に復旧できる。入っていなくても、git reflogが最後の砦として30日分の履歴を持っている。

---

次章: Autonomy——CCが自分で判断して動く仕組み
