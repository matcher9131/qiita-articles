# Type Challengesを全て打ち負かしたので、解く際に必要になるテクニックと考え方を解説してみる

TypeScriptの複雑な型システムをどれほど理解しているのかの腕試しができる[Type Challenges](https://github.com/type-challenges/type-challenges/)

[PartialByKeys](https://github.com/type-challenges/type-challenges/blob/main/questions/02757-medium-partialbykeys/README.md)のような割と実用的なものから、
[Valid Sudoku](https://github.com/type-challenges/type-challenges/blob/main/questions/35314-hard-valid-sudoku/README.md)のようにこんなのいつ使うんだよ……となるものまで
様々な問題が収録されているType Challengesですが、この度これを全て打ち負かしてきたので、解く際に必要になるテクニックや考え方をご紹介します。

## おことわり
- **一部問題のヒントがあります。**
- TypeScriptの基本的な型システムに関しての説明はありません。見慣れない用語などは以下の記事などを参照ください。

https://qiita.com/uhyo/items/e2fdef2d3236b9bfe74a

https://qiita.com/uhyo/items/da21e2b3c10c8a03952f

## 三項演算子と再帰呼び出しで全てを制御する
基本的には型引数`T`に対してMapped TypeやConditional Typeによる分岐（`T extends U ? X : Y`）でゴリゴリ書いていくことになります。
ループで何かを処理したくても`for`や`while`構文なんて便利なものはありません。諦めて再帰で書きましょう。

### タプルの要素や文字列の文字を1つずつ処理
タプルに対しては`T extends[infer F, ...infer R]`、文字列に対しては``T extends `${infer F}${infer R}` ``とすることで先頭の要素のみを`F`に、残りを`R`に格納できます。
（この際`R`は空配列あるいは空文字列になり得ます）
```
T extends [infer F, ...infer R]
    ? // タプルが空ではないときの処理
    : // タプルが空のときの処理

T extends `${infer F}${infer R}`
    ? // 文字列が空ではないときの処理
    : // 文字列が空のときの処理
```

これと再帰呼び出しを組み合わせることで、要素を1つずつ処理することができます。
```
// タプルの各要素を2つずつ列挙する型
type Double<T> = T extends [infer F, ...infer R]
    // タプルの残りRに対してDoubleを再帰的に呼び出し
    // （スプレッド演算子を使うことでタプルが階層になるのを防ぐ）
    ? [F, F, ...Double<R>]
    : [];

// type Foo = [1, 1, 2, 2, 3, 3];
type Foo = Double<[1,2,3]>;
```

途中で`break`したい場合は`F`が条件を満たしたら再帰呼び出しをせずに型を返すようにすればいいでしょう。
```
// タプルの中でもっと先頭にある非nullishな要素を返す（ない場合はneverを返す）
type FirstNonNullish<T> = T extends [infer First, ...infer Rest]
    ? First extends undefined | null
        ? FirstNonNullish<Rest>
        : First
    : never;

// type Foo = 1;
type Foo = FirstNonNullish<[null, 1, undefined]>;
// type Bar = never;
type Bar = FirstNonNullish<[undefined]>;
// type Baz = never;
type Baz = FirstNonNullish<[]>;
```

