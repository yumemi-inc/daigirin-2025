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

本稿では、 ExecuTorch という、エッジ AI を実現するためのライブラリを題材に、エッジ AI や ExecuTorch の紹介、さらには ExecuTorch 付属のサンプルコードうち、 Llama という Meta が開発している大規模言語モデル (LLM) を iOS デバイス上で動作させるサンプルを実行する方法について解説します。

筆者自身、iOS アプリにおける機械学習機能の組み込みには以前から興味はありましたが、実業務ではそこまで縁がありませんでした。したがって、本稿は iOS アプリ開発者、かつ機械学習初心者が iOS アプリに AI 機能を組み込む観点で執筆した記事となります。

なお、本稿は 2025 年 4 月 30 日時点の情報を元に執筆しました。

## エッジ AI について

**エッジAI** とは、エッジデバイスと呼ばれる、モバイル端末などを含むネットワークにつながる端末で行う AI 関連処理 (学習・推論) のことです[^1]。反対に、クラウド上のコンピューティングリソースを利用して AI 処理を行うことを **クラウドAI**　と呼びます[^2]。

エッジ AI はエッジデバイスの高性能化に伴い注目されるようになりました。エッジ AI の代表的な事例としては自動運転技術が挙げられます。

[^1]: https://www.ntt.com/bizon/glossary/j-a/edge-ai.html

[^2]: https://business.ntt-east.co.jp/content/cloudsolution/column-240.html

### エッジ AI のメリット・デメリット

エッジ AI には後述するようなメリット・デメリットがあるため、実現したいことに対してエッジ AI が適しているかどうかについては慎重に見極める必要があります。

エッジ AI のメリットとしては、ネットワークを介したサーバーへの情報送信を必要とせず端末内で AI 処理が完結するため、リアルタイム処理に向いていることやセキュリティ面でのメリットが挙げられます。特にセキュリティ面については、クラウド AI 系のサービスは送信した情報がどのように使われているのかが見えにくい点や情報漏洩のリスクがありますが、エッジ AI であればそもそもサーバーへデータを送信する必要が無いことからより安全に AI 機能を利用できるメリットがあります。

一方、エッジ AI のデメリットとしては、エッジデバイスは性能に限界があり、高度な AI 処理を実行するのには向いていないことや、エッジ AI で使用するモデル自体の漏洩リスクが挙げられます。エッジ AI ではエッジデバイスの性能限界もあり、学習済みモデルを組み込んで推論のみ実行するケースがほとんどです。また、 ChatGPT といった生成 AI 系のサービスは一般的には Web サービスとして提供されますが、これはモデルの漏洩リスクを防ぐのはもちろんのこと、生成 AI の推論処理を行うためには大量のコンピュータリソースを必要とするため、Web サービスとしての提供形態は理に適っているといえます。

エッジデバイスの制約により、エッジ AI では実現が難しいことはまだ多く存在します。一方で、エッジデバイスの性能は日々進化しており、特に GPU や Neural Engine の進化は目覚ましいものがあります。今後の性能向上により、より多くの AI 処理がエッジ AI で実現できるようになっていくのではないでしょうか。

### スマホアプリにおけるエッジ AI の活用事例

エッジ AI をスマホアプリに導入することで、ユニークなユーザー体験を実現することができます。ここでは、スマホアプリにおけるエッジ AI の活用実例として、 Apple Intelligence [^3] や Sansan のリアルタイム名刺認識技術[^4]について紹介します。

Apple Intelligence は、iPhone / iPad / macOS 上で生成 AI を活用した文章作成や画像生成を可能にする技術です。 エッジ AI のデメリットであるエッジデバイスの性能限界を Apple の AI サーバーや ChatGPT へシームレスに接続することで補っており、エッジ AI をスマホアプリに組み込む上で参考になる点が多々あるのではないでしょうか。

![Image Playground (画像生成 AI)](./images_kotetu/figure-image-playground.png "Image Playground (画像生成 AI)")

一方、 Sansan のリアルタイム名刺認識技術は、スマホのカメラを使って名刺を撮影する際の名刺領域抽出を Core ML を使うことでリアルタイムな名刺領域抽出を実現しました。リアルタイム画像認識については、エッジ AI のメリットが特に活きる分野ではないでしょうか。

[^3]: https://www.apple.com/jp/apple-intelligence/

[^4]: https://buildersbox.corp-sansan.com/entry/2022/11/01/110000

## PyTorch と ExecuTorch

エッジ AI の実現手法はいくつかありますが、本稿では ExecuTorch というライブラリに着目しました。ここでは、 ExecuTorch や元となったライブラリである PyTorch について紹介します。

### PyTorch について

**PyTorch** [^5] はオープンソースの機械学習ライブラリです。 PyTorch は TensorFlow [^6]と並んで機械学習機能の実装によく使われるライブラリのひとつです。名前のとおり Python 向けのライブラリですが、パフォーマンス向上を目的に C++ で実装されている箇所が多数あります。GPU を利用した処理の高速化や機械学習に便利な機能セットにより、多くの研究者や開発者に使用されています。

[^5]: https://pytorch.org/

[^6]: https://www.tensorflow.org/

### ExecuTorch について

**ExecuTorch** [^7] はエッジデバイス上での推論処理を実現するために開発されたライブラリです。 ExecuTorch を利用することで、既存の PyTorch モデルをエッジデバイス向け最適化された形で導入することが可能です。

ExecuTorch の特徴のひとつとして、エッジデバイスのハードウェア機能を活用したパフォーマンス向上が挙げられます。 ExecuTorch では推論を行う際に使用するバックエンドを選択することができ、iOS では Metal Performance Shader (MPS) および CoreML を選択でき、 Android では Vulkan を選択できます。モバイル端末に搭載されている GPU や Neural Engine を活用できることは、推論処理を行う上で大きなメリットといえるのではないでしょうか。

[^7]: https://pytorch.org/executorch-overview

#### PyTorch で生成された学習済みモデルを ExecuTorch で使用するには

ExecuTorch はエッジデバイス用に最適化されているため、PyTorch で生成された学習済みモデル (PyTorch モデル) を ExecuTorch でそのまま実行できないことに注意してください。

"How ExecuTorch Works"[^8] という ExecuTorch の内部構造を解説したドキュメントによれば、PyTorch モデルを実行するためには下記 3 つのステップが必要と解説されています。

1. PyTorch モデルをエッジデバイスでの実行に最適化してエクスポートする
2. エクスポートしたモデルを ExecuTorch 用のモデル (ExecuTorch モデル) としてコンパイルする
3. ExecuTorch ランタイムを用いて ExecuTorch モデルをインポートし推論処理を実行する

本稿で題材となっている Llama については 1. と 2. のステップをコマンド 1 つで処理できるツールが ExecuTorch のリポジトリに含まれており、 Python のコードを書くことなく容易に ExecuTorch モデルを出力できるようになっています。

[^8]: https://pytorch.org/executorch/stable/intro-how-it-works.html

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
