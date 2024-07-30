## 使用パッケージ
|パッケージ|バージョン|
|---|---|
|vite|5.3.4|
|react|18.3.1|
|recoil|0.7.7|
|tailwindcss|3.4.7|
|@dnd-kit/core|6.1.0|
|@dnd-kit/sortable|8.0.0|

## dnd-kitのインストール
ルートディレクトリで以下を実行します。

```bash:bash
npm install @dnd-kit/core
```

今回は要素同士の並び替えを実装したいので、以下もインストールします。

```bash:bash
npm install @dnd-kit/sortable
```

## 状態管理
今回は状態管理を`Recoil`に任せます。どのカラム内にどのアイテムがあるのかを管理するために以下の`atom`を作ります。

```typescript
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
`Container`、`Column`、`Item`コンポーネントを作成し、先ほどの`atom`を参照して各コンポーネントを配置します。ひとまず以下の画像のようになりました。

<!-- この時点でのコミット： https://github.com/matcher9131/dnd-kit-swappable-multi-column/commit/2e66ceac63fe66587e0b67cf1e0eaaa625663102 -->