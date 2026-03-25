---
title: "第4章 Monitoring——AIが何をしているかを知る"
free: false
---

# 第4章 Monitoring——AIが何をしているかを知る

「AIに任せたが、何をしているかわからない」という状態が最も危険だ。

## チェック9: コンテキスト残量を監視する

**失敗例**: CCが長時間作業していたら、突然「前の話を覚えていない」状態になった。コンテキスト窓が満杯になっていた。気づかないまま、最初から同じ作業を始めた。

**対策**: `hooks/context-monitor.sh` を PostToolUse に追加する。

**[context-monitor.sh を入手](https://github.com/yurukusa/claude-code-hooks/blob/main/hooks/context-monitor.sh)**

4段階の警告システム:

| レベル | 条件 | アクション |
|--------|------|-----------|
| CAUTION | 75%消費 | ログに記録 |
| WARNING | 85%消費 | CCに警告表示 |
| CRITICAL | 93%消費 | 自動/compactを送信 |
| EMERGENCY | 97%消費 | 緊急停止 + 状態保存 |

watchdog（`cc-solo-watchdog`）と組み合わせると、CRITICAL時に自動で `/compact` コマンドを送信して、コンテキストを要約・圧縮する。

## チェック10: 活動ログを記録する

**失敗例**: 翌朝確認したら「何をしていたかわからない」。ログがないので何が起きたか追跡できない。

**対策**: `hooks/activity-logger.sh` が全てのtool useを記録する（チェック6と同じhook）。

```bash
# ~/.claude/activity-log.jsonl に記録される例
{"ts":"2026-02-28T12:00:00Z","tool":"Edit","file":"src/main.py","lines_added":15,"lines_deleted":3}
{"ts":"2026-02-28T12:01:00Z","tool":"Bash","cmd":"npm test","exit_code":0}
{"ts":"2026-02-28T12:02:00Z","tool":"Bash","cmd":"git push","exit_code":0}
```

**何が嬉しいか**:
- 「昨日CCは何をしたか」が秒単位でわかる
- エラーが起きた時に「その前に何をしたか」を追跡できる
- コスト計算の根拠になる（何ツールコールしたか）

## チェック11: 日次サマリーを自動生成する

**失敗例**: 1週間後にぐらすが「先週のAIは何をしたの？」と聞いた。answer できなかった。ログはあるが読めない。

**対策**: `hooks/proof-log-session.sh` を Stop イベントに追加する。

**[proof-log-session.sh を入手](https://github.com/yurukusa/claude-code-hooks/blob/main/hooks/proof-log-session.sh)**

このhookは、セッション終了時に5W1Hフォーマットのサマリーを自動生成する:

```markdown
# 2026-02-28 セッションログ

## 何をしたか
- cc-health-checkのFAIL時にGitHub直リンクを追加（cli.mjs, index.html）
- DRY原則のQiita記事ドラフト作成
- Twitter告知完了（cc-health-check）

## なぜやったか
- Twitter告知後のユーザーが「どう直すか」に詰まらないよう改善
- 非エンジニアシリーズの3本目として

## 結果
- GitHub Push: 3コミット (0883205, 2d03603, 8e83603)
- コスト: 推定 $2.40
```

毎日このファイルが積み上がっていくと、「AIがどう成長しているか」が見えるようになる。

---

## Monitoring を修正した後

```
  ▸ Monitoring
    [PASS] Context window usage monitored with alerts
    [PASS] Activity logging tracks commands
    [PASS] Daily summaries are generated
    100%
```

3hookで3チェック全部カバーできる。最もコスパが高いカテゴリかもしれない。

チェック1（コンテキスト監視）は `npx cc-safe-setup` に含まれている。40%→25%→20%→15%の4段階警告で、コンテキスト枯渇前に対処できる。

---

次章: Recovery——失敗から回復する仕組み
