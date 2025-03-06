# Type Challengesを解く際に使えるテクニック集

TypeScriptの複雑な型システムをどれほど理解しているのかの腕試しができる[Type Challenges](https://github.com/type-challenges/type-challenges/)

[PartialByKeys](https://github.com/type-challenges/type-challenges/blob/main/questions/02757-medium-partialbykeys/README.md)のような割と実用的なものから、
[Valid Sudoku](https://github.com/type-challenges/type-challenges/blob/main/questions/35314-hard-valid-sudoku/README.md)のようにこんなのいつ使うんだよ……となるものまで
様々な問題が収録されているType Challengesですが、これらを解く際に使えるテクニックや考え方をご紹介します。

Type Challengesだけではなく、これってどう書けばよかったっけ…？となった際に辞書的に使えるものになっていますので、ぜひご覧ください。


## おことわり
- **一部問題（特にeasy）の解答がガッツリ載っています。**
- TypeScriptの型システムに関しての基本的な説明はありません。不明な点は以下のuhyoさんの記事を参照ください。

https://qiita.com/uhyo/items/e2fdef2d3236b9bfe74a

https://qiita.com/uhyo/items/da21e2b3c10c8a03952f


## タプル関連

### タプルの各要素を変換する（Array.prototype.map相当）
再帰呼び出しを用います。
```typescript
type Foo<T> = T extends [infer F, ...infer R]
    ? [Bar<F>, ...Foo<R>]  // Rが空でない場合
    : [];                  // Rが空の場合
```
`F`に先頭の要素、`R`に残りの要素（空になりうる）が入ります。再帰呼び出しにスプレット演算子をつけることでタプルが階層化するのを防ぎます。

なお再帰呼び出しの部分が複雑になる場合は、以下のように型引数を増やして途中経過を持たせるようにすると再帰呼び出しの制限`"Type instantiation is excessively deep and possibly infinite"`にかかりにくくなります。
```typescript
type Foo<T, A extends unknown[] = []> = T extends [infer F, ...infer R]
    ? Foo<R, [...A, Bar<F>]>
    : A;
```
この`A`を持たせるというのはType Challengesでは頻出のテクニックですが、ユーティリティ型として公開する場合には`A`に変なものを入れられる可能性があるためラップしたほうが無難です。

### タプル内で何らかの条件を満たす要素を探す（Array.prototype.find相当）
```typescript
type Foo<T> = T extends [infer F, ...infer R]
    ? Bar<F> extends true
        ? F     // Fが条件を満たす場合の返り値
        : Foo<R>
    : never;    // 条件を満たす要素が存在しない場合の返り値
```
返り値`F`や`never`は目的によって適切なものを選択してください。

### タプルの各要素を処理して複雑な型を返す（Array.prototype.reduce相当）
```typescript
type Foo<T, A = Initial> = T extends [infer F, ...infer R]
    ? Foo<R, Bar<A, F>>
    : A;
```
Array.prototype.map相当の後半で紹介したものを少しいじっただけです。`Bar<A, F>`の部分がArray.prototype.reduceにおけるコールバック関数に相当します。

## 2つのタプルの長さを比較する
`T extends [infer F, ...infer R]`を両方のタプルに対して行うことでその長さの比較ができます。
```typescript
// Xの長さがYの長さよりも大きければtrue、そうでなければfalseを返す
type LongerThan<X extends unknown[], Y extends unknown[]>
    = X extends [infer XF, ...infer XR]
        ? Y extends [infer YF, ...infer YR]
            // XもYも空ではない
            ? LongerThan<XR, YR>
            // Xは空ではないがYは空である
            : true
        // Xが空である
        : false;

type Foo = LongerThan<[1, 2, 3], ["foo", "bar"]>;  // type Foo = true
type Bar = LongerThan<[], [1]>;                    // type Bar = false
type Baz = LongerThan<["baz"], [true]>;            // type Baz = false
```
上記例の`false`を返すところで再び`Y extends [infer YF, ...infer YR]`を行えば、「長いか、そうでないか」の2値ではなく「長いか、同じか、短いか」の3値を返すことができます。

## 非負整数Nを長さNのタプルに変換する
これ自体は大して意味がありませんが、変換することによって大小比較や加算などができるようになります。[^n-to-t]

[^n-to-t]: それも大して意味があるわけではないという指摘は聞こえません
```typescript
type NumberToTuple<N extends number, A extends unknown[] = []> = A["length"] extends N
    ? A
    : NumberToTuple<N, [...A, unknown]>;

// LongerThan<X, Y>は「2つのタプルの長さを比較する」のものと同一
type GreaterThan<X extends number, Y extends number> = LongerThan<NumberToTuple<X>, NumberToTuple<Y>>;

type Foo = GreaterThan<5, 3>;     // type Foo = true
type Bar = GreaterThan<2, 2>;     // type Bar = false
type Baz = GreaterThan<-1, 0>;    // Error: Type instantiation is excessively deep and possibly infinite.
type Qux = GreaterThan<1000, 0>;  // Error: Type instantiation is excessively deep and possibly infinite.

type Add<X extends number, Y extends number> = [...ToTuple<X>, ...ToTuple<Y>]["length"];

type Quux = Add<3, 5>;  // type Quux = 8
```
タプルの長さのみが重要なので`A`の中身に関しては何でも構いません。`unknown, 0, 1`あたりが用いられることが多いようです。

なお $N < 0$ の場合`A["length"] extends N`が`true`になることはないため、再帰呼び出しが止まらず回数制限を迎えます。  
$N \geq 1000$ だとそもそも素で再帰呼び出しの回数制限に引っかかります。


## 文字列関連

### 文字列リテラル型を1文字ずつ処理する
```typescript
type Foo<T> = T extends `${infer F}${infer R}`
    ? `${Bar<F>}${Foo<R>}`
    : "";
```
`F`に先頭の文字、`R`に残りの文字列（空になりうる）が入ります。

### 文字列リテラル型を数値リテラル型に変換する
Template Literal Types内で`infer N extends number`とすることで数値型への変換が可能です。
```typescript
type Parse<S extends string> = S extends `${infer N extends number}`
    ? N
    : never;

type Foo = Parse<"42">;       // type Foo = 42
type Bar = Parse<"-3.14">;    // type Bar = -3.14
type Baz = Parse<"0xFF">;     // type Baz = number
type Qux = Parse<"2.5e2">;    // type Qux = number
type Quux = Parse<"0123">;    // type Quux = number
type Corge = Parse<"0.10">;   // type Corge = number
type Grault = Parse<"42n">;   // type Grault = never
type Garply = Parse<"0o91">;  // type Garply = never （※8進数に使えない数字がある）
type Waldo = Parse<"NaN">;    // type Waldo = never
```
ただし、数値リテラル型に変換する場合は「10進表記」かつ「浮動小数点表記ではない」かつ「余分なゼロが存在しない」ような文字列を与える必要があります。  
上記を満たさないものの数値として解釈可能な文字列が与えられた場合は単に`number`型が返ります。


## オブジェクト関連

### オブジェクトの交差型を1つのオブジェクトにまとめる
以下の2つの型は実質的に同じで相互に代入可能であるにもかかわらず、Type Challengesの正誤判定に用いられる`Equal<X, Y>`では別物とみなされてしまいます。    
それだけではなく、`Foo`はVS Codeのインテリセンスにおける表示も非常に見づらいという難点があります。
```typescript
type Foo = { x: string; } & { y: number; };
type Bar = {
    x: string;
    y: number;
};
```
この`Foo`を`Bar`のように展開するには、単にMapped Typesに通せばOKです。optionalやreadonlyなプロパティも問題なく処理できます。
```typescript
type FlattenObject<T> = {
    [P in keyof T]: T[P];
};

// type Baz = {
//     x: string;
//     y: number;
// }
type Baz = FlattenObject<Foo>;
```

### Mapped Typeのプロパティ名を変換する
Mapped Typeの`[]`内で`as`を用いることでプロパティ名を操れます。
```typescript
// Tが文字列型あるいは数値型ならUを先頭にくっつけた文字列型を返す
type Prepend<T, U extends string> = T extends string | number
    ? `${U}${T}`
    : T;

// オブジェクトの各プロパティ名にprefixをつける
type WithPrefix<T, Prefix extends string> = {
    [K in keyof T as Prepend<K, Prefix>]: T[K];
};

// type Foo = {
//     bazfoo: string;
//     baz0: number;
// }
type Foo = WithPrefix<{ foo: string; 0: number; }, "baz">;
```
このケースにおける`as`は型アサーションとは異なり、任意の型に好き勝手に変換できます。どちらかといえばプロパティ名に対するmapをたまたま同じキーワードの`as`が担っていると考えたほうがいいかもしれません。

なお、`as`以降で`extends`を使うことで条件で分岐させることができます。（後述）

### Mapped Typeで一部プロパティを除外する
Mapped Typeのプロパティ名を`never`にすると、そのプロパティはオブジェクトから削除されます。
```typescript
// オブジェクトからプロパティ名がアンダースコアで始まるものを削除する
type RemovePrivate<T> = {
    [K in keyof T as K extends `_${string}`
        ? never
        : K
    ]: T[K];
};

// type Foo = { foo: string; }
type Foo = RemovePrivate<{ foo: string; _bar: number; }>;
```

注意点として、プロパティ名ではなく値をneverにしてもオブジェクトからは削除されません。値によって条件分岐させる場合でもあくまでプロパティ名側で`extends`させる必要があります。
```typescript
// オブジェクトから値が数値型のプロパティを削除する

// 値をneverにしてしまっている（間違い）
type WrongRemoveNumberProperty<T> = {
    [K in keyof T]: T[K] extends number
        ? never
        : T[K];
};
// プロパティ名をneverにしている（正しい）
type RemoveNumberProperty<T> = {
    [K in keyof T as T[K] extends number
        ? never
        : K
    ]: T[K];
};

// type Foo = {
//     foo: string;
//     bar: never;
// }
type Foo = WrongRemoveNumberProperty<{ foo: string, bar: 0 | 1 | 2 }>;
// type Bar = { foo: string; }
type Bar = RemoveNumberProperty<{ foo: string, bar: 0 | 1 | 2 }>
```


## ユニオン型関連

### ユニオン型に含まれる要素を1つずつ処理する
```typescript
type Foo<T> = T extends T
    ? Bar<T>
    : never;  // T == neverのとき
```
`T extends T`でUnion Distributionを発生させることで1つずつ見ていくことが可能になります。[^t-ext-t]

[^t-ext-t]: もちろん`T extends unknown`でも構いませんが、`T extends T`と書くことで意図的にUnion Distributionを生じさせていることがわかりやすくなると思います。

わざわざ`T extends T`としなくても`Bar<T>`の部分でUnion Distribution自体は発生しますが、`Bar<T>`の内部で`T`が2回以上使われた場合にユニオン内の全ての組み合わせを網羅するため、1つずつ処理したい場合は`T extends T`で先にUnion Distributionを起こしておくのが無難です。（逆に言えば、`Bar<T>`の内部で`T`が1度しか使われない場合は`T extends T`は不要です。）
```typescript
// 与えられた文字列を2度繰り返す
type LooseRepeat<T extends string> = `${T}${T}`;
type StrictRepeat<T extends string> = T extends T
    ? `${T}${T}`
    : never;

// type LooseResult = "foofoo" | "barbar" | "foobar" | "barfoo"
type LooseResult = LooseRepeat<"foo" | "bar">;
// type StrictResult = "foofoo" | "barbar"
type StrictResult = StrictRepeat<"foo" | "bar">;
```





