---
title: dnd-kitでアイテムもカラムもswappableなマルチカラムを実装してみた
tags:
  - React
  - TypeScript
  - dnd-kit
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
# `dnd-kit`でアイテムもカラムもswappableなマルチカラムを実装してみた
`React`にてドラッグアンドドロップ（以下DnDと表記します）を実装するライブラリである`dnd-kit`を用いて、DnDによって

- カラムどうしの入れ替え
- 同カラム内のアイテムどうしの入れ替え
- 異なるカラム間でのアイテムどうしの入れ替え

が可能なマルチカラムを実装します。

<!-- ここに動画 -->

## 使用パッケージ
いずれも記事執筆時点での最新です。
|パッケージ|バージョン|
|---|---|
|vite|5.3.4|
|react|18.3.1|
|recoil|0.7.7|
|tailwindcss|3.4.7|
|@dnd-kit/core|6.1.0|
|@dnd-kit/utilities|3.2.2|
|@dnd-kit/sortable|8.0.0|

## `dnd-kit`のインストール
ルートディレクトリで以下を実行します。今回は要素同士の並び替えを実装したいので、`sortable`パッケージもインストールします。

```bash
npm install @dnd-kit/core @dnd-kit/utilities @dnd-kit/sortable
```


## 状態管理
今回は状態管理を`Recoil`に任せます。どのカラム内にどのアイテムがあるのかを管理するために以下の`atom`を作り、初期状態を適当に`default`にぶち込みます。

```typescript:containerChildren.ts
export const containerChildrenState = atom<ReadonlyArray<{ header: string; items: string[] }>>({
    key: "containerChildrenState",
    default: [
        { header: "A", items: ["A1", "A2", "A3", "A4"] },
        { header: "B", items: ["B1", "B2"] },
        { header: "C", items: ["C1", "C2", "C3"] },
    ],
});
```

## コンポーネントの作成
とりあえずサクっと`Container`、`Column`、`Item`コンポーネントを作ります。後の実装を考え、これらは（コンテナ・プレゼンテーションパターンにおける）プレゼンテーションコンポーネントとします。

<details>
<summary>Container</summary>

```tsx:Container.tsx
import { type ReactNode } from "react";

type ContainerProps = {
    readonly children: ReactNode;
};

const Container = ({ children }: ContainerProps): JSX.Element => {
    return <div className="flex gap-x-3">{children}</div>;
};

export default Container;
```

</details>

<details>
<summary>Column</summary>

```tsx:Column.tsx
import { type ReactNode } from "react";

type ColumnProps = {
    readonly children: ReactNode;
    readonly header: string;
};

const Column = ({ children, header: labelText }: ColumnProps): JSX.Element => {
    return (
        <div className="grow flex flex-col gap-y-3 p-3 bg-slate-200">
            <div>{labelText}</div>
            <div className="grow flex flex-col gap-y-3 ">{children}</div>
        </div>
    );
};

export default Column;
```

</details>

<details>
<summary>Item</summary>

```tsx:Item.tsx
type ItemProps = {
    readonly labelText: string;
    readonly className: string;
};

const Item = ({ labelText, className }: ItemProps): JSX.Element => {
    return <div className={`w-full flex items-center p-2 ${className}`}>{labelText}</div>;
};

export default Item;
```

</details>

先ほどの`atom`を参照して各コンポーネントを配置します。

```tsx:App.tsx(抜粋)
const App = (): JSX.Element => {
    const columns = useRecoilValue(containerChildrenState);

    return (
        <div className="w-full p-5">
            <Container>
                {columns.map((column) => (
                    <Column key={column.header} header={column.header}>
                        {column.items.map((item) => (
                            <Item key={item} labelText={item} className={getItemBgColor(item)} />
                        ))}
                    </Column>
                ))}
            </Container>
        </div>
    );
};
```

ひとまず以下の画像のようになりました。

<!-- ここに画像 -->

<!-- この時点でのコミット： https://github.com/matcher9131/dnd-kit-swappable-multi-column/commit/a12a81185692183baaac6a903b450834c2800550 -->

これをベースとして、DnD機能を実装していきます。

## `DndContext`の追加
`dnd-kit`では`Draggable`コンポーネント（ドラッグする要素）と`Droppable`コンポーネント（ドラッグ要素を受け付ける要素）を実装することでDnDを実現しますが、それらの`Context Provider`として親に`DndContext`を置く必要があります。

加えて、`dnd-kit`で提供される機能はあくまでDnDによる見た目の変化を担うものであり、実際のデータの変化などに関してはDnD終了時に呼び出される`DndContext`の`onDragEnd`イベントハンドラに記述する必要があります。が、解説の都合で今は`undefined`にしておきます。

```diff_tsx:App.tsx(抜粋)
const App = (): JSX.Element => {
    const columns = useRecoilValue(containerChildrenState);

    return (
        <div className="w-full p-5">
+           <DndContext>
                <Container>
                    {columns.map((column) => (
                        <Column key={column.header} header={column.header}>
                            {column.items.map((item) => (
                                <Item key={item} labelText={item} className={getItemBgColor(item)} />
                            ))}
                        </Column>
                    ))}
                </Container>
+           </DndContext>
        </div>
    );
};
```

## `useSortable`
`dnd-kit`においては`useDraggable`フックにて`Draggable`コンポーネントを、`useDroppable`フックにて`Droppable`コンポーネントを作成できますが、今回作る入れ替え可能な要素のように`Draggable`かつ`Droppable`なコンポーネントには`useSortable`フックを使います。使い方としては

1. `useSortable`フックを使用して必要なプロパティを取得
1. 1.の一部を用いて`CSS style object`を作成
1. 1.と2.を返す要素のrootに持たせる

といった具合で、これを`Item`と`Column`に適用させます。

しかし同じことを2度書くのも芸がないので、共通化して`SortableHolder`というコンポーネントを作ります。（ネーミングェ…）

```tsx:SortableHolder.tsx
import { useSortable } from "@dnd-kit/sortable";
import { CSS } from "@dnd-kit/utilities";
import { type CSSProperties, type ReactNode } from "react";

type SortableHolderProps = {
    readonly children: ReactNode;
    readonly id: string;
    readonly className?: string;
};

const SortableHolder = ({ children, id, className }: SortableHolderProps): JSX.Element => {
    const {
        attributes,
        listeners,
        setNodeRef,
        transform,
        transition
    } = useSortable({ id });

    const style: CSSProperties = {
        // CSSはWeb APIのインターフェスではなく@dnd-kit/utilitiesパッケージにある変数のほう
        transform: CSS.Transform.toString(transform),
        transition,
    };

    return (
        <div
            ref={setNodeRef}
            style={style}
            {...attributes}
            {...listeners}
            className={className}
        >
            {children}
        </div>
    );
};

export default SortableHolder;
```