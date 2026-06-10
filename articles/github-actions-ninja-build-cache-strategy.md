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

