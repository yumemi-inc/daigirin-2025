---
class: content
---

<div class="doc-header">
  <h1>ExecuTorch を使って iOS 上で Llama モデルを動かしてみた</h1>
  <div class="doc-author">栗山徹</div>
</div>

ExecuTorch を使って iOS 上で Llama モデルを動かしてみた
==

## はじめに

本章では、 "ExecuTorch" という、エッジ AI を実現するためのライブラリを題材に、エッジ AI や ExecuTorch の紹介、さらには ExecuTorch 付属のサンプルコードうち、 "Llama" という Meta が開発している LLM のモデルを iOS アプリで動かすサンプルを実行する方法について解説します。

なお、本章の記事は 2025 年 5 月 5 日時点の情報を元に執筆しました。

## エッジ AI について

## ExecuTorch について

## Llama について

## ビルド環境を構築する

それでは、サンプルアプリをビルドするために必要な環境の構築を始めましょう。 PyTorch のサイト内にある ExecuTorch のドキュメントの中に "Setting Up ExecuTorch" [^1]というドキュメントがあるので、こちらの手順に従って環境を構築していきます。

### 執筆にあたり使用した環境について

執筆にあたり、下記環境を用いて検証を行いました。

#### ビルド環境

- MBP Pro (M2 Pro CPU, メモリ 16GB)
- macOS Sequoia (15.4)

#### 実行環境
  
##### iOS

- iPhone 13 Pro Max (ストレージ 256 GB, メモリ 6 GB)
- iOS 18.3

##### Android

- Google Pixel 6a (ストレージ 128 GB, メモリ 8 GB)
- Android 13

#### ソフトウェア

筆者の手元で使用したソフトウェアは下記のとおりです。

- Xcode 16.2
- g++ 16.0.0 (Xcode に含まれる g++ を使用)
- asdf
- Python 3.12.0

### Anaconda　について

"Setting Up ExecuTorch" のドキュメント内では Anaconda (conda) の使用が推奨されていますが、商用利用の場合に有料となることから、本章では Anaconda を使わずに環境を構築します。

### 環境構築手順

では、 GitHub ソースコードを Clone してビルドしましょう[^2]。以下のコマンドを実行してください。 "Setting Up ExecuTorch" では `release/0.5` ブランチを使用していましたが、 4/20 現在、 `release/0.6` が release candidate 状態になっていたので、本章ではこちらを利用します。

```shell
git clone --branch release/0.6 https://github.com/pytorch/executorch.git
cd executorch
```

次に、リポジトリ内で利用しているサブモジュール関連の設定を行います。

```shell
git submodule sync
git submodule update --init
```

それでは、インストールスクリプトを実行しましょう。以下のコマンドを実行してください。

```shell
./install_executorch.sh --pybind xnnpack mps coreml
```

コマンドが正常終了すればインストール成功です。

なお、 `--pybind` オプションですが、 ExecuTorch 実行時のバックエンドの設定となります。

[^1]: https://pytorch.org/executorch/stable/getting-started-setup.html

[^2]: pip コマンドを用いてインストール ( `pip install executorch` ) することも可能ですが、 Llama のモデル変換を行う必要があることから、今回はソースコードからビルドしてインストールすることにしました。

### サンプルコードを実行する

では、動作確認を兼ねてサンプルコードを実行しましょう。

```shell
mkdir -p ../example_files
cd ../example_files
touch export_add.py
```

```python
import torch
from torch.export import export
from executorch.exir import to_edge

# Start with a PyTorch model that adds two input tensors (matrices)
class Add(torch.nn.Module):
  def __init__(self):
    super(Add, self).__init__()

  def forward(self, x: torch.Tensor, y: torch.Tensor):
      return x + y

# 1. torch.export: Defines the program with the ATen operator set.
aten_dialect = export(Add(), (torch.ones(1), torch.ones(1)))

# 2. to_edge: Make optimizations for Edge devices
edge_program = to_edge(aten_dialect)

# 3. to_executorch: Convert the graph to an ExecuTorch program
executorch_program = edge_program.to_executorch()

# 4. Save the compiled .pte program
with open("add.pte", "wb") as file:
    file.write(executorch_program.buffer)
```

```shell
python3 export_add.py
```

```shell

```


## Llama のモデルを ExecuTorch 用のモデルへ変換する

## サンプルアプリを実行してみる

## まとめ

<hr class="page-break" />
