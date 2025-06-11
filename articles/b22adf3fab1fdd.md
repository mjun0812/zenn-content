---
title: "uvでPyTorchのCPU / CUDAバージョンを環境ごとに管理する"
emoji: "😚"
type: "tech"
topics:
  - "python"
  - "pytorch"
  - "uv"
published: true
published_at: "2024-08-22 02:22"
---

こんにちは。今回はPythonのパッケージマネージャのuvで、PyTorchのCPU/CUDAのバージョンをLinuxやmacOSなどの環境に応じて切り替えてインストールする方法を紹介します。

:::message alert
uv v0.4.23でindex-urlを複数指定してパッケージをインストールする方法が追加されました。
以下の新しい記事で紹介しています。
この記事で紹介している方法だと、CUDA 12.4以降は動作しない場合があるため、
以下の新しい記事を参考にしてください。
:::

uv v0.4.23以降の新しい記事

<https://zenn.dev/mjun0812/articles/b32f870bb3cdbf>

## はじめに

先日リリースされたuvのv0.30.0で、Ryeで実装されていたようなPython自体の管理もできるようになりました。

<https://astral.sh/blog/uv-unified-python-packaging>

<https://nikkie-ftnext.hatenablog.com/entry/uv-0.3.0-release-awesome-future-python-project-package-manager>

<https://zenn.dev/turing_motors/articles/594fbef42a36ee>

コマンド体系はRyeとほぼ同じで、以下のような感じで好きなバージョンのPython環境を構築することができます。

```bash
uv python install 3.11
uv python pin 3.10
```

poetryではMulti Platformを考慮した依存関係解決とパッケージインストールが可能ですが、依存関係の解決が遅く、Ryeでは依存関係の解決はRust実装のおかげで高速なものの、特定のマシンの環境に依存したrequirements.lockが作成されてしまい、複数の環境に跨ったプロジェクトの管理がしづらいという問題がありました。
そこでpoetryと同じようにMulti Platformを考慮可能なuvがRyeのPython管理機能を吸収してくれたことで、両方の利用者が問題なく移行できるのではないかと思います。

この機能追加でpoetryやRyeで管理していたプロジェクトを移行しようと思い、PyTorchのインストール方法を試行錯誤していました。

<https://zenn.dev/mjun0812/scraps/671db64dc42ffa>

そこで、poetryのenvironment markerと同じような方法で、環境によってインストールするPyTorchを切り替える方法を発見したので共有します。

<https://zenn.dev/nakakiiro/articles/3cc5f4080a7a09>

## 成果物

まず最初に完成した`pyproject.toml`を以下に示します。

```toml
[project]
name = "new-uv"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.11"
dependencies = [
    "torch==2.4.0+cu118; sys_platform == 'linux' and platform_machine == 'x86_64'",
    "torch==2.4.0; sys_platform == 'darwin' or (sys_platform == 'linux' and platform_machine == 'aarch64')",
    "torchvision==0.19.0+cu118; sys_platform == 'linux' and platform_machine == 'x86_64'",
    "torchvision==0.19.0; sys_platform == 'darwin' or (sys_platform == 'linux' and platform_machine == 'aarch64')",
    "numpy<2.0.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.uv]
find-links = [
    "https://download.pytorch.org/whl/cu118/torch",
    "https://download.pytorch.org/whl/cu118/torchvision",
]
```

上記のpyproject.tomlは以下の環境で検証しています。

- macOS 14.6 Sonoma, arm64, M1 MacBook Air 2020
- Ubuntu 24.04, aarch64, Dockerコンテナ(上記のmacOS上)
- Ubuntu 22.04, x86_64, Ryzen 5700+RTX3090の自作PC

## 解説

uvでは[PEP508](https://peps.python.org/pep-0508/)に基づいたパッケージの依存関係の記述が可能です。

<https://docs.astral.sh/uv/concepts/dependencies/#pep-508>

この記述方法の中にEnvironment Markerという機能があり、システムの環境ごとに依存関係を記述することができます。
上記のpyproject.tomlの該当箇所は以下です。

```toml
"torch==2.4.0+cu118; sys_platform == 'linux' and platform_machine == 'x86_64'",
"torch==2.4.0; sys_platform == 'darwin' or (sys_platform == 'linux' and platform_machine == 'aarch64')",
```

この記述では、

- Linux + x86_64アーキテクチャの時はCUDA 11.8対応のPyTorch
- macOS + arm64(Apple Silicon Mac)とLinux + aarch64の時はcpu版のPyTorch
がインストールされます。

また、PyTorchのバージョン表記`2.4.0+cu118`の`+cu118`の部分はLocal Versionと呼ばれており、uvでLocal Versionを使う時はパッケージのバージョン指定に`==`しか使えないことに注意して下さい。(`>=`は使えない)

<https://docs.astral.sh/uv/pip/compatibility/#local-version-identifiers>

そして、以下でも言及されていますが、macOSでtorchをインストールする場合は`torch==2.4.0+cpu`ではなく、`torch==2.4.0`と記述しないとインストールできません。
これは今後改善されるかもしれないです。

<https://github.com/astral-sh/uv/issues/5182>

以上、uvでのPyTorchのインストール方法の紹介でした。uvはPythonの管理機能が追加されて、隙が無くなった印象です。今後注目していきたいと思います。
