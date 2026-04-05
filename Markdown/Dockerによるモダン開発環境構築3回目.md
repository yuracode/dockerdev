# Docker ハンズオン — 3回目

> **目標：** Docker Composeを使って複数コンテナのシステムを構築できるようになる

---

## 1. 前回の振り返り

前回学んだ内容を振り返りましょう。

以下の内容がスラスラ書けますか？

- Dockerfileの4大命令：FROM / RUN / COPY / CMD（それぞれのタイミングを説明できる？）
- `docker build -t my-app .`（-tと.の意味）
- `docker run -p 3000:3000 my-app`（-pの左右の意味）
- キャッシュを活かすDockerfileの書き順：package.json → npm install → ソースコード
- ビルド失敗時の対処法：どのステップで失敗したかを確認する

今日はここに「複数コンテナの連携」を追加します！

---

## 2. Docker Compose の基礎

### Docker Compose とは？

複数のサーバーで構成するシステムを、1ファイルで定義するツール。

**現実のWebシステムは複数のサーバーで動いている：**
- Webサーバー (nginx) → リクエストを転送
- アプリサーバー (Node.js) → データを読み書き
- DBサーバー (PostgreSQL)

**従来の構築：**
- 3台のサーバーを個別に設定 → 数日かかる

**Docker Compose なら：**
- `docker-compose.yml` 1ファイルに全サーバーを定義
- `docker compose up -d` で全サーバーが同時起動
- サーバー間のネットワークも自動構築

### 基本文法（Node.js / PostgreSQL）

```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - db
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
```

#### 注意：YAML の鉄則

エラーの8割はインデント。先に知っておくと全然違う。

- **鉄則：インデントはスペース2つ**
- タブ文字を使わない
- コロン(:)の後にスペースを入れる

よくあるエラー：
- `yaml.scanner.ScannerError` → インデントが揃っていない
- `mapping values are not allowed here` → コロン(:)の後にスペースがない

VS Codeを使う場合：YAML拡張機能をインストールし、タブをスペースに変換する設定にする。

### depends_on の限界

「起動した」と「使える状態」は別物。ハマりやすいポイント。

- `depends_on` の意味：app が db より後に起動する（起動順の話）
- 解決しないこと：dbコンテナが「起動した」 ≠ PostgreSQLが「接続受付中」

実際には起動から数秒〜十数秒かかる。その間にappがDB接続しようとすると接続エラー！

解決策：
1. （簡単）アプリ側でリトライ処理を入れる
2. （推奨）healthcheck + condition を使う
3. （簡易）wait-for-it.sh スクリプトを使う

---

## 3. ネットワークの概念

### ブリッジネットワークとは？

なぜコンテナ間で通信ができるのか？

- Docker Composeはデフォルトで専用のブリッジネットワークを作成
- 同じネットワーク内のコンテナは、サービス名で名前解決（DNS）が可能
- IPアドレスを意識せず、`db` や `app` といった名前で通信できる

---

## 4. ハンズオン① — Node.js + PostgreSQL システムの構築

### 4.1 サンプルアプリの用意

作業用フォルダを作り、その中に以下のファイルを作成します。

**package.json**
```json
{
  "name": "my-node-postgres-app",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.11.0"
  }
}
```

**server.js**
```javascript
const express = require('express');
const { Client } = require('pg');
const app = express();
const port = 3000;

// PostgreSQL接続設定
const client = new Client({
  host: 'db',  // Docker Composeのサービス名
  port: 5432,
  user: 'postgres',
  password: 'secret',
  database: 'postgres'
});

app.get('/', async (req, res) => {
  try {
    await client.connect();
    const result = await client.query('SELECT NOW()');
    await client.end();
    res.send(`<h1>Hello Docker Compose!</h1><p>PostgreSQL接続成功: ${result.rows[0].now}</p>`);
  } catch (err) {
    res.send(`<h1>DB接続エラー</h1><p>${err.message}</p>`);
  }
});

app.listen(port, () => {
  console.log(`App listening at http://localhost:${port}`);
});
```

**Dockerfile**
```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["node", "server.js"]
```

**.dockerignore**
```
node_modules
.git
.env
*.log
Dockerfile
docker-compose.yml
```

### 4.2 docker-compose.yml の作成

同じフォルダに `docker-compose.yml` を作成します。

```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - db
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"  # ホストからも接続可能にする場合
```

### 4.3 システムの起動

以下のコマンドを実行してシステムを起動します。

```bash
docker compose up -d
```

### 4.4 動作確認

ブラウザで `http://localhost:3000` にアクセスして、「Hello Docker Compose!」とPostgreSQLの現在時刻が表示されることを確認しましょう。

停止するには以下のコマンドを実行します。

```bash
docker compose down
```

### 4.5 ネットワークの確認

コンテナが同じネットワークに属していることを確認しましょう。

```bash
# ネットワーク一覧
docker network ls

# 特定のネットワークの詳細
docker network inspect [ネットワーク名]

# コンテナ内からping疎通確認
docker exec -it [appコンテナID] ping db
```

---

## 5. 発展課題 — カスタムネットワーク

### 5.1 カスタムネットワークの作成

docker-compose.yml を以下のように修正します。

```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - db
    networks:
      - my-network
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    networks:
      - my-network

networks:
  my-network:
    driver: bridge
```

### 5.2 再起動と確認

```bash
docker compose down
docker compose up -d
```

ネットワークがカスタムネットワークに変わっていることを確認しましょう。

---

## 6. トラブル対応

### depends_on の問題

DB接続エラーが出る場合：
- PostgreSQLの起動に時間がかかっている可能性
- アプリ側でリトライ処理を追加するか、healthcheckを使用

### ネットワークの問題

コンテナ間で通信できない場合：
- サービス名が正しいか確認（docker-compose.ymlのservices名）
- ネットワーク設定を確認（`docker network inspect`）

---

## 7. 本日のまとめ

Docker Composeで「システム全体」をコード1枚で表現できるようになりました。

- Docker Composeの基本文法で複数サービスを定義できた
- ブリッジネットワークの概念を理解した
- Node.js + PostgreSQLのシステムを構築できた

次回：データが消えない本番仕様のシステム！</content>
<parameter name="filePath">c:\Users\user\Downloads\PPT-20260403T233758Z-3-001\Markdown\Dockerによるモダン開発環境3回目.md