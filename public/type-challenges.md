# Type Challengesを解く際に使えるテクニック集

TypeScriptの複雑な型システムをどれほど理解しているのかの腕試しができる[Type Challenges](https://github.com/type-challenges/type-challenges/)

[PartialByKeys](https://github.com/type-challenges/type-challenges/blob/main/questions/02757-medium-partialbykeys/README.md)のような割と実用的なものから、
[Valid Sudoku](https://github.com/type-challenges/type-challenges/blob/main/questions/35314-hard-valid-sudoku/README.md)のようにこんなのいつ使うんだよ……となるものまで
様々な問題が収録されているType Challengesですが、これらを解く際によく使うテクニックや考え方をご紹介します。

Type Challengesだけではなく、これってどう書けばよかったっけ…？となった際に辞書的に使えるものになっていますので、ぜひご覧ください。


## おことわり
- **一部問題（特にeasy, medium）の解答がガッツリ載っています。**
- TypeScriptの型システムに関しての基本的な説明はありません。不明な点は以下のuhyoさんの記事を参照ください。

https://qiita.com/uhyo/items/e2fdef2d3236b9bfe74a

https://qiita.com/uhyo/items/da21e2b3c10c8a03952f


## タプル関連

### タプルの各要素から成るユニオン型を取得する
`T[number]`で数値でアクセスできる型、すなわちTの全要素をユニオン型で取得できます。
```typescript
type TupleToUnion<T extends unknown[]> = T[number];

// type Foo = true | 1 | "foo"
type Foo = TupleToUnion<[1, "foo", true, 1]>;
```

Type Challengesとは関係ない話ですが、他言語における`enum`相当のものを文字列リテラルのユニオン型で定義する場合には、ユニオン型を直接宣言するよりも、タプルを宣言してそこからユニオン型を作ったほうが何かと便利です。
```typescript
// as constをつけないとstring[]に推論されてしまう
const eevees = ["イーブイ", "シャワーズ", "サンダース", "ブースター"] as const;
// type Eevee = "イーブイ" | "シャワーズ" | "サンダース" | "ブースター"
type Eevee = (typeof eevees)[number];

// 与えられた文字列がEevee型かどうかを判断する型ガード関数
const isEevee = (s: string): s is Eevee => {
    return eevees.includes(s as Eevee);
    // 嘘のasが気持ち悪い場合は↓で
    // return eevees.find(eevee => eevee === s) != null;
};

console.log(isEevee("イーブイ"));    // true
console.log(isEevee("リザードン"));  // false
```
値の列挙も型ガードの実装も簡単にできます。さらに、追加の際もタプルを書き換えるだけなのもポイントです。
```diff_typescript
- const eevees = ["イーブイ", "シャワーズ", "サンダース", "ブースター"] as const;
+ const eevees = ["イーブイ", "シャワーズ", "サンダース", "ブースター", "エーフィ", "ブラッキー", "リーフィア", "グレイシア", "ニンフィア"] as const;
```

### タプルの各要素を1つずつ処理する（Array.prototype.forEach相当）
再帰呼び出しを用います。
```typescript
type Foo<T> = T extends [infer F, ...infer R]
    ? // Tが空でないとき
    : // Tが空のとき
```
`F`に先頭の要素、`R`に残りの要素（空になりうる）が入ります。後述のArray.prototype.map相当を除き、基本的にタプルの各要素を舐める際にはこのアプローチを用います。

### タプルから条件を満たす要素のみのタプルを作る（Array.prototype.filter相当）
```typescript
type Foo<T> = T extends [infer F, ...infer R]
    ? Bar<T> extends true
        ? [T, ...Foo<R>]
        : Foo<R>
    : [];
```
再帰呼び出しの際にスプレッド演算子を用いてタプルが階層化するのを防ぎます。

再帰呼び出しの制限`"Type instantiation is excessively deep and possibly infinite."`に引っかかる場合は以下がおすすめです。（詳細は後述）
```typescript
type Foo<T, A extends unknown[] = []> = T extends [infer F, ...infer R]
    ? Bar<T> extends true
        ? Foo<R, [...A, T]>
        : Foo<R, A>
    : A;
```

### タプルの各要素を変換する（Array.prototype.map相当）
単純な場合はMapped Typesを用いるのが簡便です。
```typescript
type Foo<T> = {
    [K in keyof T]: Bar<T[K]>;
};
```
`Bar<X>`が`Array.prototype.map`におけるコールバック関数に相当します。

具体的には、`T`が長さ3のタプルのとき以下のようなイメージで展開されます。（※あくまでイメージであって正確ではありません）
```typescript
K = 0 | 1 | 2;
Foo<T> = {
    0: Bar<T[0]>;
    1: Bar<T[1]>;
    2: Bar<T[2]>;
}
```

勿論`T extends [infer F, ...infer R]`でも処理できます。
```typescript
type Foo<T> = T extends [infer F, ...infer R]
    ? [Bar<F>, ...Foo<R>]
    : [];
```

### タプル内で何らかの条件を満たす要素を探す（Array.prototype.find相当）
```typescript
type Foo<T> = T extends [infer F, ...infer R]
    ? Bar<F> extends true
        ? F     // Fが条件を満たす場合の返り値
        : Foo<R>
    : never;    // 条件を満たす要素が存在しない場合の返り値
```
`Bar<X>`が`Array.prototype.find`におけるコールバック関数に相当します。返り値`F`や`never`は目的によって適切なものを選択してください。

### タプルの各要素を処理して複雑な型を返す（Array.prototype.reduce相当）
```typescript
type Foo<T, A = Initial> = T extends [infer F, ...infer R]
    ? Foo<R, Bar<A, F>>
    : A;
```
型引数に途中経過`A`を持たせることで`Array.prototype.reduce`のような操作ができます。`Bar<A, F>`の部分がコールバック関数に相当します。`Initial`には適当な初期値を入れておきます。

### タプルの各要素で条件を満たすものの個数を調べる（C++のstd::countやstd::count_if相当）
数値を直接カウントアップする術はないので、タプルの長さを用います。
```typescript
type Count<T, A extends unknown[] = []> = T extends [infer F, ...infer R]
    ? Bar<F> extends true
        ? Count<R, [...A, unknown]>
        : Count<R, A>
    : A["length"];
```
例によって`Bar<F>`はコールバック関数に相当します。

タプルの長さのみが重要なので`A`の中身に関しては何でも構いません。`unknown, 0, 1`あたりが用いられることが多いようです。
（何でも構わないので`F`をそのまま突っ込んでも勿論OKですが、`Array.prototype.filter`相当と紛らわしいため避けたほうが良いでしょう。）

### 2つのタプルの長さを比較する
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

### 非負整数Nを長さNのタプルに変換する
これ自体は大して意味がありませんが、変換することによって大小比較や加算などができるようになります。[^n-to-t] 基本的にはやることは「タプルの各要素で条件を満たすものの個数を調べる」と同じです。

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
なお $N < 0$ の場合`A["length"] extends N`が`true`になることはないため、再帰呼び出しが止まらず回数制限を迎えます。

$N \geq 1000$ だとそもそも素で再帰呼び出しの回数制限に引っかかります。[^n-to-t-2]

[^n-to-t-2]: 2の累乗の長さのタプルをあらかじめ用意しておくなどでもっと大きな非負整数を扱えるようになりますが、そこまでくると数値をタプルに変換するよりも、タプルに頼らず数値の各桁を処理したほうが楽なことが多いです。

### 配列リテラルを配列ではなくタプルとして推論させる
関数Genericsで引数を推論させる場合などで、配列リテラルを配列としてではなくタプルとして解釈して欲しいときは`[...T]`とします。
```typescript
declare function f<T extends unknown[]>(value: T): T;
declare function g<T extends unknown[]>(value: [...T]): T;

// const foo: (string | number)[]
const foo = f([1, 2, "foo"]);
// const baz: [number, number, string]
const baz = g([1, 2, "foo"]);
```

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

### 英大文字、英小文字を検出する
Utility typesの`Uppercase<S>`は`S`に含まれる英小文字を大文字に、`Lowercase<S>`は`S`に含まれる英大文字を小文字に変換しますが、対象となる文字以外は素通しするためこれで判別が可能です。
```typescript
type ContainsAlphabet<S extends string> = S extends Uppercase<S>
    ? S extends Lowercase<S>
        ? false
        : true
    : true;

type Foo = ContainsAlphabet<"A">;     // type Foo = true
type Bar = ContainsAlphabet<"a">;     // type Bar = true
type Baz = ContainsAlphabet<"_foo">;  // type Baz = true
type Qux = ContainsAlphabet<"123">;   // type Qux = false
type Quux = ContainsAlphabet<"">;     // type Quux = false
```
英大文字は`Lowercase<S>`で必ず変換され、英小文字は`Uppercase<S>`で必ず変換されるため、双方に通して変換されないのは英文字以外と判別できます。

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
このケースにおける`as`は型アサーションとは異なり、任意の型に好き勝手に変換できます。
どちらかといえばプロパティ名に対するmapをたまたま同じキーワードの`as`が担っていると考えたほうがいいかもしれません。

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

わざわざ`T extends T`としなくてもUnion Distribution自体は`Bar<T>`の内部で発生する可能性がありますが、それが意図するものかどうかはわかりません。
以下の例では`LooseRepeat<T>`内で`T`がそのままの形で2回使われたために、Union Distributionによって全ての組み合わせを網羅してしまっています。
`T extends T`で先にUnion Distributionを起こせば`` `${T}${T}` ``の部分には分配された後の型が入って意図したものになります。
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

## 関数型関連

### 関数オーバーロードを表現する
突然ですが、ここで問題です。

関数`f`は1個の`number`型あるいは`string`型の引数`x`を取り、`x`が`number`型なら`number`型を、`string`型なら`string`型を返します。この関数`f`を表す型`F`を書いてください。

「なるほど、`(x: number) => number` **または** `(x: string) => string` だから……」と思って以下のようにすると引数で型エラーが発生します。身に覚えのない`never`に襲われていますし、よく見ると返り値の型も変です。
```typescript
type F = ((x: number) => number) | ((x: string) => string);
declare const f: F;
// Error: Argument of type '3' is not assignable to parameter of type 'never'.
// const x: number | string
const x = f(42);
// Error: Argument of type 'bar' is not assignable to parameter of type 'never'.
// const x: number | string
const y = f("bar");
```

状況を整理しましょう。変数`f`には「`number`型の引数を1つ取って`number`型を返す関数」あるいは「`string`型の引数を1つ取って`string`型を返す関数」のいずれかが入ると解釈できます。よって`f`を呼び出す際はこれらのどちらであっても問題ないような引数、すなわち`number | string`ではなく`number & string`を渡す必要があります。当然`number & string == never`なので、身に覚えのない`never`はここから生じていたんですね……

よって正しくはむしろ逆で、`(x: number) => number`と`(x: string) => string`の共通部分を受け入れる型になります。[^overroad]
```typescript
type F = ((x: number) => number) & ((x: string) => string);
declare const f: F;
// const x: number
const x = f(42);
// const x: string
const y = f("bar");
```

[^overroad]: 関数の引数には反変性があることを知っていれば抵抗なく受け入れられると思います。

## その他

### 再帰呼び出し回数制限にかかりにくくする
Conditional Typesで再帰呼び出しを行う際、その型単独で書くと末尾再帰の最適化によって再帰呼び出しの回数制限`"Type instantiation is excessively deep and possibly infinite."`にかかりにくくなります。

具体的には以下の通りです。
```typescript
type IsUpperCase<S extends string> = // 省略

type Foo1<S extends string> = S extends `${infer F}${infer R}`
    ? IsUpperCase<F> extends true
        ? `${F}${Foo1<R>}`  // Foo1を単独で呼び出していないため末尾再帰の最適化の対象外
        : Foo1<R>
    : "";

type Foo2<S, A extends string = ""> = S extends `${infer F}${infer R}`
    ? IsUpperCase<F> extends true
        ? Foo2<R, `${A}${F}`>  // `${A}${F}`をFoo2の内部に閉じ込めることでFoo2を単独で呼び出しているため、末尾再帰の最適化の対象になる
        : Foo2<R, A>
    : A;

// Error: Type instantiation is excessively deep and possibly infinite.
// type X1 = `ABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVW${any}`
type X1 = Foo1<"ABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZ">;
// type X2 = "ABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZ"
type X2 = Foo2<"ABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZ">;
```

### Conditional Typesで意図しない分岐をしてしまうのを対策する
もちろん場合によりけりですが、間違ってなさそうなのに正解できない場合はConditional Typesに`never`型が紛れ込んでいるケースが多々あります。

Conditional Typesには`T extends U ? X : Y`の`T`が`never`型のとき`X`でも`Y`でもなく`never`型が返るという仕様があります。  
Union Distributionで分配するものがなくなった結果`never`型になるという扱いのようです。
```typescript
// number型に制限された型引数にneverを代入できるので
// 一見すると'never extends number'はtrueになりそうだが……
type Foo<T extends number = never> = // 省略

type Bar<T, U> = T extends U ? 1 : 0;
// 実際にはXは1でも0でもなくneverになる
// type X = never
type X = Bar<never, number>;
```

これを回避するにはUnion Distributionを起こさなければOKです。
```typescript
type Baz<T, U> = [T] extends [U] ? 1 : 0;
// type Y = 1
type Y = Baz<never, number>;
```
