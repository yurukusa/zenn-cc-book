---
title: "Claude Codeのトークン節約で本当に効いた3つの設定——800時間の運用データから"
emoji: "💰"
type: "tech"
topics: ["ClaudeCode", "AI", "トークン", "プロンプトエンジニアリング", "Claude"]
published: false
---

Claude Code Max plan（$200/月）を使い始めて最初に困ったのが、トークンの消費速度だった。朝から使い始めて昼にはquotaが半分消えていた。

GitHub Issue [#42796](https://github.com/anthropics/claude-code/issues/42796) には1,700+件のリアクションがついている。Claude Codeユーザー最大の痛みだ。

800時間の自律運用で試行錯誤した中から、**実際に効果を測定できた**3つの設定を紹介する。

## 1. CLAUDE.mdを100行→35行に凝縮する

CLAUDE.mdの内容は**毎ターン**システムプロンプトとして送信される。100行と35行では、200ターンのセッションで数万トークンの差が出る。

やったこと:

- **禁止リストをhookに移す**: 「rm -rfを使うな」等の安全ルールはCLAUDE.mdに書いても忘れられることがある。hookなら100%強制される。6行の禁止事項→1行の「hookが自動ブロック」に
- **散文をテーブルに変換**: 判断基準はテーブル形式の方が少ないトークンで同じ情報量を表現できる
- **具体例を1つ添える**: 「コミットメッセージは分かりやすく」→ `✓ fix: prevent duplicate API calls` / `✗ fix bug` のように例示すると、書き直し指示のリトライが消える

**結果**: キャッシュ読み取り率が89%→95%に改善した。

## 2. hookでトークン浪費パターンを自動ブロック

Claude Codeのhookは、コマンド実行前に割り込んで条件を検査する仕組み。トークン浪費の原因になる操作を自動でブロックできる。

例: 大きなファイルの丸ごと読み込みを防ぐ`large-read-guard`

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Read",
      "hooks": [{
        "type": "command",
        "command": "bash -c 'FILE=$(cat | jq -r \".tool_input.file_path // empty\"); [ -z \"$FILE\" ] && exit 0; SIZE=$(wc -c < \"$FILE\" 2>/dev/null || echo 0); [ \"$SIZE\" -gt 100000 ] && { echo \"BLOCKED: file too large ($SIZE bytes). Use offset/limit.\" >&2; exit 2; }; exit 0'"
      }]
    }]
  }
}
```

100KB以上のファイルを丸ごと読もうとするとブロックし、`offset`/`limit`で部分読み込みを強制する。1回のブロックで数千トークンを節約できる。

**ポイント**: これは「Claude Codeに節約をお願いする」のとは根本的に違う。CLAUDE.mdに「大きなファイルは部分的に読んで」と書いても、忘れることがある。hookは忘れない。

## 3. /compactのタイミングを意識する

`/compact`はコンテキストを圧縮するコマンド。タイミングが重要だ。

- **早すぎるcompact**: 重要な文脈が失われ、Claude Codeが同じファイルを再読み込み→トークン浪費
- **遅すぎるcompact**: コンテキストが溢れて自動compactionが走り、制御不能

実用的な目安: `/cost`でコンテキスト使用率を確認し、60-70%になったら手動で`/compact`。

## まとめ

| 設定 | 効果 | 導入時間 |
|------|------|---------|
| CLAUDE.md凝縮 | キャッシュ率89%→95% | 30分 |
| large-read-guard hook | 大ファイル読み込み防止 | 5分 |
| /compactタイミング管理 | コンテキスト最適化 | 即日 |

hookの一括インストール:

```bash
npx cc-safe-setup
```

688個のexample hookと9,200+テストが含まれている。

:::message
**もっと体系的にトークン消費を最適化したい方へ**
全10章・44,000字の[Token Book](https://zenn.dev/yurukusa/books/token-savings-guide)（¥2,500・はじめに+第1章 無料）で、800時間分の実測データと設定テンプレートを解説しています。まずは[Token Checkup（無料・30秒）](https://yurukusa.github.io/cc-safe-setup/token-checkup.html)で現状チェック。
:::
