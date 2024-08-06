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

ユーザーが選んだ値を管理する`atom`を作ります。複数の`SuggestionInput`を使用する可能性を考え、key-valueのペアで管理します。

```typescript:models/states.ts
import { atomFamily } from "recoil";
import { Eevee } from "./types";

export const eeveeStates = atomFamily<Eevee, string>({ key: "eeveeStates", default: "イーブイ" });
```