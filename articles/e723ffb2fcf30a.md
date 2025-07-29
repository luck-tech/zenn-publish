---
title: "今更Makefileについて知る"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [make]
published: true
publication_name: "ka_projects"
---

## はじめに

大規模な Web アプリケーションプロジェクトでは、開発環境のセットアップや日常的なタスクの実行が複雑になりがちです。Makefile を活用すると、これらの作業を自動化することができます。

Makefile は、もともと C 言語などのコンパイル処理を自動化するために開発されたツールですが、現在では様々な種類のタスク自動化に使用されています。シンプルな構文で複雑なワークフローを定義でき、依存関係の管理も可能です。

## 基本構文

```makefile
# 変数定義
VARIABLE_NAME=value

# ターゲット定義
target-name: dependency1 dependency2
	command1
	command2
	@echo "Silent command (@ suppresses output)"
```

## 実践的な活用例

### 1. 開発環境要件チェック

開発に必要なツールが正しくインストールされているかを自動チェック：

```makefile
check-requirements:
	type docker
	type node
	type npm
	node -v | grep 'v18'
	@echo "=====All requirements satisfied====="
```

### 2. 環境初期化の自動化

複数のステップからなる初期セットアップを一括実行：

```makefile
initial-setup:
	@echo "=====Setting up configuration files====="
	cp config/app.yml.template config/app.yml
	@echo "=====Installing dependencies====="
	npm install
	@echo "=====Building containers====="
	docker compose build
	@echo "=====Running database migrations====="
	docker compose run --rm app npm run migrate
	@echo "=====Setup complete====="
```

### 3. コード品質管理

静的解析やフォーマットチェックを統合：

```makefile
lint:
	make lint-frontend
	make lint-backend

lint-frontend:
	npm run eslint src/
	npm run prettier --check src/

lint-backend:
	./vendor/bin/phpstan analyze --memory-limit=1G
	./vendor/bin/php-cs-fixer fix --dry-run
```

### 4. ファイル生成と整形

JSON ファイルの整形など、定型作業の自動化：

```makefile
format-json:
	@cat data/config.json | jq . > data/config.tmp.json && \
	mv data/config.tmp.json data/config.json
	@echo "JSON files formatted"
```

### 5. バージョン管理

プロジェクト全体のバージョン番号を一括更新：

```makefile
set-version:
	sed "s/version: .*/version: $(VER)/" config/app.yml > tmp && mv -f tmp config/app.yml
	sed "s/\"version\": \".*\"/\"version\": \"$(VER)\"/" package.json > tmp && mv -f tmp package.json
```

実行例：

```bash
make set-version VER=2.1.0
```

## 高度なテクニック

### 変数の活用

```makefile
# デフォルト値の設定
ENVIRONMENT ?= development
PORT ?= 3000

# 環境別設定
start-server:
	NODE_ENV=$(ENVIRONMENT) PORT=$(PORT) npm start
```

### 条件分岐

```makefile
deploy:
ifeq ($(ENVIRONMENT),production)
	@echo "Deploying to production..."
	npm run build:prod
else
	@echo "Deploying to staging..."
	npm run build:staging
endif
```

### .PHONY ターゲット

**[must]** ファイル名と同じ名前のターゲットは`.PHONY`で宣言する必要があります：

```makefile
.PHONY: clean test deploy

clean:
	rm -rf dist/
	rm -rf node_modules/

test:
	npm test

deploy:
	npm run deploy
```

.PHONY: clean と宣言することによって、Make は「clean はファイルではなくタスク名だ」と理解してくれます。

### エラーハンドリング

```makefile
test-with-cleanup:
	docker compose up -d database
	npm test || (docker compose down && exit 1)
	docker compose down
```

### 並列実行の活用

```bash
# 複数のlintタスクを並列実行
make -j4 lint-frontend lint-backend lint-docker lint-docs
```
