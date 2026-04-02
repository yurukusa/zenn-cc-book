---
title: "Claude APIの回答に「見えないデータ」が隠れている——パーサーが壊れる原因はこれだった"
emoji: "🧠"
type: "tech"
topics: ["claudecode", "anthropic", "ai", "extendedthinking"]
published: true
---

Claude APIでExtended Thinking（拡張思考）を有効にしたら、たまにパーサーがエラーを吐く。JSONの形が想定と違う。でも再実行すると動く。

原因は単純だった——**レスポンスに2つの別々のブロックが入っている**のに、1つしか読んでいなかった。通常モードと同じコードで処理していたから、「推論プロセス」ブロックが丸ごと無視されていた。

[Anthropic Academy](https://anthropic.skilljar.com/)（無料）で学んだ、Extended Thinkingの正しいパース方法。

## レスポンスは2ブロックで返ってくる

Extended Thinkingを有効にすると、レスポンスに2つの別々のコンテンツブロックが含まれる。

1. **thinking block** — モデルの内部推論プロセス
2. **text block** — 最終回答

この順番で返ってくる。thinking blockが先、text blockが後。

自分のコードはたまたま最後のコンテンツブロックだけ読んでいたから問題なかった。だが「正しく動いている」のと「たまたま動いている」は違う。

:::details CC補足: 2ブロック構造の技術詳細
Extended Thinkingを有効にしたAPIレスポンスの構造:

```json
{
  "content": [
    {
      "type": "thinking",
      "thinking": "ユーザーの質問を分析すると...",
      "signature": "WJxS3P..."
    },
    {
      "type": "text",
      "text": "回答はこちらです。"
    }
  ]
}
```

thinking blockには`type: "thinking"`、text blockには`type: "text"`が設定される。パーサーが区別しないと、生の推論過程がユーザー向け出力に混入するリスクがある。

Streaming時は`thinking_delta`イベントと`text_delta`イベントで区別される。ストリーミング処理を実装する場合、イベントタイプによって書き込み先を振り分ける必要がある。
:::

## 署名（signature）がある理由

thinking blockには**署名（signature）**が付与される。暗号トークンの一種だ。

なぜ署名が必要なのか。会話の続きでthinking blockをAPIに送り返す場面がある。マルチターンの会話で、前回の思考プロセスをコンテキストとして保持したい場合だ。

このとき、thinking blockの内容が改ざんされていないことを署名で検証する。もしテキストを書き換えていたら、署名の検証が失敗する。

開発者が悪意を持って推論を書き換え、モデルの判断を誘導するのを防ぐ仕組みだ。

:::details CC補足: 署名検証の仕組み
署名はAnthropicのサーバー側で生成される暗号トークンだ。thinking blockのテキスト内容に対して生成されるため、テキストを1文字でも変更すると署名が無効になる。

マルチターン会話でthinking blockを再送する際のフロー:
1. 前回のレスポンスからthinking block（署名付き）をそのまま保持
2. 次のリクエストのmessagesに含めて送信
3. サーバー側で署名を検証
4. 検証成功 → 前回の思考コンテキストとして利用
5. 検証失敗 → エラー

実装上の注意: thinking blockの内容をログやDBに保存する場合、署名フィールドも一緒に保存すること。署名なしでは再送できない。
:::

## Redacted Thinking という概念

Academyで初めて知った概念がもう一つある。**redacted thinking**（墨消し思考）だ。

安全システムがthinking blockの内容にフラグを立てた場合、thinking textが暗号化された状態で返される。人間が読めない状態だ。

```json
{
  "type": "redacted_thinking",
  "data": "base64encodeddata..."
}
```

通常のthinking blockとは**別のブロックタイプ**（`redacted_thinking`）として返される。内容は読めないが、コンテキスト維持のために会話に含めることは可能。モデルは暗号化された思考プロセスを「覚えている」状態で次のターンに進める。

開発者としてはどう扱うべきか。`redacted_thinking`ブロックがレスポンスに含まれていても、そのままmessagesに含めて次のリクエストに送ればいい。中身は読めないが、モデルの一貫性は保たれる。

:::details CC補足: redacted thinkingの技術詳細
redacted thinkingは安全機構の一部。モデルの推論過程に、公開すべきでない内容（個人情報への言及、安全ガイドラインの内部処理など）が含まれた場合にトリガーされる。

テスト用のマジックストリング（`anthropic magic string triggered redacted thinking`で始まる特定文字列）を入力に含めると、redacted thinking blockを強制的に返させることができる。テスト環境でのハンドリング確認に使える。

redacted thinkingが返ってきたことは、モデルの回答に問題があることを意味しない。あくまでthinking *process* の一部が非公開になっただけで、最終回答（text block）は通常通り生成される。
:::

## budget_tokensの設計

Extended Thinkingの思考量は`budget_tokens`パラメータで制御する。「何トークンまで考えてよいか」の上限だ。

ここにも制約がある。

- **最小値は1024トークン**。これより少なくは設定できない
- **max_tokensはbudget_tokensより大きくなければならない**。max_tokens = 4096、budget_tokens = 8192のような設定はエラーになる

max_tokensは「thinking + textの合計上限」。budget_tokensは「thinkingだけの上限」。budget_tokensがmax_tokensを超えると、text blockに割り当てる余裕がなくなる。

```python
# 正しい設定
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000  # max_tokensより小さい
    },
    messages=[{"role": "user", "content": "..."}]
)
```

:::details CC補足: budget_tokensの実用的な設定指針
thinking tokensは出力トークンとして課金される。budgetを大きく取れば精度が上がる可能性があるが、コストも比例して増える。

実用的な指針:
- **単純な質問**: budget_tokens = 2048〜4096で十分
- **複雑な推論**: budget_tokens = 8192〜16000
- **数学・コーディング**: budget_tokens = 16000〜32000以上

budgetは上限であり、モデルが毎回使い切るわけではない。単純な質問には少ないトークンで思考を終える。

重要: Extended Thinking有効時は**temperature = 1.0に固定**される。変更不可。思考プロセスには確率分布全体が必要なため、deterministic（temperature 0）な出力は設計上サポートされていない。
:::

## 「まずプロンプト改善」が先

Academyが強調していたポイントがもう一つある。

Extended Thinkingを有効にするタイミングだ。「精度が足りない → Extended Thinking ON」ではない。

正しい順序:
1. まずプロンプトを改善する
2. それでも精度が不足する場合にExtended Thinkingを検討する

Extended Thinkingは銀の弾丸ではない。thinking tokensは出力トークンとして課金される。雑なプロンプトのまま有効にしても、モデルが長々と考えた末に同じ答えを出すだけだ。コストだけ増える。

プロンプトの構造、指示の明確さ、Few-shot Examplesの追加。これらを試し尽くした上で、それでも足りないときに初めてExtended Thinkingの出番が来る。

## まとめ

Extended Thinkingで知らなかったこと:
- レスポンスが2ブロック（thinking + text）で返る
- thinking blockに署名が付き、改ざんを防ぐ
- redacted thinkingという非公開モードがある
- budget_tokensの最小値は1024、max_tokensより小さくなければならない
- temperatureは1.0固定で変更不可
- 有効にする前にプロンプト改善が先

どれもエラーにはならない。だが知らないとコストが無駄に増えたり、パーサーが壊れたりする。

次回はRAG（Retrieval Augmented Generation）について書く。re-rankingが「ソートし直す」ではない話と、embeddingの実態を扱う。


**第2章「Safety Guards」まで無料公開中。**


🛠 Claude Codeの自律運用に使っている安全装置: [claude-code-hooks](https://github.com/yurukusa/claude-code-hooks)

---
:::message
**Claude Codeの安全対策、まだですか？**
`npx cc-safe-setup` — ワンコマンドで `rm -rf` 誤爆・秘密鍵コミット・force pushを防止。900+テスト済み。
→ [GitHub](https://github.com/yurukusa/cc-safe-setup)
:::
---
:::message
**Claude Codeの自律運用をもっと深く学びたい方へ**
トークン消費の突然の急増、ファイル消失、無限ループ——700+時間の自律稼働で起きた失敗と対策をまとめました:
[Claude Codeを本番品質にする——実践ガイド](https://zenn.dev/yurukusa/books/6076c23b1cb18b)（¥800・第3章まで無料）
:::
