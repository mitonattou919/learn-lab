# Claude Code 学習教材（2026年版）

> 公式ベストプラクティス（code.claude.com/docs/en/best-practices）に準拠した実務向け教材

---

## Chapter 0: Claude Code とは

### 1. AIコーディングエージェント

従来のAIチャット（ChatGPT、Claude.ai）は「会話」だけで、コード適用は人間の仕事やった。Claude Code は **読む・書く・実行する** を自律的にこなすエージェントや。

```
人間: 「このバグを直して」
  ↓
Claude Code: ファイル読み → 原因特定 → 修正 → テスト実行 → 結果報告
```

### 2. Claude Code の動作環境

- **CLI**: ターミナルから `claude` で起動（メイン）
- **デスクトップアプリ**: Mac / Windows の GUI、複数セッション並列管理
- **Web版**: claude.ai/code、Anthropic 管理の隔離 VM 上で実行
- **IDE拡張**: VS Code / JetBrains

### 3. 中核となる制約：コンテキストウィンドウ

ベストプラクティスの大半は **「コンテキストは早く埋まり、埋まると性能が落ちる」** という制約から導かれる。会話・読んだファイル・コマンド出力すべてが消費する。長セッションで Claude が初期指示を「忘れる」のはこれが原因。

→ **コンテキスト管理は最優先スキル**。`/clear`、`/compact`、`/rewind`、サブエージェントを駆使する。

---

## Chapter 1: 環境構築

### 1. インストール

```bash
node --version           # v18 以上
npm install -g @anthropic-ai/claude-code
claude --version
```

### 2. 認証

```bash
claude            # 初回はブラウザで OAuth
# セッション中に再認証する場合
> /login
```

### 3. 基本コマンド

| コマンド | 説明 |
|---------|------|
| `/help` | ヘルプ |
| `/clear` | 会話履歴リセット（コンテキスト解放） |
| `/compact <instructions>` | 履歴を要約して圧縮 |
| `/rewind` または `Esc Esc` | チェックポイントに巻き戻し |
| `/btw` | 履歴に残らない側面質問 |
| `/init` | CLAUDE.md の雛形を自動生成 |
| `/permissions` | 許可ルール編集 |
| `/sandbox` | OSレベル隔離の設定 |
| `/model` | モデル変更 |
| `/rename` | セッション名変更 |
| `/plugin` | プラグインマーケット |

### 4. セッション再開

会話履歴はローカルに保存される。**`/clear` しない限り次回も続けられる**。

```bash
claude --continue        # 直近のセッションを再開
claude --resume          # 一覧から選択
```

長期作業は `/rename oauth-migration` のように名前付けしておくと探しやすい。

### 5. 許可モード（Permission Modes）

| モード | 用途 |
|--------|------|
| `default` | 危険操作のみ確認（既定） |
| `auto` | 分類器モデルが自動承認、危険なものだけブロック |
| `plan` | Plan Mode（読み取り専用、計画立案） |
| `acceptEdits` | 編集を自動承認 |
| `bypassPermissions` | 全承認スキップ（要注意） |

```bash
claude --permission-mode auto -p "fix all lint errors"
```

**Sandbox** を併用すると、ファイル/ネットワークアクセスを OS レベルで制限できる。

---

## Chapter 2: プロンプト設計

### 1. 検証(Verify)ファースト ⭐ 最重要

公式が「**単一最大レバレッジ**」と明言する原則。Claude には必ず**自己検証手段**を渡す。

| 目的 | 悪い例 | 良い例 |
|------|--------|--------|
| 関数実装 | `validateEmail を実装して` | `validateEmail を実装。テスト: user@example.com→true, invalid→false, user@.com→false。実装後にテスト実行` |
| UI変更 | `ダッシュボードを良くして` | `[スクショ添付] このデザインを実装。完成後にスクショ撮って原画と差分を列挙、修正` |
| バグ修正 | `ビルドが落ちる` | `[エラー貼付] このエラーを修正してビルド成功を確認。**症状でなく根本原因**を直す` |

検証手段が無いものは出さない。テスト・リンタ・型チェック・スクショ比較が一級市民。

### 2. Explore → Plan → Implement → Commit

スコープが不明確、複数ファイルにまたがる、馴染みのないコードを触るタスクは **Plan Mode** から入る。

```
Step 1 (Plan Mode): src/auth を読んでセッション処理を把握して
Step 2 (Plan Mode): Google OAuth を追加するなら何をどう変える？計画作って
   ※ Ctrl+G で計画をエディタで直接編集できる
Step 3 (Normal Mode): 計画通り実装、コールバックのテスト書いて全テスト通して
Step 4: descriptive message でコミットして PR 作って
```

ただし**1文で diff を説明できる修正に Plan Mode は過剰**。タイポ修正、ログ追加、変数リネームは直接やる。

### 3. 効果的な指示の4要素

1. **対象**（ファイル・関数・行）
2. **目的**（なぜ）
3. **制約**（使う技術、避けること）
4. **完了条件**（何が満たされたら終わりか）

```
src/api.ts の fetchUser を、ユーザー不在時に undefined を返す代わりに
NotFoundError を throw するよう修正。既存テストを壊さず、新規テストで
不在ケースをカバー。pnpm test が全パスしたら完了。
```

### 4. リッチなコンテキストの渡し方

- **`@path/to/file`**: ファイル参照（Claude が自動で読む）
- **画像**: スクショをドラッグ&ドロップまたはペースト
- **URL**: ドキュメントURL を直接渡す（`/permissions` で常用ドメイン許可）
- **パイプ**: `cat error.log | claude -p "原因を特定"`
- **CLI ツール経由で取らせる**: `gh issue view 123 を読んで...`

### 5. インタビュー駆動の仕様化

大きめの機能は**先に Claude に質問させる**。

```
[機能の概要]を作りたい。AskUserQuestion ツールで詳細インタビューして。
技術実装、UI/UX、エッジケース、トレードオフを掘り下げて。
明白な質問はスキップ。完了したら SPEC.md に仕様を書いて。
```

仕様完成後に `/clear` して新セッションで実装に入る。

---

## Chapter 3: CLAUDE.md とプロジェクト設定

### 1. CLAUDE.md の役割

毎セッションの冒頭で自動読み込みされる**永続コンテキスト**。`/init` で雛形を生成して育てる。

### 2. 配置場所の階層

| パス | 適用範囲 |
|------|---------|
| `~/.claude/CLAUDE.md` | 全セッション共通（個人設定） |
| `./CLAUDE.md` | プロジェクト共有（git管理推奨） |
| `./CLAUDE.local.md` | 個人プロジェクト設定（`.gitignore`） |
| 親ディレクトリ | monorepo で自動継承 |
| 子ディレクトリ | そのディレクトリ配下の作業時に自動読込 |

### 3. import 構文

```markdown
See @README.md for project overview and @package.json for npm commands.

# Additional Instructions
- Git workflow: @docs/git-instructions.md
- Personal overrides: @~/.claude/my-project-instructions.md
```

### 4. 書く / 書かない

| ✅ 書く | ❌ 書かない |
|--------|-----------|
| Claude が推測できない bash コマンド | コードを読めば分かること |
| 既定と異なるコードスタイル | 言語標準の慣習 |
| テストの実行方法 | 詳細なAPIドキュメント（リンクで） |
| ブランチ命名・PR規約 | 頻繁に変わる情報 |
| プロジェクト固有の設計判断 | 長文のチュートリアル |
| 環境変数などの落とし穴 | ファイル毎の説明 |

### 5. 鉄則：肥大化させない

**長すぎる CLAUDE.md は無視される**。各行に「これを消したら Claude がミスるか？」と問うて、答えが No なら削る。守らせたいルールは `IMPORTANT` `YOU MUST` で強調する。

Claude が CLAUDE.md に書いてある質問をしてくる、または書いてあるルールを破る → 多くの場合 **長すぎる**。

### 6. CLAUDE.md の例

```markdown
# Code style
- ES modules (import/export)、CommonJS は使わない
- import は分割代入優先

# Workflow
- 一連の編集後は必ず typecheck
- テストは個別実行優先（全実行はパフォーマンス上避ける）
- マイグレーション実行前は必ず確認
```

---

## Chapter 4: セッション運用

### 1. 早期軌道修正

最速のフィードバックループが最高の結果を生む。

| 操作 | 用途 |
|------|------|
| `Esc` | 進行中の処理を中断（コンテキスト保持） |
| `Esc Esc` / `/rewind` | チェックポイントに巻き戻し |
| `Undo that` | 直前変更を取り消し |
| `/clear` | 文脈を完全リセット |

**同じ問題で2回以上修正したらセッションは汚染**。`/clear` して、学んだことを盛り込んだ良いプロンプトで再開する。

### 2. チェックポイント

Claude の各操作前に**自動でチェックポイントが作られる**。`/rewind` で：

- 会話だけ巻き戻す
- コードだけ巻き戻す
- 両方巻き戻す
- 任意のメッセージから要約する

→ 「とりあえず試してダメなら戻す」攻めのスタイルが取れる。セッションをまたいで保存される。

⚠ Claude が行った変更のみ追跡。git の代わりにはならない。

### 3. コンテキスト管理

- **`/clear`**: 無関係タスク間で必ず実行
- **自動圧縮**: 上限近辺で発火。重要なコード・決定を残す
- **`/compact <focus>`**: 指示付き圧縮（例: `/compact API変更だけ残して`）
- **`Esc Esc` → Summarize from here**: 部分圧縮
- **`/btw`**: 履歴に残さない質問（オーバーレイ表示）

CLAUDE.md に圧縮挙動も指定できる：
```
When compacting, always preserve modified file list and test commands.
```

### 4. サブエージェントによる調査委譲

調査は**メインのコンテキストを汚す**。サブエージェントは**別コンテキストで動いて要約だけ返す**。

```
サブエージェントで認証システムのトークンリフレッシュ処理と
既存の OAuth ユーティリティを調査して
```

実装後の検証にも使える：
```
サブエージェントでこのコードのエッジケースをレビューして
```

---

## Chapter 5: 拡張機構（Skills / Subagents / Hooks / MCP / Plugins）

公式の **「Extend Claude Code」** に対応。役割で使い分ける。

| 機構 | 性質 | 用途 |
|------|------|------|
| **CLAUDE.md** | 助言・全セッション読込 | プロジェクト共通の常識 |
| **Skills** | オンデマンド読込 | ドメイン知識・反復ワークフロー |
| **Subagents** | 別コンテキスト | 隔離した調査・専門タスク |
| **Hooks** | 決定論的 | 必ず実行する処理 |
| **MCP** | 外部接続 | 外部サービス連携 |
| **Plugins** | バンドル配布 | 上記をまとめてインストール |

### 1. Skills（`.claude/skills/`）

ドメイン知識や再利用ワークフローを提供。Claude が関連性で自動発火、または `/skill-name` で明示呼出。

```markdown
---
name: api-conventions
description: REST API design conventions for our services
---
# API Conventions
- URL は kebab-case
- JSON プロパティは camelCase
- 一覧 API は必ずページネーション
- バージョンは URL パスに（/v1/, /v2/）
```

副作用ある手動発火ワークフローは `disable-model-invocation: true` を付ける：

```markdown
---
name: fix-issue
description: Fix a GitHub issue
disable-model-invocation: true
---
GitHub issue $ARGUMENTS を修正:
1. gh issue view で詳細取得
2. コードベース調査
3. 修正実装
4. テスト追加・実行
5. lint / typecheck
6. コミット → push → PR 作成
```

`/fix-issue 1234` で起動。

### 2. Custom Subagents（`.claude/agents/`）

```markdown
---
name: security-reviewer
description: Reviews code for security vulnerabilities
tools: Read, Grep, Glob, Bash
model: opus
---
シニアセキュリティエンジニアとしてレビュー：
- インジェクション脆弱性（SQL, XSS, command）
- 認証・認可の欠陥
- コード内の秘密情報
- 安全でないデータ処理

具体的な行番号と修正案を提示。
```

明示呼び出し: `サブエージェントを使ってこのコードのセキュリティレビューして`

### 3. Hooks（`.claude/settings.json`）

CLAUDE.md は**助言**、Hooks は**決定論的に必ず動く**。

```
ファイル編集後に毎回 eslint を走らせる hook を書いて
migrations/ への書込をブロックする hook を書いて
```

`/hooks` で現在の設定を確認、Claude に書かせるのが速い。

### 4. MCP（外部サービス連携）

```bash
claude mcp add <name> <command> [args...]
claude mcp list
claude mcp remove <name>
```

主要サーバ: GitHub、Slack、PostgreSQL、Puppeteer、Notion、Figma、Sentry など。

**セキュリティ**: APIキーは環境変数経由、`settings.local.json` を `.gitignore`、最小権限ユーザーを使う。

### 5. Plugins

```
> /plugin
```

Skills + Hooks + Subagents + MCP のバンドル。型付き言語なら **Code Intelligence プラグイン**（正確なシンボルナビゲーション、編集後の自動エラー検出）を入れる。

### 6. CLI ツール優先

外部サービスは MCP より **CLI が context-efficient**。`gh`、`aws`、`gcloud`、`sentry-cli` を入れておく。Claude は未知の CLI も `tool --help` から学習できる。

---

## Chapter 6: 自動化とスケール

### 1. 非インタラクティブモード

```bash
claude -p "プロジェクトの説明"
claude -p "全 API エンドポイント列挙" --output-format json
claude -p "ログ解析" --output-format stream-json
```

CI、pre-commit hook、シェルスクリプトに組み込める。

### 2. Auto Mode で無人実行

```bash
claude --permission-mode auto -p "fix all lint errors"
```

分類器が承認を捌く。`-p` 併用時は分類器がブロック繰り返したら中断する（フォールバック先の人間がいないため）。

### 3. 並列セッション

| 方法 | 特徴 |
|------|------|
| デスクトップアプリ | 各セッションが独立 worktree |
| Web版 | Anthropic クラウド VM で隔離実行 |
| Agent Teams | 複数セッションをチームリーダーが調整 |

### 4. Writer / Reviewer パターン

別セッションで書かせて別セッションでレビュー。**コードを書いた直後の Claude はそのコードに肯定的バイアスがかかる**ため、フレッシュなコンテキストでレビューさせる。

| Session A (Writer) | Session B (Reviewer) |
|--------------------|----------------------|
| レート制限を実装して |  |
|  | @src/middleware/rateLimiter.ts をレビュー。エッジケース・競合・既存パターンとの整合性を見て |
| レビュー結果: [B output]。修正して |  |

テストにも使える：A がテストを書き、B が通す実装を書く。

### 5. Fan-out（大量並列処理）

```bash
# 1. 対象リスト生成
claude -p "移行対象の Python ファイルを files.txt に列挙" 

# 2. ループで個別実行
for file in $(cat files.txt); do
  claude -p "Migrate $file from React to Vue. Return OK or FAIL." \
    --allowedTools "Edit,Bash(git commit *)"
done
```

最初 2-3 ファイルでプロンプトを磨いてから全件実行。`--allowedTools` で権限スコープを絞る。

---

## Chapter 7: よくある失敗パターン

公式 **"Avoid common failure patterns"** より。

### 1. キッチンシンク・セッション
別タスクを混ぜていくうちに無関係情報で文脈が肥大化。
→ **修正**: タスク間で `/clear`。

### 2. 永遠の修正ループ
Claude が間違える → 修正 → まだ間違える → また修正…で文脈が失敗履歴で汚れる。
→ **修正**: 2回失敗したら `/clear`。学んだことを盛り込んだプロンプトで再開。

### 3. 過剰な CLAUDE.md
長すぎてルールがノイズに埋もれて無視される。
→ **修正**: 容赦なく剪定。指示なしでも正しく動くものは消す。または hook 化。

### 4. 信用したまま検証ギャップ
それっぽい実装がエッジケースを落としている。
→ **修正**: 必ず検証手段（テスト・スクリプト・スクショ）を提供。検証できないなら出荷しない。

### 5. 無限の探索
スコープなしで「調査して」→ 数百ファイル読んで文脈消費。
→ **修正**: 範囲を絞るか、サブエージェントに委譲。

---

## Chapter 8: 業務統合と運用

### 1. 適性マップ

| 作業 | 相性 |
|------|------|
| 定型反復（テスト追加、型修正、リネーム） | ◎ |
| コードベースのオンボーディング学習 | ◎ |
| 新規実装（仕様明確） | ○ |
| バグ調査・影響範囲解析 | ○ |
| ドキュメント生成 | ○ |
| 設計の意思決定 | △（人間の判断必須） |
| 本番への適用判断 | × |

### 2. オンボーディング活用

新規参画者は Claude にシニアエンジニアの如く質問する：
- 「ロギングはどう動いてる？」
- 「新 API エンドポイントの追加方法は？」
- 「foo.rs:134 の `async move { ... }` は何してる？」
- 「なぜ line 333 で foo() でなく bar() を呼んでる？」

### 3. チーム共有設定

```
git管理:    .claude/settings.json     # チーム共通設定
git管理:    CLAUDE.md                  # チーム共通プロジェクト知識
git管理:    .claude/skills/            # チーム共通スキル
git管理:    .claude/agents/            # チーム共通サブエージェント
.gitignore: .claude/settings.local.json # 個人設定（秘密情報）
.gitignore: CLAUDE.local.md             # 個人メモ
```

### 4. 人間が手放さない判断

- **設計**: アーキテクチャ選定
- **仕様**: ビジネスロジック確認
- **リスク**: 本番DB変更の可否
- **優先度**: どれから直すか

エージェントは **「実行」が得意、「判断」は人間**。

---

## Chapter 9: 直感を育てる

ガイドのパターンは出発点であって金科玉条ではない。

- **時にはコンテキストを溜めるべき**: 一つの複雑な問題に深く入っている時、履歴自体が価値を持つ
- **時には Plan を飛ばすべき**: 探索的なタスクは Claude の解釈を見てから絞る
- **時には曖昧なプロンプトが正解**: `このファイルで改善できる点は？` は思いつかなかった指摘を引き出す

うまくいった時：プロンプト構造、与えた文脈、モードを覚える。
詰まった時：文脈ノイズが多すぎたか、プロンプトが曖昧か、タスクが大きすぎたか問い直す。

継続使用でしか得られない直感を育てるのが最終ゴールや。

---

## 関連リソース

- 公式ベストプラクティス: https://code.claude.com/docs/en/best-practices
- ドキュメント索引: https://code.claude.com/docs/llms.txt
- How Claude Code works（agentic loop の内部）
- Common workflows（debugging / testing / PR のレシピ集）
