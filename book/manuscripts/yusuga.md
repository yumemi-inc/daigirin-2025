---
class: content
---

<div class="doc-header">
  <h1>Server Side Swift 実践レポート<br>2024 年に案件で採用して見えた課題と可能性</h1>
  <div class="doc-author">菅原 祐</div>
</div>

Server Side Swift 実践レポート - 2024 年に案件で採用して見えた課題と可能性
==

## Server Side Swift とは

Swift は Apple によって開発された汎用プログラミング言語で、元々は iOS や macOS, watchOS, tvOS 向けのアプリケーション開発のために設計されました。しかし、Swift はシステムプログラミングからモバイル、デスクトップアプリケーション、そして分散型のクラウドサービスに至るまで、幅広い用途に対応できるよう設計されています。特に Swift は、開発者が正確なプログラムを書くことを容易にするように設計されています。

Server Side Swift は、Swift 言語をサーバーサイド開発に使用する能力を指します。サーバー上で Swift アプリケーションを展開するためには、Vapor や Hummingbird のような Web フレームワークが利用されます。これらのフレームワークは、ルーティング、データベース統合、リクエスト処理などの重要な機能を提供し、開発者がビジネスロジックの構築に集中できるようサポートします。

詳しくは前回の `Server Side Swift - API サーバ構築` をご参照ください。

![https://zenn.dev/yusuga/articles/dc601e1f67a0b7](./images_yusuga/previous_zenn.png)

今回は、2024 年末に手掛けた開発実績をご紹介するとともに、浮かび上がった課題と今後の可能性をお伝えします。

## 実装環境

- Xcode: 16.2
- Vapor: 4.112.0

## 開発実績

- 工数は 2 人月程度
- BFF（Backend for Frontend）として設計した API サーバで、8 つのエンドポイントを提供
	- BFF からは 4 つのバックエンドサービスと API 連携
	- OAuth（Auth0）, Basic 認証
	- アナリティクス（DataDog）
	- <!-- textlint-disable -->swift-openapi-generator<!-- textlint-enable --> で Client と Server のコードを生成
	- DB なし
- AWS Fargate 上の Docker コンテナで動作

## 良かった点と得られた気づき

### Linux でビルド、動作可能

通常の iOS 開発は macOS が必須ですが、Server Side Swift は Linux で動作可能なため、Docker を使用できます。Docker を利用することで、手軽に独立した環境を構築できた点が非常に良かったです。さらに CI で安い Linux インスタンスが使用できることもメリットです（macOS インスタンスは割高）。

### ステートレスかつ、副作用を持たない構造が強制され、設計方針が明確になる

API は異なる<!-- textlint-disable -->ユーザ<!-- textlint-enable -->からリクエストされる可能性があるため、異なるリクエスト間で状態を共有するべきではありません。必ずステートレス（オンメモリのプロパティを持たない）にし、副作用（外部のプロパティによって動作が変わる）がないようにする必要があります。そのため、ステートレスを前提にした設計を強制させられることで、設計方針をより明確できます。

### Unit Tests のコードカバレッジ

API サーバの動作確認で都度 `curl` をするのは手間なため、自然と動作確認は Unit Tests となりました。その結果、特別 Unit Tests を頑張って書こうという意識はしていなかったのですが、納品時点でのコードカバレッジが 88％ と通常の iOS 開発と比べて高い水準にできました。

### Vapor の Middleware の設計が素晴らしく、各ロジックを疎結合にしやすい

Vapor における Middleware は、次のように各リクエストの直前と直後に何らかの処理を順列<!-- textlint-disable -->通り<!-- textlint-enable -->に複数挿入できるイメージです。この Middleware の仕組みにより、API ロジックとは直接関係ない処理を簡単に前後に組み合わせることが可能で、各ロジックを疎結合にしやすい仕組みがとても実装しやすかったです。<!-- textlint-disable -->例えば<!-- textlint-enable -->特定のエンドポイントのみ OAuth での認証認可が不要な場合は、そのエンドポイントのみ OAuth の Middleware を適用しないように対応することで除外できます。

**例: Vapor で API リクエストを処理する流れ**

| 実行順 | レイヤー | 処理内容 |
| :---: | --- | --- |
| 1 | OAuth Middleware | OAuth の認証認可を確認 |
| 2 | DI Middleware | エンドポイントの処理に必要な依存を DI する |
| 3 | API Controller | エンドポイントのビジネスロジック |
| 4 | Logging Middleware | ログを取る |
| 5 | Error Middleware | throw されたエラーをエラーレスポンスに変換する |

### API リクエストを受ける側の気持ちがわかった

<!-- textlint-disable -->例えば<!-- textlint-enable -->、OAuth による認証が必要な API リクエストで、クライアントが付与した access_token をサーバがどのように認証・処理しているのか、その具体的な仕組みを深く理解することができました。これにより、API リクエストの処理フローに関する解像度が一段と高まりました。また、この知見が得られたのは、自分が十分に習熟している Swift 言語を使って実装したからこそのメリットだったと感じています。

## 苦労したところ

### Foundation framework の差分がある

iOS や macOS 開発で使用している Foundation framework は Objective-C ランタイムに依存している部分があるため Linux では使用できません。そのため Linux では自動的に OSS の <!-- textlint-disable -->swift-corelibs-foundation<!-- textlint-enable --> に代替されるのですが、残念ながら一部の API は未実装です。

<!-- textlint-disable -->swift-corelibs-foundation<!-- textlint-enable --> は Apple 標準の Foundation と同一の名前空間を共有しているため、macOS 環境で開発を行っている際には特に意識する必要はありません。しかし、誤って <!-- textlint-disable -->swift-corelibs-foundation<!-- textlint-enable --> で未実装の API を使用してしまうと、Linux 環境でビルドエラーが発生します。Xcode を使った開発では macOS の標準 Foundation が用いられるため、問題に気づくのは CI やサーバなどの Linux 環境で初めてビルドした時になりがちです。そのため、Xcode をメインに開発する際には、この点に特に注意する必要があります。

具体的に直面した課題としては、String Catalog が使用できなかったことがあります。これは、<!-- textlint-disable -->swift-corelibs-foundation<!-- textlint-enable --> に LocalizedStringResource の実装が存在しないためです。String Catalog は SwiftUI と密接に連携する API ですが、仕組み的には LocalizedStringResource さえあれば利用可能なはずでした。しかし、現状では <!-- textlint-disable -->swift-corelibs-foundation<!-- textlint-enable --> 側にその実装がないため、Linux 環境では従来<!-- textlint-disable -->通り<!-- textlint-enable --> Localizable.strings を使用する方法でローカライズ対応をするしかありませんでした。

### エラーハンドリング周りの設計に苦労

サーバでは処理したリクエストが最終的に成功なら HTTP Status は 200, 300 系を、エラーなら 400, 500 系を返します。この際に <!-- textlint-disable -->swift-openapi-generator<!-- textlint-enable --> を使用している場合には Vapor 内でのエラーハンドリングと、<!-- textlint-disable -->swift-openapi-generator<!-- textlint-enable --> 内でのエラーハンドリングの ２ つがあり、どの方法を組み合わせるのがベストなのかを模索して時間がかかりました。この件は自分の中でのベストプラクティスはできたので別途記事にまとめたいと考えています。

### ビルド時間が長い

Swift は強い静的型付け言語なためビルド時間が動的型付け言語に比べて長くなる傾向です。本案件は BFF サーバなので、実装規模が比較的小さいものの、キャッシュなしのリリースビルドに約 30 分かかります。ただし、ビルド時間の内訳を確認すると、その 9 割以上が依存ライブラリのビルド時間です。そこで iOS 開発なら XCFramework のバイナリターゲットを使うことで改善できますが、残念ながら執筆時点でバイナリターゲットは Apple Platform のみサポートしているため Linux では利用できません。この件に関しては <!-- textlint-disable -->swift.org<!-- textlint-enable --> の公式フォーラムの `[Pitch] SwiftPM Support for Binary Static Library Dependencies` で議論されており、サポートされるのを待つしかありません。

### その他

インフラ周りの構築は Terraform, IAM, Route 53, ECS, Fargate など API サーバの実装とは直接関係のない異なる知識が必要です。また、ログ出力のフォーマット、環境変数の管理なども iOS とは異なり、サーバとしてのベストプラクティスがあるため、新たに学ぶ必要がありました。

## 解消した不安

### Server Side Swift を動かすための特殊な環境は不要

Server Side Swift は、Go など他のコンパイル型言語と同様に、ネイティブコードとして実行可能なバイナリを生成して動作するため、Swift を実行するための特殊な環境は必要なく、一般的な Linux 環境でそのまま動かせます。これは、ビルド時にオプションとして `--static-swift-stdlib` を指定すると、Swift ランタイムを静的リンクした単一の実行可能ファイルを生成できるため、Ubuntu などの Linux 環境においても特別な準備なしに直接動作させることを実現しています。

### エコシステムは現時点でも十分なレベル

通常の API サーバを構築する上で必要な要素は<!-- textlint-disable -->一通り<!-- textlint-enable -->揃っており、一般的なユースケースで困ることはなさそうです。具体的には次のようなライブラリが用意されています。
- <!-- textlint-disable -->ORM<!-- textlint-enable -->（PostgreSQL, MySQL, SQLite, MongoDB）
- Redis
- JWT
- Crypto

## 今後の課題

### 管理画面の作成

Vapor は管理画面の作成を直接サポートしているわけではありませんが、公式ドキュメントでは Leaf というテンプレートエンジンを使って動的に HTML ページを生成する方法が紹介されています。今回の案件では管理画面の要件がなかったため実際の検証は行っていませんが、今後の案件で必要になる可能性があるため、そのうち試してみたいと考えています。

### 依存するサービスが Swift の SDK を提供していない場合

特定のサービスを利用する必要がある場合に、そのサービスが Swift 向けの SDK を提供していないと導入が困難になり、要件を満たせない可能性があります。この問題はサービス側の対応状況に左右される面がありますが、Swift には他言語との相互運用性（Interoperability）が備わっており、他言語で実装されたコードを Swift から呼び出すことも可能です。

実際に試したことはまだありませんが、2024 年には Swift と Java を相互に利用できるライブラリで <!-- textlint-disable -->swift-java<!-- textlint-enable --> が公開されています。このように、Swift から呼び出せる言語がさらに広がっていけば、Swift 言語向けの SDK が提供されていないという問題も解決できる可能性がありそうです。

**Swift の相互運用性（Interoperability）**

| 言語 | サポート状況 |
| --- | --- |
| C, Objective-C, C++ | ネイティブでサポート |
| Rust, Go | C 互換のライブラリとしてコンパイルして使用 |
| Java | <!-- textlint-disable -->swift-java<!-- textlint-enable -->（2024年9月公開） は C 系と同様にネイティブで相互に呼び出せること目指している |

## Server Side Swift 導入に関する FAQ

### Q. 引き継ぎが難しそうですが、大丈夫ですか<!-- textlint-disable -->？<!-- textlint-enable -->

A. Server Side Swift がサーバ開発の主流になることは考えにくいため、引き継ぎや継続性が課題となる場合は採用を避ける方が無難です。

### Q. 開発者が集まりづらいのでは<!-- textlint-disable -->？<!-- textlint-enable -->

A. 開発者の確保が難しい場合、無理に採用しないほうがよいでしょう。

### Q. 他の言語と比べてメリットはありますか<!-- textlint-disable -->？<!-- textlint-enable -->

A. 特別な要件がなければ他言語と大きな差異はなく、あえて Swift を選択するメリットはあまりないでしょう。

### Q. Server Side Swift だけで本当に実用的なシステムが作れますか<!-- textlint-disable -->？<!-- textlint-enable -->

A. Amazon Prime Video が Server Side Swift を採用して実績を出している例があります。そのため、理論上は大規模なシステムでも構築可能と考えられます（個人的な見解）。

## Server Side Swift を採用する動機とメリット

Server Side Swift の主な利点として、フロントエンドとバックエンドを同一言語に統一することで、開発効率の向上や保守・運用に関わる人員の削減が期待できます。一方で、特殊性が高いため引き継ぎの難易度が上がり、ビジネス的な観点ではデメリットになることもあります。しかし、開発者の体験という視点からは、フロントエンドとバックエンドで言語が共通化されることに大きなメリットを感じています。

開発者の確保や引き継ぎといった課題はあるものの、Server Side Swift は想像以上に導入のハードルが低く、特に iOS エンジニアがサーバサイド実装に挑戦する絶好の機会といえます。ぜひ積極的にトライしてみてください<!-- textlint-disable -->！<!-- textlint-enable -->

## まとめ

Server Side Swift の開発は単純に楽しかったので、今後も力を入れて取り組んでいく予定です<!-- textlint-disable -->！<!-- textlint-enable --> 2025 年は Server Side Swift 案件を増やしていきたいと思っていますので、もしも<!-- textlint-disable -->良い<!-- textlint-enable -->話やお困りごとがありましたら是非お声がけ下さい<!-- textlint-disable -->！<!-- textlint-enable -->
