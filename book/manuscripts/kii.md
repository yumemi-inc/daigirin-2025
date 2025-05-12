---
class: content
---

<div class="doc-header">
  <h1>関数型プログラミングの一歩</h1>
  <div class="doc-author">kii</div>
</div>

# 関数型プログラミングの一歩

## はじめに
これは関数型プログラミングで登場するテクニックについて軽く紹介する内容となっています。関数型プログラミングには面白い考え方や表現の仕方があるんだなと思っていただければ幸いです。
個人的にはカリー化（本編にて後述）による連続した括弧の関数呼び出しが表現として面白くて好きです。たとえば、このようなものです。

```typescript
f(a)(b)(c);
```

カリー化をはじめて見た読者からすると中々トリッキーな書き方に見えるでしょう。しかしその考え方を知ることで普段のコーディングでの表現力を広げてくれるものだと私は感じています。
今回紹介する関数型プログラミングのテクニックはごく一部ですが、ぜひ楽しんで読んでください。
本記事の解説に用いる言語は TypeScript です。

また今回の記事を執筆するにあたって次の書籍を多いに参考にさせていただきました。
- 『関数型プログラミングの基礎』[^1]
- 『JavaScript徹底攻略 関数 〜基礎と関数型プログラミング〜』 [^2]

[^1]: 立川察理, 関数型プログラミングの基礎, リックテレコム, 2016 <br>
      <https://www.ric.co.jp/book/programming/detail/178>
[^2]: イモに聞け, JavaScript徹底攻略 関数 〜基礎と関数型プログラミング〜, 2022 <br>
      <https://techbookfest.org/product/vbZGCTWVT2FG8HAzzkiNn5?productVariantID=gftVap3gkUydhA2hSnAJFd>

## カリー化
カリー化は関数において、複数の引数を単一の引数を受け取るようにすることです。本来複数の引数を受け取る関数であったものを、ひとつの引数を受け取り、その返り値に残りの引数をひとつずつ受け取る関数を返すということです。これだけ聞いても分かりにくいので具体例を見ていきましょう。

```typescript
function add (x: number, y: number): number {
    return x + y;
}
```

このようなふたつの引数を受け取る関数があったとします。これをカリー化するということは、まず引数 x を受け取り、その返り値として y を受け取る関数を返すということです。

```typescript
function curried_add (x: number) : (y: number) => number {
    return function (y) {
        return x + y;
    }
}
```

こうすることで関数をカリー化できます。分かりにくいですが、まず引数 x を受け取りその関数の return 部分で関数を返していることが分かります。つまり最初に引数を渡すとまた関数が出てきます。

```typescript
// この res は y を受け取る関数
const res = curried_add(2);

res(3); // 5
```
この引数を異なるタイミングで適用する手法を部分適用といいます。後の章で説明します。
カリー化された関数を連続して呼び出す際は次のようになります。

```typescript
add(2, 3); // 5
curried_add(2)(3); // 5
```

ちなみに引数が 3 つ以上の場合は次のようになります。

```typescript
function curried (x) {
    return function (y) {
        return function (z) {
            // 何らかの処理
        }
    }
}

// アロー関数で書くとより簡潔
const curried_arrow = (x) => (y) => (z) => ...;
```

### 部分適用
カリー化をすることのメリットのひとつとして、引数をそれぞれ異なるタイミングで受け取れることがあります。つまり、一部の引数を先に適用できるということです。言葉で説明しても分かりにくいので具体例で見ていきましょう。先ほどの `curried_add` を使っていきます。

```typescript
const add_five = curried_add(5);

add_five(3); // 8
```

このように関数適用を部分的に行うことで値が固定されます。こうすることで関数の使い回しがしやすくなります。これを部分適用と呼びます。イメージとしてはクラスにおけるインスタンス化と近いです。抽象的な箱から具象的な値を生成しているということです。

では次に、カリー化と部分適用をメソッドに対し行うことでメソッドチェインを作ってみましょう。
カリー化は引数をひとつずつ分解することでした。それをメソッドに利用してみましょう。

```typescript
const add_chain = (n: number) => {
    let res = n;
    const method = {
        stack: (m: number) => {
            res += m;
            return method;
        },
        calc: () => res
    };
    return method;
};

add_chain(1).stack(2).stack(3).calc(); // 6
```

複数かついくつか分からない引数を受け取りたい時は、カリー化の要領で分割して受け取ると上手くいくことがあります。今回は引数の受け取りをメソッドで行うことでメソッドチェインを可能にしています。また、計算結果をクロージャに置くことで上手くいきました。計算結果については他にも引数で保持する手段もあります。

```typescript
const add_chain = (n: number) => {
    const method = (acc: number) => ({
        stack: (m: number) => {
            return method(acc + m);
        },
        calc: () => acc
    });
    return method(n);
};
```

## 遅延評価
多くの言語で関数の引数を適用する際、引数は即座に評価されます。たとえば次のようなものです。

```typescript
function print(x: number) {
    console.log(x);
}

print(2 * 3);
```

この print 関数に `2 * 3` を引数に渡すと、 `6` に評価されています。このように即座に評価する戦略を正格評価といいます。
一方で、引数が関数内で必要となるまで評価しない戦略を遅延評価といいます。 JavaScript / TypeScript では正格評価が実装されているため、遅延評価を実装するにはサンクを利用します。
サンクとは引数の必要としない関数で値をラッピングすることです。

```typescript
// 通常の値
const v = 2;

// サンクを使った場合
const thunk = () => 2;
```

遅延評価の効果を測るため竹内関数という関数をベンチマークにします。竹内関数は tarai 関数の名前であり再帰呼び出しを繰り返し行なっており、正格評価ではまともに処理できないものです。

```typescript
function tarai (x: number, y: number, z: number) {
    if (x > y) {
        return tarai(
            tarai(x -1, y, z),
            tarai(y - 1, z, x),
            tarai(z - 1, x, y)
        );
    } else {
        return y;
    }
}
```

n を正の整数とした時、 `tarai(2n, n, 0)` は処理に `n ^ n` の時間がかかるとされています。
私が手元で確認した限りでは、 n = 7 が限界で 8 以降はまともに計算ができませんでした。
これを遅延評価にすることでどれほどの効果があるでしょうか。

```typescript
type thunk = () => number;
function lazy_tarai (fx: thunk, fy: thunk, fz: thunk) {
    const x = fx();
    const y = fy();

    if (x > y) {
        return lazy_tarai(
            () => lazy_tarai(() => x - 1, fy, fz),
            () => lazy_tarai(() => y - 1, fz, fx),
            () => lazy_tarai(() => fz() - 1, fx, fy)
        );
    } else {
        return y;
    }
}
```

先ほど計算できなかった n = 8 の場合が驚くほど早く計算できたことが確認できます。遅延評価によって必要のない計算を省くことができ、これだけ高速化が実現しています。
また遅延評価以外にもメモ化という手法で高速化することが可能。メモ化はインメモリ上に計算結果を残しておくことで計算量を省略している。キャッシュのことです。

## 無名再帰関数
繰り返しが必要となる処理では多くの場合 for 文や map メソッドが使われます。ここではそれらを再帰関数で表現してみましょう。
ただ再帰関数にはひとつ注意点があります。関数呼び出しのスタックが積み重なり処理が重くなる点です。これは通常、末尾最適化と呼ばれるテクニックにより解消されます。しかし、ブラウザ側で末尾最適化をサポートしているのはごく一部です。そのため、TypeScript / JavaScript で再帰関数を使用するのはおすすめしません。

```typescript
const sum = (n: number): number => {
    let s = 0;
    for (let i=0; i<=n; i++) {
        s += i;
    }

    return s;
};
```

これは n までの自然数の和を返す関数です。これを再帰関数にするとこうなります。

```typescript
const re_sum = (n: number): number {
    if (n === 1) return 1;
    return n + re_sum(n - 1);
};
```

では次に無名関数についてです。無名関数は文字どおり名前のない関数です。

```typescript
(x) {
    return x;
}
```

これが無名関数で、これ単体ではほとんど意味をなさないです。使う場面は、高階関数として引数に渡したり即時実行をして名前空間を分ける時です。

```typescript
const arr = [0, 1, 2, 3, 4, 5];
arr.map((n) => n ** 2); // map の引数は無名関数

// 無名関数を即時実行することで処理を分けている
const res = (() => {
    const arr = [0, 1, 2, 3, 4, 5];
    const arr_tmp = arr.filter((n) => n % 2 === 0);
    return arr_tmp;
})();
```

無名関数を使うモチベーションは、名前を付けるほどの処理ではないというところにあります。小さな処理を簡単に実装するために利用しています。言ってしまえば使い捨ての関数です。
余談ですが、配列の map メソッドなどに無名関数ではなく名前付きの通常の関数を渡す書き方をポイントフリースタイルと呼んだりします。個人的には初見では分かりにくいと感じています。

```typescript
const double = (x: number): number => {
    return x * 2;
};

const arr = [0, 1, 2, 3, 4, 5];
arr.map(double); // ポイントフリースタイルによる map メソッド
```

では無名再帰関数とはなんでしょうか。もちろん無名関数を再帰関数にしたものです。どう実装したらよいでしょうか。
再帰関数を実装するとなると関数内部で自分自身を呼び出します。しかし名前が付いていないものをどうやったら呼び出せるでしょうか。

まず再帰関数に仮に `anonymous_rec` と名前を付けます。

```typescript
const anonymous_rec = (n: number): number => {
    if (n === 1) return 1;
    return n + anonymous_rec(n - 1);
};
```

次に返り値の部分にある `anonymous_rec` をなくします。ここでもし、名前をなくそうとして展開するとしたら、再帰的に展開することになり不可能です。

```typescript
const anonymous_rec = (n: number): number => {
    if (n === 1) return 1;
    return n + ((n: number): number => {
        if (n === 1) return 1;
        return n + (...)(n - 1);
    })(n - 1);
};
```

必ず何かしらの名前で置き換える必要がある。では、仮引数として扱えないだろうか。つまり再帰関数自身を引数で受け取ることで名前をつけれないだろうか。

```typescript
type AnonymRec: (
    anonymous_rec: AnonymRec,
    n: number
) => number;
function (
    anonymous_rec: AnonymRec,
    n: number
): number {
    if (n === 1) return 1;
    return n + anonymous_rec(anonymous_rec, n - 1);
}
```

これにて、名前を付けずに再帰関数を作成できました。これの関数呼び出しを即時実行でやってみます。

```typescript
(function (
    anonymous_rec: AnonymRec,
    n: number
): number {
    if (n === 1) return 1;
    return n + anonymous_rec(anonymous_rec, n - 1);
})(
    // これは引数
    function (
        anonymous_rec: AnonymRec,
        n: number
    ): number {
        if (n === 1) return 1;
        return n + anonymous_rec(anonymous_rec, n - 1);
    },
    10
)
```

関数呼び出しまでを書くと非常に冗長であると分かります。それは関数定義を二度も書いているからです。この段階で可読性が非常に落ちたので、関数定義を `F` で置き換えます。

```typescript
F(F, 10);
```

非常に簡潔になりましたね。こう見ると、最初の `F` でその定義を現し、次の括弧で関数呼び出しを行なっています（即時実行関数です）。さらに括弧の中でも `F` の関数定義を行なっています。
なぜ二度も関数定義をしているのか。それは即時実行をしている関係でどこにもその定義が保存されていないからです。では、どうしたら二度も関数定義を書かずに済むでしょうか。
関数 `F` の定義した情報を外部で保持できればよいのです。そうすれば、定義は一度で済みます。この状況は先ほどと同じだと分かります。それは、無名再帰関数を作る過程で発生した、関数の名前を取り除くために関数の名前を別の場所で、すなわち仮引数として保持したことと同じです。今回も同様、仮引数で保持すれば解決します。つまり `F` を仮引数として名前を与え、自分自身への適用を行う関数を考えればよいのです。
具体的にどうするか、まず、 `F` を引数として受け取り自分自身への適用をする関数を考えます。自分自身へ適用する際、同時に具体的な数値 `n` も要求されます。このタイミングで `n` を決めるのは不可能なのでカリー化をして `F` を関数を受け取り、その後 `n` を受け取るように変更します。後は簡単です。

```typescript
type CurriedAnonymRec = (
  curried_anonymous_rec: CurriedAnonymRec
) => (n: number) => number;
(function (
    curried_anonymous_rec: CurriedAnonymRec
) {
    return curried_anonymous_rec(curried_anonymous_rec);
})(function (
    curried_anonymous_rec: CurriedAnonymRec
) {
    return function (n: number) {
        if (n === 1) return 1;
        return n + curried_anonymous_rec(curried_anonymous_rec)(n - 1);
    };
})(10);
```

このままだと分かりづらいので、これを先ほどと同様に `F` と `C` で置き換えます。

```typescript
C(F)(10);
```

こうすることで無名再帰関数をより簡潔に表現することができました。つまり関数の呼び出し部分（ Caller ）と関数の定義（ Function ）を分けて書いています。
以上が無名関数による再帰関数です。これを一般化したもの、つまり任意の関数に対し無名再帰を可能にしたものを Y コンビネータや Z コンビネータと呼ばれています。
この話題はラムダ計算につながっていきます。

## おわり
最後まで読んでいただきありがとうございます。関数型プログラミングの面白さが少しでも伝わったでしょうか。今回紹介した話題は直接業務のコードに役立つかというとかなり微妙だと感じます。特に可読性という点で関数型プログラミングはその独特な表現のせいであまりよくないでしょう。しかし、冒頭でもお話したようにその考え方は必ず役に立つと感じています。
また今回取り上げることのできなかった話題もありますので、関数型プログラミングの世界が奥深く学びが多いと感じてくだされば幸いです。
