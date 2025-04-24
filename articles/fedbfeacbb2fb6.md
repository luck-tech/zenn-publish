---
title: "Jestを使ったテスト設計の基本と実践"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [jest, msw, graphql, test]
published: true
publication_name: "ka_projects"
published_at: 2025-05-28 09:00
---

# 1. Jest とは

Jest は、Facebook 社が開発した JavaScript 用のテスティングフレームワークです。シンプルな設定で利用できる一方、モックやタイマー制御などの高度な機能も備えています。

Jest の主な特徴は以下の通りです：

- ゼロコンフィグ：設定なしですぐに使用開始できる
- スナップショットテスティング：UI コンポーネントの変更を検出
- モック機能：外部依存の制御と分離
- 並行テスト実行：高速な実行環境
- コードカバレッジレポート：テストの網羅性確認

# 2. テスト設計の考え方

## 2.1 テストケースの洗い出し方

効果的なテストを書くためには、適切なテストケースを洗い出すことが重要です。テスト点を決める際は、主に以下のポイントに着目します：

### A. 条件分岐の結果が変わる箇所

コード内で結果が変わる分岐点をテスト対象にします：

- `if (isUserPremium) { return true; }` のような明示的な分岐
- 三項演算子 `condition ? valueA : valueB` による暗黙的な分岐
- 論理演算子 `value || defaultValue` による短絡評価

これらの分岐ごとに、異なるテストケースを作成します。

### B. 境界値とエッジケース

バグは境界条件で発生しやすいため、以下のような境界値に注目します：

- 数値が 0 か 1 以上か（または負の値への対応）
- 配列が空か要素があるか
- オブジェクトが`null`/`undefined`か有効な値か
- 日付が特定のタイムゾーンをまたぐケース

## 2.2 外部依存の有無によるテスト設計の違い

テスト設計は対象関数の特性によって大きく異なります：

| 特性     | 外部依存なし（純粋関数） | 外部依存あり（フック、API 通信など） |
| -------- | ------------------------ | ------------------------------------ |
| 複雑さ   | シンプル                 | 複雑                                 |
| 設計方法 | 入力と出力の検証のみ     | モックのセットアップが必要           |

Jest でフックや外部依存のある関数をテストする場合、外部で定義された関数やフックの挙動を制御するためにモックが必要です。テスト環境では実際の API や他のモジュールの実装は利用できないため、これらをシミュレートする必要があります。

つまり、外部依存のあるテストは、ただそのフックに引数をぽいと投げれば良いわけではなく、中で動いている外部依存システムをモックする前準備が必要になるということです。

# 3. モックする際の流れ

2 つのモックについて扱います。

1. **API モック（MSW）**: HTTP 呼び出しをインターセプトしてモックレスポンスを返す

   流れ:

   1. アプリケーションコードが API リクエストを発行
   2. MSW がリクエストをインターセプト
   3. 設定したハンドラーがモックレスポンスを返却

2. **フックモック（Jest）**: フック自体をモックして任意の戻り値を設定する

   流れ:

   1. テスト実行時に Jest がモジュールをインターセプト
   2. 実際の関数/オブジェクトの代わりにモック関数を提供
   3. アプリケーションコードはモック関数を実行

基本、API などの非同期処理のモックは MSW で良いです。ネットワークレベルでモックしてくれるので。
で、そのモックしたものを Jest のテストで呼び出して使うで問題ないのですが、環境によっては mutation を MSW でモックすると、外部からの副作用で CI 上でテストが落ちてしまうことがあるらしいです。その場合は Jest によるモックで対応する必要が出てきます。

それぞれ見ていきましょう。

## 3.1 モック by Jest

手順としては、モック関数を作成する、モック関数にアクセスする、モック関数の返す値を設定する、という流れです。

### A. モック関数を作成する方法

**① jest.fn()**

`jest.fn()`は個別の関数をモック化するための機能です：

```jsx
const mockFunction = jest.fn();
```

この関数は呼び出し回数や引数を追跡でき、テスト内で呼び出されたかどうかを検証できます。

**② jest.mock()**

`jest.mock()`はモジュール全体をモック化します：

```jsx
jest.mock('@/utils/api', () => ({
  fetchData: jest.fn()
})
```

モジュールのインポート時点で置き換えが行われるため、テスト対象のコードが依存するすべての外部モジュールをコントロールできます。

**③ jest.requireActual()**

部分的なモックを行う場合に、元のモジュールの実装を取得するために使用します：

```jsx
jest.mock("@/utils/helpers", () => {
  return {
    ...jest.requireActual("./utils/helpers"), // 元の実装を保持
    getCurrentDate: jest.fn(), // この関数だけ置き換え
  };
});
```

### B. モック関数にアクセスする方法

まずはモックしたモジュールにアクセスしましょう。

**① require()**

```jsx
jest.mock("@/hooks/userData", () => ({
  useData: jest.fn(),
}));

// テストケース内で動的に変更
it("データがロード中の場合", () => {
  require("@/hooks/userData").useData.mockReturnValue({
    loading: true,
    data: null,
  });
  // テスト内容...
});
```

**② jest.spyOn()**

```jsx
import * as userDataHooks from "@/hooks/userData";

jest.mock('@/hooks/userData', () => ({
  useData: jest.fn()}));

it('データがロード中の場合', () => {
  const selectMainAccountMock = jest.fn();
  jest.spyOn(userDataHooks, "useData").mockReturnValue({
    isPending: false,
    mutate: selectMainAccountMock,
  } as unknown as ReturnType<typeof userDataHooks.useData>);
  ...
});
```

2 者の違いは、

> **1. 型の安全性**
>
> - **require()**: 型情報が失われるリスクがある
> - **spyOn()**: ReturnType<typeof ...>などの型アサーションで型安全性を確保しやすい
>
> **2. コード補完と IDE サポート**
>
> - **require()**: 動的インポートのため、IDE のコード補完サポートが弱い
> - **spyOn()**: 静的インポートを使用するため、IDE のコード補完が効果的
>
> **3. モックの明示性**
>
> - **require()**: モジュールの参照取得がテストケース内に埋め込まれる
> - **spyOn()**: モック関数の作成や監視対象が明示的で可視性が高い
>
> **4. 関連関数の利用**
>
> - **require()**: 単純に値を設定するだけの場合に簡潔
> - **spyOn()**: 同じテスト内の他のコードで作成したモック関数を連携させやすい（例：selectMainAccountMock）

とのことです。spyOn を使っておけば良さそうですね。

### C. モックした関数が返す値を設定する方法

**mockReturnValue**, **mockImplementation()**はアクセスしたモック関数に対して、返す値を設定するためのメソッドです。

両者の違いは以下の通り。

- **mockReturnValue**：単純な固定値を返す場合
  ```jsx
  mockFunction.mockReturnValue(5);
  ```
- **mockImplementation**：引数によって返り値を変えるなど、条件によって異なる値を返す場合
  ```jsx
  mockFunction.mockImplementation((arg) => {
    if (arg === "A") return 10;
    return 20;
  });
  ```

また、非同期処理を含む機能をテストするときは`mockResolvedValue` / `mockRejectedValue` を使います。

### おまけ: **clearAllMocks(),** resetAllMocks()

各テストケース内で毎回`mockReturnValue`など返す値を明示的に設定しない場合、モックをクリーンアップする必要が出てきます。そこで使います。

2 者の違いは

- `clearAllMocks()`は、2 つの assertion 間でモックをクリーンアップしたい際に便利。
- `resetAllMocks()`は、`clearAllMocks()`の実行内容に加えて、`return values or implementations`を削除可能。

らしいです(参考: [[Jest] clearAllMocks()と resetAllMocks()の違いについて確認してみた](https://dev.classmethod.jp/articles/jest-the-difference-between-clearallmocks-and-resetallmocks/))

resetAllMocks 使っとけば間違いなさそうですね。

```jsx
// 各テストが始まる前に実行
beforeEach(() => {
  jest.resetAllMocks();
});

// 各テストが終わった後に実行
afterEach(() => {
  jest.resetAllMocks();
});
```

これによりテスト間の依存関係を排除し、各テストを独立して実行できます。

## 3.2 モック by MSW

### 3.2.1 MSW 独特の文法と主要コンセプト

**基本構造:**

MSW(Mock Service Worker)はネットワークレベルでリクエストをインターセプトするライブラリで、以下の要素で構成されています。

```tsx
// ハンドラー定義
const handlers = [
  http.get('/api/users', () => {
    return HttpResponse.json({ users: [...] })
  })
];

// サーバーセットアップ
const server = setupServer(...handlers);
```

**ハンドラー定義:**

例として、GraphQL をモックする場合のハンドラー定義。

```tsx
import { graphql } from "msw";

export const fbSdk = graphql.link(`${NEXT_PUBLIC_FB_URL}/api/graphql`);

// 基本形式
fbSdk.query<QueryType>(QueryDocument, () => {
  return HttpResponse.json({ data: { ... } })
})

// リクエスト情報にアクセスする例
fbSdk.mutation<MutationType>(MutationDocument, ({ variables }) => {
  // variablesからリクエストパラメータを取得
  return HttpResponse.json({ data: { ... } })
})
```

`fbSdk.query`と`fbSdk.mutation`の詳細：

- `fbSdk.query<T>` - GraphQL のクエリ（データ読み取り）操作をモックします
  - 第一引数：GraphQL のクエリドキュメント
  - 第二引数：レスポンスを定義するハンドラー関数
- `fbSdk.mutation<T>` - GraphQL のミューテーション（データ変更）操作をモック
  - クエリと同様の構文だが、通常は`variables`からデータを受け取り処理
  - 例：`({ variables }) => {...}` でリクエストの変数にアクセス

尚、上で使われている`HttpResponse`オブジェクトは、msw のモックが返すレスポンスを定義できるものです：

```tsx
// 正常系レスポンス
HttpResponse.json({ data: {...} })

// エラーレスポンス
HttpResponse.json(
  { data: { addTrialBlockListItem: {
    __typename: "UserError",
    message: "ドメインの形式が正しくありません。",
    code: "BAD_REQUEST",
    fields: [ /* ... */ ],
  }}},
  { status: 400 }
)
```

### 3.2.2 MSW を用いた API モックの流れ

手順としては、ハンドラーを定義する、モック関数にアクセスする、モック関数の返す値を設定する、という流れです。

**A. ハンドラーの定義**

```tsx
// mocks/handlers/user.ts
export const userHandlers = [
  http.get("/api/users", () => {
    return HttpResponse.json([
      { id: 1, name: "ユーザー1" },
      { id: 2, name: "ユーザー2" },
    ]);
  }),
];

// mocks/handlers.ts
export const handlers = [
  ...userHandlers,
  // 他のハンドラー...
];
```

**B. サーバー設定**

```tsx
// mocks/server.ts
import { setupServer } from "msw/node";
import { handlers } from "./handlers";

export const server = setupServer(...handlers);
```

`setupServer`の役割：

- 定義したすべてのハンドラーを一つのモックサーバーインスタンスにまとめます
- リクエストのインターセプト機能を提供し、マッチするハンドラーを自動的に実行
- サーバーライフサイクルメソッドを提供：
  - `server.listen()` - モックサーバーを開始
  - `server.close()` - モックサーバーを停止
  - `server.resetHandlers()` - ハンドラーをリセット（テスト間の独立性確保）

Jest 環境での MSW の設定方法：

```tsx
// jest.setup.ts
import { server } from "@/mocks/server";

// テスト実行前にMSWサーバーを開始
beforeAll(() => server.listen());

// 各テスト後にハンドラーをリセット（テスト間の干渉を防止）
afterEach(() => {
  server.resetHandlers();
  // 他のクリーンアップ処理...
});

// すべてのテスト完了後にサーバーを停止
afterAll(() => server.close());
```

そして、**jest.config.ts**に以下のように設定します。

```tsx
const customJestConfig: Config = {
  setupFilesAfterEnv: ["<rootDir>/jest.setup.ts"],
  // その他の設定...
};
```

`setupFilesAfterEnv`は重要な Jest 設定オプションで、以下の特徴があります：

- このオプションに指定されたファイルは、**テスト環境のセットアップ後、実際のテスト実行前**に読み込まれます
- 指定されたファイル内に記述された`beforeAll`、`afterEach`、`afterAll`などのグローバルフックが、すべてのテストに対して適用されるようになります
- ファイルはテスト実行前に一度だけ評価され、その中で定義されたフックが各テストの適切なタイミングで実行されます

これでテストタイミングでモックサーバーが自動でセットアップされるわけです。

実行環境（Node.js/ブラウザ）によって異なる設定も可能：

```tsx
// mocks/index.ts
export async function initMocks() {
  if (typeof window === "undefined") {
    // Node.js環境ではサーバーモードで実行
    const { server } = await import("./server");
    server.listen();
  } else {
    // ブラウザ環境ではService Workerモードで実行
    const { worker } = await import("./browser");
    await worker.start({ onUnhandledRequest: "bypass" });
  }
}
```

↑ みたいな設定を書いて、

```tsx
// appのエントリポイントで以下のような処理を追加する
useEffect(() => {
  async function setupMocks() {
    const { initMocks } = await import("../mocks");
    await initMocks();
    setShouldRender(true);
  }

  if (API_MOCKING) {
    setupMocks();
  }
}, []);
```

のようにすると設定が自動適用されます。

**C. モック用のヘルパー関数作成**

```tsx
// REST API用
export const mockUsers = (customUsers = []) => {
  server.use(
    http.get("/api/users", () => {
      return HttpResponse.json(customUsers);
    })
  );
};

// GraphQL用
export const mockMainAndSubAccounts = (
  override: Partial<
    Record<ProductAbbreviatedName, MainAndSubAccountsQuery>
  > = {}
) => {
  const emptyData = {
    account: null,
    subAccounts: [],
  };
  server.use(
    fbSdk.query<MainAndSubAccountsQuery>(MainAndSubAccountsDocument, () =>
      HttpResponse.json({ data: override.fb ?? emptyData })
    )
  );
};
```

`server.use()`は登録済みのハンドラーをレスポンスを一時的に変更できます。テストによってハンドラーに入力される値が違うので、データを override しているわけですね。

### 3.2.3 具体的な実装例

以下は、GraphQL で実装されたユーザー情報の CRUD 操作をモックする簡単な例です：

**ハンドラーの定義:**

```tsx
// GraphQLエンドポイントの設定
export const userSdk = graphql.link('<https://api.example.com/graphql>');

// 状態として保持するモックデータ
let users = [
  {
    id: '1',
    name: '山田太郎',
    email: 'taro@example.com',
    role: UserRole.ADMIN,
    createdAt: '2023-01-01T00:00:00Z'
  },
  ...
];

export const userHandlers = [
  // ユーザー一覧取得
  userSdk.query(UsersDocument, () => {
    return HttpResponse.json({
      data: {
        users: users
      }
    });
  }),
...
];

// mocks/handlers/index.ts
import { userHandlers } from './user';

export const handlers = [...userHandlers];
```

**モック用のヘルパー関数:**

```tsx
// ユーザーの一覧取得をオーバーライド
export const mockUsers = (customUsers: User[]) => {
  server.use(
    userSdk.query(UsersDocument, () => {
      return HttpResponse.json({
        data: { users: customUsers },
      });
    })
  );
};
```

実際のテストでモックを使う時はこうします。

```tsx
// __tests__/components/UserList.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import UserList from '@/components/UserList';
import { mockUsers, resetMockUsers } from '@/mocks/server';
import { UserRole } from '@/generated/graphql';

describe('UserList', () => {
  it('特定のユーザーのみを表示できること', async () => {
    // カスタムデータを使ってモックをオーバーライド
    mockUsers([{
      id: '99',
      name: '特別なユーザー',
      email: 'special@example.com',
      role: UserRole.ADMIN,
      createdAt: '2023-12-31T00:00:00Z'
    }]);

    // あとはいつも通り
    const { result } = renderHook(() => useUserList(), { wrapper });

    await waitFor(() => {
      expect(result.current).toBe(...);
    });
  });
});
```

# 4. 外部依存のない関数のテスト設計手順と具体例

まず、外部依存のない純粋関数のテスト方法から説明します。これらの関数は入力に対して予測可能な出力を返し、グローバル状態や外部リソースに依存しません。

## 4.1 テスト設計の基本手順

外部依存がない関数のテスト設計はシンプルです：

1. **テストケースの洗い出し**：関数の条件分岐や境界値を確認
2. **入力データの準備**：各テストケースに対応する入力値を用意
3. **期待値の定義**：各入力に対して期待される出力を定義
4. **テストを実行&アサーションによる検証**：関数を呼び出し、結果と期待値を比較

## 4.2 具体例：機能の利用可能性チェック関数

以下のような外部依存のない関数をテストする例を見てみましょう：

```tsx
// src/utils/subscription.ts
// サブスクリプションタイプの定義
export enum SubscriptionType {
  Free = "free",
  Basic = "basic",
  Standard = "standard",
  Premium = "premium",
}

/**
 * 特定の機能が利用可能かどうかを判定する純粋関数
 * @param subscriptionType ユーザーのサブスクリプションタイプ
 * @returns 機能が利用可能かどうか
 */
export function isFeatureAvailable(
  subscriptionType?: SubscriptionType | null
): boolean {
  if (!subscriptionType) {
    return false;
  }

  // 無料プラン以外で機能使用可能
  return subscriptionType !== SubscriptionType.Free;
}
```

**テストの実装例:**

```tsx
// src/utils/subscription.test.ts
import { isFeatureAvailable, SubscriptionType } from "./subscription";

describe("isFeatureAvailable", () => {
  it("サブスクリプションタイプが未定義の場合、falseを返す", () => {
    // 入力データの準備
    const input = undefined;

    // 関数実行
    const result = isFeatureAvailable(input);

    // 結果の検証
    expect(result).toBe(false);
  });

  it("サブスクリプションタイプがnullの場合、falseを返す", () => {
    const input = null;
    const result = isFeatureAvailable(input);
    expect(result).toBe(false);
  });

  it("無料プランの場合、falseを返す", () => {
    const input = SubscriptionType.Free;
    const result = isFeatureAvailable(input);
    expect(result).toBe(false);
  });

  it("基本プランの場合、trueを返す", () => {
    const input = SubscriptionType.Basic;
    const result = isFeatureAvailable(input);
    expect(result).toBe(true);
  });

  it("標準プランの場合、trueを返す", () => {
    const input = SubscriptionType.Standard;
    const result = isFeatureAvailable(input);
    expect(result).toBe(true);
  });

  it("プレミアムプランの場合、trueを返す", () => {
    const input = SubscriptionType.Premium;
    const result = isFeatureAvailable(input);
    expect(result).toBe(true);
  });
});
```

# 5. 外部依存のあるフックのテスト設計手順と具体例

次に、外部依存のある React フックのテスト方法です。

## 5.1 テスト設計の詳細な手順

外部依存があるフックのテストでの手順です。

1. **テストケースの洗い出し**：関数の条件分岐や境界値を確認
2. **依存関係の特定**：テスト対象にまつわる依存関係を整理
3. **モジュールレベルのモック設定**：依存関係の中でモックした方が良いものをトップレベルでモック
4. **テストケース固有のモック設定**：モック関数に対して値を挿入
5. **テストを実行&アサーションによる検証**：フックを呼び出し、結果と期待値を比較

### 5.1.1 依存関係の特定

まず、テスト対象のフックが依存するすべての外部要素を特定します：

- カスタムフック
- 外部 API やサービス呼び出し
- グローバルステートやコンテキスト

### 5.1.2 モジュールレベルのモック設定

1 で特定した外部要素が存在するモジュールを`jest.mock()`を使ってモック化し、その中でモックする必要のある関数をモックします。この際、モックする範囲は最低限にするべきです。

例えば純粋関数もモックしようと思えばできるんですが、そんなことせずに関数の返り値をモックデータとして作成してやれば良いですよね。モックと実装が密結合してしまうと、後々コードを変更するたびにテストコードをメンテナンスする必要が出てきて、リファクタを楽するためのテストなのに本末転倒ということになります。

よって、カスタムフックなど、モックしないといけないやつだけ対象にしましょう。

```tsx
jest.mock("./hooks/subscription", () => ({
  useUserSubscription: jest.fn(),
}));
```

### 5.1.3 テストケース固有のモック設定

テストケースごとに、必要なモックの戻り値や振る舞いを設定します。

### 5.1.4 **テストを実行&アサーションによる検証**

依存関係のモック設定が完了したら、テスト対象のフックを実行します：

```tsx
// フックを実行して結果を取得
const { result } = renderHook(() => useCanAccessPremiumFeatures(), { wrapper });
```

この時、renderHook を使わないと、カスタムフックを React のライフサイクル外で呼び出すなと怒られます。

最後に、フックの結果が期待通りかどうかを検証します：

```tsx
expect(result.current).toBe(expectedValue);
```

※補足:

API などの非同期処理がテスト対象内に存在する場合、jest で明示的に返り値をモックしている場合は必要ありませんが、msw を使うなどして非同期処理が存在したままテストする場合は、waitFor を使う必要があります。

```tsx
await waitFor(() => {
  expect(result.current).toBe(expectedValue);
});
```

## 5.2 具体例：機能アクセス権限判定フックのテスト

以下のような外部依存のあるカスタムフックをテストする例を見てみましょう：

```tsx
/**
 * ユーザーが機能にアクセスできるかどうかを判定するカスタムフック
 */
export const useFeatureAccessCheck = (): boolean => {
  const userProducts = useUserProductsData();
  const { products } = userProducts;

  // 試用版の製品があるかチェック
  const hasTrialProduct = Object.values(products).some(
    (product) => product.data?.subscription?.type === SubscriptionType.Trial
  );

  // 試用版またはアクセス権限があれば機能使用可能
  return hasTrialProduct || hasAvailablePermissions(userProducts);
};
```

**テストの実装例:**

```tsx
describe("useFeatureAccessCheck", () => {
  it("試用版の製品がある場合、trueを返す", async () => {
    // シンプルなモックセットアップ - 試用版製品を設定
    mockUserProductsData({
      product1: { type: SubscriptionType.Trial },
    });

    // フックを実行して結果を検証
    const { result } = renderHook(() => useFeatureAccessCheck());

    await waitFor(() => {
      expect(result.current).toBe(true);
    });
  });

  it("試用版製品がなく、利用可能な権限もない場合、falseを返す", async () => {
    mockUserProductsData({
      product1: {
        type: SubscriptionType.Standard,
        permissionLimit: 5,
        usedPermissions: 5,
      },
    });

    // フックを実行して結果を検証
    const { result } = renderHook(() => useFeatureAccessCheck());

    await waitFor(() => {
      expect(result.current).toBe(false);
    });
  });
});
```

&nbsp;
&nbsp;
&nbsp;
&nbsp;
以上です。
