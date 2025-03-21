---
title: "markdown記法メモ"
emoji: "📚"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: [zenn]
published: false
---

- あいうえお
- かきくけこ

* リスト 1
  - リスト 1-2
* リスト 2

1. リスト 1
   1. リスト 1-1
   2. リスト 1-2
2. リスト 2

- [ ] リスト 1
- [x] リスト 2

---

:::details タイトル
表示したい内容
:::

# This is an H1

> 引用本文引用本文
>
> > 入れ子

```math
\left( \sum_{k=1}^n a_k b_k \right)^{\!\!2} \leq
\left( \sum_{k=1}^n a_k^2 \right) \left( \sum_{k=1}^n b_k^2 \right)
```

[Qiita](http://qiita.com "Qiita")

~~打ち消し~~

https://qiita.com/Qiita/items/c686397e4a0f4f11683d

![discordに貼った時の見た目](https://storage.googleapis.com/zenn-user-upload/2632943c502e-20241012.png)

あいえうお[^1]
[^1]: あいうえお

| Left align | Right align | Center align |
| :--------- | ----------: | :----------: |
| This       |        This |     This     |
| column     |      column |    column    |
| will       |        will |     will     |
| be         |          be |      be      |
| left       |       right |    center    |
| aligned    |     aligned |   aligned    |

```html:sample
   <div class="radioWave">
      <p>迷いの中あてなく見上げた空彩る星たちが</p>
      <p>嘘みたいに晴れた朝に繋がることを教えてくれた</p>
   </div>
```

```diff js:sample
@@ -4,6 +4,5 @@
+	const foo = bar.baz([1, 2, 3]) + 1;
-	let foo = bar.baz([1, 2, 3]);
```

&nbsp;

```mermaid
graph TD;
    A-->B;
    A-->C;
    B-->D;
    C-->D;
```
