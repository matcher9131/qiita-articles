---
title: Reactでスライダーでも直接入力でも数値入力ができるコンポーネントを作ってみた
tags:
  - React
  - TypeScript
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
# Reactでスライダーでも直接入力でも数値入力ができるコンポーネントを作ってみた
HTMLでユーザーに数値を指定させるときに使う要素としては`<input type="number">`や`<input type="range">`がありますが、[^1]いずれも一長一短です。
[^1]: 両方とも未対応のブラウザは窓から投げ捨てましょう。

- `<input type="number">`は厳密な値を入力する際には便利ですが、マウスのみでちゃちゃっと指定したいときには面倒です。[^2]
- `<input type="range">`は逆にマウスのみで簡単に数値を選べますが、厳密な値を指定するのには向いていません。[^3]
[^2]: スピンボタンを用いればマウスのみでも指定できますが、値によってはかなり長い時間押しっぱなしにする必要があるのでお世辞にも便利とは言えません。
[^3]: マウスを1px単位で動かす簡単なお仕事はしたくありません。

しかしこの2つを組み合わせればお互いの欠点を補えるため、非常に利便性の高いコンポーネントになるはずです。

というわけでReactでこれを作ってみました。名付けて、`SliderInput`！

## 使用パッケージ
いずれも記事執筆時点での最新です。
|パッケージ|バージョン|
|---|---|
|vite|5.4.0|
|react|18.3.1|
|recoil|0.7.7|
|tailwindcss|3.4.9|
|daisyui|4.12.10|

## コンポーネントの仕様策定
細かいところは様々なやり方があるとは思いますが、今回は以下の方針にします。

※ `atom`はユーザーが指定した数値を状態に持つ変数を示します。

- 2つの`<input>`は制御コンポーネントとする
- `<input type="range">`の値が変更されたとき
    - `<input type="number">`の内容および`atom`の値を更新する
- `<input type="number">`が入力中のとき
    - 入力内容が数値として解釈できる場合のみ`<input type="range">`のおよび`atom`の値を更新する
        - ただし値が`max`を超えている場合は`max`に、`min`未満の場合は`min`に矯正する
- `<input type="number">`からフォーカスが失われたとき
    - 内容が数値として解釈できない場合、`defaultValue`に矯正する
    - 内容が数値として解釈できる場合、`max`を超えている場合は`max`に、`min`未満の場合は`min`に矯正する
    - `atom`を矯正後の値で更新する
- `<input type="number">`のスピンボタンが押されたとき
    - `<input type="range">`および`atom`の値を更新する

ポイントとして、`<input type="number">`が入力中のときは入力内容の矯正を行わないようにします。これは空文字列を含め、入力中の数値が必ず`min`以上`max`以下の範囲に収まるわけではないためです。
（例として`min`が`10`のとき、`30`を入力しようとして`3`を入力した時点では`min`以上を満たしていません。）
ただし勿論そのまま放置するとどんな値でも入力できてしまうので、入力完了＝`onBlur`のタイミングで矯正します。

## 状態管理
毎度ながら状態管理にはRecoilを使用します。他の状態管理ライブラリをお使いの方は例によって適宜読み替えてください。

ユーザーが指定した数値を状態に持つ`atom`を作ります。本コンポーネントを複数用いる可能性を考え、`atomFamily`としておきます。

```typescript:state.ts
export const valueState = atomFamily<number, string>({ key: "valueState", default: 0 });
```

## Utilの実装
いくつか便利関数を用意しておきます。

- `parseNumberOrNull`: 文字列を数値に変換しますが、変換に失敗したときは`NaN`ではなく`null`を返します。
- `withinRange`: 与えられた数値を、`min`以上`max`未満になるよう最も近い値に矯正します。

```typescript:util.ts
export const parseNumberOrNull = (s: string): number | null => {
    const num = Number(s);
    return Number.isNaN(num) ? null : num;
};

export const withinRange = (x: number, min: number, max: number): number => {
    return x > max ? max
        : x < min ? min
        : x;
};
```

## コンポーネントの実装
<!-- ここにbuttonsとかの説明 -->

```tsx:SliderInput.tsx(抜粋)
type SliderInputProps = {
    readonly id: string;
    readonly defaultValue: number;
    readonly minValue: number;
    readonly maxValue: number;
    readonly step?: number;
};

const SliderInput = ({ id, defaultValue, minValue, maxValue }: SliderInputProps): JSX.Element => {
    const [value, setValue] = useRecoilState(valueNoRefState(id));
    const [valueRaw, setValueRaw] = useState(`${defaultValue}`);

    const handleInputChange = (e: ChangeEvent<HTMLInputElement>): void => {
        setValueRaw(e.target.value);
        const tempValue = parseNumberOrNull(e.target.value);
        if (tempValue != null) {
            const newValue = withinRange(tempValue, minValue, maxValue);
            setValue(newValue);
        }
    };

    const handleInputBlur = (e: FocusEvent<HTMLInputElement>): void => {
        const newValue = withinRange(parseNumberOrNull(e.target.value) ?? defaultValue, minValue, maxValue);
        setValue(newValue);
        setValueRaw(`${newValue}`);
    };

    const handleButtonClick = (x: number) => (): void => {
        setValue(x);
        setValueRaw(`${x}`);
    };

    const handleRangeChange = (e: ChangeEvent<HTMLInputElement>): void => {
        const newValue = parseNumberOrNull(e.target.value) ?? defaultValue;
        setValue(newValue);
        setValueRaw(`${newValue}`);
    };

    return (
        <div className="flex flex-col">
            <div className="inline-flex flex-row items-center gap-x-1">
                <input
                    type="number"
                    value={valueRaw}
                    max={maxValue}
                    min={minValue}
                    onChange={handleInputChange}
                    onBlur={handleInputBlur}
                    className="input input-bordered input-sm text-right"
                />
                <button onClick={handleButtonClick(minValue)} className="btn btn-neutral btn-xs">
                    min
                </button>
                <button onClick={handleButtonClick(defaultValue)} className="btn btn-neutral btn-xs">
                    def
                </button>
                <button onClick={handleButtonClick(maxValue)} className="btn btn-neutral btn-xs">
                    max
                </button>
            </div>
            <div>
                <input
                    type="range"
                    value={value}
                    min={minValue}
                    max={maxValue}
                    step={step}
                    onChange={handleRangeChange}
                />
            </div>
        </div>
    );
};
```