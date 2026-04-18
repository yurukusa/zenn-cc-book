---
title: "Claude Codeのトークン消費を減らす方法——800時間の運用で見つけた節約テクニック"
emoji: "💰"
type: "tech"
topics: ["claudecode", "ai", "llm", "トークン", "コスト削減"]
published: false
---

Claude Codeのトークン消費を半分にした。800時間以上の自律運用で見つけた方法をまとめる。

## この記事で分かること

- CLAUDE.mdの書き方で消費が2倍変わる理由と対策
- キャッシュを壊さない運用方法
- hookで自動的にトークンを節約する設定
- `/clear`と`/compact`の正しい使い分け

## 1. CLAUDE.mdを短くする（効果: 20-40%削減）

Claude Codeの指示ファイル（CLAUDE.md）は**全ターンでコンテキストに含まれる**。100行のCLAUDE.mdを35行に凝縮したら、キャッシュ読み取り率が89%→95%に改善した。

### やること

**禁止リストを許可リストに変える。** 7行で7つの禁止事項を並べるより、6行の許可リストのほうが同じカバー範囲でトークンが少ない。しかも「リストにない操作は確認」という暗黙のルールが加わる。

```markdown
## 許可されている操作
- ファイル読み取り: 常に可
- git commit: 可（mainへの直接pushは不可）
- npm install: 可（--save-devのみ）
- ファイル削除: 自分が作成したファイルのみ

上記にない破壊的操作は実行前に確認。
```

**hookに任せるルールをCLAUDE.mdから外す。**「rm -rf禁止」はCLAUDE.mdに書くよりhookで自動ブロックする方が確実だし、CLAUDE.mdが短くなる分だけ毎ターンのトークンが減る。

**具体例を1つ添える。** 「コミットメッセージは分かりやすく」より「例: "fix: キャッシュ読み取り率が0%になるバグ修正（#42796）"」を添える方が意図が正確に伝わり、リトライが減る。

## 2. キャッシュを壊さない（効果: 10-20倍の差）

Claude Codeのプロンプトキャッシュは**5分で失効する**。キャッシュが効いているとき（cache_read）と壊れたとき（cache_creation）でコストが10-20倍違う。

### キャッシュが壊れる3大原因

1. **5分以上の空白**: 考え込んだり離席するとキャッシュ失効
2. **git status変更**: ファイルを保存するだけでコンテキストが変わる
3. **skills/ディレクトリの変更**: CLAUDE.mdと同じく毎ターン読み込まれる

### 対策

- `/cost`で定期的にキャッシュ読み取り率を確認（80%以上が健全）
- 長時間作業は`/compact`でコンテキストを圧縮してから継続
- キャッシュが壊れたら`/clear`で新セッション開始が最安

## 3. hookで自動節約

hookはClaude Codeのランタイムレベルで動く自動チェック機構。CLAUDE.mdに「大きなファイルを読むな」と書くより、hookで自動ブロックする方が確実で、しかもCLAUDE.mdが短くなる。

### large-read-guard（大きなファイル読み込みをブロック）

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Read",
      "hooks": [{ "type": "command", "command": "bash ~/.claude/hooks/large-read-guard.sh" }]
    }]
  }
}
```

```bash
#!/bin/bash
# large-read-guard.sh — 1000行以上のファイル読み込みをブロック
INPUT=$(cat)
FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty' 2>/dev/null)
[ -z "$FILE" ] && exit 0
[ ! -f "$FILE" ] && exit 0
LINES=$(wc -l < "$FILE" 2>/dev/null || echo 0)
if [ "$LINES" -gt 1000 ]; then
    echo "BLOCKED: File has ${LINES} lines. Use offset/limit to read a specific section." >&2
    exit 2
fi
exit 0
```

### 一括インストール

```bash
npx cc-safe-setup
```

[cc-safe-setup](https://github.com/yurukusa/cc-safe-setup)は691本のhookから必要なものを選んでインストールできる。トークン節約hookも含まれている。

## 4. /clearと/compactの使い分け

| コマンド | いつ使う | 効果 |
|:--|:--|:--|
| `/clear` | テーマが変わるとき | コンテキスト完全リセット |
| `/compact` | 同じテーマで長時間作業 | 会話を要約して圧縮 |
| `/cost` | 定期確認 | 現在のトークン消費量を表示 |

`/clear`はキャッシュもリセットされるが、テーマが変わるなら新しいキャッシュを作る方が安い。`/compact`は同じテーマの延長でコンテキストが膨らんだときに有効。

## 5. モデル選択で節約

`/model`でSonnetに切り替えるだけで消費が大幅に減る。日常のファイル編集、テスト実行、git操作はSonnetで十分。Opusは設計判断やデバッグなど、精度が必要な場面だけ使う。

## まとめ

効果が大きい順:

1. **CLAUDE.mdを短くする** — 毎ターンの固定コスト削減
2. **キャッシュを壊さない** — 10-20倍の差
3. **hookで自動制御** — 大きなファイル読み込み防止
4. **/clearと/compactの使い分け** — 蓄積コスト管理
5. **モデル選択** — タスクに合ったモデル

---

:::message
**この記事の内容をさらに深く知りたい方へ**

800時間の実測データ、9個のトークン節約hook設定テンプレート、Before/After比較を全10章で解説しています。

📘 [Token Book — Claude Codeのトークン消費を半分にする](https://zenn.dev/yurukusa/books/token-savings-guide)（¥2,500・はじめに+第1章 無料）

まずは[無料診断（30秒）](https://yurukusa.github.io/cc-safe-setup/token-checkup.html)で今のトークン消費状況をチェック。
:::
