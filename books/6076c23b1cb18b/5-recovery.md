---
title: "第5章 Recovery——失敗から回復する仕組み"
free: false
---

# 第5章 Recovery——失敗から回復する仕組み

AIは必ず失敗する。問題は「失敗するかどうか」ではなく「失敗した後に回復できるかどうか」だ。

## チェック12: 作業前に安全ブランチを作る

**失敗例**: CCが「リファクタリングします」と言って大量のファイルを書き換えた。動かなくなった。`git log` を見たら、元のコードはどこにもなかった。コミットしないまま作業していた。

**根本原因**: 作業前のバックアップが習慣化されていなかった。

**対策**: `hooks/branch-guard.sh` の安全ブランチ機能を使う（チェック1と同じhook）。

**[branch-guard.sh を入手](https://github.com/yurukusa/claude-code-hooks/blob/main/hooks/branch-guard.sh)**

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

**[cc-solo-watchdog を入手](https://github.com/yurukusa/claude-code-hooks/blob/main/scripts/cc-solo-watchdog)**

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

**[CLAUDE-autonomous.md テンプレート](https://github.com/yurukusa/claude-code-hooks/blob/main/templates/CLAUDE-autonomous.md)** にこのパターンが含まれている。

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

---

次章: Autonomy——CCが自分で判断して動く仕組み
