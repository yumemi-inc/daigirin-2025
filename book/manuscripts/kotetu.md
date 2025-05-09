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

**エッジAI** とは、エッジデバイスと呼ばれる、スマートフォンなどを含むネットワークにつながる端末で行う AI 関連処理 (学習・推論) のことです[^1]。反対に、クラウド上のコンピューティングリソースを利用して AI 処理を行うことを **クラウドAI**　と呼びます[^2]。

エッジ AI はエッジデバイスの高性能化に伴い注目されるようになりました。エッジ AI の代表的な事例としては自動運転技術が挙げられます。

[^1]: https://www.ntt.com/bizon/glossary/j-a/edge-ai.html

[^2]: https://business.ntt-east.co.jp/content/cloudsolution/column-240.html

### エッジ AI のメリット・デメリット

エッジ AI には後述するようなメリット・デメリットがあるため、実現したいことに対してエッジ AI が適しているかどうかについては慎重に見極める必要があります。

エッジ AI のメリットとしては、ネットワークを介したサーバーへの情報送信を必要とせず端末内で AI 処理が完結するため、リアルタイム処理に向いていることやセキュリティ面でのメリットが挙げられます。特にセキュリティ面については、クラウド AI 系のサービスは送信した情報がどのように使われているのかが見えにくい点や情報漏洩のリスクがありますが、エッジ AI であればそもそもサーバーへデータを送信する必要が無いことからより安全に AI 機能を利用できるメリットがあります。

一方、エッジ AI のデメリットとしては、エッジデバイスは性能に限界があり、高度な AI 処理を実行するのには向いていないことや、エッジ AI で使用するモデル自体の漏洩リスクが挙げられます。エッジ AI ではエッジデバイスの性能限界もあり、学習済みモデルを組み込んで推論のみ実行するケースがほとんどです。また、 ChatGPT といった生成 AI 系のサービスは一般的には Web サービスとして提供されますが、これはモデルの漏洩リスクを防ぐのはもちろんのこと、生成 AI の推論処理を行うためには大量のコンピュータリソースを必要とするため、Web サービスとしての提供形態は理に適っているといえます。

エッジデバイスの制約により、エッジ AI では実現が難しいことはまだ多く存在します。一方で、エッジデバイスの性能は日々進化しており、特に GPU や Neural Engine の進化は目覚ましいものがあります。今後の性能向上により、より多くの AI 処理がエッジ AI で実現できるようになっていくのではないでしょうか。

### スマホアプリにおけるエッジ AI の活用事例

エッジ AI をスマホアプリに導入することで、ユニークなユーザー体験を実現することができます。ここでは、スマホアプリにおけるエッジ AI の活用実例として、 Apple Intelligence [^3]や Sansan のリアルタイム名刺認識技術[^4]について紹介します。

Apple Intelligence は、iPhone / iPad / macOS 上で生成 AI を活用した文章作成や画像生成を可能にする技術です。 エッジ AI のデメリットであるエッジデバイスの性能限界を Apple の AI サーバーや ChatGPT へシームレスに接続することで補っており、エッジ AI をスマホアプリに組み込む上で参考になる点が多々あるのではないでしょうか。

![Image Playground (画像生成 AI)](./images_kotetu/figure-image-playground.png "Image Playground (画像生成 AI)")

一方、 Sansan のリアルタイム名刺認識技術は、エッジ AI がスマホアプリのユーザー体験向上に寄与する事例の一つといえるのではないでしょうか。 Sansan アプリや Eight アプリで名刺撮影をする際、机の上に置いた複数枚の名刺をリアルタイムで認識し、撮影時に認識結果を元に写真から名刺を切り出す機能です。リアルタイムでの名刺領域抽出処理を実現するため、 Sansan では Core ML を利用して画像認識を行っています。

[^3]: https://www.apple.com/jp/apple-intelligence/

[^4]: https://buildersbox.corp-sansan.com/entry/2022/11/01/110000

## PyTorch と ExecuTorch

エッジ AI の実現手法はいくつかありますが、本稿では ExecuTorch というライブラリに着目しました。ここでは、 ExecuTorch や元となったライブラリである PyTorch について紹介します。

### PyTorch について

**PyTorch** [^5]はオープンソースの機械学習ライブラリです。 PyTorch は TensorFlow [^6]と並んで機械学習機能の実装によく使われるライブラリのひとつです。名前のとおり Python 向けのライブラリですが、パフォーマンス向上を目的に C++ で実装されている箇所が多数あります。GPU を利用した処理の高速化や機械学習に便利な機能セットにより、多くの研究者や開発者に使用されています。

[^5]: https://pytorch.org/

[^6]: https://www.tensorflow.org/

### ExecuTorch について

**ExecuTorch** [^7]はエッジデバイス上での推論処理を実現するために開発されたライブラリです。 ExecuTorch を利用することで、既存の PyTorch モデルをエッジデバイス向け最適化された形で導入することが可能です。

ExecuTorch の特徴のひとつとして、エッジデバイスのハードウェア機能を活用したパフォーマンス向上が挙げられます。 ExecuTorch では推論を行う際に使用するバックエンドを選択することができ、iOS では Metal Performance Shader (MPS) および CoreML を選択でき、 Android では Vulkan を選択できます。スマートフォンに搭載されている GPU や Neural Engine を活用できることは、推論処理を行う上で大きなメリットといえるのではないでしょうか。

[^7]: https://pytorch.org/executorch-overview

#### PyTorch で生成された学習済みモデルを ExecuTorch で使用するには

ExecuTorch はエッジデバイス用に最適化されているため、PyTorch で生成された学習済みモデル (PyTorch モデル) を ExecuTorch でそのまま実行できないことに注意してください。

"How ExecuTorch Works" [^8]という ExecuTorch の内部構造を解説したドキュメントによれば、PyTorch モデルを実行するためには下記 3 つのステップが必要と解説されています。

1. PyTorch モデルをエッジデバイスでの実行に最適化してエクスポートする
2. エクスポートしたモデルを ExecuTorch 用のモデル (ExecuTorch モデル) としてコンパイルする
3. ExecuTorch ランタイムを用いて ExecuTorch モデルをインポートし推論処理を実行する

本稿で題材となっている Llama については 1. と 2. のステップをコマンド 1 つで処理できるツールが ExecuTorch のリポジトリに含まれており、 Python のコードを書くことなく容易に ExecuTorch モデルを出力できるようになっています。

[^8]: https://pytorch.org/executorch/stable/intro-how-it-works.html

## Llama について

後述するサンプルコードでは Llama のモデルを扱っているため、本節では Llama について紹介します。

**Llama (ラマ)** [^9]は Meta が開発した大規模言語モデル (LLM) です。Llama は最新の Llama 4 までの学習済みモデルが公開されており、モデルをダウンロードした上でローカルで実行可能となっていることが大きな特徴です。ただ、GPU をはじめ高いスペックを要求されるため、ローカルでの実行は容易ではありません。

一方、 **Llama 3.2** に関しては軽量モデルとして提供されており、エッジデバイスでの利用を想定して開発されたモデルです。特に 1B と呼ばれる、モデルサイズが 10 億 パラメータのモデルについては、モデルファイルのサイズが 2.5 GB 程度しかなく、スマートフォンでなどのエッジデバイスでの実行も可能です。これは、パラメータ数が公開されている DeepSeek-V3 の 6710 億パラメータ[^10]と比べると小さなものですが、テキストに特化したモデルとすることでこれだけの小ささでも十分にテキストコミュニケーションを行うことが可能です。

配布されている　Llama の各モデルは PyTorch モデルとなっており、 ExecuTorch で使用するためには変換処理が必要となります。

後述するサンプルコードでは、 Llama 3.2 の 1B モデルを使用します。

[^9]: https://www.llama.com/

[^10]: https://api-docs.deepseek.com/news/news1226

## サンプルコードを動かしてみる

それでは、 ExecuTorch リポジトリ内にある、 Llama 3.2 のモデルを使ったチャットボットのサンプルコードを iOS 端末上で動かしてみましょう。サンプルコードは ExecuTorch のリポジトリにある "ExecuTorch Llama iOS Demo App" [^11]を使用します。なお、 ExecuTorch のバージョンは `0.6` を使用します。

[^11]: https://github.com/pytorch/executorch/tree/main/examples/demo-apps/apple_ios/LLaMA

### 執筆にあたり使用した環境について

執筆にあたり、下記環境を用いて検証を行いました。

#### ビルド環境

- MBP Pro (M2 Pro CPU, メモリ 16GB)
- macOS Sequoia (15.4)

#### 実行環境 (iOS)

- iPhone 13 Pro Max (ストレージ 256 GB, メモリ 6 GB)
- iOS 18.3

#### ソフトウェア

ビルド時に筆者の手元で使用したソフトウェアは下記のとおりです。

- Xcode: 16.2
- g++: 17.0.0 (Xcode の Command Line Tools 付属の g++ を使用)
- asdf: 最新バージョン
- Python: 3.12.0
- ExecuTorch: 0.6

### ビルド手順

それでは、早速アプリをビルドするために必要なツールのインストールやモデルの変換を行いましょう。下記の順に進めていきます。

1. ExecuTorch のインストール
2. PyTorch モデルの変換に使用するツールをインストールする
3. Llama　3.2 1B モデルを取得する
4. Llama モデルを ExecuTorch 用のモデルへ変換してみる
5. LLaMA プロジェクトのビルドに必要な修正を行う

#### 1. ExecuTorch のインストール

ExecuTorch は下記 pip コマンドを用いて簡単にインストールできます。

```shell
pip install executorch
```

ただ、 Llama　のモデルを ExecuTorch 用のモデルへ変換するツールをインストールする関係で、今回は ExecuTorch をソースコードからインストールします。

ソースコードからのインストールは "Building from Source" [^12]に記載されている手順を見ながら進めていきます。

まずは ExecuTorch リポジトリを Clone してください。

```shell
git clone -b release/0.6 https://github.com/pytorch/executorch.git
cd executorch
```

インストールには `install_executorch.sh` を使用します。以下のコマンドを実行してください。

```shell
./install_executorch.sh --pybind xnnpack mps coreml
```

`--pybind` オプションですが、 ExecuTorch 実行時に選択可能なバックエンドの設定となります。本稿では xnnpack [^13]のみ使用しますが、今後 mps (Metal Performance Shaders) や coreml (coreML) を利用する可能性があるので、全て指定してビルドを行いましょう。

install_executorch.sh が正常終了したら、ビルドとインストールは成功です。

[^12]: https://pytorch.org/executorch/0.6/using-executorch-building-from-source.html

[^13]: CPU 上での推論処理を最適化するためのライブラリです。詳細は https://github.com/google/XNNPACK をご覧ください。

#### 2. PyTorch モデルの変換に使用するツールをインストールする

前述のとおり、 Llama のモデル を ExecuTorch で使用するためには ExecuTorch モデルへの変換が必要です。変換に必要なツール一式をインストールしましょう。

ExecuTorch の GitHub リポジトリに Llama 関連のツールが格納されたディレクトリ (`examples/models/llama/`) があります[^14]。ディレクトリ内に `install_requirements.sh` というスクリプトがあるので、こちらのスクリプトを実行するとインストールが開始されます。

```shell
sh ./examples/models/llama/install_requirements.sh
```

install_requirements.sh が正常終了したらインストールは完了です。

[^14]: https://github.com/pytorch/executorch/blob/v0.6.0/examples/models/llama

#### 3. Llama　3.2 1B モデルを取得する

#### 4. Llama モデルを ExecuTorch 用のモデルへ変換してみる

#### 5. LLaMA プロジェクトのビルドに必要な修正を行う

### サンプルコードを動かしてみた 〜 サンプルコードの修正

## まとめ

<hr class="page-break" />
