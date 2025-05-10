---
class: content
---

<div class="doc-header">
  <h1>AndroidアプリでRTL言語対応をする（画像編）</h1>
  <div class="doc-author">akatsuki174</div>
</div>

# AndroidアプリでRTL言語対応をする（画像編）

Androidアプリでアラビア語などのRLT（Right to Left/右から左）記述法を使用する言語（以降、RTL言語とする）に対応する場合、翻訳リソースの追加はもちろん、方向性のある一部のアイコンはRTL言語用として変更するのが適切です。

根拠：インターフェイスアイコン - Human Interface Guidelines[^human_interface_guidelines_interface-icons]

とはいえ簡単にいかない場合もあるので、対応方法について書きたいと思います。

※基本Jetpack Compose前提で書きます。

[^human_interface_guidelines_interface-icons]: インターフェイスアイコン - Human Interface Guidelines https://developer.apple.com/jp/design/human-interface-guidelines/right-to-left#Interface-icons

## 前提

RTL言語をサポートするにはAndroidManifest.xmlに以下の記述を追加する必要があります（といっても、新規にプロジェクトを作成したらデフォルトで記述されてそうです）。

```xml
<application ...
    android:supportsRtl="true">
</application>
```

実装の確認方法としては、端末の第一言語をアラビア語（地域：サウジアラビア）にして意図通りになっているかを見ています。

※開発者モードに「RTL レイアウト方向を使用」というものがあります。これは端末の言語設定に関係なく、強制的にRTLレイアウトを適用できるというものです。つまり、端末が日本語でもRTLレイアウトになります。「アラビア語にしたら設定アプリもアラビア語になって、日本語に戻すのが大変だった」ということはまああるので、デバッグ効率化のためにこちらを活用するのも手です。

cf. デバイスの開発者向けオプションを設定する[^dev_options_drawing]

↓日本語環境でアイコンが左右反転した状態になっている。

| 設定 | 実行結果 |
| :-- | :-- |
| ![](images_akatsuki174/rtl_layout_settings.png) | ![](images_akatsuki174/forced_rtl_layout.png) |

[^dev_options_drawing]: デバイスの開発者向けオプションを設定する https://developer.android.com/studio/debug/dev-options?hl=ja#drawing

## 方法の選定

アイコンを左右反転させる方法はいくつかあります。

結論から言うと、

- ベクター画像を使っている
    - ➡️autoMirroredを設定する
- ベクター画像ではない or 単純に左右を反転させればいいわけではない or 上記の方法では対応できない
    - ➡️RTL言語用リソースを別途用意する

がおすすめかなと思います。

## ベクター画像に設定を記述する

ベクター画像のxmlに1行追加するだけの方法です。

### autoMirroredを設定する

ベクター画像のxmlの `vector` 部分に `autoMirrored` を追記するだけです。

```xml
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:autoMirrored="true"
    android:viewportWidth="24"
    android:viewportHeight="24">
```

使う側は今まで通りです。

```kotlin
@Composable
fun VolumeIcon() {
    Image(
        painter = painterResource(id = R.drawable.ic_volume),
        contentDescription = null,
    )
}
```

### 動かしてみる

自作のボリュームアイコンがちゃんと左右反転しました。

| アラビア語 | 日本語 |
| :-- | :-- |
| ![](images_akatsuki174/volume_ar.png.png) | ![](images_akatsuki174/volume_ja.png) |

## RTL言語用リソースを別途用意する

左右反転したリソースを準備する方法です。strings.xmlを言語ごとに用意するのと考え方的には同じなのかなと思います。

代替リソースの指定[^providing-resources]の「レイアウト方向」の項目に沿って対応してみます。

[^providing-resources]: 代替リソースの指定 https://developer.android.com/guide/topics/resources/providing-resources?hl=ja

### 専用ディレクトリを用意して入れる

`drawable-ldrtl` ディレクトリを用意し、RTL言語用の画像リソースをここに追加します。

![](images_akatsuki174/music_skip_for_ldltr.png)
![](images_akatsuki174/music_skip_for_ldrtl.png)

実装コードはいつも通り。

```kotlin
@Composable
fun MusicSkipIcon() {
    Image(
        painter = painterResource(id = R.drawable.ic_music_skip),
        contentDescription = null,
        modifier = Modifier.size(56.dp)
    )
}
```

### 動かしてみる

ちゃんとRTL言語とLTR言語で画像が変わっています。

| アラビア語 | 日本語 |
| :-- | :-- |
| ![](images_akatsuki174/music_skip_ar.png) | ![](images_akatsuki174/music_skip_ja.png) |

## どちらの方法を使うか

ベクター画像を使っている場合は基本的に `autoMirrored` を使うのがいいと思います。

- メリット
    - デザイナー工数がいらない
    - 画像リソースを増やすことなく対応できる
- デメリット
    - ベクター画像以外にこの方法が使えない
    - 単純に左右反転ではない画像に対応できない
    - （AndroidViewにおいて）Glideを使っていると対応が難しい

ということで上記デメリットに当てはまる場合はRTL言語用リソースを用意するのがいいと思います。

## おまけ：その他の方法

## ImageViewでscaleX=-1を設定する

PNGなどの場合はこのようにコードで反転させる方法も使えます。完全Android Viewの場合も同じようにできるはず。

```kotlin
@Composable
fun ImageViewWithAndroidView() {
    val context = LocalContext.current
    AndroidView(
        factory = { ctx ->
            LayoutInflater.from(ctx).inflate(R.layout.image_view_layout, null)
        },
        update = { view ->
            val imageView = view.findViewById<ImageView>(R.id.imageView)
            if (context.resources.configuration.layoutDirection == View.LAYOUT_DIRECTION_RTL) {
                // directionがRTL言語だったらscaleXを変更
                imageView.scaleX = -1f
            }
        }
    )
}
```

結果は以下の通り。ほうら、私のプロフィール画像が左右反転してるだろ？（わかりづらい）

| アラビア語 | 日本語 |
| :-- | :-- |
| ![](images_akatsuki174/scaleX_RTL_image_view.png) | ![](images_akatsuki174/scaleX_LTR_image_view.png) |

## 最後に

今回はAndroidアプリをRTL言語対応する時に発生するアイコン周りの処理について書いてみました。アプリ全体をRTL言語対応するにはレイアウトなども対応しないといけないですが、それはまた別の機会に...。

公式ドキュメントに「各種の言語および文化をサポートする」[^support_languages]という項目があるので、ぜひそちらも参考にしてみてください。

[^support_languages]: 各種の言語および文化をサポートする https://developer.android.com/training/basics/supporting-devices/languages?hl=ja