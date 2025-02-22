---
title: "TypeScriptにおける型定義の大まかな全体像"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react, typescript]
published: true
---

## 1. 基本的な型定義

### 1-1. `type` / `interface`

```ts
type Props = {
  title: string;
  disabled?: boolean;
};

// or
interface Props {
  title: string;
  disabled?: boolean;
}
```

- **`type`** と **`interface`** はどちらもオブジェクトの構造（Props など）を定義する基本的な文法。
- `interface`は extends などで拡張が容易という特徴があるが、`type`にできることが増えてきた現状で、あえて使う必要性はなさそう（[参考: interface と type の違い、そして何を使うべきかについて](https://zenn.dev/luvmini511/articles/6c6f69481c2d17)）

---

## 2. ジェネリクス（Generics）

```ts
//ジェネリクスを使わない場合
function hoge1(input1: string) {
  console.log(input1);
  return input1;
}

function hoge2(input2: number) {
  console.log(input2);
  return input2;
}

//ジェネリクスを使う場合
function hogeGeneric<T>(inputGeneric: T): T {
  console.log(inputGeneric);
  return inputGeneric;
}
```

- **`<T>` で型パラメータ**を受け取り、呼び出し時に具体的な型を差し込める仕組み。
- **React Hooks** や **ユーティリティ型**（`React.ComponentProps<T>` など）も内側でジェネリクスを使っている。(例：`const [name, setName] = useState<string>('')`)
- 汎用的なコンポーネントやフックを作りたい場合によく使う。

---

## 3. コンポーネントの型定義

### 3-1. `React.FC<Props>`

```tsx
import { FC } from "react";

type Props = {
  title: string;
};

const MyComponent: FC<Props> = ({ title, children }) => {
  return (
    <div>
      <h1>{title}</h1>
      {children}
    </div>
  );
};
```

- **`React.FC`**（Functional Component）は、React コンポーネントであることを明示的に主張するための型定義。<>で指定したものが props の型定義, 戻り値は React.ReactElement または null になる。
- 「React 18 以降はやや非推奨」の流れがあるらしい。明示的に書くのが最近のブームらしく（？）、将来的に非推奨になるかもしれない。

### 3-2. 関数の戻り値に `React.JSX.Element` / `ReactNode` などを指定

```tsx
import { ReactNode } from "react";

type Props = {
  title: string;
  children?: ReactNode;
};

const MyComponent = (props: Props): React.JSX.Element => {
  return (
    <div>
      <h1>{props.title}</h1>
      {props.children}
    </div>
  );
};
```

- 関数の戻り値の型を **`React.JSX.Element`** などで明示する方法。FC を使う場合も、これを用いて個別に指定することもできた。
- 何を返す関数なのかをざっくり示すという意義がある。必要なのかは個人の裁量次第な気がする。
- `ReactNode` は文字列・数値・`ReactElement` など広範な要素を含むユニオン型。
- **`React.ReactElement`** は「実際の React 要素」を指すより厳密な型。場合によってはこちらを使うこともある。

---

## おまけ：　スキーマからの型生成（Zod など）

```ts
import { z } from "zod";

const userSchema = z.object({
  name: z.string(),
  age: z.number(),
});

// z.infer で自動的に型を生成
type User = z.infer<typeof userSchema>;
```

- **Zod** や **io-ts**、**runtypes** などを使って、バリデーションロジックと型定義を一貫させる。
- フロントエンドとバックエンドのスキーマを共有する場合や、型とバリデーションを同一ソースで管理したい場合に便利。

---

## まとめ

- 基本は **`type`** でオブジェクト（Props）を定義し、**ジェネリクス** で柔軟に拡張するので十分
- 関数の戻り値の型を明示したい時は、`(props: Props): React.JSX.Element` など使う（チーム規約で決まっていれば使う程度で良さそう）
- **スキーマ定義**と**型推論**を自動的に同期させたいなら、Zod 等のサードパーティーライブラリを使う

&nbsp;
&nbsp;
&nbsp;
&nbsp;
以上です。
