---
title: "uvでPyTorchのCPU / CUDAバージョンを環境ごとに管理する その2"
emoji: "😗"
type: "tech"
topics:
  - "python"
  - "pytorch"
  - "uv"
published: true
published_at: "2024-10-22 02:08"
---

こんにちは。今回は前回の記事に引き続き、Pythonのパッケージマネージャのuvを使ってPyTorchをインストールする方法について紹介します。

前回の記事

<https://zenn.dev/mjun0812/articles/b22adf3fab1fdd>

uv v0.4.23のアップデートで、1つのパッケージについて複数のindex-urlを指定する機能が追加されました。

<https://github.com/astral-sh/uv/releases/tag/0.4.23>

この機能を使うと、環境/パッケージごとに参照するindex-urlを変更できるため、より確実にPyTorchのインストールが行えます。  

:::message alert
今回わざわざこの記事を書いた理由は、[前回](https://zenn.dev/mjun0812/articles/b22adf3fab1fdd)紹介した方法が、
CUDA 12.4の場合に使用できなくなってしまったためです。
CUDA 12.4以降のPyTorchを使う場合は本記事に書いてある手法を使ってください。
:::

まずは、uvをv0.4.23以降にアップデートしましょう。以下のコマンドでアップデートできます。

```bash
uv self update
```

続いて、uvの依存関係を書いたpyproject.tomlを以下のように書きます。
今回はmacOS/aarch64のLinuxとx86_64のLinuxでCPU版とCUDA版のPyTorchを切り替える場合です。

```toml
[project]
name = "new-uv"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.11"
dependencies = [
    "torch==2.5.0+cu124; sys_platform == 'linux' and platform_machine == 'x86_64'",
    "torch==2.5.0; sys_platform == 'darwin' or (sys_platform == 'linux' and platform_machine == 'aarch64')",
]

[[tool.uv.index]]
name = "torch-cuda"
url = "https://download.pytorch.org/whl/cu124"
explicit = true

[[tool.uv.index]]
name = "torch-cpu"
url = "https://download.pytorch.org/whl/cpu"
explicit = true

[tool.uv.sources]
torch = [
    { index = "torch-cuda", marker = "sys_platform == 'linux' and platform_machine == 'x86_64'"},
    { index = "torch-cpu", marker = "sys_platform == 'darwin' or (sys_platform == 'linux' and platform_machine == 'aarch64')"},
]
```

順番に見ていきます。

まずは依存関係を書くdependenciesにenvironment marker([PEP508](https://peps.python.org/pep-0508/))を使って、環境ごとにtorchの依存関係を書きます。

```toml
dependencies = [
    "torch==2.5.0+cu124; sys_platform == 'linux' and platform_machine == 'x86_64'",
    "torch==2.5.0; sys_platform == 'darwin' or (sys_platform == 'linux' and platform_machine == 'aarch64')",
]
```

次に、CPU版とCUDA版のPyTorchのindex-urlを設定します。  
この時、`explicit = true`を忘れないようにしてください。explicitオプションの説明は以下です。

> The explicit flag is optional and indicates that the index should only be used for packages that explicitly specify it in tool.uv.sources. If explicit is not set, other packages may be resolved from the index, if not found elsewhere.  
> <https://docs.astral.sh/uv/concepts/dependencies/#index> より引用

要するに、indexで指定したurlが他のパッケージを探す際に使用されないように制限するオプションです。
今回のindex-urlはtorchのインストールにしか使わないですし、現在のアーキテクチャに対応していないパッケージを参照してエラーを起こしてしまうため、
忘れないようにしましょう。

```toml
[[tool.uv.index]]
name = "torch-cuda"
url = "https://download.pytorch.org/whl/cu124"
explicit = true

[[tool.uv.index]]
name = "torch-cpu"
url = "https://download.pytorch.org/whl/cpu"
explicit = true
```

次に、torchのインストールをindex-urlから行えるようにしましょう。
ここでもenvironment-markerを使って、環境ごとにindex-urlを設定します。
`index = "torch-cpu"`では、先ほど定義した`[[tool.uv.index]]`のnameを指定しています。

```toml
[tool.uv.sources]
torch = [
    { index = "torch-cuda", marker = "sys_platform == 'linux' and platform_machine == 'x86_64'"},
    { index = "torch-cpu", marker = "sys_platform == 'darwin' or (sys_platform == 'linux' and platform_machine == 'aarch64')"},
]
```

この`pyproject.toml`をもとに`uv sync`をすると、macOSでもLinuxでも適切なPyTorchがインストールできます。

ちなみに、`torchvision`をインストールする際のサンプルは以下です。

```toml
[project]
name = "new-uv"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.11"
dependencies = [
    "torch==2.4.0+cu124; sys_platform == 'linux' and platform_machine == 'x86_64'",
    "torch==2.4.0; sys_platform == 'darwin' or ( sys_platform == 'linux' and platform_machine == 'aarch64')",
    "torchvision==0.19.0+cu124; sys_platform == 'linux' and platform_machine == 'x86_64'",
    "torchvision==0.19.0; sys_platform == 'darwin' or ( sys_platform == 'linux' and platform_machine == 'aarch64')",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.uv.sources]
torch = [
    { index = "torch-cuda", marker = "sys_platform == 'linux' and platform_machine == 'x86_64'"},
    { index = "torch-cpu", marker = "sys_platform == 'darwin' or ( sys_platform == 'linux' and platform_machine == 'aarch64')"},
]
torchvision = [
    { index = "torch-cuda", marker = "sys_platform == 'linux' and platform_machine == 'x86_64'"},
    { index = "torch-cpu", marker = "sys_platform == 'darwin' or ( sys_platform == 'linux' and platform_machine == 'aarch64')"},
]

[[tool.uv.index]]
name = "torch-cuda"
url = "https://download.pytorch.org/whl/cu124"
explicit = true

[[tool.uv.index]]
name = "torch-cpu"
url = "https://download.pytorch.org/whl/cpu"
explicit = true
```

以上、uvでのmultiple pinned indexを使ったPyTorchのインストール方法の紹介でした。  
前回のfind-linksを使った方法と比較すると、pyproject.tomlに書く文量が長くなってしまいましたが、explicitを組み合わせてパッケージ固有のindex-urlを指定できるようになったので、より安全かつ効率的にインストール可能になりました。
