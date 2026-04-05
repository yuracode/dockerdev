# Docker ハンズオン — 2回目

> **目標：** Dockerfile を使って自分のアプリをコンテナ化できるようになる

---

## 1. 前回の振り返り

前回学んだ内容を振り返りましょう。以下のコマンドが自信を持って打てますか？

| コマンド | 確認内容 |
|----------|----------|
| `docker run hello-world` | Docker が動作していることを確認 |
| `docker run -it alpine /bin/sh` | Alpine Linux コンテナに入り、`ls` / `whoami` を実行 |
| `docker run -p 8080:80 nginx` | ブラウザで `localhost:8080` を開き Nginx を確認 |
| `docker ps` / `docker ps -a` | 起動中・停止済みコンテナの一覧を確認 |
| `docker stop [ID]` / `docker rm [ID]` | コンテナを停止・削除 |

> **キーワード：** イメージとコンテナの違い、`docker run` = pull + create + start

---

## 2. コンテナのライフサイクル

コンテナには「一生」があります。作成から破棄までの流れを理解しましょう。

| フェーズ | 意味 |
|----------|------|
| **Create（作成）** | イメージからコンテナの器を作る |
| **Start（起動）** | コンテナ内のプロセスを起動する |
| **Stop（停止）** | プロセスを安全に停止する |
| **Rm（削除）** | 不要になったコンテナを削除する |

> **ポイント：** `docker run` は Create と Start を同時に行います。

---

## 3. Dockerfile の基礎

### Dockerfile とは？

Dockerfile はサーバー設定書の進化形。コードで書けば、誰でも同じサーバーが作れます。

**これまでのサーバー設定：**
- SSH でサーバーに入る
- コマンドを 1 行ずつ手打ち
- 途中でエラー → 原因不明
- 他のサーバーで同じことを繰り返す…

**Dockerfile による革命：**
- `FROM node:18-alpine` ← OS と言語を指定
- `RUN npm install` ← パッケージ導入
- `CMD ["node", "app"]` ← 起動コマンド

このテキストがそのままサーバーの設計書。Git で管理 → チームで共有 → どこでも同じサーバーが再現可能。  
これが **Infrastructure as Code（IaC）** の入口です。

### 基本命令

| 命令 | 意味 | 例 |
|------|------|----|
| **FROM** | ベースとなるイメージを指定 | `FROM node:18-alpine` |
| **RUN** | ビルド時に実行されるコマンド | `RUN npm install` |
| **COPY** | ホストのファイルをコンテナにコピー | `COPY package.json /app/` |
| **CMD** | コンテナ起動時に実行されるデフォルトコマンド | `CMD ["node", "server.js"]` |

---

## 4. ハンズオン① — Nginx で複数ファイルの静的サイトを表示する

前回は 1 ファイルだけでしたが、今回は HTML / CSS / 画像に分けた**現実的な構成**でやってみましょう。

### 4.1 ファイル構成

```
my-nginx-site/
├── index.html
├── profile.html
├── css/
│   └── style.css
└── img/
    └── docker-logo.png   ← 任意の画像（PNG/JPGなら何でも可）
```

### 4.2 `index.html` を作成する

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>My Docker Site</title>
  <link rel="stylesheet" href="css/style.css">
</head>
<body>
  <header>
    <h1>🐳 My Docker Site</h1>
    <nav>
      <a href="index.html">ホーム</a>
      <a href="profile.html">プロフィール</a>
    </nav>
  </header>
  <main>
    <img src="img/docker-logo.png" alt="Docker Logo" class="logo">
    <p>このサイトは <strong>Nginx コンテナ</strong> で動いています。</p>
    <p>ホストのファイルをそのままマウントしているので、<br>
       ファイルを編集 → 保存 → リロードで即反映されます。</p>
  </main>
  <footer>
    <p>Docker Handson 2回目</p>
  </footer>
</body>
</html>
```

### 4.3 `profile.html` を作成する

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>プロフィール | My Docker Site</title>
  <link rel="stylesheet" href="css/style.css">
</head>
<body>
  <header>
    <h1>🐳 My Docker Site</h1>
    <nav>
      <a href="index.html">ホーム</a>
      <a href="profile.html">プロフィール</a>
    </nav>
  </header>
  <main>
    <h2>プロフィール</h2>
    <table>
      <tr><th>名前</th><td>Your Name</td></tr>
      <tr><th>学科</th><td>ICT 学科 2年</td></tr>
      <tr><th>好きな技術</th><td>Docker / Linux / JavaScript</td></tr>
    </table>
  </main>
  <footer>
    <p>Docker Handson 2回目</p>
  </footer>
</body>
</html>
```

### 4.4 `css/style.css` を作成する

```css
/* リセット */
* { box-sizing: border-box; margin: 0; padding: 0; }

body {
  font-family: 'Helvetica Neue', Arial, sans-serif;
  background: #f0f4f8;
  color: #333;
  line-height: 1.7;
}

/* ヘッダー */
header {
  background: #0d6efd;
  color: white;
  padding: 16px 32px;
  display: flex;
  align-items: center;
  gap: 32px;
}
header h1 { font-size: 1.4rem; }
nav a {
  color: white;
  text-decoration: none;
  margin-right: 16px;
  font-size: 0.95rem;
}
nav a:hover { text-decoration: underline; }

/* メイン */
main {
  max-width: 720px;
  margin: 40px auto;
  background: white;
  padding: 32px;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.08);
}
.logo {
  width: 100px;
  display: block;
  margin-bottom: 20px;
}

/* テーブル */
table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 16px;
}
th, td {
  border: 1px solid #dee2e6;
  padding: 10px 14px;
  text-align: left;
}
th { background: #e9ecef; width: 30%; }

/* フッター */
footer {
  text-align: center;
  padding: 20px;
  font-size: 0.85rem;
  color: #888;
}
```

### 4.5 Nginx コンテナを起動する

`my-nginx-site/` フォルダに移動してから実行します。

```bash
docker run --name my-nginx \
  -p 8080:80 \
  -v ${PWD}:/usr/share/nginx/html:ro \
  -d nginx:alpine
```

> **確認：** `http://localhost:8080` → トップページ、`http://localhost:8080/profile.html` → プロフィールページ

### 4.6 複数ファイル構成で気づくこと

- `-v` でフォルダごとマウントしているため、**サブフォルダ（css/ img/）も自動で見える**
- ファイルを保存してリロードするだけで即反映される（コンテナ再起動不要）
- `index.html` が Nginx のデフォルトページになる仕組みを体験できる

### 4.7 後片付け

```bash
docker stop my-nginx
docker rm my-nginx
```

---

## 5. ハンズオン② — Node.js アプリの Dockerfile 作成とビルド

### 5.1 サンプルアプリの用意

作業用フォルダを作り、その中に以下のファイルを作成します。

**package.json**
```json
{
  "name": "my-node-app",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^5.2.1"
  }
}
```

**server.js**
```javascript
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.send('<h1>Hello Docker!</h1><p>Node.js アプリがコンテナで動いています</p>');
});

app.listen(port, () => {
  console.log(`App listening at http://localhost:${port}`);
});
```

**.dockerignore**
```
node_modules
.git
.env
*.log
Dockerfile
```

### 5.2 Dockerfile を作成する

```dockerfile
FROM node:18-alpine

# 作業ディレクトリの指定
WORKDIR /app

# 依存関係のインストール（キャッシュ活用）
COPY package.json ./
RUN npm install

# ソースコードのコピー
COPY . .

# ポートの公開（ドキュメントとしての意味合い）
EXPOSE 3000

# アプリケーションの起動
CMD ["node", "server.js"]
```

> **ポイント：** `package.json` を先にコピーして `npm install` → その後ソースコードをコピーする順番が重要。  
> ソースだけ変更した場合、`npm install` のステップがキャッシュされてビルドが速くなります。

### 5.3 イメージをビルドする

```bash
docker build -t my-node-app .
```

### 5.4 コンテナを起動して動作確認する

```bash
docker run -p 3000:3000 my-node-app
```

> **確認：** `http://localhost:3000` → "Hello Docker!" が表示されれば成功！  
> 停止は `Ctrl+C`

---

## 6. トラブル対応

### ビルド失敗時

`docker build` に失敗した場合、エラーメッセージを確認してどのステップで止まったかを特定します。

よくある原因：
- ファイル名のタイポ（`Dockerfile` の大文字小文字に注意）
- `npm install` のエラー（`package.json` の記述ミス）
- `COPY` 元のパス間違い

### コンテナがすぐ終了する

`docker run` してもすぐ終了してしまう場合はログを確認します。

```bash
# ログを確認する
docker logs <コンテナ名 or ID>

# リアルタイムで追跡する
docker logs -f <コンテナ名 or ID>
```

Node.js のランタイムエラーやポートの競合などが原因として多いです。

---

## 7. 本日のまとめ

| やったこと | 学んだこと |
|------------|------------|
| Nginx で複数ファイル構成のサイトを表示 | サブフォルダごとマウントできる `-v` の仕組み |
| Dockerfile を書いて Node.js アプリをビルド | FROM / RUN / COPY / CMD の 4 命令 |
| `docker build` でオリジナルイメージを作成 | イメージとは「自分専用のサーバー設計書」 |
| `docker logs` でエラーを確認 | コンテナ内の状態をログで把握する方法 |

### 次回予告

- Node.js + PostgreSQL を **一緒に** 動かす！
- `docker-compose` で複数コンテナを一括管理する

---

*授業資料 — 専門学校北海道サイバークリエイターズ大学校*