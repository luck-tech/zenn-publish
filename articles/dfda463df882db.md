---
title: "Jotaiについての備忘録"
emoji: "🙌"
type: "tech"
topics: [jotai, state, react]
published: true
publication_name: "ka_projects"
published_at: 2025-06-04 09:00
---

## Jotai とは

Jotai は "primitive and flexible" な React 状態管理ライブラリです。シンプルで軽量、そして柔軟な設計が特徴です。

## Jotai の基本的な考え方

Jotai では「Atom」と呼ばれる状態の単位を定義し、`useAtom` フックを使ってその状態を読み書きします。

> **なぜ単なる変数ではダメなのか？**  
> 単に `export const counter = 0` のような変数を定義しても、React コンポーネントはその変化を検知できず再レンダーされません。Atom の最大の価値は「リアクティブに値を購読し、更新時に必要なコンポーネントだけを自動で再レンダーしてくれる」点にあります。

## 1. 基本的な API: atom と atomFamily

### atom: 単一の状態を管理する基本単位

```tsx
// 基本的なatomの定義
import { atom } from "jotai";
export const counterAtom = atom(0);

// 使用例
const [count, setCount] = useAtom(counterAtom);
```

`atom`は単一の値を保持するオブジェクトです。グローバルフラグや軽量な状態の管理に適しています。

### atomFamily: 動的に状態を生成するヘルパー

```tsx
// filepath: src/atoms/formFieldSetting.ts
import { atom, atomFamily } from "jotai";

export type FormFieldSetting = {
  id: string;
  options: string[];
};

// idごとに状態を分割して管理できる atomFamily
export const idToFieldSettingAtomFamily = atomFamily((id: string) =>
  atom<FormFieldSetting>({ id, options: [] })
);
```

`atomFamily`は「引数付き atom」を動的に作成するヘルパー関数です:

- 第一引数に「パラメータから atom を初期化する関数」を受け取ります
- 呼び出し時にパラメータを与えると、対応する atom を返します
- 同じパラメータで複数回呼び出しても、同じ atom インスタンスを返します（自動キャッシュ）

例えば:

```tsx
// 別々のフィールド用に独立した状態を生成
const fieldAAtom = idToFieldSettingAtomFamily("fieldA");
const fieldBAtom = idToFieldSettingAtomFamily("fieldB");

// 同じIDを渡せば、どこからでも同じ状態にアクセスできる
const [fieldSetting, setFieldSetting] = useAtom(idToFieldSettingAtomFamily(id));
```

## 2. 状態の読み書き: useAtom と useSetAtom

### useAtom: 読み書き両方のアクセス

```tsx
import { useAtom } from "jotai";

// 値と更新関数の両方を取得
const [fieldSetting, setFieldSetting] = useAtom(idToFieldSettingAtomFamily(id));

// 使用例
console.log(fieldSetting.options); // 値を読み取り
setFieldSetting({
  ...fieldSetting,
  options: [...fieldSetting.options, "新オプション"],
}); // 値を更新
```

`useAtom`フックは:

- `[値, 更新関数]` のタプルを返します
- atom 値が更新されると、このフックを使用しているコンポーネントが自動的に再レンダーされます

### useSetAtom: 書き込み専用のアクセス

```tsx
import { useSetAtom } from "jotai";

// 更新関数だけを取得
const setFieldSetting = useSetAtom(idToFieldSettingAtomFamily(id));
const removeField = useSetAtom(removeFieldAtom);
```

`useSetAtom`フックは:

- 更新関数のみを返します
- 値の読み取りが不要な場合に使用すると、不要な再レンダーを避けられます
- パフォーマンス最適化に有効です

## 3. 再レンダーの仕組み: 具体例

以下のコンポーネント階層を考えてみましょう:

```tsx
// filepath: src/components/FieldForm.tsx
import React from "react";
import { FieldSideMenuSetting } from "./FieldSideMenuSetting";

export const FieldForm: React.FC<{ fieldId: string }> = ({ fieldId }) => {
  return (
    <div className="field-form">
      <h2>Field Form</h2>
      <FieldSideMenuSetting fieldId={fieldId} />
    </div>
  );
};
```

```tsx
// filepath: src/components/FieldSideMenuSetting.tsx
import React from "react";
import { useAtom } from "jotai";
import { idToFieldSettingAtomFamily } from "../atoms/formFieldSetting";
import { SideMenuByType } from "./SideMenuByType";

export const FieldSideMenuSetting: React.FC<{ fieldId: string }> = ({
  fieldId,
}) => {
  const [fieldSetting, setFieldSetting] = useAtom(
    idToFieldSettingAtomFamily(fieldId)
  );

  return (
    <div className="side-menu-container">
      <h3>Field: {fieldSetting.id}</h3>
      <SideMenuByType
        type="radioButton"
        fieldSetting={fieldSetting}
        setFieldSetting={setFieldSetting}
      />
    </div>
  );
};
```

```tsx
// filepath: src/components/SideMenuByType.tsx
import React from "react";
import { FormFieldSetting } from "../atoms/formFieldSetting";
import { RadioButtonSideMenu } from "./RadioButtonSideMenu";

type Props = {
  type: string;
  fieldSetting: FormFieldSetting;
  setFieldSetting: (value: FormFieldSetting) => void;
};

export const SideMenuByType: React.FC<Props> = ({
  type,
  fieldSetting,
  setFieldSetting,
}) => {
  // フィールドタイプに応じたコンポーネントを選択
  if (type === "radioButton") {
    return (
      <RadioButtonSideMenu
        fieldSetting={fieldSetting}
        setFieldSetting={setFieldSetting}
      />
    );
  }

  return <div>未対応のフィールドタイプです</div>;
};
```

```tsx
// filepath: src/components/RadioButtonSideMenu.tsx
import React from "react";
import { FormFieldSetting } from "../atoms/formFieldSetting";
import { OptionsSetting } from "./OptionsSetting";
import { SingleOptionDefaultValueSetting } from "./SingleOptionDefaultValueSetting";

type Props = {
  fieldSetting: FormFieldSetting;
  setFieldSetting: (value: FormFieldSetting) => void;
};

export const RadioButtonSideMenu: React.FC<Props> = ({
  fieldSetting,
  setFieldSetting,
}) => {
  return (
    <div>
      <h4>ラジオボタン設定</h4>
      <OptionsSetting
        options={fieldSetting.options}
        setOptions={(newOptions) => {
          setFieldSetting({ ...fieldSetting, options: newOptions });
        }}
      />
      <SingleOptionDefaultValueSetting options={fieldSetting.options} />
    </div>
  );
};
```

```tsx
// filepath: src/components/OptionsSetting.tsx
import React from "react";

type Props = {
  options: string[];
  setOptions: (options: string[]) => void;
};

export const OptionsSetting: React.FC<Props> = ({ options, setOptions }) => {
  const handleDelete = (index: number) => {
    const newOptions = [...options];
    newOptions.splice(index, 1);
    setOptions(newOptions);
  };

  return (
    <div>
      <h5>オプション一覧</h5>
      <ul>
        {options.map((opt, idx) => (
          <li key={idx}>
            {opt}
            <button onClick={() => handleDelete(idx)}>削除</button>
          </li>
        ))}
      </ul>
    </div>
  );
};
```

```tsx
// filepath: src/components/SingleOptionDefaultValueSetting.tsx
import React from "react";

type Props = {
  options: string[];
};

export const SingleOptionDefaultValueSetting: React.FC<Props> = ({
  options,
}) => {
  return (
    <div>
      <h5>デフォルト値設定</h5>
      <select>
        <option value="">選択してください</option>
        {options.map((opt, idx) => (
          <option key={idx} value={opt}>
            {opt}
          </option>
        ))}
      </select>
    </div>
  );
};
```

### レンダリングの流れ

オプションが削除されたとき、どのように再レンダーが起こるのかを見てみましょう:

1. **イベント発生**: `OptionsSetting`の削除ボタンがクリックされ、`handleDelete`が実行されます

   ```jsx
   // ユーザーがボタンをクリック
   <button onClick={() => handleDelete(idx)}>削除</button>
   ```

2. **ローカル更新**: `OptionsSetting`内で`setOptions([...])`が呼ばれます

   ```jsx
   // 新しい配列を作成し、元のpropsから受け取ったsetOptionsを呼ぶ
   const newOptions = [...options];
   newOptions.splice(index, 1);
   setOptions(newOptions);
   ```

3. **親コンポーネント処理**: `RadioButtonSideMenu`が`setFieldSetting`を呼び出し、atom 全体を更新します

   ```jsx
   // RadioButtonSideMenuからOptionsSettingに渡されたコールバック
   setOptions={(newOptions) => {
     setFieldSetting({ ...fieldSetting, options: newOptions });
   }}
   ```

4. **Atom 更新とリアクティブ通知**: `FieldSideMenuSetting`で`useAtom(idToFieldSettingAtomFamily)`を呼んでいるため、このコンポーネントが再レンダーされます

5. **階層的な再レンダー**: 子コンポーネントのツリーが再レンダーされます

   ```
   FieldSideMenuSetting → SideMenuByType → RadioButtonSideMenu →
   [OptionsSetting, SingleOptionDefaultValueSetting]
   ```

6. **最終結果**: 新しい`fieldSetting.options`が子コンポーネントに渡り、UI が更新されます

## 4. Jotai を使う主なメリット

- **分割された状態管理**: `atomFamily`を使うことで、フィールドごとに状態を完全に分離できます

  - 他のフィールドに影響を与えずに局所的な更新が可能
  - コンポーネント間で状態を簡単に共有できる

- **自動的な再レンダー最適化**: `useAtom`を呼ぶコンポーネントだけが更新される

  - 状態変更に関連するコンポーネントだけが再レンダーされる
  - 明示的なメモ化（`React.memo`など）を多用せずとも効率的

- **シンプルな API**: 学習コストが低く、ボイラープレートが少ない
  - Redux のような複雑なセットアップが不要
  - アクション、リデューサー、セレクターなどの概念を覚える必要がない

## 5. 注意点

- **同期的な更新**: atom の更新は同期的にリスナー（useAtom 呼び出し箇所）へ通知されます

  - 非同期操作を扱うには別途パターンが必要

- **状態分割の重要性**: 大きな状態オブジェクトを一つの atom で管理すると非効率です

  - 更新のたびに関連する全コンポーネントが再描画される
  - 状態を論理的に分割することでパフォーマンスが向上する
