---
title: "RustでJSONから値をゆるりと取り出すマクロを書いた話"
emoji: "🦀"
type: "tech"
topics: ["rust", "json", "macro"]
published: true
---

こんにちは。突然ですが、皆さんはRustのマクロを実装した経験はありますか? **私はあります。**

https://github.com/jiftechnify/valq

このクレートが提供する `query_value!`マクロを使うと、`serde_json::Value`のような入れ子構造を持つ値から、特定の場所にあるデータを取り出すRustのコードを、まるで**JavaScriptを書いているかのように**簡潔に書けます。

![](/images/rust-macro-for-query-json/valq.png)

このように、Rustのマクロ機能を利用すれば構文規則の縛りを超越したコードを書く仕組みを作り上げることができます。Rustでコードを書いているとき、「**このコード、もっとこういい感じに書けないのかな?**」と悩んだことがある方は少なくないのではないでしょうか。**マクロを使えばそんな悩みを解決できるかもしれません!**

この記事では、先ほど紹介したマクロの実装をステップを追って解説しつつ、マクロを実装する上で使えるテクニックや考え方などを紹介していければと思います。

## こんな方に読んでほしい
- Rustのマクロでどんなことができるのか気になる方
- 以前にマクロを実装しようとしてみたが、うまくいかずに挫折した経験のある方
- 前提: Rustの基本的な構文に関する知識、`Option`型に関する知識
    + serde、特に`serde_json::Value`型に関する知識があると読みやすいかもしれません


## 書かないこと
- Rustと他の言語のマクロ機能の違いなどといった、Rustのマクロの特徴の詳細について
- 手続きマクロ(procedural macros)に関すること
:::message
Rustのマクロには `macro_rules!`で定義するいわゆる「**宣言的マクロ**(declarative macros, [macros by example](https://doc.rust-lang.org/reference/macros-by-example.html))」の他に「**手続きマクロ**(procedural macros)」という種類のものがあります。手続きマクロでは、宣言的マクロ以上に手の込んだこと(例: トレイト実装の自動導出)が可能ですが、より発展的な話題となるためこの記事ではこれ以上踏み込みません。

手続きマクロについて知りたい場合は、magurotuna氏による[こちらの記事](https://zenn.dev/magurotuna/articles/bab4db5999ebfa)がよい取り掛かりになります。
:::

# 問題提起: RustでJSONデータの一部を取り出すしんどさ
実装の話に入る前に、筆者が先程のマクロを実装しようと思ったきっかけについて書きます。

以前、外部Webサービスが提供するパブリックWeb APIを扱うツールをRustで実装することがありました。APIから返ってくる値は非常に複雑な構造をしているのですが、機能の実装に必要な値はその中のたった数個だけ、という状況でした。

RustでJSONなどの構造つきデータを扱う際の定番クレートである[serde](https://serde.rs/)には、そのようなデータをRustの構造体にマッピングしてくれる便利な機能が備わっています[^1]。しかし、**マッピング先の構造体の定義は自分で書かなければなりません**。Web APIが返す複雑~~怪奇~~なデータを表現する構造体を書くのは非常に面倒な作業です。さらに悪いことに、**Rustではフィールド名つきの無名構造体を書くことができない**[^2]ため、データの端々に出てくる中間構造体(?)のひとつひとつに名前をつけねばなりません。ソフトウェアエンジニアリングにおける「名付け」の難しさについては、語るに及びません。

次善の策として、JSONの構造をRust上のデータとして表現するデータ型である[`serde_json::Value`](https://docs.serde.rs/serde_json/value/enum.Value.html)から、欲しいデータの場所を直接指定して値を取り出すという方法があります。`serde_json::Value`からデータを取り出す手段はいくつか用意されていますが、実はいずれにも微妙な問題が潜んでいます。

- `get()`は、指定したフィールドが存在しない可能性を表現するために`Option`を返します。ネストしたデータを取得するには、以下のように`and_then()`をつなげて書くことになります。ネストが深いところの値を取り出そうと思うと、少々骨が折れます。

    ```rust
    // j: serde_json::Value
    let deep = j.get("foo")
        .and_then(|v| v.get("bar"))
        .and_then(|v| v.get("baz"))
        .and_then(|v| v.get(1))
        .and_then(|v| v.get(2));
    ```

    :::details ?演算子を使って短縮する方法
    `and_then()`を連鎖させるかわりに、[`?`演算子](https://doc.rust-jp.rs/rust-by-example-ja/error/option_unwrap/question_mark.html)とクロージャの即時実行を組み合わせることで、同等の処理をより短く書けます。
    ただ、ある程度Rustに慣れていないと意味が取りづらいかもしれません。

    ```rust
    // j: serde_json::Value
    let deep = (|| {
        Some(
            j.get("foo")?
                .get("bar")?
                .get("buz")?
                .get(1)?
                .get(2)?
        )
    })();
    ```

    (情報提供: [higumachan氏](https://zenn.dev/higumachan) ありがとうございます🙇)
    :::

- `Index`トレイトを実装しているので、角括弧記法が使えます。こちらだとコードは長くなりませんが、[指定したフィールドが存在しない場合は`Value::Null`(JSONの`null`に対応するもの)を返す](https://docs.serde.rs/serde_json/value/enum.Value.html#impl-Index%3CI%3E)という仕様のため、「フィールドが存在しない場合」と「値が`null`のフィールドが存在する場合」を区別しなければならない状況では使えません[^3]。フィールド名と配列のインデックスの両方が`[]`になるのも個人的にはマイナスポイントです。

    ```rust
    // j: serde_json::Value
    let deep = j["foo"]["bar"]["baz"][1][2];
    ```

- `pointer()`というメソッドを使うと、[JSON Pointer](https://triple-underscore.github.io/RFC6901-ja.html)記法でデータの場所を指定して値を取り出せます。しかし、この記法はちょっと見慣れない記法ですね。また、仕組み上、間違った文法のJSON Pointerを渡してしまうのを防げません。`None`が返ってきたとき、本当にそこに値が存在しないのか、文法ミスなのかを区別する方法はありません。

    ```rust
    // j: serde_json::Value
    let deep = j.pointer("/foo/bar/baz/1/2");
    ```

この中では`get()`と`and_then()`を連ねる方法が一番手堅そうですが、これとコードの簡潔さを両立する方法はないのでしょうか? 
できれば構文もJavaScriptに近い親しみやすい構文で `j.foo.bar.baz[1][2]` のように書けると嬉しいです。それに、文法が間違っていたらコンパイルエラーになってほしいですよね。

これらを全部満たす仕組みを作れるのかって? 作れるんです。**そう、Rustのマクロならね**。

[^1]: これも手続きマクロの応用例のひとつです
[^2]: RustがGoと比べて負けている部分のひとつ…だと思っています。ちなみにタプルはフィールド名がない無名構造体ですね
[^3]: 今どきそんな状況なんて存在しない、と思いたいところですが…

# マクロを使ってできること
作れるんです。といわれても、**Rustのマクロが根本的には何をするものなのか** というのを知らないと実感が湧かないかもしれませんので、ここで簡単に説明しておきます。

一言でいえば、Rustの(宣言的)マクロは**引数として与えられたRustコードのトークンの並びが特定のパターンにマッチする場合に、マッチしたパターンに対応するコードを生成する仕組み**です。この処理を「マクロの展開」と呼びます。引数の値に対してパターンマッチを行い、パターンに対応した値を返すような関数を書いた経験がある方は多いと思います。マクロとは、そのような関数の引数と返り値がRustのコードになったようなものだといえます。

簡単な例を見てみましょう。コード中の**メタ変数**(metavariables)、**フラグメント指定子**(fragment-specifier)という言葉は後の説明でも使いますので覚えておいてください。

```rust
macro_rules! safe_div {
    // パターン(1)
    // パターン内の`$`から始まる部分を「メタ変数」と呼び、
    // マッチしたコードの断片(fragment)が紐つけられる。
    ($_:expr, 0) => { 
        None 
    };
    // パターン(2)
    // メタ変数についている`expr`, `literal`は、マッチするコードの「種類」を指定する。
    // 例えば`expr`は式、`literal`はリテラルのみにマッチする。
    // この指定を総称して「フラグメント指定子」という。
    ($a:expr, $b:literal) => {
        Some($a / $b)
    };
}

let a = safe_div!(1, 0);     // (1)にマッチ => `None`に展開
let b = safe_div!(x + y, 2); // (2)にマッチ => `Some((x + y) / 2)`に展開
```

2番めの呼び出しの展開の様子を図にしてみました。
![](/images/rust-macro-for-query-json/simple_expansion.png)

より詳しい説明は[The Rust Bookのマクロの項](https://doc.rust-jp.rs/book-ja/ch19-06-macros.html)に譲るとして、ここではマクロの重要な性質についてまとめます。

- マクロの引数に渡すコードは、**Rustの構文規則や型などによる制限を受けない**。マクロ展開の結果さえ正しければよい。
- マクロの展開はコンパイル時の、型チェックなどのチェック処理よりも前に行われる。よって、**マクロ展開で生成されたコードに間違いがあればコンパイルエラー**になる。

このことが分かれば、マクロを利用することで

> - 簡潔なコードで`get()`と`and_then()`を連ねるのと同じ方法を実現する
> - 素のRustコードでは書けないような構文を実現する
> - 文法が間違っていたらコンパイルエラーにしたい

これらすべての条件を両立するコードが書ける、というのが分かってくるのではないでしょうか。


# 実際に書いてみる
まず、目標を整理しましょう。

```rust
// 目標: 「展開前」のコードから「展開後」のコードを生成するマクロを実装する
// 展開前
query_value!(j.foo.bar.baz[1][2])

// 展開後
j.get("foo")
    .and_then(|v| v.get("bar"))
    .and_then(|v| v.get("baz"))
    .and_then(|v| v.get(1))
    .and_then(|v| v.get(2))
```

一気にすべての実装を進めるのは難しいので、最小限の機能を実装するところから始めて、段階的に機能を追加していくことで、目標に近づいていきましょう。

## Step 1: JSONオブジェクトのフィールドを取り出す(1段)
とにかく、特定のフィールドを取り出せるようにしなければ始まりません。

```rust
// 展開前
query_value!(j.foo)

// 展開後
j.get("foo")
```

展開前のコードにおいて、`j`は式(expression)、`foo`は識別子(identifier)になる想定です。よって、次のようなパターンを書けばよさそうです。

```rust
macro_rules! query_value {
    // `$v:expr`は式、`$key:ident`は識別子にマッチする
    ($v:expr . $key:ident) => {
        // TODO
    };
}
```

しかし、残念ながらこれではエラーになってしまいます。

```
error: `$v:expr` is followed by `.`, which is not allowed for `expr` fragments
...
note: allowed there are: `=>`, `,` or `;`
```

つまるところ、`expr`と指定したメタ変数の直後に `.` は置けず、`=>`, `,`, `;`のいずれかしか置けないよ、ということです。これには、深遠な理由があります[^4]。

これを回避するには`$v`のフラグメント指定子を他の種類に変える必要があります。[フラグメント指定子の一覧](https://doc.rust-lang.org/reference/macros-by-example.html#metavariables)を見ると、使えそうなものは`ident`くらいに見えます。しかし、`ident`にすると以下のようなコードは通らなくなります。

```rust
// json!は、serde_jsonクレートで提供されている、JSON形式で書いたコードを
// そのJSONに対応する`serde_json::Value`の値を生成するコードに変換するマクロ
query_value!(json!({"foo": 1}).foo)
```

このように、「JSONの値になる式」から直接内部の値を取り出すようなコードを書かなければならない場面は想像しづらいです。しかし、可能性が0とは言い切れないにもかかわらず、完全に切り捨ててしまうのももったいなく思います。

実は、上記の可能性を切り捨てることなく、`expr`の制約を回避する方法が1つあります。それは、**`tt`を指定する**という方法です。`tt`は TokenTree の略で、**任意のトークン、または対になった括弧で囲まれたトークン列のひとかたまり**にマッチします。`tt`と指定されたメタ変数の後にくるものに制限はないので、以下のパターンは動作します。

```rust
macro_rules! query_value {
    // `$v:tt`は任意の1トークン or 1つの括弧でくくられた範囲にマッチ
    ($v:tt . $key:ident) => {
        // TODO
    };
}
```

`tt`を使えば制限なく何にでもマッチさせることができるのですが、裏を返せば**意図しないマッチングが発生しやすくなる**ということでもあります。そういった場合への対処方法については後述します。

パターンが書けたので、次は生成されるコードの中身を書いてみましょう。識別子を文字列リテラルに変換するマクロ[`stringify!`](https://doc.rust-lang.org/std/macro.stringify.html)を利用しています[^5]。

```rust
macro_rules! query_value {
    ($v:tt . $key:ident) => {
        // stringify!は 識別子`hoge`を文字列リテラル`"hoge"`に変換する
        $v.get(stringify!($key))
    };
}
```

**これで、このステップの目標は達成できました!** 🎉

試しに、Rust Playgroundでマクロの展開結果を確認してみましょう。Rust Playgroundの画面右上の「TOOLS」→「Expand macros」を選択すると、Standard Outputの方にマクロ展開後のコードが表示されます。

https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=3736ecb443af99b9fb05a1098fff5fab


ローカル環境でマクロ展開の結果を確認したい場合は、[cargo-expand](https://github.com/dtolnay/cargo-expand)を使うとよいです。

```sh
// インストール
cargo install cargo-expand

// マクロ展開結果表示
cargo expand
```

[^4]: 式のあとに`=>`, `,`, `;`以外のものがくるのを許すと、構文解析の結果を1つに定められなくなってしまうため、のようです。[詳しくはこちら](https://doc.rust-lang.org/reference/macros-by-example.html#follow-set-ambiguity-restrictions)
[^5]: `stringify!`を忘れると、`get()`に渡されるのが`$key`にマッチした文字列そのものではなく、その識別子が対応する「変数」になってしまう

### Step 1.5: 数字から始まるフィールド名に対応する
実は、現状では名前が数字から始まるフィールドを指定することができません。なぜならば、Rustでは識別子を数字から始めることができず、以下のように書いても`ident`と指定されたメタ変数にマッチしないからです。

```rust
// error: no rules expected the token `1st`
query_value!(j.1st)
```

この書き方をそのままサポートするのは難しいので、文字列リテラルを使って下の例のように書けるようにしてみましょう。

```rust
query_value!(j."1st")
```

もう一度[フラグメント指定子の一覧](https://doc.rust-lang.org/reference/macros-by-example.html#metavariables)を見てみると、`literal`が使えそうです。`literal`には文字列リテラルだけでなく数値リテラルなどもマッチしてしまいますが、生成コードの中でそのリテラルに対して`as &str`とすると、`&str`にキャストできない型のリテラルをエラーにできます。

```rust
macro_rules! query_value {
    ($v:tt . $key:ident) => { ... };
    ($v:tt . $key:literal) => {
        $v.get($key as &str)
    };
}
```

Rust Playgroundで確認:
https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=89533c412328315761bc32f216e754af


## Step 2: JSON配列からインデックス指定で値を取り出す(1段)
次に、JSON配列に対するインデックス指定の記法を実装してみましょう。`serde_json::Value`の`get()`メソッドは、[引数の型が`&str`の場合と`usize`の場合で動作が変わり、後者の場合は配列のインデックス指定になります](https://docs.serde.rs/serde_json/value/enum.Value.html#method.get)。よって、このステップの目標は以下のようになります。

```rust
// 展開前
query_value!(j[0])

// 展開後
j.get(0 as usize)
```

ここまでに紹介した知識を応用して書けそうですね。結果はこんな感じになります[^6]。

```rust
macro_rules! query_value {
    ($v:tt . $key:ident) => { ... };
    ($v:tt . $key:literal) => { ... };
    ($v:tt [ $idx:expr ]) => {
        $v.get($idx as usize)
    };
}
```

これで、**JSONのトップレベルにある任意の値を取得できるマクロの完成です!** 🎉🎉

Rust Playgroundで確認:
https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=5407c04053cc7fdd4bc0c4aa877c481f

[^6]: `$idx`のfragment-specifierは`literal`でもいいのですが、インデックスとして数値の変数や式を指定できると嬉しいことがありそうなので`expr`にしてみました。なお、括弧(`(`, `)`, `{`, `}`, `[`, `]`)は特別扱いされるトークンなので、先程の制限には引っかかりません。

## Step3: ネストに対応させる
いよいよネストした値を取得できるようにしていきます。

```rust
// 展開前
query_value!(j.foo.bar[0].baz)

// 展開後
j.get("foo")
    .and_then(|v| v.get("bar"))
    .and_then(|v| v.get(0 as usize))
    .and_then(|v| v.get("baz"))
```

:::message
これ以降、`query_value!`マクロの引数に渡す、取得したい値を指定するためのトークン列を「**クエリ**(query)」と呼びます。
:::

取り掛かりとして、引数として2段以上のフィールド指定・インデックス指定が連続したクエリを渡してもエラーにならないよう、パターンを修正します。

```rust
macro_rules! query_value {
    ($v:tt . $key:ident $($rest:tt)*) => { ... };
    ($v:tt . $key:literal $($rest:tt)*) => { ... };
    ($v:tt [ $idx:expr ] $($rest:tt)*) => { ... };
}
```

各パターンの末尾に`$($rest:tt)*`を追加しました。これは、パターンの前半(`$v:tt . $key:ident`などの部分)にマッチしなかった「残り」のトークンの並びを`$($rest)*`というメタ変数にまとめる、という意味になります。これで、2段以上のクエリを渡してもひとまずエラーにはならなくなりました。

### TT muncher: 再帰でトークン列を貪り食う
クエリの2段め以降を`and_then(|v| v.get(...))`の列に変換する方法を考えます。もし、クエリとして`.{フィールド名(識別子)}`というパターンだけをサポートすればよいのであれば、[同一パターンの繰り返しを一括で変換する記法](https://veykril.github.io/tlborm/decl-macros/macros-methodical.html#repetitions)を用いて比較的簡単に実装できるのですが、今回の目標は`.{フィールド名(識別子)}`, `.{フィールド名(文字列リテラル)}`, `[{インデックス(数値)}]`の3パターンが**入り混じった**クエリに対応することなので、この方法は使えません。

一旦、この処理を通常のRustコードで実装することを考えてみましょう。こんな感じになるでしょうか[^7]。

```rust
// ttsの先頭がパターンに一致する場合、マッチしたトークンを消費しつつ、
// そのパターンから生成されるコードの文字列を返す
fn match_query_pattern(tts: mut VecDeque<TokenTree>) 
    -> (Option<String>, VecDeque<TokenTree>) 
{
//  注: 雰囲気だけ表した仮想的なコード。Rustではパターンマッチで書くのは無理
//  match tt {
//      (フィールド名指定) => {
//          let _ = tts.pop_front(); // パターンのトークン数だけ繰り返し
//          (Some(".and_then(|v| v.get(...))"), tts)
//      }
//      (インデックス指定) => {
//          let _ = tts.pop_front();
//          (Some(".and_then(|v| v.get(... as usize))"), tts)
//      _ => (None, tts)
//  }
}

let mut tokens: VecDeque<TokenTree> = ...;
let mut res_code = String::new();
// クエリを1段階ずつ処理し、段階ごとの結果を最終結果に追加していく
loop {
    if tokens.is_empty() {
        break;
    }
    if let (Some(pat_code), rest) = match_query_pattern(tokens) {
        res_code.push_string(&pat_code);
        tokens = rest;
    } else {
        panic!();
    }
}
```

このようにループ処理が必要になりますが、マクロの定義において`loop`のような構文は直接はサポートされていません。さてどうしたものか…

実は、この問題は**マクロの再帰呼び出し**によって綺麗に解決できます[^8]。 

具体的な実装は次のようになります。なお、クエリの1段めと2段め以降では処理が少々異なるので、ここでは2段め以降の処理を別のマクロ`query_nested_value!`に切り出しています[^9]。

```rust
macro_rules! query_nested_value {
    // 残りのトークンがない -> クエリをすべて変換し切った
    ({ $vopt:expr }) => {
        $vopt
    };
    // 残りのトークン列の先頭がフィールド指定パターンにマッチ
    // ->「前段までの結果に`.and_then(|v| ...)`を追加した式」と「残りトークン列」を
    //   引数に query_nested_valueを再帰的に呼び出す
    ({ $vopt:expr } . $key:ident $($rest:tt)*) => {
        query_nested_value!(
            { $vopt.and_then(|v| v.get(stringify!($key))) } $($rest)*
        )
    };
    // 他のパターンも同様なので省略
    ...
}
macro_rules! query_value {
    // 1段めがマッチしたら、「1段めの変換結果の式」と「残りのトークン列」を引数に
    // 2段め以降のマッチングを行うヘルパーマクロ query_nested_value を呼び出す
    ($v:tt . $key:ident $($rest:tt)*) => {
        query_nested_value!(
            { $v.get(stringify!($key)) } $($rest)*
        )
    };
    ...
}
```

関数の再帰呼び出しに慣れている方であれば理解は難しくないと思いますが、そうでない方はこれでうまくいくのかピンとこないかもしれません。そんなときは`trace_macros!`という機能を使ってマクロの展開を1段階ずつ追ってみましょう。

:::details trace_macros!の使い方
`trace_macrons!`はnightly版でのみ使える実験的機能で、利用するには明示的に有効化する必要があります。以下のように使います。

```rust
#![feature(trace_macros)]

fn main() {
    trace_macros!(true);
    let a = query_value!(j.foo.bar.baz);
    trace_macros!(false);
}
```

このように書くと、`trace_macros!(true)`から`trace_macros!(false)`の間にあるすべてのマクロ呼び出しについて、**展開の過程を1段階ずつ表示してくれます**。引数がどのパターンにマッチして、その結果どんなコードに展開されたかをステップバイステップで追えるので、マクロの理解に大いに役立ちます。
:::


このような、マクロの再帰呼び出しを利用して任意の長さ・組み合わせのパターンの連続を1段ずつ処理していく実装パターンを、Rustコミュニティでは["TT muncher"](https://veykril.github.io/tlborm/decl-macros/patterns/tt-muncher.html)と呼んでいます[^10]。

[^7]: 実際に動作するコードではありません。また、クエリの1段めの処理に関しては省略しています。
[^8]: 関数型プログラミングの経験がある方はピンとくるのではないでしょうか。
[^9]: このような、他のマクロから内部的に呼び出されてそのマクロの処理の補助をするマクロのことを「ヘルパーマクロ」と呼びます。
[^10]: "TT"は TokenTreeの略、"muncher"は「貪り食う人」のことを指す言葉です。😋🍽🌲🌲

### 外部への公開に向けて、1つのマクロにすべての処理段階をまとめる
これで完成! といいたいところなのですが、このマクロを外部に公開することを考えたときに問題になる点が1つあります。先程の`query_value!`マクロを外部クレートから使いたくなったとします。**外部に公開するマクロには`#[macro_export]`という属性をつける必要があります**。ただし、`query_nested_value!`は外部から直接呼び出すことを想定していないので、`#[macro_export]`をつけないことにします。

```rust
macro_rules! query_nested_value {
    ...
}

#[macro_export]
macro_rules! query_value {
    // 内部でquery_nested_valueを呼び出す!
}
```

これで外部クレートからは`query_value!`だけが見える状態になりました。それでは実際に外部から`query_value!`マクロを呼び出してみましょう。すると、以下のコンパイルエラーが発生します。

```
error: cannot find macro `query_nested_value` in this scope
```

これは、`query_value!`の中で`query_nested_value!`を呼びそうとしたものの、それを`use`でインポートしていないために見つけられなかったということです。

実は、通常の関数などとは異なり、別のクレートで定義され外部に公開されているマクロを正しく呼び出すには**そのマクロが内部で呼び出すすべてのマクロをインポートしなければなりません。** しかし、`#[macro_export]`によって外部に公開されていないマクロはもちろんインポートできません。よって、**あるマクロを外部に公開する場合、そのマクロが内部で呼び出すヘルパーマクロもすべて公開しなければならない**ことになるのです。

ヘルパーマクロはそもそも外部から直接利用することを想定しないものですし、数が多いと利用側の名前空間を汚染することになるので、できればすべてを公開するのは避けたいところです。

現状、この問題に対処するには**すべてを1つのマクロ定義にまとめる**しかないようです。このときに使えるのが、マッチパターンと再帰呼び出しの工夫によって、「内部マクロ」をエミュレートする実装パターンです。ヘルパーマクロを利用していた`query_value!`の実装を1つのマクロにまとめると次のようになります。この実装パターンは[こちら](https://veykril.github.io/tlborm/decl-macros/patterns/internal-rules.html#internal-rules)を参考にしています。

```rust
macro_rules! query_value {
    // 2段め以降の処理のパターンに @trv というマーカーをつける
    (@trv { $vopt:expr }) => { $vopt };
    (@trv { $vopt:expr } . $key:ident $($rest:tt)*) => {
        query_value!(@trv
            { $vopt.and_then(|v| v.get(stringify!($key))) } $($rest)*
        )
    };
    ...
    // 1段めの処理。最初の処理段階にはマーカーをつけない
    // マーカーのないパターンをマーカーのあるパターンより上に書くと
    // マーカー部分が $v:tt にマッチしてしまうので、必ず末尾に書く
    ($v:tt . $key:ident $($rest:tt)*) => {
        query_value!(@trv // <- @trvをつけることで、2段め以降の処理段階に移行
            { $v.get(stringify!($key)) } $($rest)*
        )
    };
    ...
}
```

## 完成形
以上で、**当初の目標を達成するマクロの実装が完成しました!** 🎉🎉🎉 下のRust Playgroundにフル実装を置いておきましたので、いろいろなクエリを書いて変換結果を確認してみてください。

https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=a986a677b71c7437819cbafc0dd706f4

# もっと便利に
ここまでの実装で、「`serde_json::Value`の値から内部の`Value`を取り出す」という基本機能が実現できましたが、実は記事の最初で紹介した拙作[valq](https://github.com/jiftechnify/valq)の`query_value!`マクロはさらに便利な機能を備えています。それらについて簡単な方針を提示しますので、自分なりに実装を考えてみてください!

## `as_xxx()`による特定の型への変換に対応
`serde_json::Value`はJSONの値がとりうる「文字列、数値、bool値、オブジェクト、配列、`null`」の6種類の可能性を、Rustの`enum`の形で実装したものになっています。さらに、`as_<型名>()`というメソッドが用意されており、レシーバの`Value`が`<型名>`に一致する場合に`Value`に包まれていたその型の値を取得できます。

さて、現状`query_value!`の結果から特定の型の値を取得するには
```rust
query_value!(j...).and_then(|v| v.as_xxx())
```
と書く必要があります。ここまで読んできた皆さんなら、この`and_then()`を書く手間も省きたくなってくるのではないでしょうか? 

マクロの追加機能としてこれをサポートしてみましょう! 具体的には、以下のような展開ができるようにしたいです。

```rust
// 展開前。"->"のあとに`as_<型名>()`の<型名>を書く
query_value!(j.foo.bar -> str)

// 展開後 
j.get("foo")
    .and_then(|v| v.get("bar"))
    .and_then(|v| v.as_str())
```

### 実装方針
「残りのトークン列」`$($rest)*`の先頭が `->`であれば、`and_then(|v| -> v.as_xxx())`に展開する処理段階に移行するようにします。この新しい処理段階を識別するため、これまでに出てきた`@trv`とは別の`@`つきのパターンを使いましょう。

なお、指定された型名を`$to`として `v.as_$to()` で済ませたくなるところですが、メタ変数に紐付いたフラグメントをメソッド名の一部として埋め込むことはできないようです[^11]。したがって、残念ながらすべての`as_xxx()`に対して対応するパターンを1つ1つ書いていくしかありません。

[^11]: nightly限定の実験的APIである[`concat_idents!`](https://doc.rust-lang.org/std/macro.concat_idents.html)という複数の識別子を連結する機能を持つマクロが使えそうだと思いましたが、その結果を単純にメソッド呼び出し式に使っても上手くいきませんでした。いい方法をご存知の方がいればこっそり教えてください


## `get_mut()`による可変参照の取得への対応
`get_mut()`メソッドを使うと、`serde_json::Value`への可変参照を取得できます。さらにそこから`as_object_mut()`でオブジェクト(`Value::Object`)、`as_array_mut()`で配列(`Value::Array`)への可変参照を取ることができます。この可変参照を通して、JSON内部の好きなところにある値を上書きできます。

クエリの先頭に`mut`と書くことで、`get()`の代わりに`get_mut()`を使うように動作を変更できると便利そうです。

```rust
// 展開前
query_value!(mut j.foo.bar.baz -> object)

// 展開後
j.get_mut("foo")
    .and_then(|v| v.get_mut("bar"))
    .and_then(|v| v.get_mut("baz"))
    .and_then(|v| v.as_object_mut())
```

### 実装方針
ポイントは、`mut`が指定された場合は、**最初から最後まで`get_mut()`を使いつづける必要がある**という点です。つまり、クエリの先頭が`mut`か否かによってまったく別の処理段階を踏む必要があるわけです。考え方はこれまでと同じで、`get()`の代わりに`get_mut()`を出力するパターンを用意して、先頭が`mut`の場合はそちらに処理を進めるようにします。

また、クエリの先頭が`mut`かつ最後に`-> object`/`array`が指定された場合は、`as_object()`/`array()`ではなく`as_object_mut()`/`array_mut()`に変換したいので、ここでも処理段階の区別が必要です。

先頭に`mut`がつくパターンとつかないパターンを並べることになりますが、順序を間違えると意図しないマッチが発生し上手く動きません。上手く動いてないなと思ったら`trace_macros!`の出番です!

:::details ヒント
これは、最終的な`query_value!`の処理段階の移り変わりを、状態遷移図としてまとめた図です。
赤字で示されているのは遷移の条件になります。状態名はvalqの実装に沿っています。
![](/images/rust-macro-for-query-json/state_diagram.png)
:::

# マクロの実装に役に立つあれこれ
## 参考文献
- [The Little Book of Rust Macros](https://veykril.github.io/tlborm/)
Rustのマクロに関する知見が集められたサイトです。マクロの基本から、役に立つ実装パターンの数々、さらには言語処理系の実装まで。
なお、Daniel Keep氏による[原版](http://danielkeep.github.io/tlborm/book/)は長らく更新が止まってしまっていましたが、Veykril氏が原版の内容を引き継いだ上で内容を補完しています。

- [Macros By Example - The Rust Reference](https://doc.rust-lang.org/reference/macros-by-example.html)
Rustの詳細仕様のリファレンスであるThe Rust Referenceの宣言的マクロに関する項目です。
記事中でも何度か参照した[フラグメント指定子の一覧](https://doc.rust-lang.org/reference/macros-by-example.html#metavariables)は、マクロを実装する過程の中で何度も見返すことになると思います。それぞれが正確にはどんなトークンにマッチするのかを知るのにも役に立ちます。

- [マクロクラブ Rust支部 | κeenのHappy Hacκing Blog](https://keens.github.io/blog/2018/02/17/makurokurabu_rustshibu/)
Rustのマクロに関する数少ない日本語の文献です。ここでしっかり説明できなかった点に関して非常に詳しく書かれていますので、この記事を読んでいて分からないことが出てきたら読んでみてはいかがでしょうか(丸投げ)。

- [serde_wat](https://crates.io/crates/serde_wat)
拙作valqと同様の機能をもつ`wat!`マクロを提供するクレートです[^12]。サポートされている構文は限定的ですがその分実装が簡潔になっています。build.rsでマクロ実装のテキストファイルを生成するという手法は目から鱗です。メタ・メタプログラミングだ…

[^12]: 発見したのはvalqの実装が終わった後ですが…

## ツール
- [macro-expand](https://github.com/dtolnay/cargo-expand)
`cargo expand`とコマンドを叩くだけで、コード中のマクロ呼び出しを展開した結果のコードを出力してくれます。実装が一段落したところで、期待通りのコードに展開されることを確認するのに使うことが多いでしょう。ちなみに、よく使われるマクロ第1位(?)である`println!`の展開結果は必見です。

- [`trace_macros!`](https://doc.rust-lang.org/beta/unstable-book/library-features/trace-macros.html)
    記事中でも紹介しました、**マクロ展開の過程を1段階ずつ表示してくれる**優れものです。

    複雑なマクロを実装していると、突然思い通りに動かなくなることがあります。大抵は想定外のパターンにマッチしてしまっているのが原因ですが、コンパイラが出力するエラーからは原因がすぐに分からないことがあります。そんなときに`trace_macros!`を使えば展開過程を段階ごとに追えるので、原因が特定しやすくなります。いわばマクロのステップ実行ですね。

## 雑多な知見
- やりたいことができるか怪しくても、リファレンスなどを見て悩む前にとりあえずやりたいことをコードに書いてみましょう。一見できそうにないことでも**意外とできます**。
- TokenTreeの性質により、コードを括弧で囲めば、複数のトークンの列をひとまとめにして単一のメタ変数にマッチさせられます。うまく動かないときは、**まとめて扱いたいところを括弧で囲んでみると**道が開けるかもしれません。
- パターンマッチは上から順番に試行されるため、「当たり判定」が大きいパターン(`$t:tt`から始まるパターンなど)を上の方に書くと思いがけないマッチが発生しやすくなります。逆に特殊なフラグメント指定子(`literal`など)をもつパターンは「当たり判定」が小さいので、上の方に書いても安全です。
- **テストは可能な限り多くのパターンを網羅するように書きましょう**。通常のコードに比べてマクロは小さな変更によって壊れやすいためです。一見変更とは関係ないところに影響が出ることも多いです。


# まとめ
非常に長くなってしまいましたが、Rustのマクロを実装して理想のコードが書けるようになるまでの過程を共有してみました。この記事を読んで、**自分だけの最強のマクロを作ってみたい!** と思っていただけたならば幸いです。

最後になりますが、[valq](https://github.com/jiftechnify/valq)をどうかよろしくお願いします🙇

:::message
この記事は [Rust Advent Calendar 2021](https://qiita.com/advent-calendar/2021/rust) の18日目の記事です。
:::
