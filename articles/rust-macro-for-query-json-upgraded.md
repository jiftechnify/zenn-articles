---
title: "RustでJSONから値をゆるりと取り出すマクロをもっと便利にしてみた話"
emoji: "🦀"
type: "tech"
topics: ["rust", "json", "macro"]
published: false
---

:::message
この記事は、[Rust Advent Calendar 2025](https://qiita.com/advent-calendar/2025/rust) の22日目の記事です。
:::

JSONをはじめとする半構造化データ[^1]から一部の値を抽出するような処理をRustで書こうとすると、どうしても煩雑になりがちです。そんなRustの難点をマクロの力で解決するべく、**valq**というクレートを開発しています。

https://crates.io/crates/valq

valqが提供するマクロを使うと、巨大なJSONの奥深くにあるデータを**まるでJavaScriptを書いているかのように**簡単に取り出すことができます。

![code with valq::query_value!](/images/rust-macro-for-query-json-upgraded/valq.png)

これを4年ほど前に↓の記事で発表したところ、予想以上に大きな反響をいただくことができました。

https://zenn.dev/jiftechnify/articles/rust-macro-for-query-json

その後Rustでそういった類の処理を書く機会が減ったのもあり、しばらくの間放置気味になっていましたが、最近そんなvalqに久々に手を入れて**もっと便利に**してみたので、紹介させてください。

[^1]: スキーマによって厳密に定義された構造のかわりに、「リスト」(順序付きで値が並んだもの)や「オブジェクト」(key-valueペアを束ねたもの)の入れ子からなる「ゆるい」構造をもつデータモデルの総称。JSONの他にYAMLやTOML、CBORやMessaegPackなどがこの範疇に含まれます。cf. [Wikipedia](https://en.wikipedia.org/wiki/Semi-structured_data)


## valqが解決しようとしている課題

新機能の紹介に入る前に、改めてvalqクレートがどんな問題を解決するものなのか、そして今回の新機能が入る前のvalqがどんな機能を提供していたのかについて触れておきます。

Rustで半構造化データから値を抽出したい場合、[serde](https://serde.rs/)クレートを使ってデータをRustの構造体にデシリアライズするのが常套手段となっています。しかし、この方法ではデシリアライズしたいデータ構造ごとに別々の構造体をいちいち定義していく必要があり、場合によっては**非常に骨が折れる作業になります**。

具体例を挙げて説明します。今ここに、そのへんのSNSに生えているWeb APIが返してきそうな感じのJSONデータがあります。

```json
{
    "status": 200,
    "data": {
        "user": {
            "id": 12345,
            "name": "jiftechnify",
            "profile": {
                "age": 31,
                "occupation": "SWE",
                "hobbies": ["rhythm games", "podcasts", "walking"],
            },
        },
        "other": { ... }
    }
}
```

このJSONから `occupation` の値だけを取り出したければ、ネストしたオブジェクトそれぞれに対応する構造体を、1つずつ地道に定義していかねばなりません。

:::details 地道な構造体定義の例
```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct ApiResponse {
    data: ResponseData,
}

#[derive(Deserialize)]
struct ResponseData {
    user: User,
}

#[derive(Deserialize)]
struct User {
    profile: UserProfile,
}

#[derive(Deserialize)]
struct UserProfile {
    occupation: String,
}

fn main() {
    let json = "{...}";
    let api_resp: ApiResponse = serde_json::from_str(json)?;
    println!("user's occupation: {}", api_resp.data.user.profile.occupation);
}
```
:::

この方法の問題点を思いつく限り挙げてみましょう。

- 今後使うかもわからない**中間構造体を大量に定義しなければならない**
- Rustではフィールド名つきの無名構造体を定義できないため、**各中間構造体にいちいち名前をつける必要がある**
  - 「モノに対する命名」は、[コンピューターサイエンスにおける2つの難問](https://martinfowler.com/bliki/TwoHardThings.html)のうちの1つとして知られている
- データのスキーマが事前に分からない、あるいは頻繁に変化するような場合、そもそも正しく構造体を定義するのが困難

実は、半構造化データの奥深くにネストした値を取り出すのが地味に面倒臭いというのは、Rustコミュニティで度々話題になっているRustのPain Pointの1つだったりします。

https://users.rust-lang.org/t/how-to-parse-json-using-serde-where-target-values-are-deeply-nested/63306
https://users.rust-lang.org/t/retrieve-deeply-nested-value-with-serde/107715
https://www.reddit.com/r/rust/comments/txsguq/best_way_to_extract_data_with_serde_from_a_large/


**もう少しマシな方法はないのでしょうか?** JSONの「生」の構造をRustの値として表現する、[serde_json](https://docs.rs/serde_json/latest/serde_json/index.html)クレートの`serde_json::Value`が使えそうです。`serde_json::Value`から内部の値を抽出する手段は3つあります。

- **`get()`メソッド**で、オブジェクトのキーや配列のインデックスを指定して値を取り出す。このメソッドは、指定した場所に値が存在しない可能性を表現するために`Option<&Value>`を返す。そのため、**ネストした値を取り出すには、`and_then()`を連鎖させたり`?`演算子とクロージャの即時実行を組み合わせたりする必要があり**、冗長になりがち

    ```rust
    // 「get() + and_then()の連鎖」の例
    let v: serde_json::Value = serde_json::from_str(json)?;
    let occupation: Option<&str> = v.get("data")
        .and_then(|v| v.get("user"))
        .and_then(|v| v.get("profile"))
        .and_then(|v| v.get("occupation"))
        .and_then(|v| v.as_str());
    ```

    :::details 「get() + ?演算子 + クロージャの即時実行」の例
    ```rust
    let occupation: Option<&str> = (|| {
        Some(
            v.get("data")?
            .get("user")?
            .get("profile")?
            .get("occupation")?
            .as_str()?
        )
    })();
    ```
    :::

- **角括弧`[]`によるインデックスアクセス**を使えば、Pythonなどと同様の簡潔なコードで値を取得できる。しかし、指定した場所に値が存在しない場合、`None`ではなくJSONの`null`に対応する`Value::Null`を返す[仕様](https://docs.rs/serde_json/latest/serde_json/value/enum.Value.html#impl-Index%3CI%3E-for-Value)となっているため、**「フィールドが存在しない場合」と「値が`null`のフィールドが存在する場合」を区別できない**

    ```rust
    // インデックスアクセスの例
    let occupation = v["data"]["user"]["profile"]["occupation"].as_str();
    ```

- **`pointer()`メソッド**を使い、JSON Pointer記法で場所を指定して値を取り出す。引数のJSON Pointerが妥当かどうかはコンパイル時にチェックされないため、**文法的に間違ったJSON Pointer文字列を渡してしまう**リスクがある。また、結果が`None`だった場合に、**値が存在しないのか、JSON Pointerの文法が間違っているのかを区別できない**

    ```rust
    // pointer() の例
    let occupation = v.pointer("/data/user/profile/occupation")
        .and_then(|v| v.as_str());
    ```

以上の通り、**いずれの方法にも微妙な問題が潜んでいます**。

逆に、それぞれの方法のいいとこどりができたら、めちゃくちゃ便利そうだとは思いませんか?

**もし、インデックスアクセスやJSON Pointerのような簡潔な記法を、`get()`と`and_then()`を組み合わせたコードに変換できたとしたら…?**

まさにそれを実現するのが、valqクレートが提供する **`query_value!`マクロ**なのです。

## valq v0.1.0 時点の機能、そして限界

[前回の記事](https://zenn.dev/jiftechnify/articles/rust-macro-for-query-json)の時点(v0.1.0)で、valqの`query_value!`マクロに搭載されていた機能は以下の通りです。

- JavaScript(あるいは[JSONPath](https://en.wikipedia.org/wiki/JSONPath))風のドット記法・角括弧記法から構成される**クエリ**を、**`get()`+`and_then()`の連鎖**に展開
- クエリの後に`-> <型>`をつけた場合、取得結果を`as_<型>()`メソッドで特定の型に「キャスト」するコードに展開される
  - 例: `... -> str` → `... .and_then(|v| v.as_str())`

この`query_value!`マクロを使うと、先ほどの例は以下のように書けます。

```rust
use valq::query_value;
let occupation = query_value!(v.data.user.profile.occupation -> str);
```

このマクロ呼び出しが、コンパイル時に上記の「`get()` + `and_then()` の連鎖」の例と同等のコードに展開されるというわけです[^2]。

実は、**先ほど挙げた問題点はこれでほぼ解決できています**。

- 不要な中間構造体を定義する必要がなくなった
- データスキーマが不明な場合や頻繁に変化する場合にも、柔軟に対応しやすい
- JavaScriptと同等の簡潔さを持つ記法と安全性を両立
    - クエリの文法がコンパイル時にチェックされる
    - 指定した場所に値が存在しない場合と、`null`が存在する場合を区別可能。前者の場合は`None`が、後者の場合は`Some(Value::Null)`が返る

**ですが**、人間という生き物は欲深い生き物で、一定の利便性を手に入れるともう一段「上」を求めたくなってしまうものです。この`query_value!`マクロにも、改善できそうな点がいくつも残っています。

- 取り出した値を**任意の型に変換できない**
    - `as_<型>()`メソッドは、`u64`などの数値型や`&str`, `bool`, そしてJSON配列(`Value::Array`)やJSONオブジェクト(`Value::Object`)への軽量な「キャスト」にしか対応していない
- クエリが失敗した場合の**デフォルト値を指定する仕組みがない**
    - JavaScriptのnull合体演算子(`??`)のようなものが欲しい！
- クエリが複数の理由で失敗しうるにもかかわらず、返り値が`Option`なため**失敗の原因を区別できない**
    - 具体的には、「クエリで指定した場所に値が存在しない場合」と「型変換が失敗した場合」を区別できない

今回追加した新機能は、これらの課題を解決するためのものとなっています。

[^2]: 実装の詳細については[前回の記事](https://zenn.dev/jiftechnify/articles/rust-macro-for-query-json)で詳しく説明しています。複雑な宣言的マクロ実装の実践的なチュートリアルになっているので、興味のある方はぜひご一読ください。

## 新機能の紹介

さて、本題に入りましょう。v0.1.0から本稿執筆時点の最新版([v0.3.0](https://docs.rs/valq/0.3.0/valq/))までの間に追加された新機能を順に紹介します。

### `>>`演算子: 任意の型へのデシリアライズ

クエリの後ろに **`>> (<型>)`** [^3]をつけると、指定した型が`serde::Deserialize`トレイトを実装している前提で、抽出した値を`<型>::deserialize()`関数によってデシリアライズするようなコードに展開されます。

:::details >>演算子の展開イメージ
```rust
// 展開前
query_value!(v.aaa.bbb >> (MyStruct))

// 展開後
v.get("aaa")
    .and_then(|v| v.get("bbb"))
    .and_then(|v| <MyStruct>::deserialize(v.clone()).ok())
```
:::

例えば、先ほどのJSONデータから、`profile`内の`occupation`フィールドだけでなく`profile`全体を取得したくなった場合、こんな感じに書けます。**必要最低限の構造体だけの定義で済んでいる**点に注目です。

```rust
use serde::Deserialize;
use valq::query_value;

#[derive(Deserialize, Debug)]
struct UserProfile {
    age: u8,
    occupation: String,
    hobbies: Vec<String>,
}

fn main() {
    let json = "{...}";
    let api_resp: ApiResponse = serde_json::from_str(json)?;
    if let Some(profile) = query_value!(api_resp.data.user.profile >> (UserProfile)) {
        println!("user's profile: {profile:?}");
    }
}
```

これにより、`->`演算子では不可能だった、非常に柔軟な型変換が可能になりました。例えばJSONデータ内の文字列を`&str`ではなく`String`として取得したり、一定の構造を持ったオブジェクトを要素に持つJSON配列をまとめて`Vec<T>`にデシリアライズして取得したり、といったことができます。

```rust
let hobbies = query_value!(api_resp.data.user.profile.hobbies >> (Vec<String>));
```

:::message
`>>`演算子を使ったデシリアライズでは、抽出した値(`Value`)を`clone()`するため、`->`演算子に比べてパフォーマンス面で劣ります。`->`で事足りるのであれば、そちらを使うことをお勧めします。
:::

[^3]: `>>`演算子の後ろに続く型名は、基本的には丸括弧で囲む必要があります。技術的には、これはRustの宣言的マクロの制限に由来するものです。ただし、単一の識別子のみからなる型名(例: `String`, `MyStruct`など)については、特別に丸括弧を省略できるようにしてあります。

### `??`演算子: デフォルト値指定によるunwrapping

クエリの末尾(`->`や`>>`による型変換よりも後ろ)に **`?? <デフォルト値>`** をつけると、

- クエリが成功した場合、その結果
- クエリが失敗した場合、指定したデフォルト値

を返すコードに展開されます。

なお、具体的なデフォルト値を指定するかわりに **`?? default`** と書いた場合は、`Default::default()`がデフォルト値になります。


:::details ??演算子の展開イメージ
具体的なデフォルト値を指定した場合:

```rust
// 展開前
query_value!(v.aaa.bbb -> str ?? "default_value")

// 展開後
v.get("aaa")
    .and_then(|v| v.get("bbb"))
    .and_then(|v| v.as_str())
    .unwrap_or_else(|| "default_value")
```

`default`キーワードを指定した場合:

```rust
// 展開前
query_value!(v.aaa.bbb >> (Vec<String>) ?? default)
// 展開後
v.get("aaa")
    .and_then(|v| v.get("bbb"))
    .and_then(|v| <Vec<String>>::deserialize(v.clone()).ok())
    .unwrap_or_default()
```
:::

```rust
// `occupation`が存在しない場合は、デフォルト値 "unemployed" を返す
let occupation: &str =
    query_value!(v.data.user.profile.occupation -> str ?? "unemployed");

// `hobbies`が存在しない場合は、`Vec<String>::default()`(=空のVec)を返す
let hobbies: Vec<String> =
    query_value!(v.data.user.profile.hobbies >> (Vec<String>) ?? default);
```

ちなみに、`!`演算子を追加して`unwrap()`メソッドによる強制的なunwrappingを行えるようにすることも検討しましたが、[昨今の情勢を鑑み](https://blog.cloudflare.com/ja-jp/18-november-2025-outage/#memorinoshi-qian-ge-ridang-te)、そのような危険な動作を`!`の1文字で引き起こせるのは好ましくないと判断し、見送りました。

### `query_value_result!`: Resultを返す派生版

`Option<T>`の代わりに`Result<T, valq::Error>`を返す`query_value!`の亜種、[**`query_value_result!`マクロ**](https://docs.rs/valq/latest/valq/macro.query_value_result.html)を新たに追加しました。

[`valq::Error`](docs.rs/valq/latest/valq/enum.Error.html)は、クエリが失敗した理由を表現するエラー型で、主要なクエリの失敗理由に対応するvariantを持ちます。クエリが失敗した原因の究明に役立つペイロードも含まれています。

| variant               | クエリ失敗理由                                              | ペイロード |
| --------------------- | ----------------------------------------------------------- | ---------- |
| `ValueNotFoundAtPath` | クエリのパス部で指定した場所に値が存在しなかった            | 値が存在しないパス[^4]           |
| `AsCastFailed       ` | `->`演算子(=`as_***()`メソッド)による「キャスト」が失敗した | キャストメソッド名           |
| `DeserializationFailed`   | `>>`演算子によるデシリアライズが失敗した                    | `deserialize()`が返したエラー           |

さらに、`Result`のエラー型を`valq::Error`に固定した型エイリアス [`valq::Result<T>`](https://docs.rs/valq/latest/valq/type.Result.html) も用意されています。

```rust
use valq::{self, query_value_result};

// 指定パスに値が存在しない例(`.profile`が抜けている)
let occupation: valq::Result<&str> =
    query_value_result!(v.data.user.occupation -> str);
assert_eq!(
    occupation,
    Err(valq::Error::ValueNotFoundAtPath("data.user.profile".to_string())
));

// キャスト失敗の例(数値なのに文字列にキャストしようとしている)
let user_id = query_value_result!(v.data.user.id -> str);
assert_eq!(
    user_id,
    Err(valq::Error::AsCastFailed("as_str".to_string()))
);

// デシリアライズ失敗の例(文字列のJSON配列を数値のVecに変換しようとしている)
let hobbies = query_value_result!(v.data.user.profile.hobbies >> (Vec<u32>));
assert!(matches!(
    hobbies,
    Err(valq::Error::DeserializationFailed(_))
));
```

[^4]: 指定したパスの途中で値を「見失った」場合、見失ったところまでの部分パスが入ります。例えば、`v: Value`から`v.aaa.bbb.ccc`を取得しようとしたが、`v.aaa.bbb`フィールドが存在しなかったという状況では、エラーのペイロードは`"v.aaa.bbb"`になります。

### その他の新機能

- インデックスアクセスを行う角括弧の中に、任意の式を渡せるようになった
    - 従来は、配列状のデータに対してのみ使う想定だったため、数値(`usize`)に評価される式しか渡せなかった

    :::details 具体例
    ```rust
    use serde_json::json;
    use valq::query_value;

    let j = json!({ 
        "<exotic_key>": "<exotic_val>",
        "dict": { "one": 1, "two": 2, "three": 3 },
    });

    // Rustの識別子として使えない文字列がキーのフィールドにアクセスする例
    assert_eq!(
        query_value!(j["<exotic_key>"] -> str).unwrap(),
        "<exotic_val>"
    );

    // 文字列による動的インデクシングの例
    for (i, s) in ["one", "two", "three"].into_iter().enumerate() {
        assert_eq!(
            query_value!(j.dict[s] -> u64).unwrap(),
            (i + 1) as u64
        );
    }
    ```
    :::

- 任意個の`Option`や`valq::Result`を一括でunwrapしてタプルにまとめるヘルパーマクロ、[**`transpose_tuple!`**](https://docs.rs/valq/latest/valq/macro.transpose_tuple.html) を追加
    - 引数に1つでも`None`や`Err`があれば、全体が`None`や`Err`になる

    ```rust
    use valq::{query_value, transpose_tuple};

    let t: Option<(u64, &str, u64)> = transpose_tuple!(
        query_value!(v.data.user.id -> u64),
        query_value!(v.data.user.name -> str),
        query_value!(v.data.user.profile.age -> u64),
    );
    assert_eq!(t, Some((12345, "jiftechnify", 31)));
    ```

valqクレートの各機能についてもっと詳しく知りたい方は、[Docs.rs上のドキュメント](https://docs.rs/valq/latest/valq/)をご覧ください。

https://docs.rs/valq/latest/valq/

## 今後の展望

ますます便利になった**valq**ですが、まだまだ改善の余地があると考えています。今後実装予定の機能をいくつか紹介します(以下のコード例は想像上のものです)。

- `[*]`記法: 配列内の各要素に後続のクエリを適用し、結果をまとめて`Vec`として返す
    - 例えば、`data.items[*].name`というクエリは、配列`data.items`内の各要素から`name`フィールドの値を抜き出す
    
    ```rust
    let j = json!({
        "data": {
            "items": [
                { "id": 1, "name": "hoge" },
                { "id": 2, "name": "fuga" },
                { "id": 3, "name": "piyo" },
            ]
        }
    });
    let item_names = query_value!(v.data.items[*].name >> (String));
    // Result: Some(["hoge", "fuga", "piyo"])
    ```

- 構造体の各フィールドに対してあらかじめクエリを紐づけておいて、クエリ結果を一括で構造体にマッピングする機能
    - 巨大な半構造化データの各所から値を「チェリーピック」するような場面を想定
    - [serde-query](https://github.com/pandaman64/serde-query/)クレートの機能を、`query_value!`マクロの仕組みを使って再実装するイメージ

    ```rust
    use valq::{from_query_results, QueryMapped};

    #[derive(QueryMapped, Debug)]
    struct UserData {
        #[valq("data.user.name")]
        name: String,
        #[valq("data.user.profile.occupation")]
        occupation: String,
    }

    fn main() -> Result<(), ...> {
        let json = "{...}";
        let v: serde_json::Value = serde_json::from_str(json)?;
        let profile: UserProfile = from_query_results!(UserProfile, v)?;
        println!("user's profile: {profile:?}");
        // Output: user's profile: UserProfile { name: "jiftechnify", occupation: "SWE" }
    }
    ```

- 再利用可能クエリ
    - 例えば「`data.user.profile`を抽出するクエリ」をファーストクラスの値として扱い、複数の入力データに繰り返し適用できるようにする
    - Swiftの[Key-Path式](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/expressions/#Key-Path-Expression)や、関数型プログラミングの文脈でいう[Lens](https://techblog.recruit.co.jp/article-561/)のようなもの

    ```rust
    use valq::{query_value, lens};

    let profile_lens = lens!(data.user.profile >> (UserProfile));
    let v1: serde_json::Value = ...;
    let v2: serde_json::Value = ...;

    assert_eq!(
        profile_lens(v1),
        query_value!(v1.data.user.profile >> (UserProfile))
    );
    assert_eq!(
        profile_lens(v2),
        query_value!(v2.data.user.profile >> (UserProfile))
    );
    ```


## おわりに

半構造化データからの値の抽出処理を楽にするRustマクロライブラリ、**valq**のご紹介でした。

Rustでこういった処理を書こうと思うと、どうしても記述が重くなりがちで、億劫だな…と感じていた方は少なくないのではないかと思います。valqは、そんな皆さんの悩みごと(あるいはタイプ数)を減らす助けになってくれるはずです。

valqについては、より便利に・より多くの場面で活用できるクレートになるよう、今後も改善を続けていきたいと考えています。この記事への**Like**や[GitHubリポジトリ](https://github.com/jiftechnify/valq)への**Star**、あるいは**Issue**や**Pull Request**といった形でフィードバックをいただけますと大変励みになります。

最後までお読みいただき、ありがとうございました。

https://github.com/jiftechnify/valq
