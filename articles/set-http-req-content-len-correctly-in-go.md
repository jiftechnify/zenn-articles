---
title: "Go: HTTPリクエストのContent-Lengthを正しくセットする方法"
emoji: "📏"
type: "tech"
topics: ["go", "http"]
published: true
---

# 結論

- そもそも、ほとんどの場合はよしなにやってくれるので、`Content-Length`を自分でセットしなければならない場面は非常にまれ
    - `body`の長さが「自明」であれば、その長さが自動的にセットされる 
    - そうでなければ、自動的に`Transfer-Encoding: chunked`で送信される
- `Content-Length`を自分でセットするには、**`http.Request`の`ContentLength`フィールド**に値をセットするのが正解

```go
// ✅
var size int64 = ...
req.ContentLength = size
```

- `Header`に直接`Content-Length`をセットしても**無視される**

```go
// ❌
var size int64 = ...
req.Header.Set("Content-Length", strconv.FormatInt(size, 10))
```

# ほとんどの場合は自分でセットしなくていい

GoでHTTPリクエストを行う処理を書く際に、`Content-Length`のことを気にする必要はほぼありません。これは、標準ライブラリの`http`パッケージが「いい感じ」にやってくれているおかげです。

`http.Request`の`ContentLength`フィールドを明示的にセットしなかった場合、`body`の性質に応じて以下のどちらかの動作をします。

## `body`の長さが「自明」な場合

具体的にいうと、`body`の具体的な型が 

- `*bytes.Buffer`
- `*bytes.Reader`
- `*strings.Reader`

のうちのいずれかの場合です。これらの型の値は長さの情報を持っており、`Len()`メソッドで簡単に取得できます。

この場合は、`http.NewRequest(WithContext)`内で`body.Len()`の値が`ContentLength`フィールドに自動的にセットされます([コード](https://cs.opensource.google/go/go/+/refs/tags/go1.19.4:src/net/http/request.go;l=896))。

## `body`の長さが「自明」ではない場合

上記以外の場合が該当します。

この場合は、`Transfer-Encoding: chunked`を利用し、リクエストボディを小分けにして送信するようになります。この方法には、リクエストボディ全体の長さ(= `Content-Length`)が事前に分からなくてもよいという特長があります。


まとめると、送信するデータの長さが事前に分かっているならそれが自動的に設定されるし、長さが事前に分からなければ「長さの情報を事前に送信しなくてもいい方法」でリクエストを行うようになっている、ということです。

# 自分でセットしないといけない場合

以上を踏まえると、`Content-Length`ヘッダを自分でセットしなければならない状況というのは、**`body`の長さが自明でなく、かつ事前(リクエストボディを送り始める前)にリクエストボディ全体の長さをサーバに送信しなければならない場合**に限られます。

後半の条件に当てはまる場合、サーバは[`411 Length Required`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/411)というエラーレスポンスを返すことになっています。筆者はS3のpresigned URLを使ってファイルをアップロードしようとした際にこのエラーレスポンスに遭遇しました。

# `Header`の罠

`411`エラーというのはめったにお目にかかるものではありませんが、`Length Required`というメッセージから「`Content-Length`ヘッダをセットしてあげればよさそうだ」と推測できます。しかし、ここで焦って以下のように修正してしまうと、問題は思ったように解決してくれません。

```go
var size int64 = ...
req.Header.Set("Content-Length", strconv.FormatInt(size, 10))
resp, err := httpCli.do(req)
```

なぜなら、**`http.Request`の`Header`に直接セットされた`Content-Length`ヘッダの内容は無視されてしまう**からです。この場合は当該ヘッダをセットしなかったときと同じ挙動となるため、「`Content-Length`をセットしたはずなのに`Length Required`と怒られる」という一見不可解な状況に陥ります。

焦らず、落ち着いて、**`http.Request`の`ContentLength`フィールド**をセットしましょう。これで`Content-Length`ヘッダが意図通りに送信されます。

```go
var size int64 = ...
req.ContentLength = size
resp, err := httpCli.do(req)
```

分かってしまえば「なんだそんなことか」という感じですが、めったに見ないエラーの解決の鍵が普段あまり意識していないところにあるとなると、意外と気づけないものです。

# ちょっと深掘り
## ドキュメントを読む
この`Content-Length`ヘッダまわりの挙動については、`http`パッケージのドキュメントコメントに断片的に記されています。

- [`http.Request.Header`](https://pkg.go.dev/net/http#Request)フィールドのドキュメントより抜粋:

> For client requests, certain headers such as Content-Length and Connection are automatically written when needed and values in Header may be ignored. See the documentation for the Request.Write method.

> (抄訳)
> クライアントリクエストにおいて、Content-LengthやConnectionといったヘッダは必要に応じて自動的に書き込まれ、Headerに設定した値が無視されることがある。Request.Writeメソッドのドキュメントを参照のこと。


- [`http.Request.Write`](https://pkg.go.dev/net/http#Request.Write)メソッドのドキュメントより抜粋:

> This method consults the following fields of the request:
>
> ```
> Host
> URL
> Method (defaults to "GET")
> Header
> ContentLength
> TransferEncoding
> Body
> ```

> (抄訳)
> このメソッドはリクエストのフィールドのうち以下のものを考慮する:
>
> (略)

## コードを読む
さらに、`http`パッケージの関連コードを追うことで、実際に送信される`Content-Length`ヘッダは基本的に`http.Request.ContentLength`フィールドの値のみに基づいて決まり、`Header`に直接セットされた値は完全に無視されることがわかります。

:::details Content-Lengthヘッダを書き込む処理の概要
該当箇所: `http.Request.write`メソッド(`Write`の内部処理、[コード](https://cs.opensource.google/go/go/+/refs/tags/go1.19.4:src/net/http/request.go;l=551))

1. `newTransferWriter()`で、`transferWriter`を生成([コード](https://cs.opensource.google/go/go/+/refs/tags/go1.19.4:src/net/http/request.go;l=645))。これは`Request`の内容をHTTPプロトコルに則って書き込むメソッドを持つ。
    - ここで、`Content-Length`ヘッダの値に対応する`transferWriter.ContentLength`の値は`Request.ContentLength`のみを考慮して設定される([コード](https://cs.opensource.google/go/go/+/refs/tags/go1.19.4:src/net/http/transfer.go;l=93))
2. `transferWriter.writeHeader`メソッドで`Content-Length`ヘッダを実際に書き込む([コード](https://cs.opensource.google/go/go/+/refs/tags/go1.19.4:src/net/http/transfer.go;l=290))
3. `Header.writeSubset`メソッドで`transferWriter`が関知しないヘッダを書き込む([コード](https://cs.opensource.google/go/go/+/refs/tags/go1.19.4:src/net/http/request.go;l=654))が、このとき`Header`にセットされた`Content-Length`は無視される(`reqWriteExcludeHeader`([コード](https://cs.opensource.google/go/go/+/refs/tags/go1.19.4:src/net/http/request.go;l=89))に含まれるため)
:::

## 実験してみる
様々な設定のもとでHTTPリクエストを送信して内容を観察・比較することで、HTTPリクエスト関連の挙動に対する理解を深めることができました。実験プログラムをGitHubに置いておいたので、興味のある方はぜひ動かしてみてください。

https://github.com/jiftechnify/go-httpcli-req-observation

# おわりに
GoでHTTPリクエストを行う際に`Content-Length`ヘッダを正しく設定する方法、あるいは`http.Request.Header`にセットした`Content-Length`が無視されるという罠についてまとめました。

読者の方々が同様の問題に遭遇した際に、冷静に対処する助けになれば幸いです。

# 参考文献
- [Sending Content-Length in Go](https://medium.com/ne-digital/sending-content-length-in-go-12a1fcf41251)

:::message
この記事は [Go Advent Calendar 2022](https://qiita.com/advent-calendar/2022/go) の11日目の記事(代筆)です。
:::
