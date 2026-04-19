---
title: "CLAUDE.mdにルールを書いても守られない理由と、hookで強制する方法"
emoji: "🔐"
type: "tech"
topics: ["claudecode", "hooks", "security", "beginners"]
published: false
---

CLAUDE.mdに「mainにpushするな」と書いた。Claudeは最初は守った。2時間後、コンテキストが満杯になったタイミングで、mainにpushした。

なぜか。

## CLAUDE.mdの限界

CLAUDE.mdはプロンプトの一部としてコンテキストに読み込まれる。つまり:

1. **コンテキストが満杯になると圧縮される** — ルールが消える可能性
2. **AIの判断で無視される** — 「この状況では例外」と勝手に解釈
3. **検証手段がない** — ルールを守ったかどうか確認できない

## hookはプロセスレベルで動く

hookはCLAUDE.mdとは全く別の仕組み:

```
AIが "git push main" を実行しようとする
  ↓
Claude Code がhookスクリプトを実行
  ↓
hookが exit 2 を返す
  ↓
ツール呼び出しがブロックされる
  ↓
AIに「ブロックされました」と通知
```

hookはAIのコンテキストの外で動く。コンテキストが圧縮されても、hookは消えない。AIが「例外」と判断しても、hookはブロックする。

## 30秒でhookを入れる

```bash
npx cc-safe-setup
```

これで8個のhookが入る。mainへのpush、rm -rf、シークレット漏洩 — 全てプロセスレベルでブロック。

## CLAUDE.mdとhookの使い分け

| | CLAUDE.md | hook |
|---|---|---|
| 仕組み | プロンプト（AI判断） | プロセス（強制） |
| コンテキスト圧縮 | 影響を受ける | 影響なし |
| AIの判断で無視 | される | されない |
| 用途 | ガイドライン、スタイル | 安全ルール、禁止事項 |

**ガイドライン**（コードスタイル、命名規則）→ CLAUDE.md
**禁止事項**（rm -rf、main push、シークレット）→ hook

両方使う。CLAUDE.mdは「こうしてほしい」、hookは「これは絶対にさせない」。

---
:::message
**トークン消費が気になる方へ**
[Token Checkup（無料）](https://yurukusa.github.io/cc-safe-setup/token-checkup.html)で30秒診断。どこでトークンが消えているか可視化できる。
もっと深く知りたい方は → [Claude Codeのトークン消費を半分にする（¥2,500）](https://zenn.dev/yurukusa/books/token-savings-guide)
「AIに任せて大丈夫なのか」という不安を持ったまま800時間使い続けた記録は[非エンジニアがClaude Codeを800時間走らせた——失敗と学びの全記録](https://zenn.dev/yurukusa/books/3c3c3baee85f0a19)（¥800・第2章まで無料）に書いた。
:::
