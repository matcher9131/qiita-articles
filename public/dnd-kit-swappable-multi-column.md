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
# dnd-kitでアイテムもカラムもswappableなマルチカラムを実装してみた
Reactにてドラッグアンドドロップ（以下DnDと表記します）を実装するライブラリである[dnd-kit](https://dndkit.com/)を用いて、DnDによって

- カラムどうしの入れ替え
- 同カラム内のアイテムどうしの入れ替え
- 異なるカラム間でのアイテムどうしの入れ替え

が可能なマルチカラムを実装します。

※ 並び替えではなく入れ替えです。並び替えに関してはググるといくつか記事がヒットするので、そちらにお任せします…。

<!-- ここに動画 -->

## 使用パッケージ
いずれも記事執筆時点での最新です。
|パッケージ|バージョン|
|---|---|
|vite|5.3.4|
|react|18.3.1|
|recoil|0.7.7|
|tailwindcss|3.4.7|
|clsx|2.1.1|
|@dnd-kit/core|6.1.0|
|@dnd-kit/utilities|3.2.2|
|@dnd-kit/sortable|8.0.0|

## dnd-kitのインストール
ルートディレクトリで以下を実行します。今回は要素同士の並び替えを実装したいので、`sortable`パッケージもインストールします。

```bash
npm install @dnd-kit/core @dnd-kit/utilities @dnd-kit/sortable
```

（他のパッケージのインストールに関しては本記事の主内容から逸れるため省略させていただきます。）

## 状態管理
今回は状態管理をRecoilに任せます。勿論どの状態管理ライブラリでも大差ないので、他のものをお使いの方は適宜読み替えてください。

どのカラム内にどのアイテムがあるのかを管理するために以下の`atom`を作り、初期状態を適当に`default`にぶち込みます。
さらに指定した`header`を持つカラムの`items`を返す`selectorFamily`も用意しておきます。

```typescript:containerChildren.ts
export const containerChildrenState = atom<ReadonlyArray<{ header: string; items: string[] }>>({
    key: "containerChildrenState",
    default: [
        { header: "A", items: ["A1", "A2", "A3", "A4"] },
        { header: "B", items: ["B1", "B2"] },
        { header: "C", items: ["C1", "C2", "C3"] },
    ],
});

export const columnChildrenSelector = selectorFamily<readonly string[], string>({
    key: "columnChildrenSelector",
    get:
        (header: string) =>
        ({ get }) =>
            get(containerChildrenState).find((column) => column.header === header)?.items ?? [],
});
```

## コンポーネントの作成
とりあえずサクっと`Item`、`Column`、`Container`コンポーネントを作ります。後の実装を考え、これらは（コンテナ・プレゼンテーションパターンにおける）プレゼンテーションコンポーネントとします。

<details>
<summary>Item</summary>

```tsx:Item.tsx
type ItemProps = {
    readonly labelText: string;
    readonly className?: string;
};

const Item = ({ labelText, className }: ItemProps): JSX.Element => {
    return <div className={`w-full flex items-center p-2 ${className}`}>{labelText}</div>;
};

export default Item;
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

これをベースとして、DnD機能を実装していきます。

## `DndContext`の追加
dnd-kitでは *Draggable* コンポーネント（ドラッグする要素）と *Droppable* コンポーネント（ドラッグ要素を受け付ける要素）を実装することでDnDを実現しますが、それらの`Context Provider`として親に`DndContext`を置く必要があります。

加えて、dnd-kitで提供される機能はあくまでDnDによる見た目の変化を担うものであり、実際のデータの変化などに関してはDnD終了時に呼び出される`DndContext`の`onDragEnd`イベントハンドラに記述する必要があります。が、解説の都合で今は`undefined`にしておきます。

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
dnd-kitにおいては`useDraggable`フックにて *Draggable* コンポーネントを、`useDroppable`フックにて *Droppable* コンポーネントを作成できますが、今回作る入れ替え可能な要素のように *Draggable* かつ *Droppable* なコンポーネントには`useSortable`フックを使います。使い方としては

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
        // CSSはWeb APIのインターフェースではなく@dnd-kit/utilitiesパッケージにある変数のほう
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

## `Item`と`Column`のラッパーを作成
先ほどの`SortableHolder`を用いて各コンポーネントのラッパーを作成します。今度は（コンテナ・プレゼンテーションパターンにおける）コンテナコンポーネントとします。

```tsx:SortableItem.tsx
import Item from "./Item";
import SortableHolder from "../sortableHolder";
import { getItemBgColor } from "../../util";

type SortableItemProps = {
    readonly labelText: string;
};

const SortableItem = ({ labelText }: SortableItemProps): JSX.Element => {
    return (
        <SortableHolder id={labelText}>
            <Item labelText={labelText} className={getItemBgColor(labelText)} />
        </SortableHolder>
    );
};

export default SortableItem;
```

*Sortable* なコンポーネントは`SortableContext`の内部に置く必要があるため、`Column`の内側に配置します。`items`には内側に入る各コンポーネントを表す *id*（`useSortable`の引数に指定したものと同じ）の配列[^1]を、`strategy`には`dnd-kit`側に用意されている変数を指定します。DnDによる並び替えを目的とする場合はデフォルトの`rectSortingStrategy`で大丈夫ですが、今回は並び替えではなく入れ替えなので`rectSwappingStrategy`を指定します。
[^1]: *id* の順序はコンポーネントの順序と揃っている必要があります。なお *id* そのものだけではなく「文字列型または数値型の`id`プロパティを持つオブジェクト」も受け入れるので、コンポーネントに渡すモデルそのものを指定するのも手です。

```tsx: SortableColumn.tsx
import { rectSwappingStrategy, SortableContext } from "@dnd-kit/sortable";
import { useRecoilValue } from "recoil";
import Column from "./Column";
import SortableHolder from "../sortableHolder";
import SortableItem from "../item/SortableItem";
import { columnChildrenSelector } from "../../models/containerChildren";

type SortableColumnProps = {
    readonly header: string;
};

const SortableColumn = ({ header }: SortableColumnProps): JSX.Element => {
    const items = useRecoilValue(columnChildrenSelector(header));
    return (
        <SortableHolder id={header} className="flex-1">
            <Column header={header}>
                <SortableContext items={items} strategy={rectSwappingStrategy}>
                    {items.map((labelText) => (
                        <SortableItem key={labelText} labelText={labelText} />
                    ))}
                </SortableContext>
            </Column>
        </SortableHolder>
    );
};

export default SortableColumn;
```

`Column`も *Sortable* にしたいので`Container`の内側にも`SortableContext`を配置します。

```diff_tsx:App.tsx(抜粋)
const App = (): JSX.Element => {
    const columns = useRecoilValue(containerChildrenState);

    return (
        <div className="w-full p-5">
            <DndContext>
                <Container>
-                   {columns.map((column) => (
-                       <Column key={column.header} header={column.header}>
-                           {column.items.map((item) => (
-                               <Item key={item} labelText={item} className={getItemBgColor(item)} />
-                           ))}
-                       </Column>
-                   ))}
+                   <SortableContext items={columns.map((column) => column.header)} strategy={rectSwappingStrategy}>
+                       {columns.map((column) => (
+                           <SortableColumn key={column.header} header={column.header} />
+                       ))}
+                   </SortableContext>
                </Container>
            </DndContext>
        </div>
    );
};
```

## `onDragEnd`の実装
この状態でDnD自体はできるようになりましたが、先述の通り`DndContext`の`onDragEnd`を実装するまでは実際のデータの並びは書き換えられないため、DnDが終了するとすぐに開始前の状態に戻ってしまいます。

`onDragEnd`イベントハンドラの引数に`DragEndEvent`イベントパラメータがあり、`active`プロパティにドラッグした要素の *id* が、`over`プロパティにドロップした要素の *id* が格納されているので、それを使ってデータの入れ替えを実装します。

入れ替え操作が行数を食うので、まずはフックに切り出しておきます。

```tsx:useContainerChildren.ts
import { useRecoilCallback } from "recoil";
import { containerChildrenState } from "./containerChildren";

const toSwapped = <T>(arr: readonly T[], index1: number, index2: number): T[] => {
    const newArray = [...arr];
    [newArray[index1], newArray[index2]] = [newArray[index2], newArray[index1]];
    return newArray;
};

export const useContainerChildren = () => {
    const swap = useRecoilCallback(({ set, snapshot }) => (id1: string, id2: string) => {
        const columns = snapshot.getLoadable(containerChildrenState).getValue();

        const headerIndex1 = columns.findIndex((column) => column.header === id1);
        const headerIndex2 = columns.findIndex((column) => column.header === id2);
        if (headerIndex1 >= 0 && headerIndex2 >= 0) {
            // Columnどうしの入れ替え
            set(containerChildrenState, (prev) => toSwapped(prev, headerIndex1, headerIndex2));
        } else {
            // Itemどうしの入れ替え
            const parentColumn1 = columns.find((column) => column.items.includes(id1));
            const parentColumn2 = columns.find((column) => column.items.includes(id2));
            if (parentColumn1 == null || parentColumn2 == null) return;

            if (parentColumn1 === parentColumn2) {
                // 同Column内
                set(containerChildrenState, (prev) =>
                    prev.map((column) =>
                        column === parentColumn1
                            ? {
                                  header: column.header,
                                  items: toSwapped(column.items, column.items.indexOf(id1), column.items.indexOf(id2)),
                              }
                            : column,
                    ),
                );
            } else {
                // 異Column間
                set(containerChildrenState, (prev) =>
                    prev.map((column) =>
                        column === parentColumn1
                            ? { header: column.header, items: column.items.map((item) => (item === id1 ? id2 : item)) }
                            : column === parentColumn2
                              ? {
                                    header: column.header,
                                    items: column.items.map((item) => (item === id2 ? id1 : item)),
                                }
                              : column,
                    ),
                );
            }
        }
    });

    return { swap };
};
```

これを用いて`App.tsx`を書き換えます。

```diff_tsx:App.tsx(抜粋)
const App = (): JSX.Element => {
    const columns = useRecoilValue(containerChildrenState);
+   const { swap } = useContainerChildren();

+   const handleDragEnd = (e: DragEndEvent): void => {
+       const activeId = e.active.id.toString();
+       const overId = e.over?.id?.toString();
+       if (overId == null) return;
+
+       // 特定の要素のみ入れ替えを禁止する場合はここで早期returnさせます。
+       // 試しにA1とB1の入れ替えを禁止してみます。
+       if ((activeId === "A1" && overId === "B1") || (activeId === "B1" && overId === "A1")) return;
+
+       swap(activeId, overId);
+    };

    return (
        <div className="w-full p-5">
-           <DndContext>
+           <DndContext onDragEnd={handleDragEnd}>
                <Container>
                    <SortableContext items={columns.map((column) => column.header)} strategy={rectSwappingStrategy}>
                        {columns.map((column) => (
                            <SortableColumn key={column.header} header={column.header} />
                        ))}
                    </SortableContext>
                </Container>
            </DndContext>
        </div>
    );
};
```

## `DragOverlay`の実装
これでDnDによる入れ替えができるようになりましたが、異なる`Column`間の`Item`を入れ替える際にドラッグしている`Item`が`Column`の外側に出てくれません。（入れ替え自体は成功します。）

<!-- ここに動画 -->

どうやらこの状態では *Sortable* なコンポーネントは直近祖先の`SortableContext`から出られないようです。[^2]
[^2]: `SortableContext`を`Container`の直下の1つのみにしてしまえば良いと思いきや、`Column`と`Item`が同じ`SortableContext`に属するため（たとえ`onDragEnd`で入れ替えを禁じていても）`Column`と`Item`を入れ替えようとするアニメーションが発生してしまいます。

そこで`DragOverlay`の出番です。`DragOverlay`はドラッグする要素を別途レンダーするためのコンポーネントで、これを使用することでDnDの際の見た目をカスタマイズすることができます。ただし、デフォルトの挙動の一部がなくなるため注意が必要です。

まず、どの要素をDnDしているのかを管理したいので`atom`を追加します。

```tsx:dragTargets.ts
import { atom } from "recoil";

export const activeIdState = atom<string | null>({ key: "activeIdState", default: null });
```

`DndContext`の`onDragStart`イベントハンドラでこの`activeIdState`に対象の *id* をセットし、`DragOverlay`で参照してコンポーネントをレンダーします。この際、 *Sortable* なコンポーネントをレンダーするといろいろとややこしくなるため、 *Sortable* ではないものを用います。（`DragOverlayColumn`はそのためのただの`Column`のラッパーです。）

なお`onDragEnd`で`activeIdState`をリセットするのをお忘れなく。

```diff_tsx:App.ts(抜粋)
const App = (): JSX.Element => {
    const columns = useRecoilValue(containerChildrenState);
    const { swap } = useContainerChildren();
+   const [activeId, setActiveId] = useRecoilState(activeIdState);
+
+   const handleDragStart = (e: DragStartEvent) => {
+       setActiveId(e.active.id.toString());
+   };

    const handleDragEnd = (e: DragEndEvent): void => {
+       setActiveId(null);
+
        const activeId = e.active.id.toString();
        const overId = e.over?.id?.toString();
        if (overId == null) return;
 
        // 特定の要素のみ入れ替えを禁止する場合はここで早期returnさせます。
        // 試しにA1とB1の入れ替えを禁止してみます。
        if ((activeId === "A1" && overId === "B1") || (activeId === "B1" && overId === "A1")) return;
 
        swap(activeId, overId);
     };

    return (
        <div className="w-full p-5">
-           <DndContext onDragEnd={handleDragEnd}>
+           <DndContext onDragStart={handleDragStart} onDragEnd={handleDragEnd}>
                <Container>
                    <SortableContext items={columns.map((column) => column.header)} strategy={rectSwappingStrategy}>
                        {columns.map((column) => (
                            <SortableColumn key={column.header} header={column.header} />
                        ))}
                    </SortableContext>
+                   <DragOverlay>
+                       {activeId != null &&
+                           (columns.some((column) => column.header === activeId) ? (
+                               <DragOverlayColumn header={activeId} />
+                           ) : (
+                               <Item labelText={activeId} className={getItemBgColor(activeId)} />
+                           ))}
                    </DragOverlay>
                </Container>
            </DndContext>
        </div>
    );
};
```

これで`Item`が親`Column`から巣立つことができましたが、分身が発生してしまいます。`DragOverlay`は先述の通りあくまでドラッグ中の要素を **別途に** レンダーするだけなので、元々の要素が残ってしまいます。

流石にこれではみっともないので、`SortableItem`と`SortableColumn`を修正してドラッグ対象となるものは表示しないようにします。

```diff_tsx:SortableItem.tsx
+import clsx from "clsx";
+import { useRecoilValue } from "recoil";
import Item from "./Item";
import SortableHolder from "../sortableHolder";
+import { activeIdState } from "../../models/dragTargets";
import { getItemBgColor } from "../../util";

type SortableItemProps = {
    readonly labelText: string;
};

const SortableItem = ({ labelText }: SortableItemProps): JSX.Element => {
+   const isDragActive = useRecoilValue(activeIdState) === labelText;
    return (
-       <SortableHolder id={labelText}>
+       <SortableHolder id={labelText} className={clsx(isDragActive && "opacity-0")}>
            <Item labelText={labelText} className={getItemBgColor(labelText)} />
        </SortableHolder>
    );
};

export default SortableItem;
```

```diff_tsx:SortableColumn.tsx
import { rectSwappingStrategy, SortableContext } from "@dnd-kit/sortable";
+import { clsx } from "clsx";
import { useRecoilValue } from "recoil";
import Column from "./Column";
import SortableHolder from "../sortableHolder";
import SortableItem from "../item/SortableItem";
import { columnChildrenSelector } from "../../models/containerChildren";
+import { activeIdState } from "../../models/dragTargets";

type SortableColumnProps = {
    readonly header: string;
};

const SortableColumn = ({ header }: SortableColumnProps): JSX.Element => {
    const items = useRecoilValue(columnChildrenSelector(header));
+   const isDragActive = useRecoilValue(activeIdState) === header;
    return (
-       <SortableHolder id={header} className="flex-1">
+       <SortableHolder id={header} className={clsx("flex-1", isDragActive && "opacity-0")}>
            <Column header={header}>
                <SortableContext items={items} strategy={rectSwappingStrategy}>
                    {items.map((labelText) => (
                        <SortableItem key={labelText} labelText={labelText} />
                    ))}
                </SortableContext>
            </Column>
        </SortableHolder>
    );
};

export default SortableColumn;
```

## 異`Column`間の`Item`どうしの入れ替えをわかりやすく
現状、`Column`どうしの入れ替えと同`Column`内の`Item`どうしの入れ替えに関してはドラッグ中に入れ替えアニメーションがされるため「入れ替え可能なんだな」とユーザーに思ってもらいやすい一方で、異`Column`間の`Item`どうしの入れ替えに関しては入れ替えアニメーションが無いために直感的に入れ替え可能に見えないという問題があります。

よって、異`Column`間の`Item`どうしの入れ替えの際に *Droppable* の見た目を少し変えることでこれを解決します。

まずはどの *Droppable* がターゲットになっているかを状態管理したいので、`atom`を追加します。さらに異`Column`間の`Item`どうしの入れ替えのみを検知するために、`activeId`と`overId`のそれぞれの親`Column`を取得する`selector`を作ります。

```diff_tsx:dragTargets.ts
-import { atom } from "recoil";
+import { atom, selector } from "recoil";
+import { containerChildrenState } from "./containerChildren";

export const activeIdState = atom<string | null>({ key: "activeIdState", default: null });

+export const overIdState = atom<string | null>({ key: "overIdState", default: null });

+export const activeParentIdSelector = selector<string | null>({
+   key: "activeParentIdSelector",
+   get: ({ get }) => {
+       const activeId = get(activeIdState);
+       return get(containerChildrenState)?.find((column) => column.items.includes(activeId ?? ""))?.header ?? null;
+   },
+});

+export const overParentIdSelector = selector<string | null>({
+   key: "overParentIdSelector",
+   get: ({ get }) => {
+       const overId = get(overIdState);
+       return get(containerChildrenState)?.find((column) => column.items.includes(overId ?? ""))?.header ?? null;
+   },
+});
```

次に、追加した`overIdState`を機能させるために`DndContext`の`onDragOver`イベントハンドラを実装します。例によって`onDragEnd`でリセットするのをお忘れなく。

```diff_tsx:App.tsx(抜粋)
const App = (): JSX.Element => {
    const columns = useRecoilValue(containerChildrenState);
    const { swap } = useContainerChildren();
    const [activeId, setActiveId] = useRecoilState(activeIdState);
+   const [, setOverId] = useRecoilState(overIdState);

    const handleDragStart = (e: DragStartEvent) => {
        setActiveId(e.active.id.toString());
    };

+   const handleDragOver = (e: DragOverEvent) => {
+       setOverId(e.over?.id?.toString() ?? null);
+   };

    const handleDragEnd = (e: DragEndEvent): void => {
        setActiveId(null);
+       setOverId(null);

        const activeId = e.active.id.toString();
        const overId = e.over?.id?.toString();
        if (overId == null) return;

        // 特定の要素のみ入れ替えを禁止する場合はここで早期returnさせます。
        // 試しにA1とB1の入れ替えを禁止してみます。
        if ((activeId === "A1" && overId === "B1") || (activeId === "B1" && overId === "A1")) return;

        swap(activeId, overId);
    };

    return (
        <div className="w-full p-5">
-           <DndContext onDragStart={handleDragStart} onDragEnd={handleDragEnd}>
+           <DndContext onDragStart={handleDragStart} onDragOver={handleDragOver} onDragEnd={handleDragEnd}>
                <Container>
                    <SortableContext items={columns.map((column) => column.header)} strategy={rectSwappingStrategy}>
                        {columns.map((column) => (
                            <SortableColumn key={column.header} header={column.header} />
                        ))}
                    </SortableContext>
                </Container>
                <DragOverlay>
                    {activeId != null &&
                        (columns.some((column) => column.header === activeId) ? (
                            <DragOverlayColumn header={activeId} />
                        ) : (
                            <Item labelText={activeId} className={getItemBgColor(activeId)} />
                        ))}
                </DragOverlay>
            </DndContext>
        </div>
    );
};
```

最後に、`SortableItem`を書き換えます。

```diff_tsx:SortableItem.tsx
import { clsx } from "clsx";
import { useRecoilValue } from "recoil";
import Item from "./Item";
import SortableHolder from "../sortableHolder";
-import { activeIdState } from "../../models/dragTargets";
+import { activeIdState, activeParentIdSelector, overIdState, overParentIdSelector } from "../../models/dragTargets";
import { getItemBgColor } from "../../util";

type SortableItemProps = {
    readonly labelText: string;
};

const SortableItem = ({ labelText }: SortableItemProps): JSX.Element => {
    const isDragActive = useRecoilValue(activeIdState) === labelText;
+   const isDragOver = useRecoilValue(overIdState) === labelText;
+   const isDraggingBetweenColumns = useRecoilValue(activeParentIdSelector) !== useRecoilValue(overParentIdSelector);
    return (
-       <SortableHolder id={labelText} className={clsx(isDragActive && "opacity-0")}>
+       <SortableHolder
+           id={labelText}
+           className={clsx(isDragActive && "opacity-0", isDragOver && isDraggingBetweenColumns && "opacity-50")}
+       >
            <Item labelText={labelText} className={getItemBgColor(labelText)} />
        </SortableHolder>
    );
};

export default SortableItem;
```

これでこのようになりました。

<!-- ここに動画 -->

今回は`opacity`をいじりましたが、`brightness`や`box-shadow`あたりを変化させてもいいかもしれません。

## `onDragCancel`の実装
これにて完成！…と言いたいところですが、まだ1つ仕事が残っています。

DnDは **Escキーを押すことでキャンセル** できてその際は当然`onDragEnd`が呼び出されないので、このままだと`activeIdState`や`overIdState`がリセットされず見た目が崩壊します。

`DnDContext`に`onDragcancel`イベントハンドラがあるので、これをちょちょいと実装して解決します。

```diff_tsx:App.tsx(抜粋)
const App = (): JSX.Element => {
    const columns = useRecoilValue(containerChildrenState);
    const { swap } = useContainerChildren();
    const [activeId, setActiveId] = useRecoilState(activeIdState);
    const [, setOverId] = useRecoilState(overIdState);

    const handleDragStart = (e: DragStartEvent) => {
        setActiveId(e.active.id.toString());
    };

    const handleDragOver = (e: DragOverEvent) => {
        setOverId(e.over?.id?.toString() ?? null);
    };

+   const handleDragCancel = () => {
+       setActiveId(null);
+       setOverId(null);
+   };

    const handleDragEnd = (e: DragEndEvent): void => {
        setActiveId(null);
        setOverId(null);

        const activeId = e.active.id.toString();
        const overId = e.over?.id?.toString();
        if (overId == null) return;

        // 特定の要素のみ入れ替えを禁止する場合はここで早期returnさせます。
        // 試しにA1とB1の入れ替えを禁止してみます。
        if ((activeId === "A1" && overId === "B1") || (activeId === "B1" && overId === "A1")) return;

        swap(activeId, overId);
    };

    return (
        <div className="w-full p-5">
-           <DndContext onDragStart={handleDragStart} onDragOver={handleDragOver} onDragEnd={handleDragEnd}>
+           <DndContext
+               onDragStart={handleDragStart}
+               onDragOver={handleDragOver}
+               onDragCancel={handleDragCancel}
+               onDragEnd={handleDragEnd}
+           >
                <Container>
                    <SortableContext items={columns.map((column) => column.header)} strategy={rectSwappingStrategy}>
                        {columns.map((column) => (
                            <SortableColumn key={column.header} header={column.header} />
                        ))}
                    </SortableContext>
                </Container>
                <DragOverlay>
                    {activeId != null &&
                        (columns.some((column) => column.header === activeId) ? (
                            <DragOverlayColumn header={activeId} />
                        ) : (
                            <Item labelText={activeId} className={getItemBgColor(activeId)} />
                        ))}
                </DragOverlay>
            </DndContext>
        </div>
    );
};
```

これにてようやく完成です。お疲れ様でした！

## Github
ここまでのコードをまとめたリポジトリを公開しています。よろしければご覧ください。

<!-- ここにリポジトリへのリンク -->

## 未解決課題
### 異`Column`間の`Item`どうしの入れ替えの際に、ドロップされたほうがアニメーションせずに瞬間移動する
解決策がわかりませんでした…。ご存じの方は教えていただけると幸いです！