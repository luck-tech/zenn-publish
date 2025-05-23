---
title: "npm, Yarn, pnpm, Bunの違いについて知る"
emoji: "📦"
type: "tech"
topics: ["nodejs", "npm", "yarn", "pnpm", "bun"]
published: true
publication_name: "ka_projects"
---

## 1. はじめに

フロントエンド開発において、パッケージマネージャの選択は開発効率や環境安定性に大きく影響します。特に大規模プロジェクトや CI 環境では、インストール速度やディスク効率の最適化が重要な課題です。

現在、Node.js エコシステムには主に 4 つのパッケージマネージャがあります：

- **npm**: Node.js 標準の老舗パッケージマネージャ
- **Yarn**: Facebook が開発した代替ツール
- **pnpm**: 効率性を重視した比較的新しいマネージャ
- **Bun**: 2023 年登場の超高速新興ツール

この記事では、これら 4 つのパッケージマネージャを比較します。

## 2. 各パッケージマネージャの特徴

### 2.1 npm (Node Package Manager)

**概要:**
Node.js 公式のパッケージマネージャで、Node.js をインストールすれば自動的に使えます。世界中で最も広く使われており、莫大な数のパッケージを管理しています。

**特徴:**

- Node.js 標準のため追加インストール不要
- 圧倒的な普及度と互換性
- package-lock.json でバージョン固定
- `npm ci`コマンドで CI 環境での高速インストール

**メリット:**

- 導入コストゼロ（Node.js に同梱）
- あらゆる Node.js ツールとの互換性が最も高い
- 学習コストが最も低い

**デメリット:**

- 他ツールと比べて処理速度が遅い傾向
- プロジェクトごとにパッケージが重複して保存され、ディスク効率が悪い
- 大規模プロジェクトでは管理が煩雑になりがち

### 2.2 Yarn

**概要:**
Facebook（現 Meta）によって開発された代替パッケージマネージャ。npm の欠点を克服するために誕生し、後にバージョン系統が分かれました。特に Yarn Berry（2 系以降）では革新的な機能が多数追加されています。

**特徴:**

- Yarn Classic（1 系）: npm とほぼ同じディレクトリ構造
- Yarn Berry（2 系以降）: Plug'n'Play（PnP）方式を採用
- 並列ダウンロードとオフラインキャッシュ
- `yarn.lock`ファイルで依存関係を厳格管理

**重要な革新機能（Yarn Berry）:**

- **Plug'n'Play (PnP)**: node_modules フォルダを作らず依存関係を解決する仕組み

  - インストール時間を大幅短縮
  - ディスク使用量を劇的に削減
  - ファイルシステム操作が減少しパフォーマンス向上

- **Zero-Installs**: 依存関係を `.yarn/cache` フォルダに保存し Git などにコミット可能

  - 新規クローン後の `yarn install` が不要になる
  - CI 環境でのビルド時間短縮
  - オフライン環境でも即時開発可能

- **Virtual Packages**: 同じパッケージの異なるバージョンを同時使用可能
  - 依存関係の衝突問題を解決
  - より柔軟なバージョン管理が実現

**メリット:**

- npm より高速な並列インストール
- PnP モードによる node_modules レス開発で効率化
- プラグインシステムによる高い拡張性
- モノレポ向けワークスペース機能
- Zero-Installs によるチーム間の環境一貫性確保

**デメリット:**

- PnP 採用時は一部ツールとの互換性に課題があり設定が必要
- Yarn 1 系と 2 系以降の情報が混在して分かりにくい
- 新機能を活用するための学習コストがやや高い

### 2.3 pnpm (Performant npm)

**概要:**
「重複排除」と「高速化」を重視した npm 互換のパッケージマネージャ。特に大規模モノレポでの採用が増加しています。

**特徴:**

- グローバルストアでパッケージを一元管理
- シンボリックリンクによる効率的なパッケージ共有
- npm と互換性の高いコマンド体系
- `pnpm-lock.yaml`による依存関係管理

**メリット:**

- 圧倒的なディスク効率（同一パッケージを 1 回だけ保存）
- 2 回目以降のインストールが特に高速
- npm/Yarn からの移行が容易
- モノレポ機能が標準サポート

**デメリット:**

- 新興ツールゆえに大規模実績はまだ限定的
- 特殊なリンク構造への理解が必要

### 2.4 Bun

**概要:**
2023 年に 1.0 がリリースされた最も新しいツール。パッケージ管理機能だけでなく、JavaScript ランタイム、ビルダー、テストランナーを一体化しています。

**特徴:**

- マルチスレッドによる超高速実行
- パッケージ管理に留まらない統合開発環境
- `bun.lockb`によるロックファイル管理
- JavaScriptCore（Safari）エンジンベース

**メリット:**

- 圧倒的なインストール/実行速度（他ツールの数倍〜10 倍）
- 開発ツールチェーンの統合による効率化
- 軽量なメモリ使用

**デメリット:**

- まだ発展途上で監査機能など未実装の部分あり
- Node.js API との互換性が不完全
- 急速な変化で安定性に不安がある

## 3. 比較ポイント

### 3.1 インストール速度

実際のプロジェクトサイズやネットワーク環境によって異なりますが、一般的な速度傾向は次の通りです：

```
Bun > pnpm > Yarn > npm
```

※ Yarn に関しては、v2 以降は PnP により、node_modules がインストールされなくなり、Zero-Installs にて yarn install 自体必要なくなったので、キャッシュ状況によりけり

特に注目すべき点：

- **npm**: シングルスレッド処理で相対的に遅い
- **Yarn**: 並列ダウンロードと PnP, Zero-Installs キャッシュで高速化
- **pnpm**: グローバルストア活用で 2 回目以降が非常に速い
- **Bun**: マルチスレッド処理で圧倒的な速さを実現

### 3.2 ディスク効率

```
pnpm > Yarn(PnP) > Bun > npm/Yarn(Classic)
```

- **npm/Yarn(Classic)**: プロジェクトごとに重複保存で無駄が多い
- **Yarn(PnP)**: node_modules 不要でファイル数を大幅削減
- **pnpm**: システム全体で 1 コピーのみ保存し最も効率的
- **Bun**: 同一バージョンの重複を避けるが、pnpm ほど徹底していない

### 3.3 互換性・安定性

```
npm > pnpm > Yarn > Bun
```

- **npm**: Node.js 標準で不安要素なし
- **pnpm**: npm ライクな構造で高い互換性
- **Yarn**: 特に PnP 採用時は追加設定が必要なケースあり
- **Bun**: まだ発展途上で互換性に課題

### 3.4 セキュリティ機能

- **npm/Yarn/pnpm**: `audit`コマンドで脆弱性チェック可能
- **Bun**: 現時点でセキュリティ監査機能は未実装

### 3.5 学習・導入コスト

- **npm**: Node.js 同梱で即使用可能、学習コスト最小
- **Yarn Classic**: npm 類似でコマンドが少し異なる程度
- **pnpm**: npm 類似のコマンドで移行は容易
- **Yarn PnP/Bun**: 新しい概念理解や設定が必要

## 4. ユースケース別推奨ツール

### 4.1 小〜中規模プロジェクト

**おすすめ: npm または pnpm**

小規模プロジェクトでは、npm の使い慣れた環境を維持するメリットが大きいです。特に問題を感じなければそのまま使い続けるのもアリ。ただし、より効率的なパッケージ管理を望むなら pnpm への移行も検討価値があります。

### 4.2 大規模モノレポ

**おすすめ: pnpm または Yarn(PnP)**

大規模プロジェクト特有の課題：

- ビルド/インストール時間の長さ
- 膨大なディスク使用量
- 依存関係の複雑化

これらの課題に対して：

- **pnpm**: ディスク効率と速度を両立し、モノレポサポート
- **Yarn(PnP)**: node_modules レス環境で効率化とプラグイン拡張性

### 4.3 新規プロジェクト

**おすすめ: pnpm**

新規プロジェクトでは、将来的な拡張性を考慮すると pnpm が最もバランスが良いでしょう。npm 互換性を維持しつつ、高速・省スペースというメリットを享受できます。

### 4.4 将来を見据えた実験的採用

Bun はまだ安定性に課題があるものの、その圧倒的な速度は魅力的です。サイドプロジェクトや非重要な開発環境で試してみる価値はあります。将来的には主力ツールになる可能性も十分あります。

## まとめ

とりあえず**pnpm**選んどいて間違いないかなと思います。
&nbsp;
&nbsp;
&nbsp;
&nbsp;
以上です。
