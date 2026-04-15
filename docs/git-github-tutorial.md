# Git と GitHub チュートリアル

> Git と GitHub を初めて触る人向けのハンズオン教材。Chapter 1〜6 を順に進めれば基本操作と GitHub Flow が身につきます。
> 各テーマをより深く知りたくなったら → 公式 Pro Git Book（https://git-scm.com/book/ja/v2）へ

---

## Chapter 0: Git と GitHub とは

> この章で「なぜ Git が必要か」を理解します。手は動かしません。

---

### 1. バージョン管理がないとどうなる

ファイルを手動管理したことがある方なら、きっとこんな経験があるはずです。

```
report_final.docx
report_final2.docx
report_final_本当に最終.docx
report_final_20240301.docx
report_田中レビュー後.docx
```

ここから生まれる問題は次の 3 つです。

| 問題 | 具体例 |
|------|------|
| **どれが最新かわからない** | 「`_本当に最終`」が本当に最終かは誰にもわからない |
| **なぜその内容になったかがわからない** | 「2の内容と最終3の違いは？」が即答できない |
| **以前の内容に戻せない** | どれかを上書きすると、元の内容が永遠に失われる |

**Git** はこれらをすべて解決する「ファイルの履歴管理ツール」です。

---

### 2. Git とは何か

**Git** は**分散型バージョン管理システム**です。「ファイルの変更履歴」をスナップショットとして記録してくれます。

```
 コミット履歴

  [A] → [B] → [C] → [D]【現在】
   ↑    ↑    ↑    ↑
  初期  機能  バグ  最新
  設定  追加  修正

  → いつでも任意のコミットに戻れる
```

**「分散型」の意味**：各開発者のコンピュータにリポジトリの**全履歴が丸ごと保存される**ため、サーバーで障害が起きても作業を続けられます。

---

### 3. GitHub とは何か――Git との役割分担

Git だけでは「自分のコンピュータの中だけ」の履歴管理です。**GitHub** はその履歴をクラウド上におくプラットフォームで、チーム開発に不可欠な機能を追加で提供します。

| | **Git** | **GitHub** |
|--|---------|------------|
| **実体** | ソフトウェア（CLI ツール） | クラウドサービス |
| **動く場所** | 自分の PC | インターネット上 |
| **機能** | 履歴管理・ブランチ | コード共有・PR・Issue・CI |
| **必須か** | ✅ Git はどれに必須 | ❌ チーム開発なら事実上必須 |

比喩で言うと――**Git が「日記」、GitHub が「日記を共有するクラウドストレージ」**という関係です。

> 「他の人の変更を取り込む（pull）するようお願いする」→ **Pull Request** の語源がこれです。

---

### 4. 他のツールとの比較

| ツール | 種別 | 弱点 |
|---------|------|------|
| **ファイル名に日付を付ける** | 手動 | 「なぜその内容なのか」が不明 |
| **Google Drive / Dropbox** | クラウドストレージ | 変更理由やブランチがない |
| **SVN（集中型）** | バージョン管理 | サーバー依存、オフライン不可 |
| **Git + GitHub** | 分散型 VCS | 初期学習コストがやや高い |

---

### 5. この教材で学べること

```
Chapter 1: インストール → 初回コミット体験
  ↓
Chapter 2: 基本コマンド（status / log / diff）
  ↓
Chapter 3: ブランチで安全に開発
  ↓
Chapter 4: GitHub と繋げる
  ↓
Chapter 5: GitHub Flow でチームコラボ
  ↓
Chapter 6: ミスのリカバリ
```

各章に練習課題があります。`git-practice/` ディレクトリを作って、章を足したがら同じプロジェクトに追加していきます。

---

## Chapter 1: インストールと初回コミット体験

---

### 1. 必要な環境

- macOS / Linux / Windows（WSL2 推奨）
- アカウント: GitHub アカウント（無料）

### 2. Git のインストール

```bash
# macOS
git --version
# コマンドが見つからなければ Xcode CLI ツールのインストールプロンプトが表示される

# Linux (Ubuntu/Debian)
sudo apt install git-all

# Windows: https://git-scm.com/download/win からインストーラーを実行
```

確認：
```bash
git --version   # git version 2.x.x と出れば OK
```

### 3. 初期設定（必須）

インストール後に一度だけ実行します。コミットに記録される名前とメールアドレスです。

```bash
git config --global user.name  "名前"
git config --global user.email "メールアドレス"
git config --global core.editor "vim"   # エディタはお好みで
```

設定済み内容の確認：
```bash
git config --list
```

> **GitHub にプッシュする場合**: GitHub のメールアドレスと同じ値にすると、コミットが正しく結び付きます。

---

### 4. 5 分で成功体験

**git-practice/** ディレクトリを作って、初めてのコミットを体験しましょう。

```bash
mkdir git-practice && cd git-practice

# 1. Git リポジトリとして初期化
git init
# Initialized empty Git repository in .../git-practice/.git/ と表示される

# 2. ファイルを作成
echo "# git-practice" > README.md

# 3. ステージング（コミット候補に登録）
git add README.md

# 4. コミット（履歴に記録）
git commit -m "initial commit"
```

コミット成功するとこんな表示が出ます：
```
[main (root-commit) a1b2c3d] initial commit
 1 file changed, 1 insertion(+)
 create mode 100644 README.md
```

これが Git の基本操作です。「変更 → `add` → `commit`」をこれから繰り返すことになります。

---

### 5. add と commit の 2 ステップを理解する

初心者が最初にはまる壁がここです。

```
[Working Directory]  ――git add――→  [Staging Area]  ――git commit――→  [Repository]
  変更したファイルがある場所                次のコミットに                 履歴として
                                         含めたいものを並べる              永続保存
```

| エリア | 役割 | 操作 |
|------|------|------|
| **Working Directory** | 実際に編集するファイル | ファイルを直接編集 |
| **Staging Area** | コミットに含める候補を並べる場所 | `git add` |
| **Repository** | 履歴が永久保存される場所 | `git commit` |

> **なぜ 2 ステップなのか?** 複数ファイルを変更したときに、「コミット A にはファイル 1 だけ、コミット B にはファイル 2 だけ」と分けて記録できるからです。

---

### 6. 練習課題

1. `git-practice/` に `hello.txt` を作成し、自分の名前を書いてコミット
2. `hello.txt` を編集して 2 回目のコミット
3. `git log` でコミット履歴が 2 件記録されていることを確認

---

## Chapter 2: 基本コマンドをマスターする

---

### 1. 現在の状態を確認する――`git status`

作業中に最も頻繁に使うコマンドです。何か操作する前、コミット後に「現在何が起きているか」を確認することの習慣をつけましょう。

```bash
git status
```

出力例：
```
On branch main
Changes not staged for commit:        ← 変更済みだが add してない
  modified:   hello.txt

Untracked files:                       ← まだ Git に追跡されてない
  notes.txt
```

| 表示 | 意味 |
|------|------|
| `Changes not staged` | 変更済みだが `add` されていない |
| `Changes to be committed` | `add` 済み、次の `commit` に含まれる |
| `Untracked files` | まだ Git が知らないファイル |

---

### 2. 履歴を見る――`git log`

```bash
git log                      # 履歴一覧
git log --oneline            # 1 行表示でコンパクトに
git log --oneline --graph    # ブランチの履歴を図示化
```

出力例：
```
commit a1b2c3d
Author: 自分の名前 <email>
Date:   Mon Apr 14 2026

    initial commit
```

コミットメッセージの書き方のベストプラクティス：

| 悪い例 | 良い例 |
|--------|--------|
| `fix` | `ログイン時にパスワードが空でも通過するバグを修正` |
| `update` | `ユーザー登録フォームにメール確認フィールドを追加` |
| `tmp` | `API レスポンスのタイムアウトを 30 秒に変更` |

「**何をしたか**」でなく「**なぜそうしたか**」を書くのが理想です。

---

### 3. 差分を見る――`git diff`

```bash
git diff                  # まだ add してない変更の差分
git diff --staged         # add 済みの差分（コミット履歴との差分）
git diff HEAD~1           # 1 つ前のコミットとの比較
```

出力の読み方：
```
-元の内容   ← 削除された行（赤）
+新しい内容  ← 追加された行（緑）
```

> **よくあるミス**: `git add` した後に `git diff` しても何も表示されない。ステージ済みの差分は `git diff --staged` で見ること。

---

### 4. .gitignore でトラッキング対象を除外する

パスワードやビルド成果物、ログファイルなど、Git に追跡させたくないファイルは `.gitignore` に書きます。

```bash
# .gitignore の例
.env                # 環境変数ファイル
*.log               # 全ての .log ファイル
node_modules/       # Node.js の依存パッケージ
dist/               # ビルド成果物
.DS_Store           # macOS のシステムファイル
```

`.gitignore` はプロジェクトルートに作って `git add` することで有効になります。

> **注意**: `.gitignore` にすでに追跡中のファイルを書いても除外されません。追跡履歴を切るには `git rm --cached <file>` が必要です。

---

### 5. 基本コマンド チートシート

| コマンド | 用途 |
|---------|------|
| `git status` | 現在の変更状態を確認 |
| `git add <file>` | 指定ファイルをステージ |
| `git add .` | 全ての変更をステージ（注意: 使いすぎに注意） |
| `git commit -m "メッセージ"` | 履歴に記録 |
| `git log --oneline` | 履歴をコンパクトに一覧 |
| `git diff` | ステージ前の差分 |
| `git diff --staged` | ステージ後の差分 |

---

### 6. 練習課題

1. `git-practice/` に `feature.txt` を作ってコミット
2. `feature.txt` を編集。`git diff` で差分を確認してから `git add` し、`git diff --staged` でステージ済み差分を確認、最後にコミット
3. `.gitignore` に `*.log` を追加し、`test.log` を作成。`git status` で `test.log` が表示されないことを確認

---

## Chapter 3: ブランチで安全に開発する

---

### 1. ブランチとは何か

ブランチは「コミット履歴への軽量ポインタ」です。作成・削除が一瞬で完了し、本体のファイルは一切コピーされません。

```
 main ブランチ（正式リリース）
   [A] → [B] → [C]
               ↘
              feature/login ブランチ（開発中）
               [D] → [E]
```

- `main`（または `master`）――常に「動く状態」を保つメインブランチ
- 各機能・修正はブランチを切って作業する
- 完成したら `main` にマージ（統合）する

---

### 2. ブランチの基本操作

```bash
# ブランチ一覧
git branch

# ブランチを作成
git branch feature/login

# ブランチを切り替える（Git 2.23 以降の推奨）
git switch feature/login

# 作成＋切り替えを同時に
git switch -c feature/signup

# ブランチを削除（マージ済み）
git branch -d feature/login
```

> **`checkout` vs `switch`**: `git checkout` も同じことができますが、Git 2.23 で導入された `git switch` の方が目的が明確で推奨です。

---

### 3. HEAD とは何か

**HEAD** は「現在自分がどのブランチの先端にいるか」を示す特殊ポインタです。

```bash
git log --oneline --decorate
# a1b2c3d (HEAD -> feature/login, main) ...
#           ↑↑↑↑
#           自分は今ここ
```

---

### 4. マージする

開発したブランチの内容を `main` に取り込む操作です。

```bash
# main に戻る
git switch main

# feature/login をマージ
git merge feature/login
```

| マージの種類 | 仕組み | いつ発生するか |
|------------|------|----------------|
| **Fast-forward** | ポインタをすすめるだけ | main が分岐後にコミットしていない場合 |
| **3-way merge** | 共通祖先を使って合流 | 両ブランチが分岐後にコミットした場合 |

---

### 5. マージコンフリクトを解消する

同じファイルの同じ箇所を両方のブランチが編集すると発生します。

```
<<<<<<< HEAD
アカウント登録ページ
=======
ログインページ
>>>>>>> feature/login
```

**解消手順**：
```bash
# 1. コンフリクトファイルを確認
git status

# 2. エディタで <<<< ==== >>>> のマーカーを手動編集して削除
#    保持したい内容だけ残す

# 3. 解消後にステージ
git add <ファイル名>

# 4. マージコミットを作成
git commit

# マージを中止したい場合
git merge --abort
```

> **コンフリクトはエラーではありません。** Git が「どちらの内容を重視するか決められない」ので、人間に決めるよう表しているだけです。

---

### 6. 練習課題

1. `feature/hello` ブランチを作成して切り替え、`hello.txt` に 1 行追加してコミット
2. `main` に戻って `git merge feature/hello` を実行。`--oneline` で履歴を確認
3. （応用）`feature/conflict` ブランチを作成し、`main` と両方で同じ行を別内容に編集・コミット。マージしてコンフリクトを手動解消する

---

## Chapter 4: GitHub と繋げる

---

### 1. リモートリポジトリとは

これまでの作業はローカル（自分の PC 上）のみでした。
**リモートリポジトリ** は GitHub 上の履歴のコピーで、バックアップ・共有・チームコラボの起点になります。

```
[ローカル] ――push――→ [GitHub]
[ローカル] ←――pull――  [GitHub]
[別のローカル] ←――clone―― [GitHub]
```

---

### 2. GitHub 上にリポジトリを作る

1. GitHub.com にログイン
2. 右上の `+` → `New repository`
3. Repository name: `git-practice`、Public / Private を選択
4. **「初期化ファイルは追加しない」のまま作成**（ローカルにすでに履歴があるため）

---

### 3. リモートを設定する

```bash
# リモートを追加（origin はデフォルト名）
git remote add origin https://github.com/<ユーザー名>/git-practice.git

# リモート一覧を確認
git remote -v
# origin  https://github.com/... (fetch)
# origin  https://github.com/... (push)
```

---

### 4. push ――ローカルの履歴を GitHub に送る

```bash
# 初回プッシュ（トラッキングを設定）
git push -u origin main

# 2 回目以降
git push
```

`-u`（`--set-upstream`）は「このブランチは origin の main と対応する」と設定するオプションで、一度設定すると次回から `git push` だけで済みます。

---

### 5. pull / fetch ――GitHub の変更を取り込む

```bash
# fetch ：ダウンロードのみ（ローカルブランチに影響なし）
git fetch origin

# pull ：ダウンロード＋自動マージ
git pull origin main

# トラッキング設定済みなら
git pull
```

| コマンド | 動作 | 使いどき |
|---------|------|--------|
| `fetch` | GitHub の変更を持ってくるだけ | 内容を確認してから取り込みたい |
| `pull` | fetch + merge を一発実行 | チームの変更を素直に取り込む |

> **push が拒否される場合**: 他の人が先に push していると失敗します。先に `git pull` で変更を取り込んでから再度 push しましょう。

---

### 6. clone ――既存リポジトリをローカルにコピーする

```bash
# GitHub 上のリポジトリを全履歴ごとダウンロード
git clone https://github.com/<ユーザー名>/<リポジトリ名>.git
```

`clone` は `remote add origin` + `pull` をまとめてやってくれるコマンドです。リポジトリの**全履歴**がダウンロードされます（最新スナップショットだけではありません）。

---

### 7. 練習課題

1. GitHub に `git-practice` リポジトリを作成し、`git remote add` で設定
2. `git push -u origin main` でプッシュ。GitHub 上でコミット履歴が見えることを確認
3. GitHub の UI 上でファイルを直接編集してコミット、`git pull` でローカルに取り込む

---

## Chapter 5: GitHub Flow でコラボする

---

### 1. GitHub Flow とは

**GitHub Flow** は「ブランチベースの軽量ワークフロー」です。「main に直接コミットせず、ブランチ→PR→マージ」の流れを必ず守るというルールです。

```
Issue 作成
  ↓
ブランチ作成（main から分岐）
  ↓
コミット（小さく頻繁に）
  ↓
GitHub に push
  ↓
Pull Request 作成
  ↓
レビュー → 修正 → 承認
  ↓
main にマージ
  ↓
ブランチ削除
```

---

### 2. Issue ――タスクを記録する

Issue はバグ報告・機能要求・タスク管理の起点です。PR と結び付けることで「この変更が何の課題を解決するか」が明確になります。

**Issue 作成手順**：
1. GitHub の `Issues` タブ → `New issue`
2. タイトル：内容が一目でわかる言葉で
3. 本文：再現手順（バグの場合）や定義（機能要求の場合）を記載

---

### 3. Pull Request ――メンバーにマージを依頼する

**Pull Request（PR）** は「このブランチの変更を main に取り込んでください」とチームに提案する仕組みです。マージ単一ボタンではなく、**議論・確認・レビュー** の場を付けることが目的です。

**PR 作成手順**：
1. ブランチを push 後、GitHub 上に 「Compare & pull request」ボタンが出る
2. **タイトル**：変更内容を簡潔に
3. **本文**：レビュアーに作業背景と確認ポイントを伝える
4. `Closes #<Issue番号>` を本文に書くと、マージ時に Issue も自動クローズされる

---

### 4. レビューからマージまで

| ステップ | 作業 |
|--------|------|
| **レビュアーが確認** | `Files changed` タブで差分を確認、コメントを投稿 |
| **作者が対応** | コメントに返信、修正コミットをプッシュ |
| **承認** | `Approve` でレビュー完了 |
| **マージ** | `Merge pull request` で main に統合 |
| **ブランチ削除** | `Delete branch` で作業ブランチを片付け |

---

### 5. ドラフト PR

GitHub には**Draft Pull Request**機能があり、作業途中の状態を「レビュー不可」のまま共有できます。「まだ確定してないが共有したい」場面で便利です。

---

### 6. 練習課題

1. GitHub に Issue を作成し、`feature/issue-1` ブランチで対応コミットをプッシュ
2. PR を作成し、`Closes #1` を本文に記載
3. 自分で PR をレビューしてマージ。Issue が自動クローズされることを確認

---

## Chapter 6: ミスをリカバリする

---

### 1. Git は「戻れる」のが最大の強み

バージョン管理の目的の一つは「壊れた状態に戻せること」です。命令の種類が多いので、状況別に使い分けましょう。

---

### 2. コミット前の変更を戻す

```bash
# Working Directory の変更を捨てる（restore, Git 2.23 以降推奨）
git restore <ファイル名>

# Staging Area から取り出す（add を取り消す）
git restore --staged <ファイル名>
```

> **注意**: `git restore <ファイル名>` は Working Directory の変更を**捨てます**。元に戻せないので慎重に。

---

### 3. コミット履歴を修正する

#### 直後のコミットメッセージを修正する

```bash
# メッセージだけ編集
git commit --amend -m "修正後のメッセージ"

# ファイルも追加したい場合
git add <追加ファイル>
git commit --amend --no-edit
```

> **警告**: `--amend` はコミット履歴を書き換えます。**すでに push したコミットに対して使うのは原則 NG**（チームの履歴を破壊する）。

---

### 4. 以前のコミットを安全に取り消す

```bash
# コミット ID（ハッシュ）を確認
git log --oneline
# a1b2c3d 最新コミット
# b2c3d4e 一つ前のコミット

# 変更を取り消す（逆操作のコミットを新たに作る）
git revert <コミットID>
```

`revert` は「逆操作のコミット」を新たに作る手法です。履歴を消さず安全に取り消せるので、**push 済みのコミットにも使える安全な方法**です。

---

### 5. 作業を一時保存する――`git stash`

コミットしたくないがブランチを切り替えたいときに使います。

```bash
# 作業中の変更を一時保存
git stash

# 保存内容を一覧
git stash list

# 直近の stash を復元
git stash pop

# 特定の stash を復元
git stash apply stash@{1}
```

典型的な使い方：
```
feature/new-ui で作業中
  ↓
緊急のバグ修正が入れ！
  ↓
git stash    # 作業を一時避難場所に
  ↓
git switch main
git switch -c hotfix/urgent-bug
  ↓
バグ修正・コミット
  ↓
git switch feature/new-ui
git stash pop    # 作業を復元
```

---

### 6. まとめ：状況別リカバリガイド

| 状況 | コマンド | 安全度 |
|------|---------|--------|
| `add` 前の変更を捨てたい | `git restore <ファイル>` | ⚠ 不復元 |
| `add` を取り消したい | `git restore --staged <ファイル>` | ✅ 安全 |
| 直後のコミットメッセージを修正 | `git commit --amend` | ⚠ push 前のみ |
| コミットを安全に取り消す | `git revert <ハッシュ>` | ✅ push 後も OK |
| 変更を一時避難 | `git stash` / `git stash pop` | ✅ 安全 |

---

### 7. 練習課題

1. ファイルを変更して `git add`。`git restore --staged` でステージを取り消し、さらに `git restore` で変更自体を捨てる
2. `git commit --amend -m "..."` で直後のコミットメッセージを修正（push 前に限る）
3. `git stash` で現在の変更を逃避 → 別ブランチの作業 → `git stash pop` で戻す

---

## Chapter 7: 次の一歩

---

### 1. ここまで学んだこと

Chapter 1〜6 を通じて、Git と GitHub の基本操作が身についたはずです。

| 章 | 習得したスキル |
|-----|----------------|
| Chapter 1 | インストールと初回コミット |
| Chapter 2 | status / log / diff / .gitignore |
| Chapter 3 | ブランチ・マージ・コンフリクト解消 |
| Chapter 4 | push / pull / fetch / clone |
| Chapter 5 | Issue・PR・GitHub Flow |
| Chapter 6 | restore / revert / stash |

---

### 2. 直感を育てる

チーム開発で使い続けるうちに身につく感覚があります。

- **コミットの粒度感**：「1 コミット 1 変更」の感覚
- **ブランチの命名感**：`feature/`, `fix/`, `docs/` のプレフィックスと内容の組合せ
- **PR チェック感**：差分を小さく保つことでレビューコストが下がる
- **ローカル・リモートの同期感**：作業前と作業後の pull / push のリズム

---

### 3. この教材で扱わなかったトピック

Git と GitHub には、本教材で入門的な操作だけを扱った内容がまだ山ほどあります。

| トピック | 具体的には |
|---------|----------|
| 履歴整理 | `git rebase`, `git squash`, `git cherry-pick` |
| 認証 | SSH 鍵設定、GitHub Personal Access Token |
| GitHub Actions | PR の自動テスト・デプロイ |
| Protected branch | テストが通らないとマージできない設定 |
| GitHub CLI (gh) | `gh pr create`, `gh issue list` などターミナルから操作 |

---

### 4. 学習リソース

- **Pro Git Book**： https://git-scm.com/book/ja/v2（日本語版無料）
- **GitHub 公式ドキュメント**： https://docs.github.com/ja
- **GitHub Skills**： https://skills.github.com/（ハンズオンで学べるインタラクティブコース）

### 関連教材

- `git-github-overview.md` — Git/GitHub の全体像を俯瞰したい方向け（ブランチ戦略・コミット作法・GitHub 主要機能の概要）
