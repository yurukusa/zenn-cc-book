# zenn-cc-book

Zenn Book「Claude Codeを本番品質にする——108時間の自律稼働から生まれた実践ガイド」のリポジトリ。

## ぐらすへ: GitHubリポジトリ作成 + push手順（5ステップ）

### Step 1: GitHubでリポジトリを作成する

1. https://github.com/new を開く
2. リポジトリ名: `zenn-cc-book`（または好きな名前）
3. Public（Zenn連携には必須）
4. README, .gitignoreは追加しない（既にある）
5. 「Create repository」をクリック

### Step 2: リモートを追加してpushする

```bash
cd ~/projects/zenn-cc-book
git remote add origin https://github.com/yurukusa/zenn-cc-book.git
git push -u origin main
```

※ `yurukusa` の部分は実際のGitHubユーザー名に変える

### Step 3: ZennとGitHubを連携する

1. https://zenn.dev/dashboard/deploys を開く
2. 「GitHubからデプロイ」→「リポジトリを連携する」
3. 上記で作成した `zenn-cc-book` を選択

### Step 4: Bookとして公開設定する

1. https://zenn.dev/dashboard/books を開く
2. 連携後に `cc-production-guide` が表示される
3. 価格（¥800）・公開設定を確認
4. `config.yaml` の `published: true` に変更してpushすると公開される

### Step 5: 動作確認

- Zennダッシュボードで本の構成が正しく表示されているか確認
- プレビューで各章が読めるか確認
- 価格設定が ¥800 になっているか確認

---

## ファイル構成

```
books/
└── cc-production-guide/
    ├── config.yaml              # Zenn Book設定
    ├── intro.md                 # はじめに（無料公開）
    ├── 1-health-check.md        # 第1章（無料公開）
    ├── 2-safety-guards.md       # 第2章（有料）
    ├── 3-code-quality.md        # 第3章（有料）
    ├── 4-monitoring.md          # 第4章（有料）
    ├── 5-recovery.md            # 第5章（有料）
    ├── 6-autonomy.md            # 第6章（有料）
    ├── 7-coordination.md        # 第7章（有料）
    └── 8-putting-it-together.md # 終章（有料）
```

## 公開タイミング

`config.yaml` の `published: false` を `published: true` に変更してpushすると公開される。
pushはぐらすが手動で行う。
