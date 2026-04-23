---
title: "Claude Codeに作業させてたら`.git`が144GBに膨らんでWSLが起動しなくなった話"
emoji: "💣"
type: "tech"
topics: ["claudecode", "wsl", "git", "ai"]
published: true
---

Claude Codeに「好きにやっていいよ」と自律稼働させていたら、ある日WSLのUbuntuが起動しなくなった。

原因を辿ると、`.git/objects/pack/` に **`tmp_pack_*` ファイルが1,350個、合計144GB** 溜まっていた。`.git` ディレクトリ単体で144GB。WSL2の仮想ディスク（vhdx）は74GB → 208GBに膨張し、Cドライブを圧迫して最終的にI/Oエラーでubuntuが開けなくなった。

この記事ではその原因特定と解決手順、再発防止策をまとめる。

## TL;DR

- **症状**: WSL Ubuntuが突然起動しなくなる。PowerShellで `getpwnam() failed`、`/etc/default/locale` 読めず、などのエラー
- **実際の原因**: Claude Codeのサブエージェントが並列で `git gc` / `git repack` を実行し、ロック競合で完走しなかった一時packファイル（`tmp_pack_*`）が蓄積し続けていた
- **解決**: `find .git/objects/pack/ -name 'tmp_pack_*' -delete` → `sudo fstrim /` → PowerShellで `diskpart` の `compact vdisk`（または `Optimize-VHD -Mode Full`）
- **再発防止**: サブエージェントの並列git操作を直列化、または定期クリーンアップをcronに仕込む

## 環境

- Windows 10 + WSL2 Ubuntu 24.04
- Claude Code（自律稼働ループ `cc-loop` で複数サブエージェントが並列稼働）
- ホームディレクトリが git 管理されている

## 症状1: Cドライブが減り続ける

最初の異変はWindowsのCドライブだった。

エクスプローラーで確認すると、前は余裕があったCドライブが逼迫している。ファイル単位で大きい順に探すと、犯人はすぐ見つかった。

```
C:\Users\<user>\AppData\Local\wsl\<GUID>\ext4.vhdx   208GB
```

WSL2の仮想ディスクイメージだ。前回確認したときは74GBだったので、**134GB増えている**。

## 症状2: Ubuntuが起動しない

`wsl` コマンドを叩くと、こんなエラーが出る。

```
<3>WSL (xxxxx - Relay) ERROR: CreateProcessParseCommon:996: getpwnam(namakusa) failed 5
<3>WSL (xxxxx - Relay) ERROR: CreateProcessParseCommon:1005: getpwuid(1000) failed 5
<3>WSL (xxxxx - Relay) ERROR: ConfigUpdateLanguage:2553: fopen(/etc/default/locale) failed 5
<3>WSL (xxxxx) ERROR: I/O error @util.cpp:1358 (UtilInitGroups)
```

`/etc/passwd` や `/etc/default/locale` といった基本的なファイルすら読めない。I/Oエラー。ファイルシステムが壊れている可能性を疑った。

### 一次復旧

まず起動できる状態に戻す。

```powershell
wsl --shutdown
wsl
```

一度シャットダウンしてから起動し直すと、今回は開けるようになった。これで開けない場合は、vhdxを `diskpart` で圧縮してから再試行する（後述）。

## 偽の犯人たち

開けるようになったので、Claude Codeに「10時間以内に追加された大きなファイルを探せ」と頼んだ。候補はいくつも出た。

| 候補 | サイズ | 判定 |
|---|---|---|
| `~/.cache/huggingface/` の未使用モデル | 約7GB | 犯人ではない（普通の肥大） |
| `~/.cache/pip/http-v2/` | 4.5GB | 犯人ではない |
| `~/.vod-search/*.wav.bak` | 1.9GB | 犯人ではない |
| `logs/cdp-bridge-9222.log` | 607MB | 犯人ではない |
| Windows側のクラッシュダンプ（node, tailscaled） | 1.2GB | 犯人ではない |

全部合わせても20GB程度。**134GBの膨張を説明できない。**

「他に探せる場所はないか」と追加で指示したら、Claude Codeがホームディレクトリ直下の `.git/` を見つけた。

```
~/.git/   144GB
```

ホームディレクトリを git 管理していて、その `.git` が単体で144GBになっていた。

## 真の犯人: `tmp_pack_*` が1,350個

`.git/objects/pack/` の中身を確認。

```
$ ls .git/objects/pack/ | head
pack-abc123....pack
pack-abc123....idx
tmp_pack_xxxxxx
tmp_pack_xxxxxx
tmp_pack_xxxxxx
...

$ ls .git/objects/pack/tmp_pack_* | wc -l
1350

$ du -sh .git/objects/pack/tmp_pack_*  # 1ファイルあたり100〜145MB
```

**`tmp_pack_*` が1,350個、合計144GB。**

正規の `.pack` / `.idx` は別に2本あり、リポジトリ自体は健全。`tmp_pack_*` は全て孤立した一時ファイルだった。

### なぜ溜まったか

Gitは `git gc` や `git repack` を実行するとき、新しいpackファイルを `tmp_pack_*` という名前で作り、完成したら正式名（`pack-<hash>.pack`）にrenameする。完成前に失敗すると `tmp_pack_*` が残骸として残る。

Claude Codeの自律稼働ループでは、サブエージェントが並列で動き、それぞれがファイルを生成した後にコミット処理を走らせる。同じリポジトリに対して複数のサブエージェントが同時に `git gc` / `git repack` を叩くと、**ロック競合で片方が途中で失敗し、`tmp_pack_*` だけ残して死ぬ**。

Gitには古い `tmp_pack_*` を自動クリーンアップする仕組みはない（少なくともデフォルトでは）。溜まり続けて144GBに到達した。

## 解決手順

### 1. `tmp_pack_*` を削除

```bash
find .git/objects/pack/ -name 'tmp_pack_*' -delete
```

`rm tmp_pack_*` でもいいが、ファイル数が多いと `argument list too long` で失敗する。`find -delete` のほうが確実。

:::message alert
`.git` 配下を触るのは怖いので、実行前に以下を確認すること。

- 正規の `pack-<hash>.pack` と `.idx` が存在すること
- 削除対象が `tmp_pack_*` だけで、`pack-*` を含まないこと
- できれば `.git` 全体をバックアップしてから実行

Claude Codeに任せる場合、`.git` 配下の `rm` がhookで保護されていれば弾かれる（`destructive-guard.sh` 系）。その場合は手動でやる必要がある。
:::

削除後、`.git/` は144GB → 813MBまで縮んだ。

### 2. ext4ファイルシステムのtrim

WSL側でファイルを削除しても、Windows側のvhdxファイルのサイズは自動では縮まない。まずext4にtrimを通す。

```bash
sudo fstrim /
```

これでext4の未使用ブロックが解放される、**はず**だが、この記事の筆者環境では fstrim 単体ではvhdxのサイズは変わらなかった。

### 3. PowerShellでvhdxを圧縮

`fstrim` だけで縮まない場合、PowerShellから明示的に縮める。

```powershell
# WSLを完全停止
wsl --shutdown

# vhdxのパス確認
Get-ChildItem "$env:LOCALAPPDATA\wsl" -Recurse -Filter "ext4.vhdx"

# 圧縮（Hyper-V入っている環境）
Optimize-VHD -Path "C:\Users\<user>\AppData\Local\wsl\<GUID>\ext4.vhdx" -Mode Full
```

`Optimize-VHD` が使えない環境（Windows Homeなど）は `diskpart` で。

```
diskpart
> select vdisk file="C:\Users\<user>\AppData\Local\wsl\<GUID>\ext4.vhdx"
> compact vdisk
> exit
```

これでWindowsのCドライブに空き容量が戻る。

## 再発防止

### 対策1: 定期クリーンアップをcronに仕込む

1時間以上古い `tmp_pack_*` を定期削除する。1時間経っても完成していないpackは、ほぼ確実に失敗している。

```bash
# crontab -e
0 * * * * find ~/.git/objects/pack/ -name 'tmp_pack_*' -mmin +60 -delete 2>/dev/null
```

### 対策2: サブエージェントの並列git操作を制限する

そもそもサブエージェントが自発的に `git gc` / `git repack` を走らせる必要があるかを見直す。多くの場合、ない。

Claude Codeで並列作業する場合、同一リポジトリに対する `git` 書き込みはmutex等で直列化するのが安全。`flock` を使うなら以下のように包む。

```bash
flock ~/.git/index.lock git gc
```

### 対策3: サブエージェントに専用の作業ディレクトリを使わせる

ホームディレクトリ全体を git 管理するのをやめ、サブエージェントは各自の作業ディレクトリだけを触るように分離する。そもそも論としてはこれが正解だが、既存環境の移行コストは高い。

### 対策4: `destructive-guard` 的なhookで `.git` 配下を守る

今回の復旧時、Claude Codeに `rm` を頼んだら `destructive-guard.sh` でブロックされた。`.git` をうっかり消し飛ばす事故を防げるのでhook自体は正しい。ただし**特定ファイル限定で許可する例外パスを用意しておく**と、今回のような「tmp_pack_* だけ消したい」ケースで詰まらない。

```bash
# 例: .git/objects/pack/tmp_pack_* は許可
if [[ "$CMD" =~ \.git/objects/pack/tmp_pack_ ]]; then
    exit 0
fi
```

## おわりに

自律稼働するAIに作業を任せると、人間が意図しない形でリソースが食われる。今回の `tmp_pack_*` 1,350個は「悪意」ではなく、**並列実行というパターンとGit内部の一時ファイル運用が噛み合わなかった結果の副作用**だ。

AIに任せる範囲を広げるほど、こういう「誰も悪意を持たないが誰も掃除しない」ゴミが溜まる。定期クリーンアップと、並列性を前提にした設計が必要になる。

もし自律稼働ループで同じような現象に遭ったら、まず `du -sh ~/.git/` を見てみてほしい。GBオーダーならだいたいこれだ。

## 関連

- Gitのドキュメント: [git-gc(1)](https://git-scm.com/docs/git-gc) — `gc.pruneExpire` で孤立オブジェクトの削除期限を変更できるが、`tmp_pack_*` には効かない
- WSL2のvhdx縮小: [Microsoft Docs — Compact a virtual hard disk](https://learn.microsoft.com/en-us/windows/wsl/disk-space)
- [cc-safe-setup](https://github.com/yurukusa/cc-safe-setup) — `npx cc-safe-setup` で安全hookをインストール。`destructive-guard` が `.git` 配下の破壊操作をブロックする
- [Token Checkup](https://yurukusa.github.io/cc-safe-setup/token-checkup.html) — 無料。自分のClaude Code環境のリスクを30秒で診断
