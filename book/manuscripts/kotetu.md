---
class: content
---

<div class="doc-header">
  <h1>ExecuTorch の Llama サンプルコードを動かしてみた</h1>
  <div class="doc-author">栗山徹</div>
</div>

ExecuTorch の Llama サンプルコードを動かしてみた
==

## はじめに

なお、本章の記事は 2025 年 4月時点の情報を元に執筆しました。

## エッジ AI について

## ExecuTorch について

## Llama について

## ビルド環境を構築する

それでは、サンプルアプリをビルドするために必要な環境の構築を始めましょう。 PyTorch のサイト内にある ExecuTorch のドキュメントの中に "Setting Up ExecuTorch" [^1]というドキュメントがあるので、こちらの手順に従って環境を構築していきます。

### 執筆にあたり使用した環境について

執筆にあたり、下記環境を用いて検証を行いました。

#### ビルド環境

- MBP Pro (M2 Pro CPU, メモリ 16GB)
- macOS Sonoma (14.5)

#### 実行環境
  
##### iOS

iPhone 13 Pro Max (ストレージ 256 GB, メモリ 6 GB)

##### Android

Google Pixel 6a (ストレージ 128 GB, メモリ 8 GB)

#### ソフトウェア

以下のバージョンのソフトウェアを利用しました。

- Xcode 16.2
- g++ 16.0.0 (Xcode に含まれる g++ を使用)
- Python 3.12.0

### Anaconda　について

"Setting Up ExecuTorch" のドキュメント内では Anaconda (conda) の使用が推奨されていますが、有料化の問題があることから、本章では Anaconda を使わずに環境を構築する方法について解説します。

### 環境構築手順

[^1]: https://pytorch.org/executorch/stable/getting-started-setup.html

## Llama のモデルを ExecuTorch 用のモデルへ変換する

## サンプルアプリを実行してみる

## まとめ

<hr class="page-break" />
