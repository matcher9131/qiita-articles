---
title: suggestion-input
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
HTMLでいくつかの与えられた候補から1つの値を選ぶときは`<select>`および``<option>`要素を使いますが、候補がたくさんあると探すのも面倒だし、場所がわかっていても下のほうにあるとスクロールをするのも面倒です。

今の時代流石にこれでは不便すぎる、ということで標準のHTMLでも入力内容に応じて候補がドロップダウンで表示されて選択できるシステムは実装されており、`<input>`と`<datalist>`の組み合わせがそれに当たります。

ですがこの`<datalist>`……まったくもって日本語フレンドリーではないんですよね。[^1]
[^1]: 中国語も変換が必要なので漢字フレンドリーではないと言ったほうが正しいかもしれない。

<!-- ここにdatalistの動画 -->

利便性を求めるなら、当然変換前の文字にも反応してサジェストを表示してほしいものです。

ということで、ふりがなでもサジェストを表示できる`<input>`、名付けて`SuggestionInput`を作ります。

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

今回の目的である「変換前の文字（ふりがな）にも反応するサジェスト」を達成するために、値の候補（変数`eevees`）は「値そのものである`value`プロパティとふりがなを示す`furigana`プロパティからなるオブジェクト」の配列として定義します。この配列を元にユーザーが選ぶべき1つの値を表すユニオン型（型`Eevee`）を作ることで、候補が増えたときに2重に管理する手間を省きます。[^2][^3]
[^2]: ここの型テクニックがよくわからない！という方は[配列から型を生成する | TypeScript入門『サバイバルTypeScript』](https://typescriptbook.jp/tips/generates-type-from-array)をご覧ください。
[^3]: これでブイズが増えても安心！（意訳：新しいブイズの追加はよ）

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
先述の通り、ドロップダウン部分にはdaisyUIの[Dropdown](https://daisyui.com/components/dropdown/)コンポーネントを用います。使い方としては

- root要素のクラスに`dropdown`を指定する。
- root直下の最初の子要素にドロップダウンの親要素を置く。
- ドロップダウンの親要素と同じ階層にドロップダウン本体を置き、クラスに`dropdown-content`を、`tabindex`属性に`0`を指定する。

といった具合です。親要素として（フォーカスを得られる）どんな要素でも指定できる点が優秀です。

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

### 叩き台の作成
先ほど作った`suggestInputValueState`とこのDropdownコンポーネントを用いて、ひとまず叩き台を作ります。まだサジェストの絞り込みや選択時の動作は実装していません。

今回はユーザーに選んでもらう値として`Eevee`型しか用意していませんが、実際には複数種類の値の選択をすることもあるので、
型引数を与え、サジェスト候補の`dataList`やユーザーが選んだ値をセットする関数`setValue`を外から`props`経由で渡すようにします。

さらに関数`validateValue`も`props`経由で渡して、ユーザーが（サジェストに関係なく）選ぶべき値と一致する入力をした際に`setValue`を呼ぶようにします。

```tsx:components/SuggestionInput.tsx
import { useState, type ChangeEvent } from "react";
import { useRecoilState } from "recoil";
import { suggestInputValueState } from "../models/states";

type SuggestionInputProps<T extends string> = {
    readonly id: string;
    readonly dataList: ReadonlyArray<{ readonly value: T; readonly furigana: string }>;
    readonly setValue: (newValue: T) => void;
    readonly validateValue: (s: string) => s is T;
};

const SuggestionInput = <T extends string>({
    id,
    dataList,
    setValue,
    validateValue,
}: SuggestionInputProps<T>): JSX.Element => {
    const [inputValue, setInputValue] = useRecoilState(suggestInputValueState(id));
    const [suggestions, setSuggestions] = useState<readonly T[]>(dataList.map((data) => data.value));

    const handleChange = (e: ChangeEvent<HTMLInputElement>) => {
        const newValue = e.target.value;
        setInputValue(newValue);
        if (validateValue(newValue)) {
            setValue(newValue);
        }
    };

    return (
        <div className="dropdown">
            <input value={inputValue} onChange={handleChange} className="input input-bordered input-sm" />
            <ul tabIndex={0} className="dropdown-content max-h-60 z-10 overflow-auto bg-base-200 p-2 shadow">
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
入力内容に応じて適切なサジェストを表示するようにします。

今回は単純に入力内容と同じ文字列がサジェスト候補の`value`または`furigana`に含まれるかどうかで判定します。
（入力内容を分割した部分一致、すなわち「エフ」が「エーフィ」に反応するような判定はしません。）  
利便性を考慮し、サジェストの表示順は完全一致→前方一致→前方一致ではない部分一致（同カテゴリ内では元の表示順）とします。

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
        setValue(newValue);
    }
+    setSuggestions(filterSuggestions(dataList, newValue));
};
```