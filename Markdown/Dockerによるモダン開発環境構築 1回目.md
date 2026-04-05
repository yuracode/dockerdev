# Docker ハンズオン — 初回

> **目標：** Docker の基本概念を理解し、コンテナの起動・確認・停止を自力で行えるようになる

---

## 1. Docker とは何か

### 仮想マシン（VM）との違い

Linux・VM を使ってきた皆さんへ向けた説明から始めます。

| 比較項目 | 仮想マシン（VM） | Docker コンテナ |
|----------|----------------|----------------|
| 起動時間 | 数十秒〜数分 | 数秒以内 |
| サイズ | GB 単位 | MB 単位（軽量） |
| OS | ゲスト OS を丸ごと持つ | ホスト OS のカーネルを共有 |
| 用途 | 完全な環境の隔離 | アプリ単位の隔離・配布 |

> **ポイント：** VM は「家ごと引っ越す」、Docker は「荷物だけ持ち運ぶ」イメージです。

### 重要な用語

| 用語 | 意味 |
|------|------|
| **イメージ** | コンテナの設計図（読み取り専用） |
| **コンテナ** | イメージを実行した実体（インスタンス） |
| **DockerHub** | イメージの公開レジストリ（GitHub のイメージ版） |
| **ポートマッピング** | ホスト側のポートとコンテナ内のポートを繋ぐ設定 |
| **ボリュームマウント** | ホスト側のフォルダをコンテナ内に繋ぐ設定 |

---

## 2. よく使う Docker コマンド

```bash
# イメージを取得する
docker pull <イメージ名>:<タグ>

# コンテナを起動する（イメージがなければ自動でpull）
docker run <オプション> <イメージ名>

# 起動中のコンテナ一覧
docker ps

# 全コンテナ一覧（停止中も含む）
docker ps -a

# コンテナを停止する
docker stop <コンテナ名 or ID>

# コンテナを削除する
docker rm <コンテナ名 or ID>

# イメージ一覧
docker images

# イメージを削除する
docker rmi <イメージ名>
```

---

## 3. ハンズオン① — Nginx で静的サイトを表示する

### 3.1 HTML ファイルを用意する

作業用フォルダを作り、その中に `index.html` を作成します。

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Nginx Handson</title>
</head>
<body>
    <h1>Hello Nginx!</h1>
    <p>index.html を表示中</p>
</body>
</html>
```

### 3.2 Nginx コンテナを起動する

`index.html` があるフォルダに移動してから以下を実行します。

```bash
docker run --name my-nginx -p 8080:80 -v ${PWD}:/usr/share/nginx/html:ro -d nginx:alpine
```

#### オプションの意味

| オプション | 意味 |
|------------|------|
| `--name my-nginx` | コンテナに名前をつける |
| `-p 8080:80` | ホスト:8080 → コンテナ:80 へポートを繋ぐ |
| `-v ${PWD}:/usr/share/nginx/html:ro` | 現在のフォルダを Nginx のドキュメントルートにマウント（読み取り専用） |
| `-d` | バックグラウンドで起動（デタッチモード） |
| `nginx:alpine` | Alpine Linux ベースの軽量 Nginx イメージ |

> **確認：** ブラウザで `http://localhost:8080` にアクセス → "Hello Nginx!" が表示されれば成功！

### 3.3 リアルタイム編集を体験する

`index.html` を書き換えて保存し、ブラウザをリロードしてみましょう。  
コンテナを再起動しなくても即座に反映される（`-v` マウントの効果）ことが確認できます。

### 3.4 後片付け

```bash
docker stop my-nginx
docker rm my-nginx
```

---

## 4. ハンズオン② — Node.js でランタイム情報を表示する

### 4.1 `index.js` を作成する

```javascript
const http = require('http');
const os = require('os');

const server = http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });

    const info = {
        'Node.js Version': process.version,
        'V8 Engine':       process.versions.v8,
        'Platform (OS)':   process.platform,   // コンテナ内は 'linux' と表示される
        'Architecture':    process.arch,
        'Container Hostname': os.hostname()    // コンテナ ID が表示される
    };

    let html = '<h1>Docker Runtime Check</h1><ul>';
    for (const [key, value] of Object.entries(info)) {
        html += `<li><b>${key}:</b> ${value}</li>`;
    }
    html += '</ul><p>※ イメージのタグだけ変えて起動し直してみよう！</p>';

    res.end(html);
});

server.listen(3000, () => {
    console.log('Server is running...');
});
```

### 4.2 Node.js 20（Alpine）で起動する

```bash
docker run --name node-test -p 3000:3000 -v ${PWD}:/app -w /app -d node:20-alpine node index.js
```

> **確認：** `http://localhost:3000` を開き、`v20.x.x` / `linux` が表示されることを確認。

### 4.3 Node.js 18（Debian）に切り替える

```bash
# 停止・削除
docker stop node-test && docker rm node-test

# Node 18（Debian ベース）で起動し直す
docker run --name node-test -p 3000:3000 -v ${PWD}:/app -w /app -d node:18-bookworm node index.js
```

> **確認：** リロードすると `v18.x.x` に変わる。コードは変えていないのにバージョンが変わることを確認しましょう。

### 4.4 このデモで学べること

1. **環境のポータビリティ**：イメージのタグを変えるだけで Node のバージョンを統一できる
2. **OS の隠蔽**：Windows 上で動かしていても `process.platform` は `linux` になる
3. **使い捨ての手軽さ**：`nvm` などのバージョン管理ツールなしに「その場限り」の環境が作れる

---

## 5. ハンズオン③ — PostgreSQL をコンテナで動かす

### 5.1 なぜ apt install より Docker が便利か

`apt` でインストールする場合、複数バージョンを共存させるには設定が複雑になります。  
Docker ならタグを変えるだけでバージョンを切り替えられます。

```bash
# apt でインストールした場合のバージョン共存イメージ（参考：面倒な例）
# /etc/postgresql/14/main/postgresql.conf
# /etc/postgresql/16/main/postgresql.conf
# ポートが競合するため個別に設定が必要 → 管理コストが高い
```

### 5.2 PostgreSQL 16 を起動する

```bash
docker run --name my-postgres16 \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -d postgres:16-alpine
```

#### オプションの意味

| オプション | 意味 |
|------------|------|
| `-e POSTGRES_PASSWORD=...` | 環境変数でパスワードを設定（コンテナ固有の設定渡し方） |
| `postgres:16-alpine` | PostgreSQL 16 の軽量イメージ |

> **接続確認：**
> ```bash
> psql -h localhost -U postgres -p 5432
> ```

### 5.3 PostgreSQL 14 に切り替える

```bash
docker stop my-postgres16

docker run --name my-postgres14 \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -d postgres:14-alpine
```

### 5.4 PostgreSQL 14 に切り替える(複数のオプション)

```bash
psql -h localhost -U postgres -p 5432
```

```bash
docker stop my-postgres16

docker run --name my-postgres14 \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -e POSTGRES_USER=sasaki \
  -d postgres:14-alpine
```

```bash
psql -h localhost -U sasaki -p 5432
```

### 5.5 後片付け

```bash
docker stop my-postgres14
docker rm my-postgres14
```

> **DockerHub の公式イメージ：** https://hub.docker.com/_/postgres/tags  
> タグ一覧からバージョンや OS ベースを選べます。

---

## 6. まとめ

| 今日やったこと | 学んだこと |
|----------------|------------|
| Nginx コンテナを起動 | `-p`（ポートマッピング）と `-v`（ボリュームマウント）の意味 |
| Node.js のバージョンを切り替え | イメージタグで実行環境をコントロールできる |
| PostgreSQL を Docker で動かす | 環境変数 `-e` でコンテナに設定を渡す方法 |

### 次回予告

- `Dockerfile` を自分で書いてオリジナルイメージを作る
- `docker-compose` で複数コンテナを一括管理する

---

*授業資料 — 専門学校北海道サイバークリエイターズ大学校*