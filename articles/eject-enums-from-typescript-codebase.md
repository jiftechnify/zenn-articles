---
title: "TypeScriptコードから一撃でenumを「追放」するツールを作った"
emoji: "⏏️"
type: "tech" 
topics: ["TypeScript"]
published: false
---

:::message
この記事は、[TypeScript Advent Calendar 2025](https://qiita.com/advent-calendar/2025/typescript) の21日目の記事です。
:::

## TL; DR

TypeScriptの`enum`を[等価なオブジェクトリテラル+union型の組](https://www.typescriptlang.org/docs/handbook/enums.html#objects-vs-enums)に一括で書き換えるツール、その名も[`eject-enum`](https://github.com/jiftechnify/eject-enum)を作りました。

コマンド一発で、あなたのTSコードベースから`enum`を「追放(eject)」できます。

```bash
npx eject-enum
```

## 背景

TypeScriptの`enum`を避けるべき理由についてはさんざん語り尽くされているので詳しくは説明しませんが、一言でいえば **「JavaScriptに型情報を付加するだけ」というTypeScriptの設計思想から逸脱する存在である**、というのが最大の問題です。

https://typescriptbook.jp/reference/values-types-variables/enum/enum-problems-and-alternatives-to-enums
https://zenn.dev/rikson/articles/2025-08-13_rikson_let-me-use-enums-in-typescript#enum%E3%81%B8%E3%81%AE%E6%89%B9%E5%88%A4

最近のNode.jsでは[TypeScriptのコードをそのまま実行](https://nodejs.org/ja/learn/typescript/run-natively)できるようになりましたが、これは内部的には**TypeScriptの型注釈を剥がしてJavaScriptとして実行**しているにすぎません(type stripping)。ところが、`enum`をはじめとするTypeScript独自構文[^1]は単純には剥がせないため、現在のNode.jsではそういった構文が使われたコードは実行できません。TypeScript 5.8では、Node.jsがそのまま実行できないコードをエラーとみなすコンパイラオプション `erasableSyntaxOnly` が導入される始末です。

https://zenn.dev/ubie_dev/articles/ts-58-erasable-syntax-only

さて、`enum`構文を使わずに列挙型を表現する手法はいくつかありますが、その中で`enum`を最も「忠実」に再現できるのは、以下のように**オブジェクトリテラルとそこから導出されるunion型を組み合わせる**手法です。これはTypeScript Handbookでも紹介されており、広く受け入れられている印象です。

```ts
// オリジナルのenum
enum Direction {
  Up,
  Down,
  Left,
  Right,
}
```
```ts
// 「代替手法」で書き換えたもの
const Direction = {
  Up: 0,
  Down: 1,
  Left: 2,
  Right: 3,
} as const;

type Direction = (typeof Direction)[keyof typeof Direction]; // 0 | 1 | 2 | 3
```

これで万事解決―――**本当にそうでしょうか?** 

特に、`typeof`や`keyof`を使ってunion型を導出する部分はそれなりに複雑で、既存のコードベースに`enum`が大量に散らばっている場合、いちいち書き換えるのは非常に面倒です。また、生成AIにTypeScriptのコードを書かせたら`enum`を使ってきた、なんて話もちらほら耳にします。

この世のすべての`enum`をまとめて葬り去る"力"が欲しい――― **`eject-enum`は、そんなあなたの望みを叶えてくれます**。

[^1]: `enum`のほかに、`namespace`や[クラスのパラメータプロパティ](https://www.typescriptlang.org/docs/handbook/2/classes.html#parameter-properties)などが該当

## 使い方
### CLIツールとして

シンプルなプロジェクトであれば、`npx eject-enum`と唱えるだけでOKです。
デフォルトでは、カレントディレクトリにある`tsconfig.json`がincludeするすべてのTypeScriptが書き換え対象となります。もちろん、`pnpx`や`bunx`、[`deno x`(`dx`)](https://deno.com/blog/v2.6)などでも動作します。

オプションで、`tsconfig`のパスを指定したり、globにマッチするファイルだけを書き換え対象にしたり、逆に対象から除外したりもできます。

```bash
# tsconfigのパスを指定
npx eject-enum --project path/to/tsconfig.json

# globによるファイル指定
# srcディレクトリ以下の.tsファイルのうち、テストファイル以外のすべてを書き換える
npx eject-enum --include "src/**/*.ts" --exclude "src/**/*.test.ts"
```

### ライブラリとして

`eject-enum`は、JS/TSスクリプトからライブラリとして使うこともできます。他の開発ツールとの統合などに使えるかもしれません。

```bash
npm install eject-enum
```

```ts
import { ejectEnum, EjectEnumTarget } from "eject-enum";

// tsconfigのパスを指定
await ejectEnum(
    EjectEnumTarget.projects([
        "path/to/tsconfig.json",
    ]),
);

// globによるファイル指定
// srcディレクトリ以下の.tsファイルのうち、テストファイル以外のすべてを書き換える
await ejectEnum(
    EjectEnumTarget.srcPaths({
        include: ["src/**/*.ts"],
        exclude: ["src/**/*.test.ts"],
    }),
);
```

## アピールポイント
### enumメンバーを型として参照するコードにも対応
`enum`のすべてのメンバーがリテラル型に推論できる場合、各メンバーをリテラル型のように使える仕様が存在します(enum member types)。例えば、discriminated unionのタグとして`enum`メンバーを使ったりできます。

https://www.typescriptlang.org/docs/handbook/enums.html#union-enums-and-enum-member-types

```ts
// Before
enum ShapeKind {
    Circle = "circle",
    Square = "square",
}

interface Circle {
    kind: ShapeKind.Circle;
    radius: number;
}

interface Square {
    kind: ShapeKind.Square;
    sideLength: number;
}

type Shape = Circle | Square;
```

単純に`enum ShapeKind`だけをオブジェクトに書き換えると、オブジェクトのフィールドは型としては使えないため、コードが壊れてしまいます。

`eject-enum`は、このような「enumメンバーをリテラル型として参照している部分」を検出し、`typeof`をつけることで、元のコードの挙動を維持します。

```ts
// After
const ShapeKind = {
    Circle: "circle",
    Square: "square",
} as const;
type ShapeKind = (typeof ShapeKind)[keyof typeof ShapeKind];

interface Circle {
    kind: typeof ShapeKind.Circle;
    radius: number;
}

interface Square {
    kind: typeof ShapeKind.Square;
    sideLength: number;
}

type Shape = Circle | Square;
```

### 低侵襲
`eject-enum`は、可能な限り既存のコードを尊重した書き換えを行うように設計されています。

- `enum`の代わりとして単純なunion型を使う方法もあるが、`eject-enum`はあえてオブジェクトリテラル+union型の組を生成する。これは、(上記のような例外を除き)`enum`を使う側のコードを変更せずに済むようにするため
- `enum`自体や、各メンバーについたコメントをできるだけ保存する
- `enum`メンバーの値が定数式で初期化されている場合、元の定数式をコメントとして残す
    - オブジェクトリテラルのフィールドを定数式で初期化した場合リテラル型ではなく`number`や`string`型に推論されてしまうため、そのまま書き換えられないという問題があり、それに対する苦肉の策

```ts
// Before
enum ConstExprs {
    A = 1 + 2 + 3,
    B = "hello" + " " + "world",
}

// After
const ConstExprs = {
    A: 6, // 1 + 2 + 3
    B: "hello world", // "hello" + " " + "world"
} as const;
type ConstExprs = (typeof ConstExprs)[keyof typeof ConstExprs];
```

## 実装詳細、あるいはハマり話

[ts-morph](https://ts-morph.com/)を使ってTypeScriptのASTをガリガリ書き換える形で実装しています。具体的にどんな実装になっているかについては[ソースコード](https://github.com/jiftechnify/eject-enum/blob/0.5.1/src/EjectEnum.ts#L135)を覗いていただければと思います。

ts-morphでコードの書き換え処理を書く体験は、TypeScript Compiler APIを直接使って生ASTをそのまま扱うよりは間違いなくマシだが、Rustとかにあるリッチなマクロ機構を触ったことがある身としてはどこか物足りない、みたいなところです。特に、ASTを巨大な可変状態として扱うような作りになっており、ある書き換えが即座に反映されて後続の処理に影響を及ぼす点に苦労させられました。

例えば、enumメンバーをリテラル型として扱うコードを書き換えるにあたり、「`enum`本体」と「それを利用する箇所」の2箇所を変更する必要があるのですが、ここで「`enum`本体 → 利用箇所」の順で書き換えてしまうと、利用箇所の書き換え時点で`enum`本体がすでにオブジェクトリテラルに変わってしまっているため、正しく型参照を検出できなくなってしまいます。これに気づくまでに数時間かかりました…


## 余談: `as enum`は福音となるか?

オブジェクトリテラルに`as enum`というアサーションをつけられるようにして、これをつけるとTypeScript上で`enum`と同様に振る舞うようにしよう、という提案がなされています。

https://github.com/Microsoft/TypeScript/issues/60790

JavaScriptとして実行したければ単純に`as enum`を剥がすだけで済むという寸法ですね。現状のworkaroundに比べて見た目もスッキリしています。個人的にはかなり良いアイデアだと思うので、ぜひ採用されてほしいところです。

## おわりに

TypeScriptのコードベースから`enum`を一掃するのに役立つツール、`eject-enum`の紹介でした。

実はこのツールは3年ほど前から作り始めて、中途半端な状態で放置していたものだったりします。type strippingでTypeScriptを任意の場所で動かそうという機運が高まっている今だからこそ輝くところがあるのではないかと思い、ここ数日で一気にfeature completeな状態に持っていきました。

ぜひ、`eject-enum`をTypeScriptコードベースの改善の一手にお役立てください。バグ報告や機能要望などもお待ちしております。

https://github.com/jiftechnify/eject-enum
