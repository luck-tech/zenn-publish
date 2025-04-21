---
title: "Jestを使ったテスト設計の基本と実践"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
publication_name: "ka_projects"
published_at: 2025-05-28 09:00
---

## Jest とは

Jest は、Facebook 社が開発した JavaScript 用のテスティングフレームワークです。シンプルな設定で利用できる一方、モックやタイマー制御などの高度な機能も備えています。

Jest の主な特徴は以下の通りです：

- ゼロコンフィグ：設定なしですぐに使用開始できる
- スナップショットテスティング：UI コンポーネントの変更を検出
- モック機能：外部依存の制御と分離
- 並行テスト実行：高速な実行環境
- コードカバレッジレポート：テストの網羅性確認

## テスト設計の考え方

### テストケースの洗い出し方

効果的なテストを書くためには、適切なテストケースを洗い出すことが重要です。テスト点を決める際は、主に以下のポイントに着目します：

### 1. 条件分岐の結果が変わる箇所

コード内で結果が変わる分岐点はすべてテスト対象になります：

- `if (isUserPremium) { return true; }` のような明示的な分岐
- 三項演算子 `condition ? valueA : valueB` による暗黙的な分岐
- 論理演算子 `value || defaultValue` による短絡評価

これらの分岐ごとに、異なるテストケースを作成することでコードパスの網羅率を高められます。

### 2. 境界値とエッジケース

バグは境界条件で発生しやすいため、以下のような境界値に注目します：

- 数値が 0 か 1 以上か（または負の値への対応）
- 配列が空か要素があるか
- オブジェクトが`null`/`undefined`か有効な値か
- 日付が特定のタイムゾーンをまたぐケース

### 外部依存の有無によるテスト設計の違い

テスト設計は対象関数の特性によって大きく異なります：

| 特性     | 外部依存なし（純粋関数） | 外部依存あり（フック、API 通信など） |
| -------- | ------------------------ | ------------------------------------ |
| 複雑さ   | シンプル                 | 複雑                                 |
| 設計方法 | 入力と出力の検証のみ     | モックのセットアップが必要           |

Jest でフックや外部依存のある関数をテストする場合、外部で定義された関数やフックの挙動を制御するためにモックが必要です。テスト環境では実際の API や他のモジュールの実装は利用できないため、これらをシミュレートする必要があります。

### テスト設計に影響するその他の要因

- **非同期処理の有無**：`async/await`や`Promise`を扱うテストは特殊なアサーション方法が必要
- **状態管理の複雑さ**：Redux のような状態管理が関わるとテストもより複雑になる
- **UI コンポーネント vs ビジネスロジック**：UI のテストは見た目の検証も必要になるため複雑
- **サイドエフェクトの有無**：ローカルストレージへの保存など、副作用があるとモックが必要

それぞれのやり方は別途調べてください。

## Jest 固有の機能と文法

### モックの種類と使い分け

### jest.fn()

`jest.fn()`は個別の関数をモック化するための機能です：

```jsx
const mockFunction = jest.fn();
mockFunction.mockReturnValue("テスト結果");
```

この関数は呼び出し回数や引数を追跡でき、テスト内で呼び出されたかどうかを検証できます。

### jest.mock()

`jest.mock()`はモジュール全体をモック化します：

```jsx
jest.mock("./utils/api", () => ({
  fetchData: jest.fn().mockResolvedValue({ data: "モックデータ" }),
}));
```

モジュールのインポート時点で置き換えが行われるため、テスト対象のコードが依存するすべての外部モジュールをコントロールできます。

### require()とモックの組み合わせ

`jest.mock()`と`require()`を組み合わせると、テストケース別にモックの挙動を変更できます：

```jsx
jest.mock("./hooks/useData", () => ({
  useData: jest.fn(),
}));

// テストケース内で動的に変更
it("データがロード中の場合", () => {
  require("./hooks/useData").useData.mockReturnValue({
    loading: true,
    data: null,
  });
  // テスト内容...
});
```

### jest.resetAllMocks()

テスト間でモック状態をリセットするために使用します：

```jsx
beforeEach(() => {
  jest.resetAllMocks();
});
```

これによりテスト間の依存関係を排除し、各テストを独立して実行できます。

### jest.requireActual()

部分的なモックを行う場合に、元のモジュールの実装を取得するために使用します：

```jsx
jest.mock("./utils/helpers", () => {
  const originalModule = jest.requireActual("./utils/helpers");
  return {
    ...originalModule, // 元の実装を保持
    getCurrentDate: jest.fn().mockReturnValue("2025-04-21"), // この関数だけ置き換え
  };
});
```

これは以下のような場合に便利です：

- ユーティリティ関数や定数など、変更する必要がない部分は元のまま使用
- テスト対象と直接関係ない補助的な機能は実装を保持
- 一部の挙動だけを制御したい場合

### mockReturnValue, mockImplementation

両方ともモック関数の振る舞いを設定するメソッドですが、使用目的が異なります：

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

使い分けの基準：

- シンプルに固定値を返すだけなら`mockReturnValue`
- 条件分岐や引数を使った計算が必要なら`mockImplementation`

また、非同期処理を含む機能をテストするときは`mockResolvedValue` / `mockRejectedValue` を使います。ほぼ変わらないのでここでは省略します。

## 外部依存のない関数のテスト設計手順と具体例

まず、外部依存のない純粋関数のテスト方法から説明します。これらの関数は入力に対して予測可能な出力を返し、グローバル状態や外部リソースに依存しません。

### テスト設計の基本手順

外部依存がない関数のテスト設計は比較的シンプルです：

1. **テストケースの洗い出し**：関数の条件分岐や境界値を確認
2. **入力データの準備**：各テストケースに対応する入力値を用意
3. **期待値の定義**：各入力に対して期待される出力を定義
4. **テストの実装**：関数を呼び出し、結果と期待値を比較

### 具体例：機能の利用可能性チェック関数

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

### テストの実装例

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

## 外部依存のあるフックのテスト設計手順と具体例

次に、より複雑な外部依存のある React フックのテスト方法を詳しく説明します。

### テスト設計の詳細な手順

外部依存があるフックのテストでは、テストケースを見定めた後に、さらに以下の 5 つのステップに従います：

### 1. 依存関係の特定とモック関数の作成

まず、テスト対象のフックが依存するすべての外部要素を特定します：

- 他の React フック（`useState`、`useEffect`など以外の独自フック）
- 外部 API やサービス呼び出し
- ユーティリティ関数
- グローバルステートやコンテキスト

特定したら、これらの依存関係ごとにモック関数を作成します：

```tsx
// 各依存関係に対応するモック関数を作成
const mockIsUserActive = jest.fn();
const mockCountAvailableFeatures = jest.fn();
const mockUseUserSubscription = jest.fn();
```

このステップで、テスト対象が必要とするすべての外部依存をコントロールするための準備が整います。

### 2. モジュールレベルのモック設定

次に、`jest.mock()`を使って依存しているモジュールをモック化します。この際、以下の点に注意します：

- モジュールのパスを正確に指定
- 返す関数やオブジェクトの構造を元のモジュールと一致させる
- TypeScript を使用している場合は型も考慮

```tsx
// 依存するモジュールをモック化
jest.mock("./utils/user", () => ({
  isUserActive: (user) => mockIsUserActive(user),
}));

jest.mock("./utils/features", () => {
  const originalModule = jest.requireActual("./utils/features");
  return {
    ...originalModule, // 元の実装を保持（テスト対象外の関数用）
    countAvailableFeatures: (subscription, userData) =>
      mockCountAvailableFeatures(subscription, userData),
  };
});

// フックのモック
jest.mock("./hooks/subscription", () => ({
  useUserSubscription: jest.fn(),
}));
```

このステップでは、フックが利用する外部モジュールの振る舞いを制御できるようにします。

### 3. テストケース固有のモック設定

テストケースごとに、必要なモックの戻り値や振る舞いを設定します。具体的には：

- `require`で既にモック化したモジュールにアクセス
- `mockReturnValue`や`mockImplementation`でテストケース固有の値を設定
- 必要に応じてモックデータオブジェクトを作成

```tsx
// テストケース内でモック設定
it("プレミアムユーザーの場合のテスト", () => {
  // モックデータを準備
  const mockData = createTestData({ isPremium: true });

  // テストケース固有のモック設定
  require("./hooks/subscription").useUserSubscription.mockReturnValue(mockData);
  mockIsUserActive.mockReturnValue(true);
});
```

何回もモックデータ作るのが面倒な場合は、ヘルパー関数を作成するのもあり：

```tsx
// モックデータ作成ヘルパー関数
const createTestData = ({ isPremium = false }) => ({
  subscription: {
    type: isPremium ? SubscriptionType.Premium : SubscriptionType.Free,
    expirationDate: "2025-12-31",
    features: isPremium ? ["feature1", "feature2", "feature3"] : ["feature1"],
  },
  // ...他のデータ
});
```

### 4. フックの実行と結果の取得

依存関係のモック設定が完了したら、テスト対象のフックを実行します：

```tsx
// フックを実行して結果を取得
const result = useCanAccessPremiumFeatures();
```

### 5. アサーションによる検証

最後に、フックの結果が期待通りかどうかを検証します：

```tsx
// 結果を検証
expect(result).toBe(true);

// 必要に応じてモック関数の呼び出し状況も検証
expect(mockIsUserActive).toHaveBeenCalledTimes(1);
expect(mockIsUserActive).toHaveBeenCalledWith(
  expect.objectContaining({
    type: SubscriptionType.Premium,
  })
);
```

### 具体例：機能アクセス権限判定フックのテスト

実際に`useCanAccessPremiumFeatures`フックのテスト実装例を見てみましょう：

```tsx
// src/hooks/useFeatures.ts
import { useUserSubscription } from "./subscription";
import { isUserActive } from "../utils/user";
import { countAvailableFeatures } from "../utils/features";
import { SubscriptionType } from "../utils/subscription";

/**
 * プレミアム機能へのアクセス権を判定するフック
 * @returns プレミアム機能へアクセスできるかどうか
 */
export const useCanAccessPremiumFeatures = (): boolean => {
  // 別のフックに依存
  const userSubscriptionData = useUserSubscription();
  const { subscription, userData } = userSubscriptionData;

  // プレミアムサブスクリプションかチェック
  const isPremium = subscription?.type === SubscriptionType.Premium;

  if (isPremium) {
    return true; // プレミアムなら即true
  }

  // 利用可能な機能数をチェック
  const featureCount = isUserActive(userData)
    ? countAvailableFeatures(subscription, userData)
    : 0;

  // 特定の条件を満たせばアクセス可能
  return featureCount > 5;
};
```

### 完全なテスト実装例

```tsx
// src/hooks/useFeatures.test.ts
import { useCanAccessPremiumFeatures } from "./useFeatures";
import { SubscriptionType } from "../utils/subscription";

describe("useCanAccessPremiumFeatures", () => {
  // ステップ1: テスト環境のリセットと準備
  beforeEach(() => {
    jest.resetAllMocks();
  });

  // モックデータ作成ヘルパー関数
  const createMockUserSubscriptionData = ({ isPremium = false }) => ({
    subscription: {
      type: isPremium ? SubscriptionType.Premium : SubscriptionType.Standard,
      expirationDate: "2025-12-31",
      features: isPremium
        ? [
            "feature1",
            "feature2",
            "feature3",
            "feature4",
            "feature5",
            "feature6",
          ]
        : ["feature1", "feature2"],
    },
    userData: {
      id: "user123",
      name: "テストユーザー",
      isActive: true,
    },
  });

  // ステップ2: モック関数の作成
  const mockIsUserActive = jest.fn().mockImplementation(() => true);
  const mockCountAvailableFeatures = jest.fn();

  // ステップ3: モジュールのモック化
  jest.mock("../utils/user", () => ({
    isUserActive: (userData) => mockIsUserActive(userData),
  }));

  jest.mock("../utils/features", () => {
    const originalModule = jest.requireActual("../utils/features");
    return {
      ...originalModule,
      countAvailableFeatures: (subscription, userData) =>
        mockCountAvailableFeatures(subscription, userData),
    };
  });

  jest.mock("./subscription", () => ({
    useUserSubscription: jest.fn(),
  }));

  // ステップ4: テストケースの実装
  it("プレミアムサブスクリプションの場合、trueを返す", () => {
    // テストケース固有のモック設定
    const mockData = createMockUserSubscriptionData({ isPremium: true });
    require("./subscription").useUserSubscription.mockReturnValue(mockData);

    // フックを実行して結果を取得
    const result = useCanAccessPremiumFeatures();

    // 結果を検証
    expect(result).toBe(true);
  });

  it("プレミアムでなく、利用可能な機能が5以下の場合、falseを返す", () => {
    // テストケース固有のモック設定
    const mockData = createMockUserSubscriptionData({ isPremium: false });
    require("./subscription").useUserSubscription.mockReturnValue(mockData);

    // 機能数の設定
    mockCountAvailableFeatures.mockReturnValue(3);

    // フックを実行して結果を取得
    const result = useCanAccessPremiumFeatures();

    // 結果を検証
    expect(result).toBe(false);
  });

  it("プレミアムでないが、利用可能な機能が5を超える場合、trueを返す", () => {
    // テストケース固有のモック設定
    const mockData = createMockUserSubscriptionData({ isPremium: false });
    require("./subscription").useUserSubscription.mockReturnValue(mockData);

    // 機能数の設定
    mockCountAvailableFeatures.mockReturnValue(6);

    // フックを実行して結果を取得
    const result = useCanAccessPremiumFeatures();

    // 結果を検証
    expect(result).toBe(true);
  });

  it("ユーザーがアクティブでない場合、falseを返す", () => {
    // テストケース固有のモック設定
    const mockData = createMockUserSubscriptionData({ isPremium: false });
    require("./subscription").useUserSubscription.mockReturnValue(mockData);

    // ユーザーがアクティブでない設定
    mockIsUserActive.mockReturnValue(false);

    // フックを実行して結果を取得
    const result = useCanAccessPremiumFeatures();

    // 結果を検証
    expect(result).toBe(false);

    // 特定の関数が呼ばれていないことを検証
    expect(mockCountAvailableFeatures).not.toHaveBeenCalled();
  });
});
```

### テストの各ステップの意図と役割

1. **モック関数の作成**：フックが依存する外部関数（`isUserActive`など）をコントロールするためのモック
2. **モジュールのモック化**：インポートされるモジュールの振る舞いを置き換え
3. **モックデータ作成関数**：複雑なデータ構造を簡単に生成するためのヘルパー
4. **テストケース固有設定**：各テストケースで必要な条件を設定するためのモック値
5. **フック実行と検証**：テスト対象のフックを実行し、結果を期待値と比較

## まとめ

効果的な Jest テストを設計するためのポイントは以下の通りです：

1. **テストの種類**
   - 純粋関数：入力と出力の検証を中心に、シンプルなテスト
   - 外部依存あり：モックを使った制御された環境でのテスト
2. **テストの種類に応じた設計手順を理解する**
3. **モック機能を適切に使い分ける**
   - `jest.fn()`：個別関数のモック
   - `jest.mock()`：モジュール全体のモック
   - `jest.requireActual()`：部分的なモック
   - `require()`：モックインスタンスへのアクセス
