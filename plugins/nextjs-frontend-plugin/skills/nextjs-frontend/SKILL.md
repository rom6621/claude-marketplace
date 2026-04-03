---
name: nextjs-frontend
description: Next.js App Routerフロントエンド開発ガイド。フォーム実装、コンポーネント設計、スタイリングの標準フロー。新機能実装時に自動的に使用。
---

# Next.js Frontend Development Skill

このスキルは、Next.js App Router製フロントエンドの開発に必要な知識とワークフローを提供します。

## 技術スタック

- **パッケージマネージャー**: pnpm
- **フレームワーク**: Next.js (App Router)
- **言語**: TypeScript
- **フォームライブラリ**: Conform + Zod
- **スタイリング**: Tailwind CSS
- **UIコンポーネント**: react-aria-components
- **状態管理**: React hooks (useState, useTransition, useActionState)
- **Server Actions**: Next.js Server Actions

## 絶対に守るべき3つのルール

### 1. HTMLセマンティックルールの厳守

**CRITICAL**: 実装前に必ず[MDN](https://developer.mozilla.org/ja/)でHTML仕様を確認すること。

```tsx
// ❌NG: <a>の中に<button>を入れてはいけない
<Link href="/items/1">
  <Button>開く</Button>
</Link>

// ✅OK: LinkButtonコンポーネントを使う
<LinkButton href="/items/1">開く</LinkButton>
```

**理由**: HTMLの仕様上、`<a>`要素はインタラクティブコンテンツ（`<button>`, `<a>`, `<input>`など）を含むことができません。

### 2. `name` 属性は必須

FormDataに含めたい全ての入力要素に `name` 属性を設定する。

```tsx
// ❌NG: name属性がない
<input id={field.id} defaultValue={field.initialValue} />

// ✅OK: name属性を設定
<input name={field.name} id={field.id} defaultValue={field.initialValue} />
```

**理由**: これがないとServer ActionでFormDataを受け取れず、フォームが機能しません。

### 3. UIコンポーネントを使う

プロジェクトで定義された `Input`, `Button`, `LinkButton` 等のUIコンポーネントを使用する。

```tsx
import { Input } from "@/components/ui/input";
import { Button } from "@/components/ui/button";
import { LinkButton } from "@/components/ui/link-button";
```

**理由**: これらには重要なデフォルト設定（アクセシビリティ、スタイル等）が含まれています。

## アーキテクチャ

### ディレクトリ構造

```
frontend/src/
├── app/                    # Next.js App Router
│   ├── (auth)/            # 認証が必要なページ
│   │   ├── page.tsx       # ダッシュボード
│   │   └── layout.tsx     # 認証レイアウト
│   ├── globals.css        # グローバルスタイル
│   └── layout.tsx         # ルートレイアウト
├── components/            # 共通UIコンポーネント
│   ├── ui/               # 汎用UIコンポーネント
│   ├── header/           # ヘッダー
│   └── sidebar/          # サイドバー
├── features/             # 機能別ディレクトリ
│   └── [feature]/        # 各機能
│       ├── actions/      # Server Actions
│       ├── components/   # コンポーネント
│       ├── schema.ts     # Zodスキーマ
│       └── types.ts      # 型定義
├── libs/                 # ライブラリ・ユーティリティ
│   └── api/             # APIクライアント
└── utils/               # ユーティリティ関数
```

### レイヤー構造

- **Page Layer (Server Component)**: データフェッチとルーティング
- **Component Layer (Client Component)**: UI とインタラクション
- **Action Layer (Server Actions)**: フォーム送信とデータ更新
- **API Layer**: バックエンドAPI呼び出し

## 新機能実装フロー

1. **型定義とスキーマ定義**: `features/[feature]/types.ts`, `schema.ts`
2. **Server Actions 実装**: `features/[feature]/actions/`
3. **Page 実装 (Server Component)**: `app/(auth)/[path]/page.tsx`
4. **Component 実装 (Client Component)**: `features/[feature]/components/`
5. **スタイリング調整**: Tailwind CSS + cn()
6. **動作確認**: `pnpm dev`

### 重要なポイント

- **Server Actions**: `parseWithZod` → `submission.reply()` → `revalidatePath()`
- **Client Components**: `useActionState` + `useForm` で状態管理
- **UIコンポーネント**: プロジェクトの共通UIを使用

## フォーム実装パターン

### 基本構造: Conform + Zod + Server Actions

#### Step 1: スキーマ定義

```typescript
// features/[feature]/schema.ts
import z from "zod";

export const updateItemSchema = z.object({
  id: z.string(),
  name: z.string().min(1, "名前を入力してください"),
});
```

#### Step 2: Server Action 実装

```typescript
// features/[feature]/actions/updateItem.ts
"use server";

import { parseWithZod } from "@conform-to/zod";
import { updateItemSchema } from "../schema";
import { revalidatePath } from "next/cache";
import { apiClient } from "@/libs/api/client";

export async function updateItem(_: unknown, formData: FormData) {
  // 1. バリデーション
  const submission = parseWithZod(formData, {
    schema: updateItemSchema,
  });
  if (submission.status !== "success") {
    return submission.reply();
  }

  const { id, name } = submission.value;

  // 2. API呼び出し
  try {
    await apiClient.put(`/items/${id}`, { name });

    // 3. キャッシュ再検証
    revalidatePath("/items");

    // 4. 成功レスポンス
    return submission.reply({ resetForm: true });
  } catch (error) {
    // 5. エラーハンドリング
    const message = error instanceof Error ? error.message : "Unknown error";
    return submission.reply({
      formErrors: [message],
    });
  }
}
```

#### Step 3: Client Component 実装

```typescript
// features/[feature]/components/item-form.tsx
"use client";

import { useActionState } from "react";
import { useForm } from "@conform-to/react";
import { parseWithZod } from "@conform-to/zod";
import { updateItem } from "../actions/updateItem";
import { updateItemSchema } from "../schema";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { cn } from "@/utils/cn";
import type { Item } from "../types";

type Props = {
  item: Item;
};

export function ItemForm({ item }: Props) {
  const [lastResult, action, isPending] = useActionState(updateItem, undefined);

  const [form, { id, name }] = useForm({
    lastResult,
    defaultValue: item,
    onValidate({ formData }) {
      return parseWithZod(formData, { schema: updateItemSchema });
    },
  });

  return (
    <form
      id={form.id}
      onSubmit={form.onSubmit}
      action={action}
      className={cn("p-6", "flex flex-col gap-3", "border rounded-lg")}
    >
      <input type="hidden" name={id.name} value={item.id} />

      <label htmlFor={name.id}>名前</label>
      <Input
        type="text"
        name={name.name}
        id={name.id}
        key={name.key}
        defaultValue={name.initialValue}
      />

      <Button type="submit" isDisabled={isPending}>
        {isPending ? "更新中..." : "更新"}
      </Button>
    </form>
  );
}
```

### useActionState の使い方

```typescript
const [lastResult, action, isPending] = useActionState(myAction, undefined);
```

- **lastResult**: Server Actionからの最新のレスポンス（エラー情報を含む）
- **action**: フォームのaction属性に渡す関数
- **isPending**: フォーム送信中かどうかのBoolean値

### useForm の使い方

```typescript
const [form, { field1, field2 }] = useForm({
  lastResult,              // Server Actionからのレスポンス
  defaultValue: initialData, // 初期値
  onValidate({ formData }) {  // クライアント側バリデーション
    return parseWithZod(formData, { schema: mySchema });
  },
});
```

- **form**: フォームのメタデータ（id, onSubmit, errors など）
- **fields**: 各フィールドのメタデータ（name, id, key, initialValue など）

### ローディング状態の管理

#### フォーム送信: useActionState

`isPending` を使用してボタンの disabled 状態とテキストを制御。

#### その他の非同期処理: useTransition

```typescript
const [isDeleting, startTransition] = useTransition();
const [deletingId, setDeletingId] = useState<string | null>(null);

const handleDelete = (id: string) => {
  setDeletingId(id);
  startTransition(async () => {
    await deleteAction(id);
    setDeletingId(null);
  });
};

<Button
  onPress={() => handleDelete(item.id)}
  isDisabled={isDeleting}
>
  {isDeleting && deletingId === item.id ? "削除中..." : "削除"}
</Button>
```

### エラーハンドリング

```typescript
// Server Action
return submission.reply({
  formErrors: ["エラーメッセージ"],
});

// Component
{form.errors && <ErrorText>{form.errors}</ErrorText>}
```

### 実装チェックリスト

#### スキーマ定義
- [ ] `features/[feature]/schema.ts` に Zod スキーマを定義
- [ ] エラーメッセージを適切に記述

#### Server Action
- [ ] "use server" ディレクティブ
- [ ] `parseWithZod` でバリデーション
- [ ] submission.status チェック
- [ ] try-catch でエラーハンドリング
- [ ] submission.reply() でレスポンス
- [ ] revalidatePath() でキャッシュ更新

#### Client Component
- [ ] "use client" ディレクティブ
- [ ] useActionState + useForm を使用
- [ ] field.name, field.id, field.key, field.initialValue を設定
- [ ] isPending でローディング状態を管理
- [ ] UIコンポーネントを使用

## コンポーネント設計パターン

### Server Component でデータフェッチ

```typescript
// app/(auth)/items/[itemId]/settings/page.tsx
export default async function SettingsPage({ params }: Props) {
  const { itemId } = await params;

  const result = await getItem(itemId);
  if (!result.success) {
    throw new Error(result.error);
  }

  return (
    <div className={cn("w-4xl mx-auto", "flex flex-col gap-5")}>
      <h1>設定</h1>
      <ItemForm item={result.result} />
    </div>
  );
}
```

### 1ファイルで完結

- 関連する処理は1つのファイルにまとめる
- 不要なコンポーネント分割をしない
- 200〜300行程度なら分割不要

### JSX設計パターン

#### 浅い階層を維持

```tsx
// ❌NG
<div className="container">
  <div className="wrapper">
    <div className="inner">
      <label>ラベル</label>
      <div className="input-wrapper">
        <input />
      </div>
    </div>
  </div>
</div>

// ✅OK
<div className="container">
  <label>ラベル</label>
  <input />
</div>
```

#### セマンティックな構造

```tsx
<form className={cn("p-6", "flex flex-col gap-6", "border rounded-lg")}>
  <h2>フォームタイトル</h2>

  <label htmlFor="name">名前</label>
  <input id="name" type="text" />

  <hr />

  <label htmlFor="email">メール</label>
  <input id="email" type="email" />

  <div className={cn("flex justify-end gap-2")}>
    <Button type="submit">送信</Button>
  </div>
</form>
```

## データ取得パターン

### 複数エンドポイントからの並列取得

```typescript
export async function getItem(id: string): Promise<Result<Item>> {
  try {
    const response = await apiClient.get<ItemResponse>(`/items/${id}`);

    // 関連データを並列取得
    const [owner, category] = await Promise.all([
      apiClient.get<User>(`/users/${response.ownerId}`),
      apiClient.get<Category>(`/categories/${response.categoryId}`),
    ]);

    return {
      success: true,
      result: { ...response, owner, category },
      error: null,
    };
  } catch (error) {
    return {
      success: false,
      result: null,
      error: error instanceof Error ? error.message : "Unknown error",
    };
  }
}
```

### 型設計パターン（Response型とフロントエンド型の分離）

```typescript
// features/[feature]/types.ts

// バックエンドAPIのレスポンス型（IDのみ）
export type ItemResponse = {
  id: string;
  ownerId: string;
  categoryId: string;
  name: string;
  createdAt: string;
  updatedAt: string;
};

// フロントエンド型（完全なオブジェクト）
export type Item = {
  id: string;
  owner: User;
  category: Category;
  name: string;
  createdAt: string;
  updatedAt: string;
};
```

**メリット**:
- Server Actionでの変換が明示的
- 型安全性の向上（IDの誤使用を防ぐ）
- UIコンポーネントが常に完全なオブジェクトを前提にできる

## トラブルシューティング

### 1. FormDataが空になる

**原因**: `name` 属性の欠落

**デバッグ**:
```typescript
export async function myAction(_: unknown, formData: FormData) {
  console.log("FormData entries:", Array.from(formData.entries()));
}
```

### 2. APIが空レスポンスを返す（204 No Content）

**症状**: `Unexpected end of JSON input` エラー

**解決策**:
```typescript
const text = await response.text();
if (!text) {
  return null as T;
}
return JSON.parse(text) as T;
```

### 3. revalidatePathで画面が更新されない

**解決策**: 関連するパスを全て revalidate
```typescript
revalidatePath(`/items/${id}/settings`);
revalidatePath("/items");
```

### 4. Conformのフィールドが更新されない

**原因**: `key` 属性の欠落

```tsx
// ✅OK: keyを設定
<Input
  name={field.name}
  id={field.id}
  key={field.key}
  defaultValue={field.initialValue}
/>
```

### 5. useActionStateのisPendingが動作しない

**原因**: `action`属性に渡していない

```tsx
<form action={action} onSubmit={form.onSubmit}>
  <Button type="submit" isDisabled={isPending}>
    {isPending ? "送信中..." : "送信"}
  </Button>
</form>
```

### 6. 配列フィールドの追加・削除が動作しない（React Aria + Conform）

**原因**: Conformの`getButtonProps()`はReact Aria Buttonと互換性がない

```tsx
// ❌NG
<Button {...form.insert.getButtonProps({ name: fields.items.name })}>

// ✅OK: onPressで直接呼び出す
<Button
  type="button"
  onPress={() => {
    form.insert({
      name: fields.items.name,
      defaultValue: { name: "", amount: 0 }
    });
  }}
>
  追加
</Button>
```

## 避けるべきこと

- **過度な設計**: 要求された機能以外の追加や改善を避ける
- **過度なコンポーネント分割**: 小さすぎるコンポーネントに分割しない
- **不要なネスト**: div の入れ子を最小限に
- **早すぎる抽象化**: 1回限りの操作のためのヘルパーやユーティリティを作らない
- **手動の状態管理**: useActionState と Conform を使う
- **HTMLルール違反**: インタラクティブ要素のネストなど、MDNで確認すること

## 日付処理

**全ての日付処理はタイムゾーンを明示的に指定する。**

### 送信時

Server Actionで日付を送信する際は、タイムゾーン付きのISO形式で送信する。

```typescript
// yyyy-mm-dd形式をISO文字列に変換してAPIに渡す
body: {
  eventDate: `${eventDate}T00:00:00+09:00`,
}
```

### 表示時

日付を表示する際は、必ずタイムゾーンを指定する。

```typescript
// ✅OK: タイムゾーン指定あり
new Intl.DateTimeFormat("ja-JP", {
  year: "numeric",
  month: "long",
  day: "numeric",
  timeZone: "Asia/Tokyo",
}).format(date);

// ❌NG: タイムゾーン指定なし（サーバーのTZに依存）
new Date(dateString).toLocaleString("ja-JP");
```

## コーディング規約

- **TypeScript**: strict モードを使用
- **フォーマット**: Biome でフォーマット
- **命名規則**:
  - コンポーネント: PascalCase
  - 関数: camelCase
  - 定数: UPPER_SNAKE_CASE
- **エラーハンドリング**: 明示的なエラー表示
- **バリデーション**: Zod スキーマで定義

## 参考資料

### Context7 で最新ドキュメントを参照

ライブラリのAPIを確認する際は `use context7` を使用すること。LLMの学習データより新しい情報を取得できる。

対象ライブラリ:
- **Conform** - フォームAPI（field.initialValue等）
- **Zod** - バリデーションスキーマ
- **Next.js** - App Router、Server Actions
- **React Aria Components** - データ属性、アクセシビリティ
- **Tailwind CSS** - ユーティリティクラス
