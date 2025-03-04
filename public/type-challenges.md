# Type Challengesを解くうえで使えるテクニック集

TypeScriptの複雑な型システムをどれほど理解しているのかの腕試しができる[Type Challenges](https://github.com/type-challenges/type-challenges/)

[PartialByKeys](https://github.com/type-challenges/type-challenges/blob/main/questions/02757-medium-partialbykeys/README.md)のような割と実用的なものから、
[Valid Sudoku](https://github.com/type-challenges/type-challenges/blob/main/questions/35314-hard-valid-sudoku/README.md)のようにこんなのいつ使うんだよ……となるものまで
様々な問題が収録されているType Challengesですが、これらを解く際に使えるテクニックや考え方をご紹介します。

Type Challengesだけではなく、これってどう書けばよかったっけ…？となった際に辞書的に使えるものになっていますので、ぜひご覧ください。

## おことわり
- **一部問題（特にeasy）のガッツリ解答が載っています。**
- TypeScriptの基本的な型システムに関しての説明はありません。用語などは以下の記事などを参照ください。

https://qiita.com/uhyo/items/e2fdef2d3236b9bfe74a

https://qiita.com/uhyo/items/da21e2b3c10c8a03952f

## タプルの各要素を1つずつ処理

### 各要素を他の型に変換する場合（Array.prototype.map相当）
```typescript
type Foo<T> = T extends [infer F, ...infer R]
    ? [Bar<F>, ...Foo<R>]  // Rが空でない場合
    : [];                  // Rが空の場合
```
- `F`に先頭の要素、`R`に残りの要素が入る
  - `R`は空タプルになりうる
- 再帰呼び出しにスプレット演算子をつけることでタプルが階層化するのを防ぐ

### 何らかの条件を満たす要素を探す場合（Array.prototype.find相当）
```typescript
type Foo<T> = T extends [infer F, ...infer R]
    ? Bar<F> extends true
        ? F     // Fが条件を満たす場合の返り値
        : Foo<R>
    : never;    // 条件を満たす要素が存在しない場合の返り値
```
- 返り値`F`や`never`は目的によって適切なものを選択する

### 返す型が単純ではない場合（Array.prototype.reduce相当）
```typescript
type Foo<T, A = Initial> = T extends [infer F, ...infer R]
    ? Foo<R, Bar<A, F>>
    : A;
```
- 型引数を増やすことで途中経過を持たせられるようになる
  - すべての要素を見終わったらそれを返すだけ

## 文字列を1文字ずつ処理
```typescript
type Foo<T> = T extends `${infer F}${infer R}`
    ? `${Bar<F>}${Foo<R>}`
    : "";
```
- `F`に先頭の文字、`R`に残りの文字列が入る
  - `R`は空文字列になりうる

