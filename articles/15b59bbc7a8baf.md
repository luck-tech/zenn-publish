---
title: "アプリのメンテナンスページを設定する"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [nextjs]
published: true
published_at: 2024-12-17 09:00
---

## 記事の目的

メンテナンスページとは、メンテナンス中のアプリへのアクセスを一時的に制限し、ユーザーに状況を通知するためのページです。本記事では、Next.js を使用してメンテナンスページを設定する方法を解説します。

## メンテナンスページのコード例

まずはメンテナンスページの中身です。シンプルなメンテナンスページの実装例を載せときます。

### `maintenance.tsx`

```tsx
import React from "react";

const MaintenancePage: React.FC = () => {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen bg-gray-100 text-gray-800">
      <h1 className="text-4xl font-bold mb-4">🚧 メンテナンス中 🚧</h1>
      <p className="text-lg mb-6">
        サイトは現在メンテナンス中です。しばらくお待ちください。
      </p>
    </div>
  );
};

export default MaintenancePage;
```

## メンテナンスモードの設定方法

メンテナンスページは常に表示するわけにはいかないので、手動で設定する必要があります。
設定方法は、主に以下の 2 つが考えられると思います。

### 1. 環境変数を使用する方法

Next.js では、環境変数を利用してメンテナンスモードを設定する方法です。以下の手順で実装します。

#### 1.1 環境変数を定義する

まず、プロジェクトルートに`.env`ファイルを作成し、以下のように環境変数を定義します。

```env
IS_IN_MAINTENANCE_MODE=true
```

#### 1.2 環境変数を読み込む

次に、`next.config.ts`で環境変数を読み込みます。

```ts
module.exports = {
  env: {
    IS_IN_MAINTENANCE_MODE: process.env.IS_IN_MAINTENANCE_MODE,
  },
};
```

#### 1.3 `middleware.ts`を作る

```ts
import { NextRequest, NextResponse } from "next/server";

export const config = {
  matcher: "/big-promo", // 特定のURLでメンテナンスページを表示する。不要なら消してください
};

export function middleware(req: NextRequest) {
  // 環境変数をチェックしてメンテナンスモードを確認
  const isInMaintenanceMode = process.env.IS_IN_MAINTENANCE_MODE === "true";

  if (isInMaintenanceMode) {
    req.nextUrl.pathname = `/maintenance`;
    return NextResponse.rewrite(req.nextUrl);
  }
}
```

---

### 2. Vercel Edge Config を使用する方法

1 よりもこちらを推奨します。
![](https://storage.googleapis.com/zenn-user-upload/5e6d8e6464c7-20241213.png)
画像を見ればわかりますが、読み込み遅延がとても少なく、再デプロイが必要ないため、変更の反映が早いです。

ちなみに Vercel Edge Config は、Vercel のプラットフォームで提供される機能で、アプリケーションの動作をリアルタイムで動的に管理できる仕組みを提供します。
今回はこれを使って、メンテナンスモードのフラグを管理しようという話です。

#### 2.1 `middleware.ts`を作る

```ts
import { NextRequest, NextResponse } from "next/server";
import { get } from "@vercel/edge-config";

export const config = {
  matcher: "/big-promo", // 特定のURLでメンテナンスページを表示する。不要なら消してください
};

export async function middleware(req: NextRequest) {
  // Edge Configを使用してメンテナンスモードを確認
  const isInMaintenanceMode = await get("isInMaintenanceMode");

  // メンテナンスモードが有効な場合、メンテナンスページにリダイレクト
  if (isInMaintenanceMode) {
    req.nextUrl.pathname = `/maintenance`;
    return NextResponse.rewrite(req.nextUrl);
  }
}
```

#### 2.2 Vercel Edge Config を設定する

1. Vercel のプロジェクトページに飛ぶ
2. 上のタブから Storage を選択
3. データベースを作って繋ぐ。オプションはつけなくていいと思います
4. データベースをクリックして設定ページに飛ぶ
5. Items タブの Store Items に環境変数を置けるので、以下のように設定する

```
{
  "isMaintenance": true
}
```

## おまけ： `middleware.ts`の置き場所

Next.js では、`middleware.ts`は以下のいずれかのディレクトリに配置する必要があります。

- プロジェクトルート
- `app/`
- `pages/`
- `src/`

**注意**: `src/`フォルダを作成している場合は、`src/middleware.ts`に配置してください。
&nbsp;
&nbsp;
&nbsp;
&nbsp;
以上です。
