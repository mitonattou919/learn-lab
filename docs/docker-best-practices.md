# Docker ベストプラクティス

> 基本操作を習得済みの人向けの実務・運用設計教材。「動く」から「壊れない・速い・安全・チームで使える」へのステップアップを目指します。
> 概念をゼロから学びたい方 → `docker-overview.md`
> 手を動かして基礎を習得したい方 → `docker-tutorial.md`

---

## Chapter 0: Docker の中核制約とトレードオフ

### 1. Docker の本質的な制約

ベストプラクティスの大半は、以下の 3 つの制約から導かれます。

**① カーネル共有による分離の限界**
コンテナは VM と違いホスト OS のカーネルを共有します。隔離はプロセスレベルであり、カーネル脆弱性はすべてのコンテナに波及します。「コンテナ ≠ セキュリティバウンダリ」という認識が運用設計の出発点です。

**② イメージの不変性とデータの揮発性**
コンテナのファイルシステムはコンテナ削除と同時に消えます。「状態をどこに持たせるか」を設計段階で決めないと、本番データが消えます。

**③ ビルドの再現性とキャッシュの両立**
Dockerfile はビルドを再現可能にしますが、`latest` タグや外部からのダウンロードはその再現性を壊します。「速さと確実さのトレードオフ」を意識した Dockerfile 設計が必要です。

### 2. 主要なトレードオフ一覧

| 観点 | 手軽さ重視 | 堅牢性重視 | 推奨 |
|------|-----------|-----------|------|
| ベースイメージ | `ubuntu:latest` | `ubuntu:22.04` | **バージョン固定** |
| イメージサイズ | フルイメージ | Alpine / scratch | **用途に応じて選択** |
| ユーザー権限 | root で実行 | 非 root ユーザー | **非 root** |
| シークレット管理 | ENV に直書き | Docker Secrets / 外部注入 | **外部注入** |
| ネットワーク | デフォルト bridge | ユーザー定義ネットワーク | **ユーザー定義** |
| データ永続化 | bind mount | Volume | **Volume（本番）** |

### 3. ⭐ 最重要原則：「イミュータブルインフラ」

Docker の価値は「**動いているコンテナを変更しない**」という設計思想にあります。コンテナを `docker exec` で直接変更するのは、VM でファイルを直接編集するのと同じアンチパターンです。

```
✅ 正しい運用フロー
Dockerfile 変更 → docker build → docker push → docker pull → docker run（新コンテナ）

❌ アンチパターン
docker exec で設定ファイルを直接編集
```

この原則を守ることで、環境差異・デプロイ事故・「このコンテナだけ特殊な状態」が消えます。

---

## Chapter 1: 環境構築と前提

### 1. Docker のバージョン管理

チームメンバーの Docker バージョンを揃えないと、動作の差異が生まれます。

```bash
# バージョン確認
docker --version
docker compose version

# Linux: 特定バージョンをインストール
sudo apt-get install docker-ce=5:26.1.4-1~ubuntu.22.04~jammy
```

**推奨**: `docker-compose.yml` に `version` を明示し、README に「必要な Docker バージョン: 26.x 以上」と記載する。

### 2. プロジェクトの標準ファイル構成

```
project/
├── Dockerfile                 # 本番用
├── Dockerfile.dev             # 開発用（デバッグツール込み）
├── .dockerignore              # ビルドに含めないファイルを指定
├── compose.yaml               # 開発環境
├── compose.override.yaml      # ローカル個人設定（.gitignore 推奨）
├── compose.prod.yaml          # 本番用差分
└── .env.example               # 環境変数のサンプル（実値は .env に、git 管理外）
```

### 3. .dockerignore は必ず作る

`.dockerignore` がないと `COPY . .` でビルドコンテキスト（Docker Daemon に送るファイル群）が肥大化し、ビルドが遅くなります。また `.env` や秘密鍵がイメージに混入するリスクがあります。

```
# .dockerignore の基本テンプレート
node_modules/
.git/
.env
.env.*
*.log
dist/
build/
coverage/
.DS_Store
README.md
```

| ないと起きる問題 | 対処 |
|---------------|------|
| `node_modules`（数百 MB）がビルドコンテキストに含まれる | `.dockerignore` に追加 |
| `.env` がイメージに混入してシークレットが漏洩 | `.dockerignore` に追加 |
| `COPY . .` のたびにキャッシュが無効化される | 変化しないファイルを除外 |

---

## Chapter 2: イメージ設計原則

### 1. ⭐ マルチステージビルドを使う

マルチステージビルドは、ビルド用の依存（コンパイラ・テストツール）を最終イメージに含めないための仕組みです。イメージサイズを劇的に削減でき、攻撃対象面も減ります。

```dockerfile
# ステージ 1: ビルド
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci                    # npm install より再現性が高い
COPY . .
RUN npm run build             # dist/ を生成

# ステージ 2: 本番実行（ビルドツールを含まない）
FROM node:20-alpine AS runner
WORKDIR /app

# 非 root ユーザーを作成（Chapter 3 で詳述）
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./
RUN npm ci --omit=dev         # devDependencies は含めない

USER appuser
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

| 比較 | シングルステージ | マルチステージ |
|------|---------------|--------------|
| イメージサイズ | 数百 MB（ビルドツール込み） | 数十 MB（実行に必要なものだけ） |
| セキュリティ | コンパイラ等の脆弱性を引き継ぐ | 攻撃対象が最小 |
| ビルド複雑性 | シンプル | やや複雑だが自動化で吸収 |

### 2. ベースイメージの選び方

| イメージ系統 | 特徴 | 推奨場面 |
|------------|------|---------|
| `node:20` | Debian ベース・フル装備 | 開発環境・互換性優先 |
| `node:20-slim` | 不要なパッケージを省いた Debian | バランス重視 |
| `node:20-alpine` | Alpine Linux ベース（超軽量） | 本番・サイズ最優先 |
| `distroless` | シェルすら含まない最小イメージ | セキュリティ最優先 |
| `scratch` | 空のイメージ（Go などの静的バイナリ向け） | 静的リンク済みバイナリ |

> ⚠ **Alpine の注意点**: Alpine は `musl libc` を使うため、`glibc` 依存のネイティブモジュール（特定の npm パッケージなど）でエラーになることがあります。互換性トラブルが出たら `-slim` に切り替えてください。

### 3. `npm ci` vs `npm install`

| コマンド | 挙動 | 推奨場面 |
|---------|------|---------|
| `npm install` | `package-lock.json` がなければ生成・更新する | ローカル開発 |
| `npm ci` | `package-lock.json` と完全一致を要求・高速 | Dockerfile・CI（再現性優先） |

Dockerfile では必ず `npm ci` を使ってください。

---

## Chapter 3: Dockerfile 最適化

### 1. レイヤーキャッシュ戦略

Docker はレイヤーを上から評価し、**変更があった最初のレイヤー以降はすべてキャッシュが無効**になります。「変わりにくいものを下に」が鉄則です。

```dockerfile
# ❌ 悪い例：コードを先にコピーするとソース変更のたびに npm ci が走る
COPY . .
RUN npm ci

# ✅ 良い例：依存定義だけ先にコピーしてキャッシュを活かす
COPY package*.json ./
RUN npm ci
COPY . .
```

**キャッシュを意識した命令順序の原則:**

```
1. ベースイメージ（最も変わらない）
2. システムパッケージのインストール
3. 依存関係の定義ファイル（package.json など）
4. 依存関係のインストール（npm ci など）
5. アプリコード（最も変わりやすい）
6. メタデータ（EXPOSE, CMD など）
```

### 2. 非 root ユーザーで実行する

コンテナがデフォルトの `root` で動いていると、コンテナ脱出の脆弱性を突かれた際にホストへのダメージが大きくなります。

```dockerfile
# Alpine の場合
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Debian/Ubuntu の場合
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser
```

```bash
# 実行確認
docker run --rm myapp whoami
# appuser（root ではないことを確認）
```

### 3. RUN 命令はまとめる

各 `RUN` 命令が 1 レイヤーを作ります。無駄にレイヤーを増やすとイメージサイズが膨らみます。

```dockerfile
# ❌ 悪い例：不要なファイルが残り、3レイヤー生成
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ✅ 良い例：1レイヤーでインストールとクリーンアップをまとめる
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/*
```

### 4. COPY と ADD の使い分け

| 命令 | 追加機能 | 推奨場面 |
|-----|---------|---------|
| `COPY` | なし | ローカルファイルのコピー（基本これを使う） |
| `ADD` | URL からのダウンロード・tar 自動展開 | tar ファイルの展開が必要なときだけ |

`ADD` を URL ダウンロードに使うとキャッシュが効きにくくなります。URL からのダウンロードは `RUN curl` か `RUN wget` を使い、`COPY` でローカルファイルをコピーする方針が明確です。

---

## Chapter 4: データ設計（Volume と永続化戦略）

### 1. 本番での Volume 設計原則

| 原則 | 内容 |
|------|------|
| **データとコンテナを分離する** | DB データは必ず Volume に。コンテナ削除でデータが消えない構成に |
| **名前付き Volume を使う** | パス依存のない名前付き Volume（`-v pgdata:/var/lib/...`）が移植性高い |
| **Bind Mount は開発限定** | 本番では使わない。ホストパスへの依存はポータビリティを壊す |
| **Volume は用途ごとに分ける** | DB データ・ログ・アップロードファイルは別 Volume に |

```yaml
# compose.yaml での Volume 設計例
services:
  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data   # データ永続化
      - pglog:/var/log/postgresql          # ログ分離

  app:
    image: myapp:1.0
    volumes:
      - uploads:/app/uploads              # アップロードファイル

volumes:
  pgdata:
  pglog:
  uploads:
```

### 2. バックアップ戦略

Volume の中身は `docker run` でダンプ・リストアできます。

```bash
# PostgreSQL のバックアップ
docker exec postgres pg_dump -U postgres mydb > backup.sql

# Volume ごとバックアップ（汎用）
docker run --rm \
  -v pgdata:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/pgdata-$(date +%Y%m%d).tar.gz -C /data .

# リストア
docker run --rm \
  -v pgdata:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/pgdata-20240101.tar.gz -C /data
```

### 3. tmpfs でメモリ上の一時領域を使う

秘密情報やセッションデータなど「再起動後に残してはいけないデータ」には `tmpfs` が有効です。

```bash
docker run --tmpfs /run/secrets:rw,noexec,nosuid,size=100m myapp
```

---

## Chapter 5: ネットワーク・シークレット管理

### 1. ネットワーク分離設計

すべてのコンテナを同一ネットワークに置くのは、内部サービスが意図せず外部に露出するリスクがあります。フロントエンド・バックエンド・DBを別ネットワークに分離する設計が推奨です。

```yaml
# ネットワーク分離の例
services:
  nginx:
    networks:
      - frontend    # インターネットと接続

  app:
    networks:
      - frontend    # nginx と通信
      - backend     # db と通信

  db:
    networks:
      - backend     # app のみと通信（外部非公開）

networks:
  frontend:
  backend:
    internal: true  # ← 外部インターネットへの通信を遮断
```

| 問題 | 対処 |
|------|------|
| DB がインターネットに公開される | `internal: true` ネットワークに配置 |
| 全コンテナが同一ネットワークで相互通信できる | 用途別ネットワークで最小権限に |

### 2. ⭐ シークレット管理：ENV に書いてはいけない

`ENV` や `ARG` に秘密情報を書くと、イメージ履歴（`docker history`）に平文で残ります。

```dockerfile
# ❌ 絶対にやってはいけない
ENV DATABASE_PASSWORD=secret123
ARG API_KEY=sk-xxxxxxx
```

**正しいシークレット管理の方法:**

| 方法 | 概要 | 向いている場面 |
|------|------|-------------|
| **実行時 ENV 注入** | `docker run -e DB_PASS=$DB_PASS` で渡す | シンプルな構成 |
| **`.env` ファイル** | Compose の `env_file:` で読み込む（git 管理外） | 開発環境 |
| **Docker Secrets** | Swarm モードで使える暗号化シークレット | Docker Swarm 本番 |
| **外部シークレット管理** | AWS SSM / Vault / Kubernetes Secret | クラウド本番環境 |

```yaml
# compose.yaml での .env ファイル読み込み
services:
  app:
    image: myapp:1.0
    env_file:
      - .env          # git 管理外の .env から読み込む
```

```bash
# .gitignore
.env
.env.*
!.env.example    # サンプルだけ管理
```

---

## Chapter 6: Compose 運用（環境別設定・healthcheck）

### 1. 環境別 Compose の設計

開発・ステージング・本番で設定を変えるには `compose.override.yaml` と複数ファイルの `-f` 合成を使います。

```
compose.yaml           ← 共通設定（全環境共通）
compose.override.yaml  ← ローカル開発用（自動適用・git 管理外）
compose.prod.yaml      ← 本番差分
```

```yaml
# compose.yaml（共通）
services:
  app:
    image: myapp:${APP_VERSION:-latest}
    environment:
      NODE_ENV: production

  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
```

```yaml
# compose.override.yaml（開発時に自動適用）
services:
  app:
    build: .            # 開発時はローカルビルド
    volumes:
      - .:/app          # ソースコード Bind Mount
    environment:
      NODE_ENV: development
    ports:
      - "3000:3000"

  db:
    ports:
      - "5432:5432"     # 開発時だけ DB を外部公開
```

```bash
# 本番デプロイ
docker compose -f compose.yaml -f compose.prod.yaml up -d
```

### 2. healthcheck を必ず設定する

`depends_on` だけでは「コンテナが起動した」ことしか保証できず、DB が受け付け準備できていない状態でアプリが接続しようとしてエラーになります。

```yaml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s   # 起動直後のチェック猶予期間

  app:
    image: myapp:1.0
    depends_on:
      db:
        condition: service_healthy   # ← healthcheck が通るまで待つ
```

### 3. リソース制限を設定する

設定なしだとコンテナがホストリソースを食い尽くすリスクがあります。

```yaml
services:
  app:
    image: myapp:1.0
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 128M
```

---

## Chapter 7: CI/CD 統合

### 1. イメージタグ戦略

| タグ戦略 | 例 | 目的 |
|---------|---|------|
| セマンティックバージョン | `myapp:1.2.3` | リリース管理・ロールバック |
| Git コミットハッシュ | `myapp:a3f5c9d` | デプロイの完全な追跡性 |
| ブランチ名 | `myapp:main` | ブランチの最新ビルドを示す |
| `latest` | `myapp:latest` | **本番では使わない**（再現性がない） |

**推奨**: `コミットハッシュ` タグで管理し、`latest` は CI での動作確認用にとどめる。

```bash
# GitHub Actions でのタグ付け例
TAG=$(git rev-parse --short HEAD)
docker build -t myapp:${TAG} .
docker tag myapp:${TAG} myapp:latest
docker push myapp:${TAG}
docker push myapp:latest
```

### 2. GitHub Actions でのビルド・プッシュ

```yaml
# .github/workflows/docker-build.yml
name: Docker Build & Push

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}    # PAT を使う（パスワードは使わない）

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            myorg/myapp:${{ github.sha }}
            myorg/myapp:latest
          cache-from: type=gha    # GitHub Actions キャッシュを活用
          cache-to: type=gha,mode=max
```

### 3. イメージの脆弱性スキャン

本番にデプロイする前にイメージのセキュリティスキャンを CI に組み込みます。

```yaml
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myorg/myapp:${{ github.sha }}
          format: table
          exit-code: 1             # 高・重大な脆弱性があれば CI を失敗させる
          severity: HIGH,CRITICAL
```

> **Docker Scout**: Docker 公式の脆弱性スキャンツール。`docker scout cves myapp:latest` でローカルでも確認できます。

---

## Chapter 8: よくある失敗パターン集

### 1. イメージ関連

| パターン | 何が起きるか | 対策 |
|---------|------------|------|
| `FROM ubuntu:latest` を使う | ビルドごとに異なるベースになり再現性が壊れる | バージョン固定（`ubuntu:22.04`） |
| `node_modules` を `.dockerignore` に入れない | ホストの `node_modules` がコンテナに混入し動作が変わる | 必ず `.dockerignore` に追加 |
| 巨大イメージをそのままデプロイ | push/pull に時間がかかり、デプロイが遅い | マルチステージビルド・Alpine 検討 |
| `COPY . .` を `RUN npm ci` より前に書く | ソース変更のたびに `npm ci` が走りビルドが遅い | 依存定義ファイルを先に COPY |

### 2. セキュリティ関連

| パターン | 何が起きるか | 対策 |
|---------|------------|------|
| `ENV DATABASE_PASSWORD=secret` を Dockerfile に書く | `docker history` で平文が見える | 実行時に `-e` または `env_file` で注入 |
| root で実行 | コンテナ脱出時の被害が最大になる | 非 root ユーザーを作成して `USER` を指定 |
| `--privileged` フラグを安易に使う | ホスト OS のほぼすべてにアクセスできる危険な状態 | 必要な capability だけ `--cap-add` で付与 |
| 古いベースイメージを放置 | 既知の CVE を抱えたまま本番稼働 | 定期的なイメージ再ビルド・スキャン導入 |

### 3. 運用関連

| パターン | 何が起きるか | 対策 |
|---------|------------|------|
| `docker exec` でコンテナを直接変更 | 「このコンテナだけ特殊な状態」になり再現不可 | Dockerfile を変更して再ビルド |
| `depends_on` だけで依存管理 | DB 起動前にアプリが接続してクラッシュ | `healthcheck` + `condition: service_healthy` |
| Volume なしで DB を動かす | コンテナ再起動でデータが消える | 必ず名前付き Volume を設定 |
| ログをコンテナ内ファイルに書く | コンテナ削除でログが消え、障害調査が不可能 | stdout/stderr に出力し Docker のログドライバに任せる |
| コンテナに `latest` タグで本番デプロイ | どのコードがデプロイされているか追跡不能 | コミットハッシュやバージョンタグを使う |

### 4. パフォーマンス関連

| パターン | 何が起きるか | 対策 |
|---------|------------|------|
| ビルドキャッシュを活かしていない | 毎回フルビルドで CI が遅い | 依存定義ファイルを先に COPY する順序最適化 |
| イメージを定期的に `prune` しない | ディスクが満杯になる | `docker system prune` を定期実行（CI 後など） |
| 大きなビルドコンテキスト | `docker build` の開始が遅い | `.dockerignore` を整備 |

---

## Chapter 9: 直感を育てる

### 1. 判断の問いかけ

設計に迷ったとき、以下を自問してください。

**「このコンテナが突然消えても問題ないか？」**
→ NO なら Volume やデータ設計を再確認。コンテナはいつでも消えうる前提で設計する。

**「このイメージを3ヶ月後にビルドして同じものが出てくるか？」**
→ NO なら `latest` タグや外部 URL からのダウンロードを見直す。再現性が信頼の根拠。

**「この設定はコードで管理されているか？」**
→ NO（手作業の変更が混じっている）なら Dockerfile や compose.yaml に落とし込む。

**「root でなければならない理由があるか？」**
→ 大抵の場合 NO。非 root ユーザーで動かす原則を守る。

### 2. 成長のロードマップ

| レベル | できること |
|-------|----------|
| **入門** | `docker run`・基本イメージの利用・Compose で開発環境 |
| **中級**（本教材の対象） | マルチステージビルド・Volume 設計・環境別 Compose・CI 統合 |
| **上級** | Kubernetes 移行・サービスメッシュ・GitOps・ゼロダウンタイムデプロイ |
| **専門** | コンテナランタイムの内部構造・カスタム CNI・セキュリティ強化（seccomp/AppArmor） |

### 3. 「うまくいっている」サイン

- Dockerfile を変更せずに本番デプロイが失敗したことがない
- 新メンバーが `docker compose up` だけで開発環境を立ち上げられる
- コンテナ内を `docker exec` で直接変更する必要が生じない
- イメージサイズとビルド時間が改善を続けている
- CI でセキュリティスキャンが自動実行されている

---

## 関連リソース

- [Docker 公式ベストプラクティス](https://docs.docker.com/develop/dev-best-practices/)
- [Dockerfile ベストプラクティス](https://docs.docker.com/build/building/best-practices/)
- [Docker セキュリティ](https://docs.docker.com/engine/security/)
- [Trivy（脆弱性スキャナー）](https://github.com/aquasecurity/trivy)
- [Docker Scout](https://docs.docker.com/scout/)
- 概念を俯瞰したい → `docker-overview.md`
- 基礎を手を動かして学びたい → `docker-tutorial.md`
