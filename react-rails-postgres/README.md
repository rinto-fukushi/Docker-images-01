# 環境構築手順

## 1 プロジェクトディレクトリの作成

ターミナルで作業用ディレクトリを作成し、移動します。
```bash
mkdir react-fastapi-postgres
cd react-fastapi-postgres
```

## 2 ディレクトリ構造の設定

以下のディレクトリを作成します。

    frontend/: フロントエンド（React）用
    backend/: バックエンド（FastAPI）用
    db/: データベース用（設定ファイルは特に不要）

```bash
mkdir frontend backend db
```

## 3 Frontend: React の設定

### 3-1: Reactアプリの作成
frontend ディレクトリに移動し、Reactアプリを作成します。

```bash
cd frontend
npx create-react-app .
```

npxとは？ Node.jsのパッケージを実行するためのコマンドです。create-react-app はReactのプロジェクトを素早く作成するためのツールです。

### 3-2: Dockerfileの作成
frontend ディレクトリ内に Dockerfile を作成します。

```Dockerfile
# ベースイメージとしてNode.jsを使用
FROM node:14

# アプリケーションディレクトリを作成
WORKDIR /app

# 依存関係をインストール
COPY package*.json ./
RUN npm install

# アプリケーションコードをコピー
COPY . .

# アプリケーションをビルド
RUN npm run build

# Nginxを使用してビルドしたファイルを提供
FROM nginx:alpine
COPY --from=0 /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

## 4 Backend: Ruby on Rails の設定

### 4-1: Railsアプリの作成
backend ディレクトリに移動し、必要なファイルを作成します。

### 4-2: Gemfileの作成
Gemfile を作成し、以下を記述します。

```Gemfile
source 'https://rubygems.org'

gem 'rails', '~> 6.1.4'
```

### 4-3: Gemfile.lockの作成
Gemfile.lock を空の状態で作成します。


### 4-4: Dockerfileの作成
Dockerfile を作成します。

```Dockerfile
FROM ruby:3.1

# 必要なパッケージをインストール
RUN apt-get update -qq && apt-get install -y build-essential libpq-dev nodejs yarn

# RubyGems を最新バージョンに更新
RUN gem update --system

WORKDIR /app

# Gemfile と Gemfile.lock をコピー
COPY Gemfile Gemfile.lock ./

# bundle install を実行
RUN bundle install

# アプリケーションコードをコピー
COPY . .

EXPOSE 3000

CMD ["rails", "server", "-b", "0.0.0.0"]
```

### 4-5: Railsプロジェクトの初期化
Dockerコンテナ内でRailsプロジェクトを作成します。

```bash
docker run --rm -v "$PWD":/app -w /app ruby:2.7 bash -c "bundle install && rails new . --api --database=postgresql --skip-bundle"
```

> 説明：
> - ```--rm```：コンテナ実行後に自動的に削除します。
> - ```-v "$PWD":/app```：現在のディレクトリをコンテナ内の /app にマウントします。
> - ```-w /app```：コンテナ内での作業ディレクトリを /app に設定します。
> - ```rails new . --api --database=postgresql --skip-bundle```：現在のディレクトリに新しい Rails アプリケーションを作成します。

### 4-6: Gemfileの確認と編集
Gemfile が Rails によって更新されたので、内容を確認します。
> 確認ポイント：
> - ```gem 'pg'``` が含まれていることを確認してください。
> - ```gem 'bootsnap'``` が含まれていることを確認してください。

### 4-7: データベース設定の変更
config/database.yml を編集し、以下のように変更します。

```yml
default: &default
  adapter: postgresql
  encoding: unicode
  host: db       # コンテナ名 'db' を指定
  username: user
  password: password
  pool: 5

development:
  <<: *default
  database: mydatabase

# 以下、test 環境や production 環境の設定があれば追記
```


## 5 Docker Composeファイルの作成
プロジェクトのルートディレクトリに戻ります。
```bash
cd ..
```
```docker-compose.yml``` を作成します。

```yml
services:
  frontend:
    build: ./frontend
    ports:
      - "80:80"
    depends_on:
      - backend

  backend:
    build: ./backend
    ports:
      - "3000:3000"
    depends_on:
      - db
    environment:
      DATABASE_HOST: db
      DATABASE_USER: user
      DATABASE_PASSWORD: password
      DATABASE_NAME: mydatabase

  db:
    image: postgres:13
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydatabase
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

## 6 Docker イメージのビルド
Docker イメージをビルドします。

```bash
docker-compose build
```

## 7 データベースの作成
データベースを作成します。

```bash
docker-compose run backend rails db:create
```

## 8 アプリケーションの起動
すべてのサービスを起動します。

```bash
docker-compose up
```

## 9 動作確認
ブラウザで http://localhost:3000 にアクセスして、Rails のデフォルトページが表示されることを確認します。