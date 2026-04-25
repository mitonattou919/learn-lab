# Docker チュートリアル

> Docker を初めて触る人向けのハンズオン教材。8章を順に進めれば「コンテナを動かす・作る・繋げる」基本が身につきます。
> 全体像を先に俯瞰したい方 → `docker-overview.md`
> 本番運用・チーム設計を深掘りしたい方 → `docker-best-practices.md`

---

## Chapter 0: Docker とは

### 1. 「自分の環境では動くのに」を終わりにする

```
「ローカルでは動くのに、本番で動かない」
「新メンバーの環境セットアップに半日かかる」
「Node のバージョン違いでエラーになる」
```

こういった「環境差異」の問題に心当たりがあれば、Docker はその答えになります。

**Docker** はアプリとその実行環境をまるごと「コンテナ」という箱に詰めて持ち運ぶ仕組みです。コンテナを使えば、自分のMacでも、CIサーバーでも、本番Linuxでも、まったく同じ動作が保証されます。

### 2. 仮想マシンとの違い

| | 仮想マシン（VM） | Docker コンテナ |
|--|----------------|----------------|
| ゲスト OS | 各VMが丸ごと持つ（数GB） | ホストのカーネルを共有（数MB） |
| 起動時間 | 数分 | 数秒以下 |
| 用途分け | OS が違う環境・強い分離が必要な場面 | アプリの配布・CI・高密度デプロイ |

### 3. このチュートリアルで作るもの

全8章を通じて `docker-practice/` ディレクトリを育てていきます。

```
docker-practice/
├── hello/          ← Chapter 3: Node.js アプリのイメージをビルド
├── volumes/        ← Chapter 4: Volume でデータ永続化
├── network/        ← Chapter 5: コンテナ間通信
└── compose/        ← Chapter 6: Docker Compose でまとめる
```

---

## Chapter 1: インストールと初回体験

### 1. 必要な環境

| OS | 推奨手段 |
|----|---------|
| **macOS** | Docker Desktop for Mac |
| **Windows** | Docker Desktop for Windows（WSL2 有効化が必要） |
| **Linux** | Docker Engine（apt / yum でインストール） |

### 2. Docker Desktop のインストール（macOS / Windows）

[https://docs.docker.com/desktop/install/](https://docs.docker.com/desktop/install/) からインストーラーをダウンロードして実行してください。インストール後、Docker Desktop を起動するとメニューバー（macOS）やタスクトレイ（Windows）にクジラアイコンが表示されます。

### 3. Docker Engine のインストール（Ubuntu の場合）

```bash
# 公式スクリプトで一発インストール
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# ログアウト→ログインで反映
```

### 4. インストール確認

```bash
docker --version
# Docker version 26.x.x, build xxxx

docker info
# サーバーの情報が表示されれば OK
```

> **うまくいかない時**: `Cannot connect to the Docker daemon` が出たら Docker Desktop が起動しているか確認。Linux では `sudo systemctl start docker` で起動できます。

### 5. ⭐ 5分で成功体験

まずは公式の `hello-world` イメージを動かしてみましょう。

```bash
docker run hello-world
```

```text
Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```

この一行で何が起きたか：

```
1. Docker Hub から hello-world イメージを pull（ダウンロード）
2. そのイメージからコンテナを作成
3. コンテナの中でプログラムを実行
4. 終了してコンテナが停止
```

おめでとうございます。これが Docker の基本サイクルです。

### 練習課題

1. `docker run hello-world` を2回実行してみる（2回目はキャッシュが使われることを確認）
2. `docker images` を実行して `hello-world` イメージが保存されているか確認する
3. `docker ps -a` を実行して実行済みコンテナの一覧を確認する

---

## Chapter 2: コンテナを動かす基本操作

### 1. 最初に覚える 6 コマンド

| コマンド | 何をするか |
|---------|----------|
| `docker run` | イメージからコンテナを作って起動 |
| `docker ps` | 実行中のコンテナ一覧 |
| `docker ps -a` | 停止中も含めた全コンテナ一覧 |
| `docker stop` | コンテナを停止 |
| `docker rm` | 停止中のコンテナを削除 |
| `docker logs` | コンテナのログを表示 |

### 2. nginx を動かしてみる

```bash
docker run -d -p 8080:80 --name my-nginx nginx:1.25
#           ↑        ↑          ↑              ↑
#     バックグラウンド  ポートマッピング  コンテナ名     イメージ名
```

ブラウザで `http://localhost:8080` を開くと nginx の Welcome ページが表示されます。

```bash
docker ps          # 実行中を確認
docker logs my-nginx    # アクセスログを確認
```

### 3. コンテナの止め方・消し方

```bash
docker stop my-nginx    # コンテナを停止
docker rm my-nginx      # コンテナを削除

# 一発で停止＆削除
docker rm -f my-nginx
```

> **`--rm` フラグ**: `docker run --rm ...` とすると、コンテナ停止時に自動削除されます。使い捨てのテスト実行に便利です。

### 4. コンテナの中を覗く

実行中のコンテナにシェルで入れます。

```bash
docker run -d --name my-nginx nginx:1.25
docker exec -it my-nginx bash
# コンテナの中に入った状態
ls /etc/nginx/
cat /etc/nginx/nginx.conf
exit   # コンテナから出る
```

| オプション | 意味 |
|---------|------|
| `-i` | 標準入力を開いたまま（インタラクティブ） |
| `-t` | 疑似ターミナルを割り当て |
| `-it` | シェルで操作するときは常にこの組み合わせ |

### 5. 便利な run オプション

| オプション | 例 | 意味 |
|---------|---|------|
| `-d` | `docker run -d nginx` | バックグラウンドで起動 |
| `-p` | `-p 8080:80` | ホスト:コンテナ のポートマッピング |
| `--name` | `--name my-nginx` | コンテナに名前をつける |
| `--rm` | `docker run --rm alpine` | 終了時に自動削除 |
| `-e` | `-e ENV_VAR=value` | 環境変数を渡す |

### 練習課題

`docker-practice/` ディレクトリを作って以下を実施してください。

```bash
mkdir -p docker-practice && cd docker-practice
```

1. `docker run -d -p 8080:80 --name practice-nginx nginx:1.25` で nginx を起動し、ブラウザで確認する
2. `docker logs practice-nginx` でログを確認する
3. `docker exec -it practice-nginx bash` でコンテナに入り、`nginx -v` でバージョンを確認する
4. `exit` で出て、`docker stop practice-nginx && docker rm practice-nginx` で後片付けする

---

## Chapter 3: 自分のイメージをビルドする

### 1. なぜ自前のイメージが必要か

公式イメージはベースに過ぎません。自分のアプリを乗せた独自イメージを作ることで、「どこでも同じ動作」が実現します。

```
公式イメージ（node:20）
    +
自分のアプリコード
    ↓ docker build
自分のイメージ（myapp:1.0）
    ↓ docker run
コンテナ（本番と同じ動作）
```

### 2. シンプルな Node.js アプリを作る

```bash
mkdir -p docker-practice/hello && cd docker-practice/hello
```

`app.js` を作成：

```javascript
// app.js
const http = require('http');
const PORT = process.env.PORT || 3000;

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello from Docker!\n');
});

server.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

`package.json` を作成：

```json
{
  "name": "hello-docker",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  }
}
```

### 3. Dockerfile を書く

```dockerfile
# Dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["node", "app.js"]
```

各命令の意味：

| 命令 | 何をするか |
|-----|-----------|
| `FROM node:20-alpine` | Node.js 20 の軽量ベースイメージを使用 |
| `WORKDIR /app` | コンテナ内の作業ディレクトリを設定 |
| `COPY package.json ./` | 依存定義だけ先にコピー（キャッシュ最適化） |
| `RUN npm install` | 依存ライブラリをインストール |
| `COPY . .` | アプリコード全体をコピー |
| `EXPOSE 3000` | 使用ポートを宣言（ドキュメント目的） |
| `CMD ["node", "app.js"]` | コンテナ起動時のコマンド |

> ⭐ **キャッシュ最適化のポイント**: `COPY package.json` → `RUN npm install` → `COPY . .` の順にすることで、コードだけ変えた場合に `npm install` がキャッシュから再利用されます。逆順にすると毎回インストールが走って遅くなります。

### 4. ビルドして実行

```bash
# docker-practice/hello/ にいる状態で
docker build -t hello-docker:1.0 .
#              ↑                ↑
#          イメージ名:タグ    Dockerfileの場所（現在ディレクトリ）

docker images   # ビルドされたイメージを確認

docker run -d -p 3000:3000 --name hello hello-docker:1.0
```

ブラウザで `http://localhost:3000` を開くと `Hello from Docker!` と表示されます。

```bash
docker logs hello    # ログ確認
docker rm -f hello   # 後片付け
```

> **うまくいかない時**: `COPY failed` が出たら Dockerfile と同じディレクトリで `docker build` を実行しているか確認。`package.json` が存在するか `ls` で確認してください。

### 練習課題

1. `app.js` のメッセージを自分の名前を使った文言に変更し、`docker build -t hello-docker:1.1 .` で新しいバージョンをビルドする
2. `docker run --rm -p 3000:3000 hello-docker:1.1` で確認する（`--rm` で自動削除）
3. `docker images` で `:1.0` と `:1.1` の2つのイメージが存在することを確認する
4. `docker rmi hello-docker:1.0` で古いイメージを削除する

---

## Chapter 4: データを残す（Volume と Bind Mount）

### 1. コンテナのデータは消える

コンテナを削除するとその中に書き込んだデータも消えます。データベースのデータやアップロードファイルを永続化するには **Volume** または **Bind Mount** を使います。

| 方式 | 使い場面 |
|------|---------|
| **Volume** | DB データなど永続化すべきデータ（本番・CI） |
| **Bind Mount** | 開発中のソースコードのホットリロード |

### 2. Volume を使う：PostgreSQL の例

```bash
# Volume を作成
docker volume create pg-data

# PostgreSQL コンテナを Volume 付きで起動
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=mydb \
  -v pg-data:/var/lib/postgresql/data \
  postgres:16
```

データを書き込む：

```bash
docker exec -it postgres psql -U postgres -d mydb -c \
  "CREATE TABLE users (id SERIAL, name TEXT); INSERT INTO users (name) VALUES ('Alice');"
```

コンテナを削除して再作成しても、データが残ることを確認：

```bash
docker rm -f postgres

# 同じ Volume を指定して再起動
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=mydb \
  -v pg-data:/var/lib/postgresql/data \
  postgres:16

docker exec -it postgres psql -U postgres -d mydb -c "SELECT * FROM users;"
# Alice が表示される → データが永続化されている！
```

### 3. Bind Mount を使う：開発時のホットリロード

```bash
mkdir -p docker-practice/volumes && cd docker-practice/volumes
echo "<h1>Hello Bind Mount</h1>" > index.html

# カレントディレクトリを nginx の公開ディレクトリにマウント
docker run -d -p 8080:80 --name web \
  -v $(pwd):/usr/share/nginx/html \
  nginx:1.25
```

ブラウザで `http://localhost:8080` を開き、`index.html` を編集してリロードすると即座に変更が反映されます。

```bash
echo "<h1>Updated!</h1>" > index.html
# ブラウザをリロード → 変更が反映される
docker rm -f web
```

### 4. Volume の管理コマンド

```bash
docker volume ls              # Volume 一覧
docker volume inspect pg-data # 詳細確認
docker volume rm pg-data      # 削除
docker volume prune           # 不要な Volume を一括削除
```

### 練習課題

1. `docker volume create myvolume` で Volume を作成する
2. `docker run --rm -v myvolume:/data alpine sh -c "echo 'hello volume' > /data/test.txt"` でファイルを書き込む
3. `docker run --rm -v myvolume:/data alpine cat /data/test.txt` で読み出して `hello volume` と表示されることを確認する
4. `docker volume rm myvolume` で Volume を削除して、再度 3 のコマンドを実行するとどうなるか確認する

---

## Chapter 5: ネットワークとコンテナ間通信

### 1. コンテナはデフォルトで通信できない

別々のコンテナは、同じホストで動いていても直接通信できません。**ユーザー定義ネットワーク**を作ってコンテナを同じネットワークに参加させると、コンテナ名で通信できます。

```bash
mkdir -p docker-practice/network && cd docker-practice/network
```

### 2. ユーザー定義ネットワークを作る

```bash
# ネットワーク作成
docker network create mynetwork

# バックエンド（Python/Flask） を起動
docker run -d \
  --name backend \
  --network mynetwork \
  -e FLASK_APP=app.py \
  python:3.12-slim \
  sh -c "pip install flask -q && python -c \"
from flask import Flask
app = Flask(__name__)
@app.route('/')
def hello(): return 'Response from backend!'
app.run(host='0.0.0.0', port=5000)
\""

# フロントエンド（curl で確認）
docker run --rm \
  --network mynetwork \
  alpine \
  sh -c "apk add curl -q && curl backend:5000"
# → Response from backend!
```

コンテナ名（`backend`）がそのままホスト名として使えることが分かります。

### 3. デフォルト bridge vs ユーザー定義ネットワーク

| 比較 | デフォルト bridge | ユーザー定義ネットワーク |
|------|-----------------|----------------------|
| コンテナ名で通信 | ❌ 不可 | ✅ 可能 |
| 分離性 | 全コンテナが同一ネットワーク | 明示的に同じネットワークのみ |
| `docker compose` | 自動でユーザー定義を作成 | — |

### 4. ネットワーク管理コマンド

```bash
docker network ls                      # ネットワーク一覧
docker network inspect mynetwork       # 詳細（接続コンテナを確認）
docker network connect mynetwork 他のコンテナ  # 後から接続
docker network rm mynetwork            # 削除
```

```bash
# 後片付け
docker rm -f backend
docker network rm mynetwork
```

### 練習課題

1. `docker network create practice-net` でネットワークを作る
2. `docker run -d --name db --network practice-net -e POSTGRES_PASSWORD=pass postgres:16` で PostgreSQL を起動する
3. `docker run --rm --network practice-net postgres:16 pg_isready -h db -U postgres` を実行して `db:5432 - accepting connections` と表示されることを確認する（コンテナ名 `db` で到達できることの確認）
4. `docker rm -f db && docker network rm practice-net` で後片付けする

---

## Chapter 6: Docker Compose でまとめて管理

### 1. 複数コンテナを毎回 `docker run` するのはつらい

これまでの章でネットワーク・Volume・複数コンテナを組み合わせてきました。毎回コマンドを順番に打つのは大変です。**Docker Compose** は `compose.yaml` 一ファイルに全構成を書いて、`docker compose up` 一発で起動できます。

### 2. Web + DB の構成を作る

```bash
mkdir -p docker-practice/compose && cd docker-practice/compose
```

`compose.yaml` を作成：

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5

  web:
    image: nginx:1.25
    ports:
      - "8080:80"
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./html:/usr/share/nginx/html

volumes:
  pgdata:
```

`html/index.html` を作成：

```bash
mkdir html
echo "<h1>Hello Docker Compose!</h1>" > html/index.html
```

### 3. 起動・確認・停止

```bash
# 起動（バックグラウンド）
docker compose up -d

# 状態確認
docker compose ps

# ログを見る（全サービス）
docker compose logs

# 特定サービスのログ
docker compose logs -f web

# ブラウザで http://localhost:8080 を確認
```

DB に接続できることを確認：

```bash
docker compose exec db psql -U postgres -d appdb -c "\l"
```

停止と削除：

```bash
docker compose down       # コンテナ停止・削除（Volume は残る）
docker compose down -v    # Volume も含めて全削除
```

### 4. よく使う Compose コマンド

| コマンド | 動作 |
|---------|------|
| `docker compose up -d` | バックグラウンドで全サービス起動 |
| `docker compose down` | 全サービス停止・コンテナ削除 |
| `docker compose ps` | サービスの状態確認 |
| `docker compose logs -f` | リアルタイムログ表示 |
| `docker compose exec db bash` | 実行中コンテナにシェルで入る |
| `docker compose restart web` | 特定サービスを再起動 |
| `docker compose build` | `build:` 指定のサービスを再ビルド |

### 5. Compose で自前イメージを使う

Chapter 3 で作った `hello-docker` アプリを Compose に組み込む例：

```yaml
services:
  app:
    build: ../hello    # Dockerfile があるディレクトリを指定
    ports:
      - "3000:3000"
    environment:
      PORT: 3000
```

`docker compose up --build` でイメージのビルドと起動を同時に行えます。

### 練習課題

1. `docker-practice/compose/` の `compose.yaml` と `html/index.html` を作成し、`docker compose up -d` で起動する
2. ブラウザで `http://localhost:8080` が表示されることを確認する
3. `html/index.html` の内容を変更してブラウザをリロードし、変更が即座に反映されることを確認する（Bind Mount の効果）
4. `docker compose down -v` で後片付けする

---

## Chapter 7: 次の一歩

### 1. このチュートリアルで学んだこと

| 章 | 習得内容 |
|----|---------|
| Chapter 1 | インストール・`hello-world` で動作確認 |
| Chapter 2 | `run / ps / stop / rm / exec` の基本操作 |
| Chapter 3 | Dockerfile 作成・`docker build` でイメージをビルド |
| Chapter 4 | Volume / Bind Mount でデータ永続化 |
| Chapter 5 | ユーザー定義ネットワークでコンテナ間通信 |
| Chapter 6 | Docker Compose でマルチコンテナを一元管理 |

### 2. よく直面するエラーと対処

| エラー | 原因 | 対処 |
|--------|------|------|
| `Cannot connect to the Docker daemon` | Docker が起動していない | Docker Desktop を起動、または `systemctl start docker` |
| `port is already allocated` | ポートが使用中 | 使用中プロセスを停止するか、別のポートを指定 |
| `image not found` | イメージ名・タグのタイポ | `docker images` で確認、または `docker pull` で取得 |
| `no such file or directory` in build | Dockerfile のパスが間違っている | `docker build` を実行するディレクトリを確認 |
| コンテナがすぐ終了する | メインプロセスが終了している | `docker logs コンテナ名` でエラーを確認 |

### 3. 次のステップ

基本操作を身につけたら、以下の発展トピックに進みましょう。

| 学習トピック | 概要 |
|------------|------|
| **マルチステージビルド** | ビルド用イメージと実行用イメージを分けてサイズを削減 |
| **Docker Scout** | イメージの脆弱性スキャン |
| **プライベートレジストリ** | ECR / GCR など自組織のレジストリへの push |
| **本番運用パターン** | ヘルスチェック・ログ設計・シークレット管理 |
| **Kubernetes 入門** | 複数ホストでのコンテナオーケストレーション |

詳しくは `docker-best-practices.md`（予定）を参照してください。

---

## 関連リソース

- [Docker 公式ドキュメント](https://docs.docker.com/)
- [Docker Compose 仕様](https://docs.docker.com/compose/)
- [Docker Hub（公式イメージ検索）](https://hub.docker.com/)
- 全体像を俯瞰したい → `docker-overview.md`
- 本番・チーム運用を深掘りしたい → `docker-best-practices.md`（予定）
