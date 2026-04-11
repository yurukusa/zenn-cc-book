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

「AIに任せて大丈夫なのか」という不安を持ったまま800時間使い続けた記録は[非エンジニアがClaude Codeを800時間走らせた——失敗と学びの全記録](https://zenn.dev/yurukusa/books/3c3c3baee85f0a19)（¥1,500・第2章まで無料）に書いた。
:::
