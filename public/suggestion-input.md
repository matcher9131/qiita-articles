---
title: Reactでふりがなでもサジェストを表示できるinputを実装してみた
tags:
  - HTML
  - React
  - TypeScript
  - tailwindcss
  - daisyUI
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
# Reactでふりがなでもサジェストを表示できるinputを実装してみた
HTMLでいくつかの与えられた候補から1つの値を選ぶときは`<select>`および`<option>`要素を使いますが、候補がたくさんあると探すのも面倒だし、場所がわかっていても下のほうにあるとスクロールをするのも面倒です。

今の時代流石にこれでは不便すぎる、ということで標準のHTMLでも入力内容に応じて候補がドロップダウンで表示されて選択できるシステムは実装されており、`<input>`と`<datalist>`の組み合わせがそれに当たります。

ですがこの`<datalist>`……まったくもって日本語フレンドリーではないんですよね。[^1]
[^1]: 中国語も変換が必要なので漢字フレンドリーではないと言ったほうが正しいかもしれない。

<!-- ここにdatalistの動画 -->

利便性を求めるなら、当然変換前の文字にも反応してサジェストを表示してほしいものです。

ということで、ふりがなでもサジェストを表示できる`<input>`、名付けて`SuggestionInput`を作ります。

<!-- ここにSuggestionInputの動画 -->

※ 今回は「ユーザーが選ぶ値は必ずこちらで用意した候補に一致する必要がある」ケースを想定しています。[^2]
[^2]: 「サジェストは表示するが、入力内容はユーザーに任せる」ケースはこれより緩いので、適当にいじっていただければ…

## 使用パッケージ
いずれも記事執筆時点での最新です。
|パッケージ|バージョン|
|---|---|
|vite|5.3.4|
|react|18.3.1|
|recoil|0.7.7|
|tailwindcss|3.4.7|
|daisyui|4.12.10|

## daisyUIのインストール
今回はドロップダウン部分の実装に、Tailwind CSSのコンポーネントライブラリであるdaisyUIを使います。

https://daisyui.com/

まずはパッケージをインストールして

```bash
npm install -D daisyui@latest
```

`tailwind.config.js`を書き換えます。

```diff_js:tailwind.config.js
+import daisyui from "daisyui";

/** @type {import('tailwindcss').Config} */
export default {
    content: ["./index.html", "./src/**/*.{js,ts,jsx,tsx}"],
    theme: {
        extend: {},
    },
-   plugins: [],
+   plugins: [daisyui],
};
```

（他のパッケージのインストールに関しては本記事の主内容から逸れるため省略させていただきます。）

## 値の候補と型の実装
まずはユーザーに選んでもらう値とそれを型として実装します。

```typescript:models/types.ts
// as constを付けることで配列リテラルとして定義する（配列リテラルにしないと↓の型がユニオン型にならない）
export const eevees = [
    { value: "イーブイ", furigana: "いーぶい" },
    { value: "シャワーズ", furigana: "しゃわーず" },
    { value: "サンダース", furigana: "さんだーす" },
    { value: "ブースター", furigana: "ぶーすたー" },
    { value: "エーフィ", furigana: "えーふぃ" },
    { value: "ブラッキー", furigana: "ぶらっきー" },
    { value: "リーフィア", furigana: "りーふぃあ" },
    { value: "グレイシア", furigana: "ぐれいしあ" },
    { value: "ニンフィア", furigana: "にんふぃあ" },
] as const;

// type Eevee = "イーブイ" | "シャワーズ" | "サンダース" | "ブースター" | "エーフィ" | "ブラッキー" | "リーフィア" | "グレイシア" | "ニンフィア";
export type Eevee = (typeof eevees)[number]["value"];

export const isEevee = (s: string): s is Eevee => {
    return eevees.some((eevee) => eevee.value === s);
};
```

今回の目的である「変換前の文字（ふりがな）にも反応するサジェスト」を達成するために、値の候補（変数`eevees`）は「値そのものである`value`プロパティとふりがなを示す`furigana`プロパティからなるオブジェクト」の配列として定義します。[^3]この配列を元にユーザーが選ぶべき1つの値を表すユニオン型（型`Eevee`）を作ることで、候補が増えたときに2重に管理する手間を省きます。[^4][^5]
[^3]: 値がカタカナばかりになっていますが、もちろん漢字かな交じりでも`furigana`を正しく設定すれば問題ありません。
[^4]: ここの型テクニックがよくわからない！という方は[配列から型を生成する | TypeScript入門『サバイバルTypeScript』](https://typescriptbook.jp/tips/generates-type-from-array)をご覧ください。
[^5]: これでブイズが増えても安心！（意訳：新しいブイズの追加はよ）

さらに、与えられた文字列がユニオン型`Eevee`のいずれかに一致するかを判定するカスタム型ガード関数を実装しておきます。

## 状態管理
今回も状態管理にRecoilを使います。例によって他の状態管理ライブラリを使っている方は適宜読み替えてください。

`SuggestionInput`を制御コンポーネントとして実装したいので、入力内容を保存する`suggestInputValueState`を作ります。複数の`SuggestionInput`を使用する可能性を考え、key-valueのペアで管理します。

続けてユーザーが選んだ値を保存する`eeveeState`を作ります。これら2つを別々にすることで、ユーザーが自由に入力できる一方で選択する値はこちらで用意したものに強制できるようにします。

```typescript:models/states.ts
import { atomFamily } from "recoil";
import { Eevee } from "./types";

export const suggestInputValueState = atomFamily<string, string>({
    key: "suggestInputValueState",
    default: "イーブイ",
});

export const eeveeState = atomFamily<Eevee, string>({ key: "eeveeState", default: "イーブイ" });
```

## `SuggestionInput`の作成
いよいよコンポーネント本体に取り掛かります。

### daisyUIのDropdownコンポーネント
先述の通り、ドロップダウン部分にはdaisyUIの[Dropdown](https://daisyui.com/components/dropdown/)コンポーネントを用います。使い方としては以下の通りです。

- root要素のクラスに`dropdown`を指定する。
- root直下の最初の子要素にドロップダウンの親要素を置く。
- ドロップダウンの親要素と同じ階層にドロップダウン本体を置き、クラスに`dropdown-content`を、`tabindex`属性に`0`を指定する。

```html
<div class="dropdown">
    <!-- 親要素 -->
    <div tabindex="0">Foo</div>
    <!-- ドロップダウン本体 -->
    <ul tabindex="0" class="dropdown-content">
        <li>Bar</li>
        <li>Baz</li>
    </ul>
</div>
```

root以下がフォーカスを得るとドロップダウンが表示され、フォーカスを失うとドロップダウンが非表示になるという振る舞いをCSSのみで実現しています。よってドロップダウンの親要素として（フォーカスを得られる）どんな要素でも指定できます。

### 叩き台の作成
先ほど作った`suggestInputValueState`とこのDropdownコンポーネントを用いて、ひとまず叩き台を作ります。まだサジェストの絞り込みや選択時の動作は実装していません。

今回はユーザーに選んでもらう値として`Eevee`型しか用意していませんが、実際には複数種類の値の選択をすることもあるので、
型引数を与え、サジェスト候補の`dataList`やユーザーが選んだ値をセットする関数`setSelectedValue`を外から`props`経由で渡すようにします。

さらに関数`validateValue`も`props`経由で渡して、ユーザーが（サジェストに関係なく）選ぶべき値と一致する入力をした際に`setSelectedValue`を呼ぶようにします。

```tsx:components/SuggestionInput.tsx
import { useState, type ChangeEvent } from "react";
import { useRecoilState } from "recoil";
import { suggestInputValueState } from "../models/states";

type SuggestionInputProps<T extends string> = {
    readonly id: string;
    readonly dataList: ReadonlyArray<{ readonly value: T; readonly furigana: string }>;
    readonly setSelectedValue: (newValue: T) => void;
    readonly validateValue: (s: string) => s is T;
};

const SuggestionInput = <T extends string>({
    id,
    dataList,
    setSelectedValue,
    validateValue,
}: SuggestionInputProps<T>): JSX.Element => {
    const [inputValue, setInputValue] = useRecoilState(suggestInputValueState(id));
    const [suggestions, setSuggestions] = useState(dataList.map((data) => data.value));

    const handleChange = (e: ChangeEvent<HTMLInputElement>) => {
        const newValue = e.target.value;
        setInputValue(newValue);
        if (validateValue(newValue)) {
            setSelectedValue(newValue);
        }
    };

    return (
        <div className="dropdown">
            <input
                value={inputValue}
                onChange={handleChange}
                className="input input-bordered input-sm"
            />
            <ul
                tabIndex={0}
                className="dropdown-content max-h-60 z-10 overflow-auto bg-base-200 p-2 shadow"
            >
                {suggestions.map((suggestion) => (
                    <li key={suggestion}>
                        <button className="btn btn-sm w-full justify-start">{suggestion}</button>
                    </li>
                ))}
            </ul>
        </div>
    );
};

export default SuggestionInput;
```

### サジェストの絞り込み
まずは入力内容に応じて適切なサジェストを表示するようにします。

今回は単純に入力内容と同じ文字列がサジェスト候補の`value`または`furigana`に含まれるかどうかで判定します。
（入力内容を分割した部分一致、すなわち「エフ」が「エーフィ」に反応するような判定はしません。）  
利便性を考慮し、サジェストの表示順は完全一致→前方一致→部分一致（同カテゴリ内では元の表示順）とします。

まずは判定の関数を作り、

```typescript:util/filterSuggestions.ts
export const filterSuggestions = <T extends string>(
    dataList: ReadonlyArray<{ readonly value: T; readonly furigana: string }>,
    inputValue: string,
): T[] => {
    const completeMatchResults: T[] = [];
    const forwardMatchResults: T[] = [];
    const partialMatchResults: T[] = [];
    for (const data of dataList) {
        if (data.furigana === inputValue || data.value === inputValue) {
            completeMatchResults.push(data.value);
        } else if (data.furigana.startsWith(inputValue) || data.value.startsWith(inputValue)) {
            forwardMatchResults.push(data.value);
        } else if (data.furigana.includes(inputValue) || data.value.includes(inputValue)) {
            partialMatchResults.push(data.value);
        }
    }
    return [...completeMatchResults, ...forwardMatchResults, ...partialMatchResults];
};
```

`onChange`のタイミングでサジェストを更新します。

```diff_tsx:components/SuggestionInput.tsx(抜粋)
const handleChange = (e: ChangeEvent<HTMLInputElement>) => {
    const newValue = e.target.value;
    setInputValue(newValue);
    if (validateValue(newValue)) {
        setSelectedValue(newValue);
    }
+    setSuggestions(filterSuggestions(dataList, newValue));
};
```

### サジェストをクリックしたときの動作
次にサジェストをクリックしたときに、`<input>`の内容およびユーザーが選んだ値を管理するstateを更新するようにします。

`handleSuggestionClick`をカリー化しておき、イベントハンドラを指定する際に`suggestion`を部分適用して呼び出せるようにします。

関数の中身は単に`setSelectedValue`と`setInputValue`を呼び出すだけです。ついでに`document.activeElement.blur()`を呼び出してフォーカスを失わせることでドロップダウンを非表示にします。

```diff_tsx:components/SuggestionInput.tsx(抜粋)
const SuggestionInput = <T extends string>({
    id,
    dataList,
    setSelectedValue,
    validateValue,
}: SuggestionInputProps<T>): JSX.Element => {
    const [inputValue, setInputValue] = useRecoilState(suggestInputValueState(id));
    const [suggestions, setSuggestions] = useState(dataList.map((data) => data.value));

    const handleChange = (e: ChangeEvent<HTMLInputElement>) => {
        const newValue = e.target.value;
        setInputValue(newValue);
        if (validateValue(newValue)) {
            setSelectedValue(newValue);
        }
        setSuggestions(filterSuggestions(dataList, newValue));
    };

+    const handleSuggestionClick = (suggestion: T) => () => {
+        setSelectedValue(suggestion);
+        setInputValue(suggestion);
+        // ドロップダウンを閉じる
+        (document.activeElement as HTMLElement | null)?.blur();
+    };

    return (
        <div className="dropdown">
            <input
                value={inputValue}
                onChange={handleChange}
                className="input input-bordered input-sm"
            />
            <ul
                tabIndex={0}
                className="dropdown-content max-h-60 z-10 overflow-auto bg-base-200 p-2 shadow"
            >
                {suggestions.map((suggestion) => (
                    <li key={suggestion}>
-                        <button className="btn btn-sm w-full justify-start">{suggestion}</button>
+                        <button
+                            onClick={handleSuggestionClick(suggestion)}
+                            className="btn btn-sm w-full justify-start"
+                        >
+                            {suggestion}
+                        </button>
                    </li>
                ))}
            </ul>
        </div>
    );
};
```

### 入力内容の矯正
ここまでで概ね目的は達成していますが、さらなるブラッシュアップを考えます。

今回は「ユーザーが選ぶ値は必ずこちらで用意した候補に一致する必要がある」としていますが、`<input>`が自由に入力できるため、ユーザーが選択した値と`<input>`の内容が一致しないケースがどうしても出てきます。

入力中以外は両者を一致させたいので、`SuggestionInput`がフォーカスを失った時に`<input>`の内容を矯正する方針にします。具体的には`<input>`の`onBlur`イベントハンドラを実装し、入力文字列が値の候補のいずれとも一致しない場合、最後に選択した値を表示するようにします。[^6]
[^6]: ここは場合によりけりで、「常にデフォルト値に矯正」や「入力文字列と最もよく一致する値に矯正」なども考えられます。

しかし現状`SuggestionInput`はユーザーが最後に選択した値を知る術がないので、これも`props`経由で外部から与えます。

```diff_tsx:components/SuggestionInput.tsx(抜粋)
type SuggestionInputProps<T extends string> = {
    readonly id: string;
    readonly dataList: ReadonlyArray<{ readonly value: T; readonly furigana: string }>;
+    readonly selectedValue: T;
    readonly setSelectedValue: (newValue: T) => void;
    readonly validateValue: (s: string) => s is T;
};

const SuggestionInput = <T extends string>({
    id,
    dataList,
+    selectedValue,
    setSelectedValue,
    validateValue,
}: SuggestionInputProps<T>): JSX.Element => {
    const [inputValue, setInputValue] = useRecoilState(suggestInputValueState(id));
    const [suggestions, setSuggestions] = useState(dataList.map((data) => data.value));

    const handleChange = (e: ChangeEvent<HTMLInputElement>) => {
        const newValue = e.target.value;
        setInputValue(newValue);
        if (validateValue(newValue)) {
            setSelectedValue(newValue);
        }
        setSuggestions(filterSuggestions(dataList, newValue));
    };

+    const handleBlur = () => {
+        if (!validateValue(inputValue)) {
+            setInputValue(selectedValue);
+        }
+    };

    const handleSuggestionClick = (suggestion: T) => () => {
        setSelectedValue(suggestion);
        setInputValue(suggestion);
        (document.activeElement as HTMLElement | null)?.blur();
    };

    return (
        <div className="dropdown">
            <input
                value={inputValue}
                onChange={handleChange}
+                onBlur={handleBlur}
                className="input input-bordered input-sm"
            />
            <ul
                tabIndex={0}
                className="dropdown-content max-h-60 z-10 overflow-auto bg-base-200 p-2 shadow"
            >
                {suggestions.map((suggestion) => (
                    <li key={suggestion}>
                        <button onClick={handleSuggestionClick(suggestion)} className="btn btn-sm w-full justify-start">
                            {suggestion}
                        </button>
                    </li>
                ))}
            </ul>
        </div>
    );
};
```

これで一旦目的は果たせていますが、サジェストをクリックすると一瞬だけ直前に選んだ値が表示されてから新たな値に矯正されます。具体的には、「エーフィ」を選択してから入力内容を消してサジェストから「ブラッキー」を選択すると、一瞬だけ「エーフィ」が表示されてから「ブラッキー」が表示されます。

これはイベントの順序が`<input>のonBlur`→`<button>のonClick`となっているためです。イベントの順序を操作することはできませんが、`onClick`の代わりに`onMouseDown`にすれば`onBlur`より発生が早くなるので、ディテールにこだわる場合はこちらを採用します。[^7]
[^7]: 当然ながら`MouseDown`で処理をするということは`MouseUp`を待たなくなるため、操作感が変わる点には注意が必要です。

ただし`onMouseDown`を採用した場合、`<button>のonMouseDown`→`<input>のonBlur`の間は再レンダリングが起きないため`selectedValue`が1つ前の値を参照してしまう、すなわち上記の例だと「ブラッキーを選択したのに表示がエーフィのまま」となってしまいます。これを回避するために`useRef`を用いて`selectedValueRef`を作り、`<button>のonMouseDown`で更新してから`<input>のonBlur`で参照することにします。

```diff_tsx:components/SuggestionInput.tsx(抜粋)
const SuggestionInput = <T extends string>({
    id,
    dataList,
    selectedValue,
    setSelectedValue,
    validateValue,
}: SuggestionInputProps<T>): JSX.Element => {
    const [inputValue, setInputValue] = useRecoilState(suggestInputValueState(id));
    const [suggestions, setSuggestions] = useState(dataList.map((data) => data.value));
+    const selectedValueRef = useRef(selectedValue);

    const handleChange = (e: ChangeEvent<HTMLInputElement>) => {
        const newValue = e.target.value;
        setInputValue(newValue);
        if (validateValue(newValue)) {
            setSelectedValue(newValue);
        }
        setSuggestions(filterSuggestions(dataList, newValue));
    };

    const handleBlur = () => {
        if (!validateValue(inputValue)) {
-            setInputValue(selectedValue);
+            setInputValue(selectedValueRef.current);
        }
    };

    const handleSuggestionClick = (suggestion: T) => () => {
        setSelectedValue(suggestion);
        setInputValue(suggestion);
+        selectedValueRef.current = suggestion;
        (document.activeElement as HTMLElement | null)?.blur();
    };

    return (省略)
};
```

### 再選択時の利便性の向上
ユーザーが何度も選択内容を変えるような使い方が想定される場合、入力内容を全選択して消してからでないとサジェストが表示されないというのは不便です。よってフォーカスを得た瞬間は入力内容に関係なく全サジェストを表示することにします。ついでにテキストを全選択する手間も省いておきましょう。

```
const SuggestionInput = <T extends string>({
    id,
    dataList,
    selectedValue,
    setSelectedValue,
    validateValue,
}: SuggestionInputProps<T>): JSX.Element => {
    const [inputValue, setInputValue] = useRecoilState(suggestInputValueState(id));
    const [suggestions, setSuggestions] = useState(dataList.map((data) => data.value));
    const selectedValueRef = useRef(selectedValue);

    const handleChange = (e: ChangeEvent<HTMLInputElement>) => {
        const newValue = e.target.value;
        setInputValue(newValue);
        if (validateValue(newValue)) {
            setSelectedValue(newValue);
        }
        setSuggestions(filterSuggestions(dataList, newValue));
    };

+    const handleFocus = (e: FocusEvent<HTMLInputElement>) => {
+       e.target.select();
+       setSuggestions(filterSuggestions(dataList, ""));
+   };

    const handleBlur = () => {
        if (!validateValue(inputValue)) {
            setInputValue(selectedValueRef.current);
        }
    };

    const handleSuggestionClick = (suggestion: T) => () => {
        setSelectedValue(suggestion);
        setInputValue(suggestion);
        selectedValueRef.current = suggestion;
        (document.activeElement as HTMLElement | null)?.blur();
    };

    return (
        <div className="dropdown">
            <input
                value={inputValue}
                onChange={handleChange}
+                onFocus={handleFocus}
                onBlur={handleBlur}
                className="input input-bordered input-sm"
            />
            <ul
                tabIndex={0}
                className="dropdown-content max-h-60 z-10 overflow-auto bg-base-200 p-2 shadow"
            >
                {suggestions.map((suggestion) => (
                    <li key={suggestion}>
                        <button
                            onMouseDown={handleSuggestionClick(suggestion)}
                            className="btn btn-sm w-full justify-start"
                        >
                            {suggestion}
                        </button>
                    </li>
                ))}
            </ul>
        </div>
    );
};
```
