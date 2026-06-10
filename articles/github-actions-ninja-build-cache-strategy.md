---
title: "GitHub ActionsでGitHub-hosted Runnerの限界を超えてCUDAのNinja Buildを通す方法"
emoji: "🙂‍↕️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

こんにちは。今回は、GitHub ActionsでGitHub-hosted Runnerの6時間の実行時間の限界を超えて
CUDAのNinja Buildを通す方法について説明します。
まずは、この方法が必要になった背景と、GitHub-hosted Runnerの制限、
Ninja BuildのCacheの仕組みについて説明し、
その後、どのようにして長時間のビルドをGitHub Actionsで成功させたかについて説明します。

## 背景

私のGitHubのリポジトリの1つとして、[mjun0812/flash-attention-prebuild-wheels](https://github.com/mjun0812/flash-attention-prebuild-wheels)というリポジトリがあります。

https://github.com/mjun0812/flash-attention-prebuild-wheels

flash-attentionは、TransformerのAttention機構を高速化するためのCUDAカーネルを提供するライブラリで、
非常に多くのLLMや画像生成モデルで利用されています。

しかし、このライブラリは、Python, PyTorch, CUDAのバージョンや
Windows/Linuxやx86/arm64などのプラットフォームの組み合わせごとにビルドする必要がある上に、
CUDAカーネルをビルドするため、非常にリソースと時間がかかります。

そこで、私のリポジトリである[mjun0812/flash-attention-prebuild-wheels](https://github.com/mjun0812/flash-attention-prebuild-wheels)では、
GitHub Actionsを利用して、複数のプラットフォーム向けのwheelファイルをビルドして配布しています。
これによって、ユーザーは自分の環境に合ったwheelファイルをダウンロードしてインストールするだけで、
flash-attentionを利用できるようになり、苦痛と時間を伴うビルドプロセスを回避できます。

https://x.com/mjun0812/status/1850706663982137615?s=20

非常にニッチな問題を解決するためのリポジトリですが、
2026/6/10時点で、1.5kのスターと700万以上のダウンロードがあり、多くのユーザーに利用されています。

## 課題

1. 6h cap で cancel された「キャッシュなし時代」の失敗例

Run 26828098662 (PR #114, ARM64 FA2 GitHub-hosted, 2026-05-02)
https://github.com/mjun0812/flash-attention-prebuild-wheels/actions/runs/26828098662

- 24 ジョブのうち 1 つが 6:00:16 cancelled ← GitHub の 6h ハードリミット直撃
- 周辺ジョブは 5h30m〜5h59m で完走、限界ギリギリの実例として象徴的

2. timeout --signal=TERM 345m 導入後の attempt 1 失敗例（cap内で意図的に止めた）

Run 26864061027 (PR #120, 2026-06-03)
https://github.com/mjun0812/flash-attention-prebuild-wheels/actions/runs/26864061027

- [5:49:17] failure — 5h45m で TERM、cache save 成功、attempt 1 で意図的に失敗させた最初の動作例

3. キャッシュ戦略が完成した「失敗→rerun→成功」のセット例 ← 記事の主役

Run 27098881346 (v0.9.40 release, 2026-06-07)
https://github.com/mjun0812/flash-attention-prebuild-wheels/actions/runs/27098881346

- Attempt 1: 5:48:42 failure × 2 (cap で TERM、cache save)
- Attempt 2: 1:23:36 success × 2 (cache restore → 完走)
- 同一 run ID で attempt 1/2 を見比べられるのが強い

attempt 個別 URL:
- Attempt 1 (失敗): https://github.com/mjun0812/flash-attention-prebuild-wheels/actions/runs/27098881346/attempts/1
- Attempt 2 (成功): https://github.com/mjun0812/flash-attention-prebuild-wheels/actions/runs/27098881346/attempts/2

4. self-hosted で 17h かけて完走した例（24h超ではないが長時間例）

Run 24447352042 (2026-04-15, ARM64 self-hosted)
https://github.com/mjun0812/flash-attention-prebuild-wheels/actions/runs/24447352042

- [17:02:56] success — Linux ARM64 self-hosted FA3 build
- GitHub の self-hosted 上限は 5日なので 6h cap がない、という対比例

GitHub Actions の 6h 上限に収まらない FA3 ARM64 ビルドを、ninja の build dir を actions/cache@v4 で attempt 間に持ち越して解決した。5h45m で timeout --signal=TERM、build dir を save → rerun の attempt 2 で restore して差分だけ追加コンパイルする戦略。

ninja は .ninja_deps (binary) に .o と依存 header の mtime を nanosecond 精度で記録し「記録 mtime と一致するか」「output > input か」で差分判定するため、attempt 跨ぎで噛み合わない。対策:

1. ninja は nanosecond mtime で照合するが、tar は ustar で秒精度に落とすので、save 前に build dir の mtime を整数秒に truncate する
2. ninja は .o の mtime が記録とズレると全 rebuild するので、restore 後の .o は絶対に touch しない
3. cutlass / torch / CUDA include は cache 対象外で毎回 fresh インストールされ「header > .o」で全 rebuild されるので、1970-01-02 に touch して .o より古くする
4. ヘッダー mtime も .ninja_deps に入っており単に古くするだけでは記録と不一致になるので、restore 後に .ninja_deps を削除し per-target の .o.d (gcc-style depfile) から graph を再構築させる