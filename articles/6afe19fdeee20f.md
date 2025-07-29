---
title: "Next.js: 開発 Tips のハッカソンメモ"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [nextjs, font, dotenv]
published: true
publication_name: "ka_projects"
---

## はじめに

こないだ参加したハッカソンで得た知見の備忘録です。

## Google Fonts の導入

`next/font/google`を利用することで、パフォーマンスを最適化しつつ Google Fonts を容易に導入できる。`layout.tsx`で設定し、CSS 変数として全体に適用するのが基本パターン。

```tsx
// layout.tsx
import { Hachi_Maru_Pop } from "next/font/google";

const hachiMaruPop = Hachi_Maru_Pop({
  weight: "400",
  variable: "--font-hachi-marupop",
  subsets: ["latin"],
});

// bodyタグのclassNameに`${hachiMaruPop.variable}`などを指定して使用
```

---

## `curl`による API の疎通確認

UI を介さずに API エンドポイントを直接テストする場合、`curl`コマンドが有効。ログイン API を叩いてクッキーをファイル(`cookies.txt`)に保存し、そのクッキーを使って認証が必要な API を叩く、といった連携テストが可能。

```bash
# ログインAPIを叩き、セッションクッキーをcookies.txtに保存
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}' \
  -c cookies.txt

# 保存したクッキーを使って認証が必要なAPIにPOSTリクエスト
curl -X POST http://localhost:3000/api/posts \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d '{"content":"テスト投稿です"}'
```

---

## Node.js スクリプトでの`.env`利用

DB マイグレーションなど、Next.js の実行コンテキスト外で Node.js スクリプトを実行する場合、`.env`ファイルは自動で読み込まれない。`dotenv`パッケージを利用して明示的に読み込む必要がある。

```ts
// scripts/some-script.ts
import dotenv from "dotenv";

dotenv.config();

// これでprocess.env.DATABASE_URLなどが利用可能になる
console.log("DATABASE_URL:", process.env.DATABASE_URL ? "OK" : "NOT SET");
```

---

## レイアウトファイルの制約

`layout.tsx`ファイルでは、`metadata`や`default`エクスポートなど、Next.js が予約した特定のエクスポートのみが許可される。カスタムの名前付きエクスポートを追加することはできない。

---

## `middleware.ts`による全体的なルート制御

認証状態に応じたリダイレクトなど、複数のページにまたがる共通のアクセス制御ロジックは`middleware.ts`に集約する。

```ts
// middleware.ts
import { NextRequest, NextResponse } from "next/server";
import { jwtVerify } from "jose";

const AUTH_PATHS = ["/login", "/signup"];
const SECRET = new TextEncoder().encode(process.env.JWT_SECRET!);

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const token = request.cookies.get("auth-token")?.value;

  // ログイン済みのユーザーがログインページなどにアクセスした場合
  const isAuthPath = AUTH_PATHS.some((path) => pathname.startsWith(path));
  if (isAuthPath && token) {
    try {
      await jwtVerify(token, SECRET);
      return NextResponse.redirect(new URL("/", request.url));
    } catch (error) {
      /* NOOP */
    }
  }

  // 保護されたルートに未ログインでアクセスした場合
  if (!isAuthPath && !token) {
    return NextResponse.redirect(new URL("/login", request.url));
  }

  // トークンはあるが、無効な場合
  if (!isAuthPath && token) {
    try {
      await jwtVerify(token, SECRET);
    } catch (err) {
      return NextResponse.redirect(new URL("/login", request.url));
    }
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/((?!api|_next/static|_next/image|favicon.ico).*)"],
};
```
