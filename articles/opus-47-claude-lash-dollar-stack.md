---
title: "Max $200 vs $45 DIY stack ——「Claude-lash」後の費用対効果を真顔で比較する"
emoji: "💸"
type: "tech"
topics: ["claudecode", "anthropic", "llm", "cost", "opus47"]
published: true
---

2026年4月、Anthropicエコシステムで5つの事件が2週間で連続した:

1. **4/16 Opus 4.7 launch** — tokenizer inflation 1.35〜1.46×
2. **4/21 $20プランから排除騒動** — 翌日撤回
3. **Issue #46829 cache TTL 1h→5m regression** — 4/12「not planned」close
4. **3rd-party tools ban強化** — OpenClaw 等
5. **4/23 weekly quota reset バグ** — `/usage` 表示そのものが不正確 (4 OPEN issues)

この5重奏の結果、HNで「Claude-lash」という言葉が生まれた ([#47855832](https://news.ycombinator.com/item?id=47855832)、400+コメント、$100/day burn層の離脱相談)。

本記事では、Max $200/月を払っている私が、**Tyler Folkman の $45 DIY stack** 提案と真剣に比較した結果をまとめる。

## Tyler Folkman の $45 DIY stack とは

4/12公開、数千ブックマーク。内訳:

| component | cost/month |
|---|---|
| Claude Pro ($20/月) | $20 |
| Anthropic API (overflow 時) | $10-15 |
| Gemini 2.5 Pro (補助) | $5-10 |
| Continue / Aider / Zed (OSS CLI) | $0 |
| **total** | **$35-45** |

Max $200 から -$155/月、年間 -$1,860。

## 真顔で比較してみる

### 比較軸 1: 並列実行

Max plan の最大の利点は subagent 10並列 + quota buffer だ。Opus 4.7 でも subagent を 5並列 で走らせれば、Pro の逐次実行に比べて体感3〜5倍速い。

$45 stack は逐次実行が基本。Continue などで subagent 的なワークフローは組めるが、**Anthropic製の最適化された並列フォークとは別物**。

**結論**: 複数ブランチ同時進行が必要な workflow なら Max に軍配。

### 比較軸 2: Opus 4.7 tokenizer inflation

Opus 4.7 は 4.6 比で 1.35〜1.46× の tokenizer 変化がある ([Simon Willison 計測](https://simonwillison.net/2026/apr/20/claude-token-counts/))。

- Max: quota は月次 fix、inflation の影響は quota 枯渇速度にのみ影響
- $45 stack: API pay-as-you-go、inflation がそのまま +40% コスト増に

**結論**: inflation を考慮すると、**重い coding workflow では Max のほうが「quota さえ持てば」お得**。しかし 4/23 weekly quota reset バグで quota 計測そのものが不安定だと効くか不明。

### 比較軸 3: 3rd-party tool ban リスク

$45 stack は Continue/Aider/Zed などの OSS tool 依存。これらは Anthropic の公式利用規約内で動作するかどうか毎回チェックが必要 (OpenClaw は4月中旬 ban された)。

Max は公式経路。ban リスクゼロ。

**結論**: Max のほうが持続可能性で勝る。

### 比較軸 4: CLAUDE.md 削減による quota buffer

4/23 Anthropic postmortem で明らかになったこと:
- 3/26-4/10 の 15日間、extractMemories で毎ターン背景に追加 Opus 呼び出しが入っていた (倍近い消費)
- 3/4-4/7 の 34日間、default effort が silent downgrade (medium → high 逆?)
- 4/16-4/20 の 5日間、verbosity 制限文字が inject された (出力品質 3% 低下)

これらは全て「設定やプロンプトでユーザー側が削減できる余地があった」。Max plan で費用対効果を取り戻す現実的な方法は:

1. `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`
2. effort を `xhigh` (coding) に明示指定 (`max` は避ける)
3. CLAUDE.md の必須行 <= 200行
4. subagent は summarize してから read (file 丸投げしない)
5. `/usage` を毎セッション start でログ (cf. Token Book ch8 症状55)

**結論**: Max plan で「払うだけ」にすると $45 stack の3倍払って3倍の価値を得られない。削減運用とセットで初めて元が取れる。

## 実務的な結論

私の今の判断 (2026-04-24 時点):

**Max plan を継続する**、ただし以下の条件で:

- [ ] `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` set (月間トークン-50%体験報告あり)
- [ ] `/config` で effort 明示、reflex の `max` を禁止
- [ ] CLAUDE.md 200行以下に絞る
- [ ] `/usage` を session start で記録 (weekly quota reset バグ検出用)
- [ ] Opus 4.7 の長文 retrieval は subagent に分離 (MRCR 78.3→32.2% 対策)

これで月の quota 消費が半減し、実質 $100/月相当の Max plan として使える。

## 移行検討が妥当な層

一方、Tyler Folkman の $45 stack を真剣に検討すべき層は以下:

- 月間 subagent 並列が月10回未満 = Max benefit を使い切らない
- Pro plan で普段は足りている = overflow のみ API で対応
- ban リスクがビジネスクリティカルでない (個人開発者)
- Opus 4.7 でなく 4.6 で十分 (多くの用途で実際そう)

## さらに深掘り

本記事は「費用対効果」単一軸での比較。トークン消費削減のより詳細な 65 症状カタログは拙著 Token Book にまとめた:

[Claude Codeのトークン消費を減らす本](https://zenn.dev/yurukusa/books/token-savings-guide) (¥2,500、第1章無料)

4月の `cache TTL` / `extractMemories` / `weekly quota reset` / `Opus 4.7 tokenizer` の全症状 + 対処法 + 実用 hook が入っている。4/21の騒動で pro→Max や Max→DIY の移行を検討している人も、**まず手元の CLAUDE.md と workflow を見直すほうが ROI が高い**というのが私のスタンスだ。

---

本記事が参考になったら:

- [cc-safe-setup](https://github.com/yurukusa/cc-safe-setup) — 安全運用 hook 700+本 (MIT)
- [ccusage](https://github.com/ryoppippi/ccusage) — Max quota 可視化 (13.2k★ OSS)
- [Tyler Folkman の元記事](https://tylerfolkman.substack.com/) — 英語読める方向け
