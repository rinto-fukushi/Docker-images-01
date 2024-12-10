# 環境構築手順
- フロントエンド: React
- バックエンド: Fast API
- DB: Postgres

のreact-fast-postgresテンプレートの作成手順

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
または
```bash
npm install react@18 react-dom@18
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

## 4 Backend: Fast API の設定

### 4-1: Fast APIアプリの作成
backend ディレクトリに移動し、必要なファイルを作成します。
```bash
cd ../backend
```

### 4-2: main.pyの作成
main.pyを作成し、以下を記述します。

```Python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"Hello": "World"}
```
### 4-3: 依存関係の設定
requirements.txt を作成します。
```txt
fastapi
uvicorn[standard]
psycopg2-binary
```
> psycopg2-binaryとは？ PostgreSQLデータベースに接続するためのPythonライブラリです。


### 4-4: Dockerfileの作成
Dockerfile を作成します。

```Dockerfile
FROM python:3.8-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

```
> uvicornとは？ 高速なASGI（Asynchronous Server Gateway Interface）サーバーで、FastAPIと組み合わせて使用します



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
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:password@db:5432/mydatabase
    depends_on:
      - db

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

## 6-1 buildに失敗した場合
### フロントエンドでエラーが出た場合
Reactのバージョンをダウングレードさせる

#### 1. 現在のReactをアンインストール
現在のreactとreact-domを削除します。

```bash
npm uninstall react react-dom
```
これにより、既存のReactとその依存関係が削除されます。

#### 2. React 18.3.1 をインストール
ReactとReact DOMのバージョンを明示的に指定してインストールします。
```bash
npm install react@18.3.1 react-dom@18.3.1
```

#### 3. package-lock.json を確認
もしエラーが発生する場合、package-lock.json と node_modules が競合を引き起こしている可能性があります。その場合は以下を実行します。
```bash
rm -rf node_modules package-lock.json
npm install
```

#### 4. 依存関係を再インストール
プロジェクトに他の依存パッケージがある場合は、再インストールして全体の整合性を確保します。
```bash
npm install
```
#### 5. peerDependencies のエラー解消
もし依然としてpeerDependenciesのエラーが発生する場合は、以下のオプションを使用してインストールします。
  - ```--legacy-peer-deps``` を使用（推奨）
```bash
npm install --legacy-peer-deps
```
  - ```--force``` を使用（最終手段）
```bash
npm install --force
```

#### 6. インストールされたReactのバージョン確認
以下のコマンドで正しいバージョンがインストールされていることを確認します。
```bash
npm list react react-dom
```
期待される出力（例）
```bash
react@18.3.1
react-dom@18.3.1
```

#### 7. web-vitals をインストール
web-vitals パッケージをプロジェクトに追加します。
Docker環境で直接インストールする場合、以下を実行します：
```bash
npm install web-vitals --save
```

#### 8. 必要なパッケージを一括インストール
Reactの基本的な依存関係を再確認し、欠けているパッケージをインストールします。
以下を実行してください：
```bash
npm install react-scripts web-vitals @testing-library/react @testing-library/user-event --save
```

## 8 アプリケーションの起動
すべてのサービスを起動します。

```bash
docker-compose up
```

## 9 動作確認
ブラウザで http://localhost:3000 にアクセスして、Rails のデフォルトページが表示されることを確認します。