# VS Code における Devcontainer チュートリアル

> 「コンテナの中で開発する」ってどこか難しそうに聞こえますが、
> 実際には「プロジェクトの環境設定をファイル 1枚にまとめ、
> チーム全員が同じ環境で開発できるようにする」仕組みです。
> Docker を触ったことがない方 → まず Docker Desktop をインストールしてから
> チーム運用・パフォーマンス最適化を深掘りしたい方 → `devcontainer-best-practices.md` へ

---

## Chapter 0: Devcontainer とは

### 1. 「環境の身分証明書」だと思って付き合う

新しいチームメンバーがプロジェクトに参加するとき、いつもこんな問題が起きていませんか？

```
「なぜか Node.js のバージョンが違うんだけど？」
「私の環境だとビルドできないんだけど？」
「山田さんの Mac では動くのに、磯田さんの Windows では動かない」
```

この問題の根本にあるのは、**開発環境が各自のマシンにインストールされた「ローカルの何か」に依存している**ことです。

**Devcontainer** はそれを解決します。
プロジェクトリポジトリの中に **「このプロジェクトはこういう環境で動かす」と書かれた身分証明書** を同居させ、
開く人全員がその環境のコンテナの中で開発できるようになります。

### 2. 3つの技術の役割分担

「コンテナ」と関係する技術は複数あります。混同しやすいので整理を。

| 技術 | 対象レイヤ | 問い |
|------|--------------|------|
| **Docker** | インフラ・コンテナランタイム | イメージをビルド・実行する基盤 |
| **Dev Containers 拡張** | VS Code 側のブリッジ | Docker と VS Code をつなぐ |
| **Devcontainer** | 開発環境定義 | 「この環境で開発する」を宣言する |

→ Docker を基盤に、Devcontainer が環境定義を持ち、Dev Containers 拡張が VS Code と結ぶ、という流れです。

### 3. Devcontainer が解決する問題

| 従来の問題 | Devcontainer で解決 |
|---------|------------------|
| 「私の環境だと動く」 | コンテナが環境ごと持ち運びされる |
| README のインストール手順が古い | `postCreateCommand` で自動化 |
| OS 差による挙動の違い | コンテナ内は常に Linux |
| 拡張機能のインストールリストが口伝 | `devcontainer.json` に一元管理 |

### 4. この教材の進め方

```
Chapter 1 → 最小構成で 5 分で動かす
Chapter 2 → devcontainer.json の主要プロパティを辞書的に知る
Chapter 3 → Dockerfile で実務環境に近づける
Chapter 4 → Features で部品を追加する
Chapter 5 → Docker Compose でデータベースも加える
Chapter 6 → ライフサイクルと Rebuild
Chapter 7 → チームで使う・次の一歩
```

---

## Chapter 1: インストールと初回ハロー体験

### 1. 必要な環境

| ソフトウェア | バージョン | 確認コマンド |
|------------|----------|-----------|
| **Docker Desktop** | 最新安定版 | `docker --version` |
| **VS Code** | 最新推奨 | — |
| **Dev Containers 拡張** | — | VS Code 拡張マーケット |

```bash
# Docker が起動しているか確認
docker ps
```

> **うまくいかない時**: `Cannot connect to the Docker daemon` が出たら Docker Desktop が
> 起動していません。タスクトレイ（Windows）またはメニューバー（Mac）から起動してください。

### 2. Dev Containers 拡張のインストール

VS Code を開き、拡張機能（Ctrl+Shift+X）で `ms-vscode-remote.remote-containers` を検索してインストール。

またはコマンドラインから：
```bash
code --install-extension ms-vscode-remote.remote-containers
```

インストール後、左下に緑色の `><` アイコンが表示されれば準備完了です。

### 3. 5分で成功体験

練習用ディレクトリを作って最小構成を試してみましょう。

```bash
mkdir devcontainer-practice && cd devcontainer-practice
mkdir .devcontainer
```

`.devcontainer/devcontainer.json` を以下の内容で作成：

```json
{
  "name": "My First Devcontainer",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu"
}
```

VS Code でフォルダを開き、コマンドパレット（Ctrl+Shift+P）から：

```
Dev Containers: Open Folder in Container
```

しばらくすると、左下のアイコンが `>< Dev Container: My First Devcontainer` に変わり、
コンテナ内での開発が始まります。

### 4. コンテナの中にいることを確認する

ターミナル（Ctrl+`）を開いて：

```bash
uname -a       # Linux カーネルの情報が表示される
whoami         # vscode と表示される
echo $HOME     # /home/vscode
```

あなたの PC が Mac でも Windows でも、コンテナ内は Ubuntu Linux です。これが Devcontainer の核心です。

### 5. コンテナを閉じる・再び開く

| 操作 | 方法 |
|------|------|
| コンテナを閉じてローカルに戻る | `Dev Containers: Reopen Locally` |
| コンテナを再び開く | `Dev Containers: Reopen in Container` |
| VS Code ごと閉じる | 次回フォルダを開いたとき自動でコンテナが起動 |

### 6. 練習課題

1. `devcontainer-practice/` に `.devcontainer/devcontainer.json` を作り、コンテナを起動してください。
2. コンテナ内のターミナルで `cat /etc/os-release` を実行して Ubuntu のバージョンを確認してください。
3. `Dev Containers: Reopen Locally` でローカルに戻り、再度 `Dev Containers: Reopen in Container` でコンテナに入ってみてください。

---

## Chapter 2: devcontainer.json の基本

### 1. devcontainer.json の役割

`.devcontainer/devcontainer.json` は Devcontainer の心臓です。
**コンテナの詳細を一元管理できる設定の集積地**で、これをリポジトリに混ぜておけばチーム全員が同じ環境を使えます。

### 2. 主要プロパティ一覧

#### `image` — ベースイメージの指定

一番シンプルな方法です。Docker Hub や MCR（Microsoft Container Registry）のイメージをそのまま使います。

```json
{
  "name": "Node.js 開発環境",
  "image": "mcr.microsoft.com/devcontainers/node:20"
}
```

Microsoft の公式イメージは `mcr.microsoft.com/devcontainers/` 以下に横断な言語・ランタイムで展開されています。

| 使いたい環境 | image の一例 |
|------------|----------------|
| Ubuntu だけ | `mcr.microsoft.com/devcontainers/base:ubuntu` |
| Node.js | `mcr.microsoft.com/devcontainers/node:20` |
| Python | `mcr.microsoft.com/devcontainers/python:3.12` |
| Go | `mcr.microsoft.com/devcontainers/go:1.22` |
| Rust | `mcr.microsoft.com/devcontainers/rust:latest` |

#### `customizations.vscode.extensions` — 拡張機能の自動インストール

コンテナを起動するたびに指定の拡張機能が自動インストールされます。

```json
{
  "name": "Node.js 開発環境",
  "image": "mcr.microsoft.com/devcontainers/node:20",
  "customizations": {
    "vscode": {
      "extensions": [
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode",
        "eamodio.gitlens"
      ]
    }
  }
}
```

> **拡張機能 ID の調べ方**: VS Code の拡張機能パネルで拡張機能を右クリック → 「ID をコピー」で取得できます。

#### `customizations.vscode.settings` — VS Code 設定の上書き

コンテナ内に局所的な VS Code 設定を上書きできます。

```json
{
  "customizations": {
    "vscode": {
      "settings": {
        "editor.formatOnSave": true,
        "editor.defaultFormatter": "esbenp.prettier-vscode"
      }
    }
  }
}
```

#### `forwardPorts` — ポートフォワーディング

コンテナ内で起動したサーバーにホストからアクセスできるようにする設定です。

```json
{
  "forwardPorts": [3000, 5432]
}
```

これだけで `localhost:3000` と打つとコンテナ内のサーバーに届きます。

#### `postCreateCommand` — コンテナ作成後の初期化

コンテナが初めて作られたときに**1回だけ**実行されるコマンドです。npm install や pip install などの初期セットアップに使います。

```json
{
  "postCreateCommand": "npm install"
}
```

複数コマンドを走らせたい場合：

```json
{
  "postCreateCommand": "npm install && npm run build"
}
```

### 3. まとめて書いた例

```json
{
  "name": "Node.js 20 開発環境",
  "image": "mcr.microsoft.com/devcontainers/node:20",
  "forwardPorts": [3000],
  "postCreateCommand": "npm install",
  "customizations": {
    "vscode": {
      "extensions": [
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode"
      ],
      "settings": {
        "editor.formatOnSave": true
      }
    }
  }
}
```

### 4. よくあるアンチパターン

- **`postCreateCommand` で対話的コマンドを使う**: `sudo apt-get install -y vim` などは OK ですが、パスワードを問うコマンドは初期化をブロックします。
- **`forwardPorts` を書かずにポートを使わせる**: VS Code はコンテナ内のリッスンポートを自動検知しますが、明示的に書いたほうが確実です。

### 5. 練習課題

1. `devcontainer-practice/` の `devcontainer.json` に `customizations.vscode.extensions` を追加し、お気に入りの拡張機能を自動インストールさせてください。
2. `forwardPorts: [8080]` を追加してリビルド。コンテナ内で `python3 -m http.server 8080` を実行し、ブラウザから `localhost:8080` にアクセスしてください。
3. `postCreateCommand` に `echo 'Devcontainer 起動完了！' > /tmp/setup.txt` を設定し、リビルド後にそのファイルができていることを確認してください。

---

## Chapter 3: Dockerfile で環境をカスタマイズする

### 1. なぜ Dockerfile が必要になるのか

Chapter 2 の `image` 指定だけでは足りないケースが出てきます。

```
「Python のイメージに、mecab も必要なんだよね」
「デフォルトのイメージにはないインストール済み CLI ツールがある」
「会社の独自ライブラリをプリインストールしたイメージを使いたい」
```

こういった場合は、公式イメージをベースにした **Dockerfile** を書いて追加できます。

### 2. `build` で Dockerfile を指定する

`image` の代わりに `build` を使います。

#### ファイル構成

```
my-project/
├── .devcontainer/
│   ├── devcontainer.json
│   └── Dockerfile          ← 追加する
└── src/
```

#### `.devcontainer/Dockerfile`

```dockerfile
FROM mcr.microsoft.com/devcontainers/python:3.12

# システムパッケージの追加
RUN apt-get update && apt-get install -y \
    mecab \
    libmecab-dev \
    mecab-ipadic-utf8 \
    && rm -rf /var/lib/apt/lists/*

# Python パッケージのインストール
# 要件定義があれば requirements.txt 経由のほうが一般的
COPY requirements.txt /tmp/requirements.txt
RUN pip install -r /tmp/requirements.txt
```

#### `.devcontainer/devcontainer.json`

```json
{
  "name": "Python + MeCab 環境",
  "build": {
    "dockerfile": "Dockerfile"
  },
  "forwardPorts": [8000],
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-python.pylance"
      ]
    }
  }
}
```

> **`image` と `build` は両方指定できません**。どちらか一方です。

### 3. `build.context` でビルドコンテキストを指定する

Dockerfile とプロジェクトルートを別のディレクトリに置きたい場合や、
`COPY` でルートのファイルを持ち込みたい場合に使います。

```json
{
  "build": {
    "dockerfile": "Dockerfile",
    "context": ".."
  }
}
```

`context: ".."` にすると、`COPY requirements.txt /tmp/` のようにプロジェクトルートからファイルを持ち込めます。

### 4. Dockerfile 書き方のコツ

#### Docker レイヤーキャッシュを意識する

```dockerfile
# 悪い例：毎回全部再実行される
RUN apt-get update
RUN apt-get install -y vim
RUN apt-get install -y git

# 良い例：一度にまとめる（レイヤーが 1 つになりキャッシュが利く）
RUN apt-get update && apt-get install -y \
    vim \
    git \
    && rm -rf /var/lib/apt/lists/*
```

#### `FROM` は公式 MCR イメージから

```dockerfile
# 良い例：公式 devcontainers イメージをベースに
FROM mcr.microsoft.com/devcontainers/python:3.12

# 注意：素の Ubuntu や Alpine から始めると vscode ユーザーなどの設定を自分で行う必要がある
```

### 5. よくあるアンチパターン

- **`image` と `build` を両方書く**: どちらか一方にしてください。
- **Dockerfile を変更しても Rebuild しない**: Dockerfile を改変したら必ず「Dev Containers: Rebuild Container」が必要です。
- **ベースイメージのバージョンを `latest` にする**: チーム間でバージョンがずれるので、番号を固定する方が安全です。

### 6. 練習課題

1. `devcontainer-practice/` に `.devcontainer/Dockerfile` を作り、
   `FROM mcr.microsoft.com/devcontainers/base:ubuntu` に `curl` を `apt-get install` する層を追加してください。
   `devcontainer.json` を `build` に切り替えてリビルド。
2. コンテナ内のターミナルで `curl --version` を実行してインストールされたことを確認してください。
3. 「Dockerfile を変更→ Rebuild なし」と「Dockerfile を変更→ Rebuild あり」で振る舞いの違いを確認してください。

---

## Chapter 4: Features で部品を追加する

### 1. Features とは

**Dev Container Features** は、ツール・ランタイム・言語サポートを「部品」として宣言的に追加できる仕組みです。

Dockerfile で `apt-get install` や `curl | sh` を山のように書くことなく、
`devcontainer.json` の `features` ブロックに **宣言的に追加するだけ**。

```
「この環境に Node.js を入れて」→ features に追加
「この環境に GitHub CLI を入れて」→ features に追加
「この環境で Docker-in-Docker を使いたい」→ features に追加
```

### 2. features の書き方

```json
{
  "name": "Python + 追加ツール環境",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu",
  "features": {
    "ghcr.io/devcontainers/features/python:1": {
      "version": "3.12"
    },
    "ghcr.io/devcontainers/features/node:1": {
      "version": "20"
    },
    "ghcr.io/devcontainers/features/github-cli:1": {}
  }
}
```

**書き方のルール**:
- キー: Feature の OCI レジストリ URL
- 値: オプションのオブジェクト（オプションなしなら `{}` で OK）

### 3. 理解しておくべき公式 Features

| Feature ID | 内容 |
|-----------|------|
| `ghcr.io/devcontainers/features/python:1` | Python + pip |
| `ghcr.io/devcontainers/features/node:1` | Node.js + npm |
| `ghcr.io/devcontainers/features/go:1` | Go |
| `ghcr.io/devcontainers/features/rust:1` | Rust + cargo |
| `ghcr.io/devcontainers/features/github-cli:1` | GitHub CLI (`gh`) |
| `ghcr.io/devcontainers/features/docker-in-docker:2` | Docker-in-Docker |
| `ghcr.io/devcontainers/features/aws-cli:1` | AWS CLI |
| `ghcr.io/devcontainers/features/azure-cli:1` | Azure CLI |

→ 全リストは https://containers.dev/features で検索できます。

### 4. `image` と `features` の組み合わせ

Features は `image` とも `build` とも併用できます。
「Base イメージ + features で必要なものを追加」と「Dockerfile で独自ツール + features でその他のもの」の両方が成り立ちます。

```json
{
  "name": "Node.js + AWS CLI 環境",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu",
  "features": {
    "ghcr.io/devcontainers/features/node:1": { "version": "20" },
    "ghcr.io/devcontainers/features/aws-cli:1": {}
  },
  "customizations": {
    "vscode": {
      "extensions": ["ms-vscode.vscode-typescript-next"]
    }
  }
}
```

### 5. features vs Dockerfile の使い分け

| 状況 | 推奨 |
|------|---------|
| Node.js や Python など公式 Feature がある | `features` |
| `apt-get install` で展開する独自ツール | `Dockerfile` |
| 素のシェルスクリプトでインストールするツール | `features` または `Dockerfile` |
| 会社固有のライブラリ/バイナリ | `Dockerfile` |

### 6. よくあるアンチパターン

- **Feature のバージョンを指定しない**: `:1` と書いたところはメジャーバージョン固定です。パッチバージョンの大きな変更で環境が変わる可能性があるため、気になるなら `"version": "20"` 等でピン止めしましょう。
- **Feature を追加しても Rebuild しない**: `features` の変更は必ず Rebuild が必要です。

### 7. 練習課題

1. `devcontainer-practice/` の `devcontainer.json` に `features` を追加し、GitHub CLI (`gh`) をインストールしてください。リビルド後に `gh --version` で確認。
2. `python:1` feature を追加してリビルド。`python3 --version` でバージョンを確認してください。
3. features の `version` オプションを変えてリビルドし、別のバージョンが入ることを体感してください。

---

## Chapter 5: Docker Compose でデータベースも加える

### 1. コンテナが 1 つでは足りないケース

Chapter 2〜4 で作る環境は **アプリケーションコンテナが 1 つ**でした。
しかし実務では多くの場合、データベースやキャッシュも一緒に起動したいです。

```
「開発環境に PostgreSQL も起動しておきたい」
「アプリ + Redis + PostgreSQL の構成で開発したい」
「本番と同じミドルウェア構成ですぐできるのが理想」
```

これを解決するのが **Docker Compose** との組み合わせです。

### 2. ファイル構成

```
my-project/
├── .devcontainer/
│   ├── devcontainer.json
│   └── docker-compose.yml    ← 追加する
└── src/
```

### 3. docker-compose.yml を書く

```yaml
# .devcontainer/docker-compose.yml
services:
  app:
    image: mcr.microsoft.com/devcontainers/node:20
    volumes:
      - ..:/workspace:cached
    command: sleep infinity   # コンテナを起動したままにする

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: devpassword
      POSTGRES_USER: devuser
      POSTGRES_DB: devdb
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

**ポイント**:
- `app` サービスに `command: sleep infinity` を入れるのが定番です。VS Code はこのコンテナにアタッチします。
- `..:/workspace:cached` はプロジェクトルートを `/workspace` にマウント。
- `:cached` は Mac 環境でのパフォーマンス改善を期待するヒント。

### 4. devcontainer.json を Docker Compose 対応に変更する

```json
{
  "name": "Node.js + PostgreSQL 環境",
  "dockerComposeFile": "docker-compose.yml",
  "service": "app",
  "workspaceFolder": "/workspace",
  "forwardPorts": [3000, 5432],
  "postCreateCommand": "npm install",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-vscode.vscode-typescript-next",
        "mtxr.sqltools",
        "mtxr.sqltools-driver-pg"
      ]
    }
  }
}
```

**必須プロパティ**:

| プロパティ | 内容 |
|---------|------|
| `dockerComposeFile` | 参照する compose ファイル名 |
| `service` | VS Code がアタッチするコンテナ名 |
| `workspaceFolder` | VS Code が開くコンテナ内のパス |

### 5. コンテナ間の接続

`app` コンテナ内から PostgreSQL に接続するときのホスト名は `localhost` **ではありません**。
**サービス名**（`db`）がホスト名になります。

```env
# 接続文字列の例
DATABASE_URL=postgresql://devuser:devpassword@db:5432/devdb
#                                               ^サービス名
```

> **よくあるハマりポイント**: `localhost` で接続しようとして失敗するパターンが非常に多いです。
> Docker Compose のサービス名をホスト名として使うと覚えておいてください。

### 6. よくあるアンチパターン

- **`command: sleep infinity` を書かない**: `app` コンテナがすぐに退出し、VS Code のアタッチが失敗する。
- **`workspaceFolder` を書かない**: 開くディレクトリが決まらずコンテナのルートに落ちることがある。
- **サービス間接続で `localhost` を使う**: 接続が失敗する。サービス名を使うこと。

### 7. 練習課題

1. `devcontainer-practice/` に `.devcontainer/docker-compose.yml` と PostgreSQL サービスを追加し、`devcontainer.json` を `dockerComposeFile` 構成に切り替えてリビルドしてください。
2. `app` コンテナ内から `psql -h db -U devuser devdb` で接続を試みてください（`psql` がなければ features で入れる）。
3. 一度 VS Code を閉じて再起動した後、PostgreSQL のデータが残っているか確認してください（named volume の効果）。

---

## Chapter 6: ライフサイクルと Rebuild

### 1. ライフサイクルコマンドの全体像

Devcontainer は起動時のフェーズに応じて、指定のコマンドを自動実行できます。

```
initializeCommand
    ↓
  [Docker ビルド・起動]
    ↓
 onCreate (初回のみ)
    ↓
 postCreateCommand (初回のみ)
    ↓
 postStartCommand (起動ごと)
    ↓
 postAttachCommand (VS Code 接続ごと)
```

### 2. 各フェーズの役割

| コマンド | 実行タイミング | 実行場所 | 代表的用途 |
|----------|------------|------------|----------------|
| `initializeCommand` | Docker 起動前 | **ホスト** | `.env` ファイルの構築チェック |
| `onCreate` | コンテナ作成時（1回） | コンテナ | 初回からの重いセットアップ |
| `postCreateCommand` | コンテナ作成後（1回） | コンテナ | `npm install` / `pip install` |
| `postStartCommand` | **VS Code 起動ごと** | コンテナ | DB マイグレーション / バックグラウンドデーモン |
| `postAttachCommand` | **VS Code 接続ごと** | コンテナ | ウェルカムメッセージ表示 |

**使い分けの目安**:

- `postCreateCommand` → 初回起動時のセットアップ（パッケージインストール等）
- 自動起動が必要なデーモン系 → `postStartCommand`
- 接続アナウンス → `postAttachCommand`

### 3. postCreateCommand の実用例

#### 単一コマンド

```json
{
  "postCreateCommand": "npm install"
}
```

#### 複数コマンド

```json
{
  "postCreateCommand": "npm install && cp .env.example .env"
}
```

#### シェルスクリプトに委譲（複雑な場合）

```json
{
  "postCreateCommand": "bash .devcontainer/setup.sh"
}
```

### 4. Rebuild をするタイミング

以下の変更をしたときは **Rebuild が必要**です。Rebuild しないと変更が反映されません。

| 変更したもの | Rebuild 必要？ |
|------------|-------------|
| `devcontainer.json` の `image` / `build` / `features` | ✅ 必要 |
| `Dockerfile` の内容 | ✅ 必要 |
| `docker-compose.yml` の内容 | ✅ 必要 |
| `postCreateCommand` / `postStartCommand` | ⚠ 次回起動時に自動実行 |
| `customizations.vscode.extensions` | ⚠ 次回接続時に自動インストール |
| `forwardPorts` | ⚠ 次回起動時に反映 |

#### Rebuild の方法

```
コマンドパレット（Ctrl+Shift+P） → Dev Containers: Rebuild Container
```

コンテナを完全に再構築する場合（イメージキャッシュを破棄する）：

```
Dev Containers: Rebuild Container Without Cache
```

### 5. よくあるエラーと対処法

#### `postCreateCommand` が失敗する

```
エラー: 「postCreateCommand が退出コード 1 で失敗しました」

対処:
1. VS Code の「出力」ペインを開いてエラーの内容を確認
2. コンテナ内のターミナルで同じコマンドを手動実行して原因を特定
3. 修正したら Rebuild
```

#### コンテナ起動時に Docker エラーが出る

```
最初に診断すること:
1. Docker Desktop が起動しているか？
   → `docker ps` が通るか確認
2. イメージ名が間違っていないか？
   → `docker pull <image>` で手動ダウンロードして確認
3. ARM64 Mac で x86 専用イメージを使っていないか？
   → multi-arch イメージか、`platform: linux/amd64` を compose に指定
```

### 6. 練習課題

1. `devcontainer.json` に `postStartCommand` を追加し、VS Code を再起動するたびにターミナルに「コンテナ起動完了」と出るようにしてください。
2. `Dockerfile` を少し変更して、`Rebuild Container` と「コンテナを再起動」の差分を体験してください。
3. あえて `postCreateCommand` に存在しないコマンド（`false_command`）を書いて Rebuild し、エラーメッセージの読み方を練習してください。

---

## Chapter 7: チームで共有する・次の一歩

### 1. .devcontainer/ をリポジトリにコミットする

Devcontainer の核心は **設定ファイルをリポジトリに混ぜる**ことです。

```
.devcontainer/
├── devcontainer.json     ← git 管理する
├── Dockerfile            ← git 管理する
└── docker-compose.yml    ← git 管理する
```

この 3 ファイルをコミットするだけで、リポジトリを clone した人全員が VS Code で「Dev Containers: Reopen in Container」を実行するだけで全く同じ環境になります。

### 2. 環境変数の扱い方

API キーやパスワードは `devcontainer.json` に直書きしてはいけません。

#### 推奨パターン：`.env` ファイル + `runArgs`

```bash
# .devcontainer/.env（.gitignore に追加する）
DATABASE_PASSWORD=my_secret_password
AWS_ACCESS_KEY_ID=AKIA...
```

```json
{
  "runArgs": ["--env-file", ".devcontainer/.env"]
}
```

`.env.example` を git 管理してテンプレートとして活用する方法もよく使われます。

```json
{
  "postCreateCommand": "cp .devcontainer/.env.example .devcontainer/.env"
}
```

> **絶対に避ける**: 秘密鍵・トークン・接続文字列を `devcontainer.json` や `Dockerfile` に直書きして git push すること。

### 3. チームで使うときの管理決め

| 項目 | git 管理 | 理由 |
|------|-----------|------|
| `devcontainer.json` | ✅ する | チーム共通の設定 |
| `Dockerfile` | ✅ する | チーム共通の環境 |
| `docker-compose.yml` | ✅ する | チーム共通のサービス構成 |
| `.env` | ❌ しない | 認証情報・パスワード |
| `.env.example` | ✅ する | テンプレートとして |
| 個人メモ類 | ❌ しない | 個人のローカル設定 |

### 4. GitHub Codespaces への拡張

`devcontainer.json` は **GitHub Codespaces でもそのまま使えます**。
ローカルに Docker を入れず、ブラウザですぐ開発コンテナを起動できます。

```
GitHub のリポジトリページ → Code ボタン → Codespaces タブ → Create codespace
```

リポジトリに `.devcontainer/` があればその設定で Codespaces 環境が起動します。

### 5. よくあるアンチパターン

- **`.devcontainer/` を .gitignore に入れる**: 逆効果です。ファイルをコミットするのが正解です。
- **`devcontainer.json` に API キーを直書き**: `.env` 経由で渡すこと。
- **チームに共有せず口伝**: README に `devcontainer.json` を起動する手順をやっておくことで共有しましょう。

### 6. 次の一歩

ここまでで Devcontainer の基本は身につきました。実務に組み込んでいくには次のトピックがあります。

- **パフォーマンスの最適化**: Windows/Mac でのボリュームマウントの遅さ、WSL2 の活用
- **複数の Devcontainer 構成**: monorepo でサブフォルダごとに環境を分ける
- **CI との統合**: GitHub Actions で Devcontainer を使ったビルド
- **Devcontainer のデバッグ**: 起動時のスローなコンテナを高速化するテクニック

### 7. 練習課題

1. `devcontainer-practice/` の `.devcontainer/` を git リポジトリにコミットし、別の場所に clone して `Dev Containers: Reopen in Container` でコンテナが起動することを確認してください。
2. `.env.example` を作り、`postCreateCommand` で自動コピーされる仕組みを構築してください。
3. GitHub に push して Codespaces から開いてみてください（Codespaces が利用可能な場合）。

---

## 関連リソース

- **Dev Containers 公式**: https://containers.dev/
- **VS Code Devcontainer ドキュメント**: https://code.visualstudio.com/docs/devcontainers/containers
- **公式 Features 一覧**: https://containers.dev/features
- **devcontainer.json リファレンス**: https://containers.dev/implementors/json_reference/
- **Microsoft 公式 devcontainers イメージ**: https://mcr.microsoft.com/en-us/catalog?search=devcontainers

### 関連教材

- `devcontainer-best-practices.md` — パフォーマンス・CI 統合・チーム運用など中級以上向け（未作成）
