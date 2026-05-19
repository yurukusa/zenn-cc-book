---
title: "Claude Codeが7GBのデータを消した——rm -rf事故をhookで防ぐ"
emoji: "🛡️"
type: "tech"
topics: ["claudecode", "hooks", "セキュリティ", "ai"]
published: true
---

「PDFを整理して」と頼んだだけだった。Claude Codeはファイルを分類し、フォルダ構造を作り、3,467個のPDFを移動した。再整理が必要になった時、確認なしで`rm -rf`を実行した。7GBのデータが消えた。

GitHub Issue [#46058](https://github.com/anthropics/claude-code/issues/46058)（2026-04-10報告、high-priorityラベル付き）。

これは孤立した事例ではない。

| Issue | 被害 |
|---|---|
| [#46058](https://github.com/anthropics/claude-code/issues/46058) | 3,467ファイル / 7GB消失 |
| [#36339](https://github.com/anthropics/claude-code/issues/36339) | ユーザープロファイル全体（165GB、NTFSジャンクション経由） |
| [#37331](https://github.com/anthropics/claude-code/issues/37331) | 未pushソースコード全体 |
| [#36640](https://github.com/anthropics/claude-code/issues/36640) | 本番Nextcloudデータ（NFSマウント経由） |

## hookで防ぐ

PreToolUseフックは、Claudeがコマンドを実行する前に検査し、危険なら止める。

```bash
#!/bin/bash
# destructive-guard-minimal.sh
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
[ -z "$CMD" ] && exit 0

if echo "$CMD" | grep -qiE 'rm\s+(-[a-z]*[rf][a-z]*\s+|--force|--recursive)|git\s+(reset\s+--hard|clean\s+-[a-z]*f)'; then
    echo "BLOCKED: destructive command" >&2
    exit 2
fi
```

settings.jsonに登録:

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{"type": "command", "command": "bash .claude/hooks/destructive-guard-minimal.sh"}]
    }]
  }
}
```

## 30秒でインストール

手動設定が面倒なら:

```bash
npx cc-safe-setup
```

8個の安全フックを一括インストール。rm -rfブロックに加えて、mainへのpush防止、シークレット漏洩検出、構文チェックも含まれる。

[cc-safe-setup](https://github.com/yurukusa/cc-safe-setup) — 650+ hooks / 9,200+ tests

:::message
**トークン消費も気になる方へ**
rm -rfの次に多い痛みは「トークンが10倍消える」問題（[#42796](https://github.com/anthropics/claude-code/issues/42796)、2,130+リアクション）。
[Claude Codeを本番品質にする——実践ガイド](https://zenn.dev/yurukusa/books/6076c23b1cb18b)（¥800・第3章まで無料）のch10で診断と対策を解説。

「AIに任せて大丈夫なのか」という不安を持ったまま800時間使い続けた記録は[非エンジニアがClaude Codeを800時間走らせた——失敗と学びの全記録](https://zenn.dev/yurukusa/books/3c3c3baee85f0a19)（¥800・第2章まで無料）に書いた。
:::
:::message
**トークン消費が気になる方へ**
[Token Checkup（無料）](https://yurukusa.github.io/cc-safe-setup/token-checkup.html)で30秒診断。どこでトークンが消えているか可視化できる。
もっと深く知りたい方は → [Claude Codeのトークン消費を半分にする（¥2,500）](https://zenn.dev/yurukusa/books/token-savings-guide)
:::
---
5月22日に新刊の事例集 [Claude Code Claim-Verify Handbook](https://yurukusa.gumroad.com/l/claim-verify-handbook?utm_source=zenn-rmrf-prevention-hook&utm_medium=article&utm_campaign=handbook-zenn-bulk) ($19、 約89頁、 約113,000字) を発売します。 道具が「成功した」 「比較した」 「設定された」 と主張する一方で実態が乖離していた事例を GitHubの起票の集まりから130件 (本文15件+付録D 115件) 整理した本で、 14件の防衛の手順と5件の自動の点検の道具と一緒に提供しています。 試し読みのGist (約16,000字、 章1の全件) は [こちら](https://gist.github.com/yurukusa/6dd608049064ed66c54f1a545a7b47a8?utm_source=zenn-rmrf-prevention-hook&utm_medium=article&utm_campaign=handbook-preview-zenn-bulk)。
予防 hook の集まりは [cc-safe-setup](https://github.com/yurukusa/cc-safe-setup) (MIT、 745件以上の hook、 30,000件以上のインストール)。 月額の継続の媒体 [CC Safety Lab](https://yurukusa.github.io/cc-safe-setup/safety-lab.html?utm_source=zenn-rmrf-prevention-hook&utm_medium=article&utm_campaign=safety-lab-zenn-bulk) (¥500/月、 Ko-fi) は毎月の事故の整理を配信。
