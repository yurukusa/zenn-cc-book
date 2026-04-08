---
title: "第4章 Monitoring——キャッシュ読取率が89%→4%に暴落してトークンが20倍消えた"
free: false
---

# 第4章 Monitoring——トークンがどこで消えているかを可視化する

ある朝、Max Planの残り時間を確認したら、5時間あったはずの枠が15分で消えていた。プロンプトキャッシュの読取率が89%から4%に暴落していた。トークン消費が20倍。GitHub Issue [#40524](https://github.com/anthropics/claude-code/issues/40524)には同じ現象の報告が220件以上ある。

「AIに任せたが、何をしているかわからない」——この状態が最も危険だ。この章では、暴走を検知する仕組みを作る。

## チェック9: コンテキスト残量を監視する

**失敗例**: CCが長時間作業していたら、突然「前の話を覚えていない」状態になった。コンテキスト窓が満杯になっていた。気づかないまま、最初から同じ作業を始めた。

**対策**: `hooks/context-monitor.sh` を PostToolUse に追加する。

**入手方法**: `npx cc-safe-setup`（[cc-safe-setup](https://github.com/yurukusa/cc-safe-setup)）

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

**入手方法**: `npx cc-safe-setup --install-example proof-log-session`（[cc-safe-setup](https://github.com/yurukusa/cc-safe-setup)）

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

## 補足: トークン消費を追跡する

**失敗例**: Max Planの5時間制限が1時間で枯渇。同じ使い方をしているのに、以前の数倍のスピードでトークンが減っていく。GitHubには同じ報告が山のように上がっている。

**なぜ起きるか**: トークンを消費する隠れた要因が複数ある。

| 原因 | 仕組み | 影響度 |
|------|--------|--------|
| **プロンプトキャッシュ無効化** | CCが自分の会話履歴(.jsonl)を読むとキャッシュのプレフィックスが変わり、毎ターン全履歴を再送信する | **最大（10〜20倍）** |
| Tool Search | 毎ターン、MCPサーバーのツール定義がコンテキストに追加される | MCP多い環境で大 |
| 大ファイルRead | 1万行のファイルを丸ごと読む→1行しか使わなくても全行分消費 | 大きいプロジェクトで大 |
| Auto-compact循環 | 閾値到達→要約→再構築→すぐまた閾値→繰り返し | 長時間セッションで大 |
| サブエージェント | Agent1回で新しいコンテキスト窓が作られる | 乱用すると大 |

プロンプトキャッシュの無効化が最も破壊的だ。キャッシュ読み取り率が89%→4%に暴落した実例がある。防御hookは後の章（「10のやらかしパターン」のパターン10）で詳しく解説する。

**対策**: プロンプトログhookを仕掛けて「何にトークンが消えているか」を可視化する。

```bash
#!/bin/bash
# ~/.claude/hooks/prompt-usage-logger.sh
# UserPromptSubmitに登録して全プロンプトを記録
INPUT=$(cat)
PROMPT=$(printf '%s' "$INPUT" | jq -r '.prompt // empty' 2>/dev/null | head -c 100)
TOOLS=$(printf '%s' "$INPUT" | jq -r '.tool_name // empty' 2>/dev/null)
printf '%s prompt=%s tool=%s\n' "$(date -u +%H:%M:%S)" "$PROMPT" "$TOOLS" \
  >> /tmp/claude-token-log.txt
exit 0
```

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [{ "type": "command", "command": "bash ~/.claude/hooks/prompt-usage-logger.sh" }]
      }
    ]
  }
}
```

セッション後に `/tmp/claude-token-log.txt` を確認すると、どの操作がどの頻度で行われているかが見える。usage dashboardと突き合わせれば、消費の多いパターンを特定できる。

**すぐにできる節約策**:

```bash
# 1. Tool Searchを無効化（MCP多い場合に効果大）
# settings.jsonの"env"に設定する:
#   "env": { "ENABLE_TOOL_SEARCH": "false" }

# 2. 手動/compactを閾値到達前に実行（auto-compactより安い）
# 3. plan modeで計画→実装（trial-and-errorより総コスト低い）
# 4. 大ファイルのReadにはoffsetとlimitを指定
```

**Notificationフック**で「auto-compactが何回起きているか」を把握する:

```bash
#!/bin/bash
# ~/.claude/hooks/compact-alert.sh → Notificationに登録
INPUT=$(cat)
MSG=$(printf '%s' "$INPUT" | jq -r '.message // empty' 2>/dev/null)
if printf '%s' "$MSG" | grep -qi "compact"; then
  printf '⚠ Compaction発生 — 手動/compactか/clearを検討\n' >&2
fi
exit 0
```

1セッションで3回以上auto-compactが発火しているなら、作業を分割するか `/clear` で新しく始めた方がトータルのトークン消費は少ない。

---

## Monitoring を修正した後

```
  ▸ Monitoring
    [PASS] Context window usage monitored with alerts
    [PASS] Activity logging tracks commands
    [PASS] Daily summaries are generated
    [PASS] Token consumption tracked and visible
    100%
```

4hookで4チェック全部カバーできる。最もコスパが高いカテゴリかもしれない。

コンテキスト監視は `npx cc-safe-setup` に含まれている。75%→85%→93%→97%消費の4段階警告で、コンテキスト枯渇前に対処できる。トークン消費ログは、使い始めて1日分溜まれば「どの操作が高コストか」が見えてくる。

---

次章: Recovery——ファイル消失から5分で復旧する仕組み
