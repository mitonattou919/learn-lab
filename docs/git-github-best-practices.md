# Git と GitHub ベストプラクティス

> 基本操作（add / commit / push / PR）は習得済みの方向け。「使える」から「運用できる」レベルへの引き上げを目指します。
> Git と GitHub を初めて触る方 → `git-github-tutorial.md` を先に
> 全体像を俯瞰したい方 → `git-github-overview.md` を先に

---

## Chapter 0: Git / GitHub とは

> この教材の前提：`git-github-tutorial.md` の基本操作（add / commit / push / PR）は習得済みとします。
> ここでは「使える」から「運用できる」レベルへの引き上げを目指します。

### 1. 中核となる制約：分散型の本質

Git が「分散型」である意味を実務レベルで理解すると、多くの操作判断が一貫して見えてきます。

```
全員がリポジトリの「フルコピー」を持っている
        ↓
コミットは「各自のコピー上で独立して存在する」
        ↓
push で他者のコピーと同期する
        ↓
「共有済み = 変更してはいけない」という黄金律が生まれる
```

| 観点 | 集中型 (SVN) | 分散型 (Git) |
|------|------------|------------|
| 履歴の保存場所 | サーバーのみ | 全員のマシン |
| オフライン作業 | 不可 | 可 |
| ブランチコスト | 重い | 極めて軽い |
| 履歴書き換えリスク | 低（サーバー管理） | **高（要注意）** |

### 2. ベストプラクティスの根拠：3 つのトレードオフ

best-practices が存在する理由は「自由度が高い = 間違えやすい」からです。

| 自由度 | リスク | ベストプラクティス |
|-------|--------|------------------|
| ローカルで歴史改変できる | push 済みに適用すると他者の作業が破壊される | **push 前のみ rebase** |
| ブランチを自由に作れる | 戦略がないと管理不能になる | **ブランチ命名規約・戦略の選択** |
| コミットメッセージは自由 | 履歴が読めなくなる | **1コミット・1変更・意味あるメッセージ** |

### 3. この教材で扱うトピック

```
コミット品質         → Chapter 2
ブランチ戦略         → Chapter 3
rebase と merge      → Chapter 4
GitHub 活用          → Chapter 5
自動化               → Chapter 6
失敗パターン         → Chapter 7
チーム運用           → Chapter 8
```

→ Chapter 1 の設定最適化から始めて、各層の原則を積み上げていきます。

---

## Chapter 1: 環境・設定の最適化

### 1. .gitconfig を整える

インストール後は最低限の設定だけしている人が多いです。実務で役立つ設定を `~/.gitconfig` に追加しましょう。

```ini
[user]
    name  = 自分の名前
    email = github に登録したアドレス

[core]
    editor   = vim           # デフォルトエディタ
    autocrlf = input         # macOS/Linux: 改行コード正規化

[pull]
    rebase = true            # git pull をリベース型に（履歴が線形になる）

[push]
    default = current        # 同名のリモートブランチに自動プッシュ

[init]
    defaultBranch = main
```

### 2. よく使うエイリアス

長いコマンドを短縮してリズムを作ります。

```ini
[alias]
    st  = status
    co  = switch
    br  = branch
    lg  = log --oneline --graph --decorate --all
    pu  = push
    pl  = pull
```

使い方：

```bash
git lg     # ブランチツリー付きの履歴表示
git st     # git status の短縮
```

### 3. 認証：HTTPS vs SSH

| 方式 | メリット | デメリット |
|------|---------|----------|
| **HTTPS + PAT** | 設定が簡単、企業プロキシを通る | トークン管理が必要 |
| **SSH 鍵** | パスワード不要で快適 | 初期設定にひと手間 |

SSH 鍵を登録する手順（推奨）：

```bash
# 1. 鍵を生成（パスフレーズあり推奨）
ssh-keygen -t ed25519 -C "your_email@example.com"

# 2. 公開鍵を表示して GitHub に登録
cat ~/.ssh/id_ed25519.pub
# → GitHub > Settings > SSH and GPG keys > New SSH key に貼り付け

# 3. 接続確認
ssh -T git@github.com
# Hi <username>! と返れば OK
```

> **Personal Access Token (PAT)** を使う場合は `repo` スコープのみに絞り、有効期限を設定すること。無期限の全権トークンは最小権限違反です。

### 4. グローバル .gitignore

OS・エディタ固有のファイルはプロジェクトの `.gitignore` に書かず、個人のグローバル設定で除外します。

```bash
git config --global core.excludesfile ~/.gitignore_global
```

```bash
# ~/.gitignore_global の例
.DS_Store
Thumbs.db
.idea/
.vscode/
*.swp
```

→ チームの `.gitignore` には「プロジェクト固有のもの」だけを残せます。

### 5. 現在の設定を確認・リセットする

```bash
git config --list --show-origin   # どのファイルで設定されているか確認
git config --global --unset <key> # 特定の設定を削除
```

---

## Chapter 2: コミット設計の原則

### 1. コミットは「最小の論理単位」で切る

「1コミット = 1論理変更」が基本原則です。これが守られると：

- 特定の変更だけを `git revert` で取り消せる
- `git bisect` でバグ混入コミットを二分探索で特定できる
- コードレビューで変更の意図が一目でわかる

```
❌ やりがちな悪い例
[ユーザー登録機能を追加] というコミット1つに
  → APIエンドポイント追加
  → バリデーション実装
  → メール送信機能
  → UIフォーム
  → 無関係なタイポ修正
  がすべて混入している

✅ 正しい切り方
[feat: ユーザー登録APIエンドポイントを追加]
[feat: ユーザー登録フォームのバリデーションを実装]
[feat: 登録確認メールの送信を追加]
[fix: ログインページのタイポを修正]
```

複数ファイルを触ったときに「関係ないもの」が混入していたら：

```bash
git add --patch          # ファイルの一部だけをステージ
# e: 手動編集 / s: ハンクを分割 / y/n: ハンクごとに選択
```

### 2. コミットメッセージの書き方

**Pro Git 推奨フォーマット**：

```
件名（50文字以内・命令形・末尾にピリオドなし）

本文（任意、72文字折り返し）：
- 何をしたかではなく「なぜそうしたか」を書く
- 以前の挙動との差分を説明する

フッター（任意）：
Fixes #123
```

| 悪い例 | 良い例 |
|--------|--------|
| `fix` | `Fix: ログイン時に空パスワードで通過するバグを修正` |
| `update api` | `Refactor: fetchUser を NotFoundError を投げる設計に変更` |
| `wip` | `WIP: OAuth コールバック処理を実装中（動作未確認）` |
| `add test` | `Test: fetchUser の存在しないユーザーケースを追加` |

> **「なぜ」を書く実践**：`git log --oneline` を 6 ヵ月後の自分が見たとき、変更の意図がわかるかを基準に。

### 3. Conventional Commits（推奨）

チームで統一しやすい型付きプレフィックスの規約です（`conventionalcommits.org`）。

| プレフィックス | 用途 | 例 |
|------------|------|-----|
| `feat:` | 新機能 | `feat: ダッシュボードにグラフを追加` |
| `fix:` | バグ修正 | `fix: 日本語入力時の文字化けを修正` |
| `docs:` | ドキュメント | `docs: README にインストール手順を追記` |
| `refactor:` | リファクタリング | `refactor: getUserById をクラスメソッドに整理` |
| `test:` | テスト追加・修正 | `test: ユーザー登録APIのE2Eテストを追加` |
| `chore:` | ビルド・設定変更 | `chore: ESLint を v9 にアップグレード` |
| `ci:` | CI/CD 関連 | `ci: main マージ時のデプロイを自動化` |

**破壊的変更**は本文か件名末尾に `BREAKING CHANGE:` を付記。

### 4. コミット前チェック習慣

```bash
git diff --staged            # add 済みの変更を目視確認
git diff --check             # 余分な空白を検出
git log --oneline -5         # 直近コミットと粒度を比較
```

コミット前に「このコミットを一文で説明できるか？」を問う。説明に「and」が入るなら分割を検討。

---

## Chapter 3: ブランチ戦略

### 1. 戦略を選ぶ前に問う 3 つの質問

1. **デプロイ頻度はどのくらいか**（1日複数回 vs リリース週次・月次）
2. **並行する機能開発の数はどのくらいか**（1〜2本 vs 10本以上）
3. **長期間マージできないブランチが発生するか**（フィーチャーフラグなしの大規模機能開発）

答えによって「どの戦略が合うか」が変わります。

### 2. 3 つの主要戦略の比較

| 観点 | GitHub Flow | Git Flow | Trunk-based |
|------|------------|---------|-------------|
| **main の状態** | 常にデプロイ可 | 安定（タグ付きリリース） | 常にデプロイ可 |
| **ブランチ数** | 少（feature + main） | 多（main/develop/feature/release/hotfix） | 最少（feature + main のみ） |
| **デプロイ頻度** | 高〜中 | 中〜低 | 非常に高（CD 前提） |
| **向いているチーム** | 小〜中規模 Web | 複数バージョン維持・大規模 | DevOps 成熟チーム |
| **ブランチ存在期間** | 短命（数日） | 長命あり（develop が永続） | 非常に短命（数時間〜1日） |

### 3. GitHub Flow（推奨スタート地点）

ほとんどのチームはここから始めると失敗が少ないです。

```
1. main から feature/xxx ブランチを作成
2. コミットを積む（小さく・頻繁に）
3. push して PR を開く（早めの Draft PR も有効）
4. レビュー → 修正 → 承認
5. main にマージ
6. ブランチを削除
（→ デプロイ）
```

**ブランチ命名規約の例**：

| プレフィックス | 用途 | 例 |
|------------|------|-----|
| `feature/` | 新機能 | `feature/user-login` |
| `fix/` | バグ修正 | `fix/null-pointer-on-logout` |
| `docs/` | ドキュメント | `docs/api-reference` |
| `chore/` | 保守・設定 | `chore/upgrade-node-v20` |
| `hotfix/` | 緊急修正 | `hotfix/critical-auth-bypass` |

> **ブランチ名にスラッシュを使うと** GitHub 上でフォルダ的にグループ化されて見通しが良くなります。

### 4. Git Flow（複数バージョンを維持する場合）

`main`（安定） / `develop`（統合） / `feature/*`（開発） / `release/*`（リリース準備） / `hotfix/*`（緊急修正）の5種類のブランチを使う戦略です。

```
          hotfix/*
         ↗        ↘
main ←←←←←←←←←←←← release/*
  ↕                      ↑
develop ← feature/A      |
develop ← feature/B ─────┘
```

**使い時の目安**：

- 本番・ステージング・開発の 3 環境が並立する
- パッケージ（npm, App Store）のバージョン管理が必要
- リリースノートを明示的に管理したい

### 5. Trunk-based Development（CI/CD を最大活用）

全員が `main`（trunk）に直接コミット、またはごく短命のブランチ（数時間〜1日）を使う戦略です。Feature Flag で未完成の機能を本番に混在させるのが前提。

**採用の前提条件**：

- CI が常時グリーンを維持できる（テストが速く・安定している）
- Feature Flag の基盤がある
- コードレビューが PR 以外の方法でも回る

### 6. よくあるアンチパターン

- **ゾンビブランチ**: マージもされず削除もされないブランチが何週間も放置される → 定期的に `git branch --merged` で棚卸し
- **策略なしの `main` 直 push**: 緊急だと言い訳しながら習慣化する → ブランチ保護で強制
- **ブランチ名が「fix2」「test-tmp」**: 何の作業か数日後にわからなくなる → 命名規約をチームで合意

---

## Chapter 4: rebase vs merge の使い分け

### 1. 黄金律（最重要）

> **「他の人がベースにしている可能性があるコミットは rebase しない」**

これだけ守れば、rebase によるチームへの被害は防げます。

| 状況 | 許可 |
|------|------|
| ローカルのみのコミット | ✅ rebase OK |
| push 済み・自分しか使っていないブランチ | ⚠ 慎重に（force push が必要） |
| チームが参照している共有ブランチ | ❌ rebase 禁止 |

### 2. rebase と merge の違い

```
元の状態:
  A --- B --- C  (main)
         \
          D --- E  (feature/xxx)

merge の結果:
  A --- B --- C --- M  (main、M はマージコミット)
         \        /
          D --- E

rebase の結果:
  A --- B --- C --- D' --- E'  (線形、D・E は再作成)
```

| 観点 | merge | rebase |
|------|-------|--------|
| 履歴の形 | 並行の事実を保存（マージコミットあり） | 線形・クリーン |
| 衝突対応 | 一度（マージ時） | コミットごとに発生することがある |
| 安全性 | 常に安全 | ローカルのみ安全 |
| 使い時 | 共有ブランチの統合 | PR 作成前のローカル整理 |

### 3. 実践パターン：PR 前の整理に rebase を使う

```bash
# main の最新を取り込みながら自分のコミットを線形に保つ
git fetch origin
git rebase origin/main

# コンフリクトが発生した場合
# → 解消 → git add → git rebase --continue
# → 中止する場合は git rebase --abort

# ローカルコミットをまとめる（スカッシュ）
git rebase -i HEAD~3    # 直近 3 コミットを対象に
# pick → squash に変えて複数コミットを1つに
```

> **`git pull --rebase`** を `~/.gitconfig` に設定しておくと、`git pull` が自動的にリベース型になり、無駄なマージコミットが発生しなくなります。

### 4. スカッシュマージ（GitHub の `Squash and merge`）

PR のコミット履歴をすべて1つに圧縮して main にマージする方法。

```
feature ブランチの履歴:
  feat: APIエンドポイント追加
  fix: タイポ修正
  wip: 途中保存
  chore: lint エラー対応
         ↓ Squash and merge
main の履歴:
  feat: ユーザー登録機能を追加 (#42)
```

**使い時の判断**：

| マージ方法 | 特徴 | 向いている場面 |
|----------|------|-------------|
| Merge commit | 全コミットを保存（マージコミットあり） | 詳細な開発履歴を残したい |
| Squash and merge | PR = 1コミット（履歴がシンプル） | コミット品質が不揃いの PR を綺麗に統合 |
| Rebase and merge | 全コミットを線形に追加（マージコミットなし） | 全コミットが品質基準を満たしているとき |

チームでどれを使うかを `CONTRIBUTING.md` に明記しておくと混乱が減ります。

### 5. よくあるアンチパターン

- **`git push --force` で共有ブランチを上書き** → 他者の作業が消える。共有ブランチは `force push` 禁止がデフォルト
- **rebase 後に `--force-with-lease` を使わず force push** → 誰かが push した内容を上書きしてしまう。自分のブランチでも `--force-with-lease` を使う習慣に
- **コンフリクトが怖くて rebase を避け、merge を積み重ねる** → タコ足マージで履歴が追いにくくなる

---

## Chapter 5: GitHub 活用

### 1. Pull Request を「小さく・早く」開く

PR の品質を決める最大の要因は「サイズ」です。

| PR のサイズ | レビュー所要時間（目安） | バグ見逃し率 |
|-----------|----------------------|------------|
| 〜200行 | 数分〜30分 | 低 |
| 200〜400行 | 30分〜1時間 | 中 |
| 400行超 | 1時間超、またはスキップ | 高 |

→ 「1 PR = 1 目的」を守り、大きな機能は複数 PR に分割する。

**PR テンプレートを置くと一貫性が上がります**（`.github/pull_request_template.md`）：

```markdown
## 変更の目的
<!-- なぜこの変更が必要か -->

## 変更内容
<!-- 何を変えたか -->

## 動作確認
- [ ] ローカルで動作確認済み
- [ ] テスト追加・更新済み

## 関連 Issue
Closes #
```

### 2. Draft PR を活用する

「作業途中だが共有したい」「方向性のフィードバックが欲しい」場面では、マージ不可の **Draft Pull Request** を早めに開く。

```
機能開発開始 → Draft PR を開く（進捗の可視化・早期フィードバック）
      ↓
レビュー準備完了 → Ready for review に変更
      ↓
レビュー・マージ
```

WIP コミットを積む代わりに Draft PR で作業を見せることで、方向性のズレを早期に発見できます。

### 3. Code Owners でレビュアーを自動アサイン

`.github/CODEOWNERS` ファイルに設定すると、特定ファイル・ディレクトリへの変更時に担当者が自動でレビュアーに追加されます。

```
# .github/CODEOWNERS

# 全ファイルのデフォルトオーナー
*                   @org/core-team

# セキュリティ関連は必ずセキュリティチームが確認
src/auth/           @org/security-team
*.env.example       @org/security-team

# インフラは SRE チームへ
infra/              @org/sre-team
.github/workflows/  @org/sre-team
```

### 4. ブランチ保護ルール

`Settings > Branches > Branch protection rules` で `main` に設定しておく最低限のルール：

| 設定 | 効果 |
|------|------|
| **Require a pull request before merging** | 直接プッシュ禁止 |
| **Require approvals（1〜2人）** | レビュー必須 |
| **Require status checks to pass** | CI が通らないとマージ不可 |
| **Require branches to be up to date** | マージ前に最新 main との同期を強制 |
| **Do not allow bypassing above settings** | 管理者にも適用 |

> **最低でも** 「PR 必須」と「CI ステータスチェック必須」の 2 つは設定する。これだけでデグレ事故の大半は防げます。

### 5. Issue テンプレートとラベル運用

**Issue テンプレート**（`.github/ISSUE_TEMPLATE/`）で報告の品質が安定します。

**ラベルの最小セット**：

| ラベル | 用途 |
|-------|------|
| `bug` | 不具合 |
| `feature` | 新機能要求 |
| `good first issue` | 初心者向け |
| `needs triage` | 未確認・要調査 |
| `blocked` | 別 Issue・PR に依存して進められない |

### 6. よくあるアンチパターン

- **PR の本文が空**：レビュアーは変更の意図を差分から推測するしかなくなる
- **大量の「fix」コミットが混入した PR をマージコミットで統合**：履歴が汚染される → スカッシュマージを選択
- **CODEOWNERS なしで全員が全ファイルをレビュー**：専門知識が必要な変更が見落とされる

---

## Chapter 6: 自動化と CI/CD

### 1. GitHub Actions の基本構造

ワークフローは `.github/workflows/*.yml` に置きます。「イベント → ジョブ → ステップ」の 3 層構造です。

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:           # PR を開いた・更新したときに発火

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm test
```

### 2. PR に CI を仕込む（最低限の自動化）

PR ごとに以下を自動で走らせるのがベースラインです。

| チェック | 目的 |
|---------|------|
| テスト実行 | 機能の壊れを検知 |
| Lint / フォーマット | コードスタイルの統一 |
| 型チェック | 型エラーの早期発見 |
| セキュリティスキャン | 既知の脆弱性の検出 |

→ `Settings > Branches` の「Required status checks」に CI ジョブを追加すると、CI グリーンなしにマージできなくなります。

### 3. デプロイの自動化

`main` マージ後に自動デプロイする典型パターン：

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production       # 環境ごとに承認者を設定可
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - name: Deploy to production
        run: ./scripts/deploy.sh
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

> **シークレットは `Settings > Secrets` に登録し、コードに直書きしない。** ログにも出力されないようになっています。

### 4. pre-commit フック（ローカル側の最終防衛線）

CI を通す前にローカルで問題を検出するために、Git フックを活用します。

```bash
# .git/hooks/pre-commit（実行権限が必要）
#!/bin/sh
npm run lint
npm run typecheck
```

または `husky`（Node.js プロジェクト）などのツールを使うとチームで共有できます：

```bash
# husky の例
npx husky init
echo "npm run lint" > .husky/pre-commit
```

> **フックは `--no-verify` で回避できます。** 緊急時に使う場合は「なぜ回避したか」をコミットメッセージや PR に明記する文化を持つことが重要です。

### 5. GitHub CLI（`gh`）でターミナルから操作する

GitHub の操作をターミナルで完結させると、コンテキストスイッチが減ります。

```bash
# インストール（macOS）
brew install gh
gh auth login

# よく使うコマンド
gh pr create --fill              # PR を作成（タイトル・本文をコミット情報から自動生成）
gh pr view                       # 現在ブランチの PR を表示
gh pr checkout 123               # PR #123 のブランチにチェックアウト
gh pr merge --squash             # PR をスカッシュマージ
gh issue create                  # Issue 作成
gh issue list --assignee @me     # 自分にアサインされた Issue 一覧
gh run list                      # 最近の Actions 実行一覧
gh run watch                     # 実行中のワークフローをウォッチ
```

### 6. よくあるアンチパターン

- **シークレットをコードに直書き** → 一度 git 履歴に入ると `git revert` では消えない。`git filter-repo` での完全除去が必要になる
- **CI が遅すぎて誰もウォッチしない** → キャッシュ活用・並列化で5分以内を目標に
- **`--no-verify` が常態化** → フックが壊れているか、チェック内容がブランチの実態に合っていないサイン。根本原因を直す

---

## Chapter 7: よくある失敗パターン

### 1. 共有ブランチへの `force push`（最大のリスク）

```
開発者Aが feature/xxx を rebase → force push
  ↓
開発者Bは rebase 前のコミットを base にしていた
  ↓
Bが push すると「マージ済み」のコミットが復活
  ↓
コンフリクトの嵐 + 「どれが正しいか」が不明に
```

**対策**：

- 共有ブランチに対する `force push` をブランチ保護で禁止
- 自分のブランチに force push が必要な場合は `--force-with-lease` を使う（他者が追加コミットしていたら失敗してくれる）

```bash
git push --force-with-lease   # 安全な force push
```

### 2. 秘密情報のコミット

一度コミットした情報は `git revert` では消せません。**コミット履歴に存在し続けます**。

**よく発生するパターン**：

- `.env` を `.gitignore` に追加する前にコミット
- `config.yml` に直書きしたAPIキー
- テスト用ハードコードのパスワードを本番コードにそのまま入れた

**発生した場合の対処（3ステップ）**：

```bash
# 1. まず流出した認証情報を即座に無効化・ローテーション（最優先）

# 2. git-filter-repo で履歴から完全除去
pip install git-filter-repo
git filter-repo --path .env --invert-paths

# 3. --force-with-lease で全ブランチにプッシュ（チームへの周知も必須）
```

> **予防**：`git-secrets` や GitHub の「Secret scanning」を有効化すると、コミット前・push 時にスキャンされます。

### 3. 巨大 PR によるレビュー品質の低下

```
1000行の PR → レビュアーが疲弊 → 「LGTM」だけで承認
  ↓
バグが通過 → 本番障害
```

**対策**：

- 1 PR を 400行以内に分割（`git add --patch` で細かくコミット分割）
- Feature Flag を使って「機能は main に入っているが OFF」の状態でマージ
- `CONTRIBUTING.md` に PR サイズの上限を明記

### 4. `main` への直接 push

「急いでいたから」が理由の大半ですが、テストなしで壊れたコードが入る確率が高まります。

**対策**：ブランチ保護を設定したら「例外なく」適用。管理者も含めて PR を通す。

### 5. コンフリクト解消ミス

コンフリクトマーカーを消し忘れたままコミット・デプロイは意外と多発します。

```python
<<<<<<< HEAD       ← これが残ってると動かない
def authenticate():
=======
def login():
>>>>>>> feature/rename
```

**対策**：

```bash
git diff --check       # コンフリクトマーカーの残存を検出
```

→ CI の lint・ビルドステップでも引っかかる設定にしておくと安全。

### 6. ブランチの放置（ゾンビブランチ）

マージも削除もされないブランチが増えると、誰かが古いブランチから新機能を分岐して最新の変更と大幅にズレた PR を出す事態が起きます。

```bash
# マージ済みブランチを一覧（main への統合済み）
git branch --merged main

# リモートの古いブランチを棚卸し
git fetch --prune        # リモートで削除済みのローカル参照を削除
git branch -vv | grep ': gone]'   # リモートが消えているローカルブランチを一覧
```

GitHub の `Settings > Automatically delete head branches` を有効にすると、マージ後にブランチを自動削除できます。

### 7. タグなしリリースによる「いつ何が出たか不明」問題

```bash
# リリース時に必ずタグを打つ
git tag -a v1.2.0 -m "Release v1.2.0: ユーザー登録機能を追加"
git push origin v1.2.0
```

GitHub の `Releases` 機能でタグに変更ログを紐付けると、誰でも「いつ何が出たか」を追跡できます。

---

## Chapter 8: チーム運用

### 1. チームで合意しておくべき最小セット

ツールより先に「規約」を固めると、あとで揉めません。`CONTRIBUTING.md` に書いて git 管理します。

```markdown
# CONTRIBUTING.md の最小テンプレート

## ブランチ戦略
- 戦略: GitHub Flow
- ブランチ命名: feature/xxx, fix/xxx, chore/xxx
- main は常にデプロイ可能な状態を保つ

## コミット
- 規約: Conventional Commits
- 粒度: 1コミット = 1論理変更

## Pull Request
- サイズ: 差分 400行以内を目安
- マージ方法: Squash and merge（main 履歴をクリーンに保つ）
- レビュー必須人数: 1人以上

## CI
- すべての PR で CI（lint + test + build）が通ることを必須とする
```

### 2. コードレビューの作法

**レビュアー側**：

| レビューの種類 | 書き方の例 |
|-------------|----------|
| **提案**（強制でない） | `nit: このループは Array.map で書けると思います` |
| **質問**（理解のため） | `なぜここで同期処理にしているのでしょうか？` |
| **必須修正** | `REQUIRED: この入力値はサニタイズが必要です（XSS リスク）` |
| **褒める** | `こういう抽象化、好きです！` |

> **「コードへのフィードバック」と「人への批判」を混同しない**。差分に話しかけるイメージで書く。

**作者側**：

- レビューコメントへの返信は「対応した / 対応しない（理由）」を明示して閉じる
- `Resolved` ボタンは作者が押す（「対応済み」の宣言）
- 反論する場合は根拠を添える。設計判断の違いなら直接話す場を設ける

### 3. オンボーディング活用

新規参画者が Git / GitHub に慣れるための段階的な受け入れ：

| 段階 | タスク |
|------|--------|
| 1日目 | clone → ローカルビルド → `good first issue` に取り組む |
| 1週間 | PR を出してレビューを受ける（Draft PR → Ready の流れを体験） |
| 1ヶ月 | 他の PR にレビューコメントをつける |
| 3ヶ月 | ブランチ保護・CI ルールの説明ができる |

### 4. チームで共有する設定ファイル

```
git 管理する（全員に適用）:
  .gitignore              # プロジェクト固有の除外ルール
  .github/CODEOWNERS      # レビュアー自動アサイン
  .github/pull_request_template.md
  .github/ISSUE_TEMPLATE/
  .github/workflows/      # CI/CD ワークフロー
  CONTRIBUTING.md         # 開発規約

個人設定（git 管理しない）:
  ~/.gitconfig            # エイリアス・エディタ
  ~/.gitignore_global     # OS/エディタ固有の除外ルール
  .git/hooks/             # ローカルのみのフック（husky で共有可）
```

### 5. ブランチ保護を「仕組み」として設計する

「文化として守る」から「技術的に守れない」への移行が運用の成熟を示します。

```
文化的制約（守られない可能性がある）:
  × 「main には直接 push しない」というルール

技術的制約（守らざるを得ない）:
  ✅ ブランチ保護: force push 禁止 + PR 必須 + CI 通過必須
  ✅ CODEOWNERS による自動レビュアーアサイン
  ✅ GitHub Actions による自動チェック
```

---

## Chapter 9: 直感を育てる

### 1. ガイドは出発点

ここまでのベストプラクティスはすべて「出発点」です。状況によっては原則を曲げる判断が正解になることもあります。

| 原則 | 曲げるべき状況の例 |
|------|-----------------|
| 1コミット = 1変更 | WIP コミットを積んで後で squash する探索的開発 |
| PR は400行以内 | 機械的な一括リネームで1000行変わるが、変更の意図は明確 |
| force push 禁止 | 自分しか使っていないブランチで、rebase 後の push が必要 |
| スカッシュマージ | 各コミットが品質基準を満たし、履歴として残す価値がある場合 |

**「なぜそのルールがあるのか」を理解していれば**、例外を正当化できます。

### 2. 判断の精度を上げる振り返り習慣

| タイミング | 問い |
|----------|------|
| PR マージ後 | 「コミット粒度は適切だったか？レビュアーに意図が伝わったか？」 |
| コンフリクト解消後 | 「このコンフリクトは防げたか？ブランチの分岐タイミングが遅かったか？」 |
| インシデント後 | 「どのコミットで壊れたか `git bisect` で特定できたか？」 |
| 長い PR レビュー後 | 「分割できたか？Draft PR を早めに開いていれば方向性の確認ができたか？」 |

### 3. 指標として意識する数字

明確な目標値を持つと改善が測定しやすくなります。

| 指標 | 目安 |
|------|------|
| PR のサイズ | 差分 400行以内 |
| PR のマージまでの時間 | 24時間以内（小さい PR で達成しやすい） |
| CI の実行時間 | 10分以内（超えると待ちが発生し始める） |
| ブランチの存在期間 | 数日以内（長ければ main との乖離が大きくなる） |
| `git log --oneline` の読みやすさ | 6ヵ月後の自分が意図を理解できるか |

### 4. 継続使用でしか得られない感覚

使い続けるうちに体感的にわかるようになること：

- **コミットを切るタイミング**：「ここで一度コミットしておくと後で安心」という感覚
- **rebase を使うタイミング**：「この履歴は整理してから PR にしたい」という感覚
- **PR を分割すべきタイミング**：「これレビューされる側に立つと辛いな」という感覚
- **force push を避けるタイミング**：「誰かが見ているかもしれない」という慎重さ

**うまくいった時**（レビューが速い、コンフリクトが少ない）：何がよかったかを意識する。

**詰まった時**（大きなコンフリクト、難解な履歴）：「もっと早いタイミングでできたことは何か」を問い直す。

---

## 関連リソース

- **Pro Git Book（日本語版）**: https://git-scm.com/book/ja/v2
- **GitHub Docs**: https://docs.github.com/ja
- **Conventional Commits**: https://www.conventionalcommits.org/ja/v1.0.0/
- **GitHub Flow ガイド**: https://docs.github.com/ja/get-started/using-github/github-flow
- **GitHub Actions ドキュメント**: https://docs.github.com/ja/actions

### 関連教材

- `git-github-overview.md` — Git/GitHub の全体像を俯瞰したい方向け（ブランチ戦略・コミット作法・GitHub 主要機能の概要）
- `git-github-tutorial.md` — インストールから GitHub Flow まで手を動かして学ぶハンズオン教材
