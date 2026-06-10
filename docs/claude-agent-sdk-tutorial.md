# Claude Agent SDK チュートリアル

> Claude Code をプログラムから操作したい開発者向けのハンズオン教材。
> 8章を順に進めれば基本操作と最低限の作法が身につきます。
> Claude Code CLI 自体の使い方 → `claude-code-tutorial.md` を先に
> 運用ノウハウを深掘りしたい方 → `claude-agent-sdk-best-practices.md`（予定）へ

---

## Chapter 0: Claude Agent SDK とは

### 1. 「CLI で使う Claude」と「SDK で使う Claude」の違い

**Claude Code CLI** はターミナルを開いて人間が対話する道具です。
**Claude Agent SDK** は Python/TypeScript からコードで Claude Code を呼び出し、
CI/CD・自動化パイプライン・プロダクションアプリに組み込む道具です。

```
【Claude Code CLI】
開発者 → ターミナル → claude → 対話 → 結果を目で確認

【Claude Agent SDK】
プログラム → query() → Claude がツールを自律実行 → ResultMessage → 次の処理へ
```

### 2. 3 つの「Claude を使う方法」の役割分担

混同しやすいので整理を。

| 方法 | パッケージ | 向いているケース |
|------|-----------|-----------------|
| **Claude Code CLI** | `claude` (npm) | 人間がインタラクティブに作業する |
| **Claude Agent SDK** | `claude-agent-sdk` (pip) | アプリから自動化・バッチ処理 |
| **Anthropic Client SDK** | `anthropic` (pip) | ツールループを自分で細かく制御する |

→ 「自律的に動くエージェントをアプリに埋め込みたい」なら Agent SDK が最適。

### 3. Claude Agent SDK が得意なこと

```
✅ CI/CD パイプラインへの組み込み（PR レビュー自動化 etc.）
✅ バッチ処理（100 ファイルを一括マイグレーション）
✅ カスタムツールを持たせて特定業務に特化させる
✅ Amazon Bedrock / Google Vertex AI / Azure との統合
❌ 人間がリアルタイムに確認しながら進めたい → CLI を使う
❌ ツールループを細かく制御したい → Anthropic Client SDK を使う
```

### 4. 全体の仕組みを掴む

`query()` を呼ぶと非同期イテレータが返ります。Claude がツールを使いながらタスクをこなし、ステップごとにメッセージが yield されます。

```
query(prompt="src/ 内の型エラーをすべて直して") を呼ぶ
  ↓
SystemMessage(subtype="init")    … セッション開始
  ↓
AssistantMessage                 … Claude が考える・ツールを呼ぶ
  ↓
UserMessage                      … SDK がツールを実行して結果を返す
  ↓
AssistantMessage                 … 次のステップを考える
  ↓
  …（ loop ）…
  ↓
ResultMessage(subtype="success") … 完了・最終結果
```

4 種類のメッセージをひとつずつ処理するだけで、エージェントの全ステップをプログラムから制御できます。

### 5. 「自律的なコーディングアシスタント」だと思って付き合う

- 指示が明確であれば確実にこなす
- ファイル操作・コマンド実行・Web 検索を組み合わせて動く
- 最終的な確認と責任は人間（呼び出し側）が持つ

---

## Chapter 1: インストールと初回ハロー体験

### 1. 必要な環境

- Python 3.10 以上
- Anthropic アカウント（API キー）

```bash
python --version   # 3.10 以上を確認
```

### 2. インストール

```bash
pip install claude-agent-sdk
```

`uv` を使う場合（推奨）：

```bash
uv add claude-agent-sdk
```

### 3. 認証

```bash
export ANTHROPIC_API_KEY=your-api-key-here
```

`.env` ファイルを使う場合：

```bash
pip install python-dotenv
```

```python
# main.py の先頭に追加
from dotenv import load_dotenv
load_dotenv()
```

> **API キーの取得**: [console.anthropic.com](https://console.anthropic.com/) でアカウントを作り、「API Keys」から発行してください。

### 4. 5 分で成功体験

練習用ディレクトリを作って試してみましょう。

```bash
mkdir agent-practice && cd agent-practice
echo "print('hello')" > hello.py
```

`main.py` を作成：

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="このディレクトリにあるファイルを教えて",
        options=ClaudeAgentOptions(allowed_tools=["Bash", "Glob"]),
    ):
        if hasattr(message, "result"):
            print(message.result)

asyncio.run(main())
```

実行：

```bash
python main.py
```

期待される出力例：

```
このディレクトリには以下のファイルがあります：
- hello.py
- main.py
```

これが Claude Agent SDK の基本動作です。**`query()` にやってほしいことを書くだけ**で、Claude が必要なツールを選んで実行し、結果を返します。

### 5. もう少し試してみる

```python
# ファイルを読んで説明させる
async def main():
    async for message in query(
        prompt="hello.py を読んで、何をしているか説明して",
        options=ClaudeAgentOptions(allowed_tools=["Read"]),
    ):
        if hasattr(message, "result"):
            print(message.result)
```

```python
# ファイルを修正させる
async def main():
    async for message in query(
        prompt="hello.py を「こんにちは、世界！」と出力するよう修正して",
        options=ClaudeAgentOptions(allowed_tools=["Read", "Write", "Edit", "Bash"]),
    ):
        if hasattr(message, "result"):
            print(message.result)
```

これで Claude が **読む・編集する・実行する** を自律的にこなします。

### 6. よくあるエラーと対処

| エラー | 原因 | 対処 |
|--------|------|------|
| `ModuleNotFoundError: No module named 'claude_agent_sdk'` | インストール未実施 | `pip install claude-agent-sdk` |
| `AuthenticationError` | API キー未設定 | `export ANTHROPIC_API_KEY=...` |
| `Python 3.10+ required` | Python バージョンが古い | `python --version` で確認、アップグレード |
| 何も出力されない | `result` 以外のメッセージタイプを受け取っている | Chapter 2 でメッセージの種類を学ぶ |

### 7. 練習課題

`agent-practice/` の中で：

1. `numbers.py` を「1 から 10 までの合計を計算して出力する」スクリプトとして作らせる
2. `python numbers.py` を実行させる
3. 出力を「合計: XX」という形式に変更させる

---

## Chapter 2: エージェントループの仕組み

### 1. `query()` が返す 4 種類のメッセージ

Claude Agent SDK の核心は、`query()` が返す**非同期イテレータ**です。Claude がタスクをこなす過程で、以下の 4 種類のメッセージが順番に届きます。

```
SystemMessage(subtype="init")  ← ① セッション開始
AssistantMessage               ← ② Claude が考える・ツールを呼ぶ
UserMessage                    ← ③ SDK がツールを実行して結果を返す
AssistantMessage               ←   ② と ③ を繰り返す
    …
AssistantMessage               ← ④ テキストのみ（最終回答）
ResultMessage                  ← ⑤ ループ終了・コスト・セッションID
```

「テスト修正タスク」を例にすると、こんな順番になります：

```
1. SystemMessage(init)
2. AssistantMessage  … Bash でテスト実行
3. UserMessage       … 3 件の失敗を受け取る
4. AssistantMessage  … Read でファイル読み込み
5. UserMessage       … ファイル内容を受け取る
6. AssistantMessage  … Edit で修正 → Bash で再テスト
7. UserMessage       … テスト全通過を受け取る
8. AssistantMessage  … 「修正しました」という最終テキスト
9. ResultMessage     … 完了
```

### 2. 各メッセージの主要フィールド

```python
from claude_agent_sdk import (
    query, ClaudeAgentOptions,
    SystemMessage, AssistantMessage, UserMessage, ResultMessage,
)

async for message in query(prompt="...", options=...):
    if isinstance(message, SystemMessage):
        # subtype: "init" または "compact_boundary"
        print(f"セッション開始: {message.data}")

    elif isinstance(message, AssistantMessage):
        # content: テキストブロックとツール呼び出しのリスト
        for block in message.content:
            if hasattr(block, "text"):
                print(f"Claude: {block.text}")

    elif isinstance(message, ResultMessage):
        # result: 最終回答テキスト
        # subtype: "success" / "error_max_turns" / "error_during_execution" 等
        # total_cost_usd: 今回のコスト
        # num_turns: 使ったターン数
        print(f"結果: {message.result}")
        print(f"コスト: ${message.total_cost_usd:.4f}")
```

### 3. ループを観察する（デバッグモード）

ステップが見えると何が起きているか把握しやすくなります。

```python
import asyncio
from claude_agent_sdk import (
    query, ClaudeAgentOptions,
    AssistantMessage, ResultMessage,
)

async def run_with_trace(prompt: str):
    print(f"\n>>> {prompt}\n")

    async for message in query(
        prompt=prompt,
        options=ClaudeAgentOptions(allowed_tools=["Bash", "Glob", "Read"]),
    ):
        if isinstance(message, AssistantMessage):
            for block in message.content:
                # ツール呼び出し
                if hasattr(block, "name"):
                    print(f"  [ツール] {block.name}({dict(block.input)})")
                # テキスト
                elif hasattr(block, "text") and block.text:
                    print(f"  [Claude] {block.text[:80]}...")

        elif isinstance(message, ResultMessage):
            print(f"\n完了 ({message.num_turns} ターン / ${message.total_cost_usd:.4f})")

asyncio.run(run_with_trace("このディレクトリの Python ファイルを数えて"))
```

### 4. ループを制御するオプション

| オプション | 型 | 説明 |
|-----------|-----|------|
| `max_turns` | `int` | ツール呼び出しの最大ターン数。超えると `error_max_turns` |
| `max_budget_usd` | `float` | コスト上限（USD）。超えると `error_max_budget_usd` |
| `effort` | `str` | `"low"` / `"medium"` / `"high"` / `"xhigh"` / `"max"` |

```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Edit", "Bash"],
    max_turns=10,          # 10 ターンで強制終了
    max_budget_usd=0.50,   # $0.50 で強制終了
    effort="medium",       # 速度・品質のバランス
)
```

> **`max_turns` の数え方**: ツール呼び出しを含むターンのみカウント。テキストのみの最終応答はカウントされません。

### 5. ResultMessage の終了理由を確認する

ループが終わった理由は必ず確認しましょう。（→ Chapter 6 で詳しく扱います）

```python
if isinstance(message, ResultMessage):
    match message.subtype:
        case "success":
            print(f"✅ 完了: {message.result}")
        case "error_max_turns":
            print("⚠ ターン上限に達しました。max_turns を増やすか、タスクを分割してください")
        case "error_max_budget_usd":
            print("⚠ コスト上限に達しました")
        case "error_during_execution":
            print("❌ 実行中にエラーが発生しました")
```

### 6. 練習課題

`agent-practice/` の中で：

1. `run_with_trace()` を使って「`hello.py` のバグを見つけて修正して」を実行し、何ターンで完了したか確認してください
2. `max_turns=2` に制限して同じタスクを実行し、`error_max_turns` を意図的に発生させてみてください
3. タスクが 2 ターンで収まるような短いプロンプトに書き直して成功させてください

---

## Chapter 3: 組み込みツールを使いこなす

### 1. 組み込みツール一覧

Claude Agent SDK には最初から使えるツールが揃っています。

| カテゴリ | ツール | できること |
|---------|--------|-----------|
| **ファイル操作** | `Read` | ファイルを読む |
| | `Write` | 新規ファイルを作成する |
| | `Edit` | 既存ファイルを編集する |
| **検索** | `Glob` | パターンでファイルを検索する（`**/*.py` 等） |
| | `Grep` | 正規表現でファイル内容を検索する |
| **実行** | `Bash` | ターミナルコマンドを実行する |
| | `Monitor` | バックグラウンドプロセスを監視する |
| **Web** | `WebSearch` | Web 検索する |
| | `WebFetch` | Web ページを取得・解析する |
| **エージェント** | `Agent` | サブエージェントを起動する |
| | `AskUserQuestion` | ユーザーに確認を取る |
| | `Skill` | スキルを呼び出す |
| | `ToolSearch` | ツールをオンデマンドで検索・ロードする |
| **タスク管理** | `TaskCreate` / `TaskUpdate` | タスクを追跡する |

### 2. `allowed_tools` でスコープを絞る

ツールを絞ることで、エージェントが意図しない操作をするリスクを減らせます。

```python
# 読み取りのみ（安全）
options = ClaudeAgentOptions(allowed_tools=["Read", "Glob", "Grep"])

# ファイル操作も許可
options = ClaudeAgentOptions(allowed_tools=["Read", "Write", "Edit", "Glob", "Grep"])

# コマンド実行も許可（強力だが要注意）
options = ClaudeAgentOptions(allowed_tools=["Read", "Edit", "Bash"])

# Bash を特定コマンドのみに絞る
options = ClaudeAgentOptions(allowed_tools=["Read", "Bash(pytest *)"])
```

> **基本方針**: タスクに必要な最小限のツールだけ渡す。`Bash` は強力なのでスコープを絞るか、テスト環境で使う。

### 3. よく使うツールの組み合わせ

タスクに応じて典型的な組み合わせがあります。

```python
# コードレビュー（変更なし）
ClaudeAgentOptions(allowed_tools=["Read", "Glob", "Grep"])

# バグ修正
ClaudeAgentOptions(allowed_tools=["Read", "Edit", "Bash"])

# 新機能の実装
ClaudeAgentOptions(allowed_tools=["Read", "Write", "Edit", "Glob", "Grep", "Bash"])

# ドキュメント生成
ClaudeAgentOptions(allowed_tools=["Read", "Glob", "Write"])

# Web 情報収集
ClaudeAgentOptions(allowed_tools=["WebSearch", "WebFetch", "Write"])
```

### 4. パーミッションモードと組み合わせる

`allowed_tools` はスコープを決め、`permission_mode` は確認の挙動を決めます。

| モード | 動作 | 向いているケース |
|--------|------|-----------------|
| `"default"` | 危険操作のみ確認（既定） | インタラクティブな UI から呼ぶとき |
| `"acceptEdits"` | ファイル編集を自動承認 | 開発マシンで自律実行させるとき |
| `"plan"` | 読み取り専用、提案のみ | レビューや計画立案タスク |
| `"bypassPermissions"` | 全操作を無条件実行 | CI / 隔離コンテナ環境のみ |

```python
# 開発マシンでファイル変更タスク
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Edit", "Bash"],
    permission_mode="acceptEdits",
)

# CI パイプライン（隔離コンテナ前提）
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Edit", "Bash"],
    permission_mode="bypassPermissions",
)
```

> ⚠ `bypassPermissions` は隔離された環境（Docker コンテナ・CI）でのみ使うこと。ローカルマシンでは `"acceptEdits"` が安全側のデフォルトです。

### 5. 実践：ファイル操作タスク

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage

async def review_code(filepath: str):
    """ファイルのコードをレビューして問題点を報告する"""
    async for message in query(
        prompt=f"{filepath} をレビューして、バグや改善点を日本語で報告して。変更はしない。",
        options=ClaudeAgentOptions(
            allowed_tools=["Read"],
            permission_mode="plan",   # 読み取りのみ
        ),
    ):
        if isinstance(message, ResultMessage):
            print(message.result)

asyncio.run(review_code("hello.py"))
```

### 6. 実践：Web 検索タスク

```python
async def research_topic(topic: str, output_file: str):
    """トピックを調べて Markdown ファイルに保存する"""
    async for message in query(
        prompt=f"""
        「{topic}」について調べて、以下の構成で {output_file} に保存して：
        - 概要（3行）
        - 主要な特徴（箇条書き）
        - 参考 URL
        """,
        options=ClaudeAgentOptions(
            allowed_tools=["WebSearch", "WebFetch", "Write"],
        ),
    ):
        if isinstance(message, ResultMessage):
            print(f"保存しました: {output_file}")

asyncio.run(research_topic("Python asyncio", "asyncio-notes.md"))
```

### 7. 練習課題

`agent-practice/` の中で：

1. `Read` と `Glob` だけを使って「このディレクトリ内の全 `.py` ファイルの行数を合計して」を実行してください
2. `permission_mode="plan"` で `hello.py` の改善案を提案させ、次に `permission_mode="acceptEdits"` で実際に変更させてください
3. `WebSearch` と `Write` を使って好きなプログラミングトピックを調べ、メモファイルを作らせてください

---

## Chapter 4: カスタムツールを作る

### 1. なぜカスタムツールが必要か

組み込みツールはファイル操作・Web 検索など汎用的な操作を担います。自社 API・社内 DB・独自の計算ロジックを Claude に使わせたいときは**カスタムツール**を定義します。

```
【カスタムツールなし】
「在庫データベースで製品Aの在庫を確認して」
  → Claude には DB アクセス手段がない → 答えられない

【カスタムツールあり】
「在庫データベースで製品Aの在庫を確認して」
  → Claude が get_inventory("製品A") を呼ぶ → DB照会 → 結果を回答
```

### 2. ツールの定義：`@tool` デコレーター

`@tool` デコレーターで Python 関数をツールに変換します。

```python
from claude_agent_sdk import tool

@tool(
    "get_inventory",                                        # ツール名（一意）
    "指定した製品の在庫数を返す。在庫確認タスクで使用する。",  # Claude への説明
    {"product_name": str, "warehouse": str},               # 引数の型定義
)
async def get_inventory(args):
    # args は辞書で渡される
    product = args["product_name"]
    warehouse = args.get("warehouse", "東京")

    # 実際の開発ではここで DB クエリ等を実行
    stock = {"製品A": 150, "製品B": 43, "製品C": 0}
    count = stock.get(product, -1)

    # 戻り値は content リスト形式
    return {
        "content": [{"type": "text", "text": f"{product}（{warehouse}）: {count} 個"}]
    }
```

**3 つのポイント：**

| 要素 | 役割 |
|------|------|
| **ツール名** | Claude がツールを識別する ID |
| **説明文** | Claude が「いつ使うか」を判断する根拠。具体的に書くほど精度が上がる |
| **引数の型定義** | Claude が正しい引数を生成するためのスキーマ |

### 3. MCP サーバーとして登録する

定義したツールは `create_sdk_mcp_server` でまとめて `mcp_servers` に渡します。

```python
import asyncio
from claude_agent_sdk import (
    query, ClaudeAgentOptions,
    tool, create_sdk_mcp_server,
    ResultMessage,
)

@tool(
    "get_inventory",
    "指定した製品の在庫数を返す。在庫確認タスクで使用する。",
    {"product_name": str},
)
async def get_inventory(args):
    stock = {"製品A": 150, "製品B": 43, "製品C": 0}
    count = stock.get(args["product_name"], -1)
    return {"content": [{"type": "text", "text": f"在庫: {count} 個"}]}


# ツールをサーバーとしてまとめる
inventory_server = create_sdk_mcp_server(
    name="inventory",          # サーバー名
    version="1.0.0",
    tools=[get_inventory],     # 複数渡せる
)


async def main():
    async for message in query(
        prompt="製品Aと製品Cの在庫を確認して",
        options=ClaudeAgentOptions(
            mcp_servers=[inventory_server],
            # カスタムツールは mcp__{サーバー名}__{ツール名} で参照される
            allowed_tools=["mcp__inventory__get_inventory"],
        ),
    ):
        if isinstance(message, ResultMessage):
            print(message.result)

asyncio.run(main())
```

> **ツール名の命名規則**: `mcp__{server_name}__{tool_name}` の形式になります。`allowed_tools` にはこの完全名を指定してください。

### 4. エラーをループ内で処理する

ツールで例外を `raise` するとエージェントループ全体が止まります。`is_error: True` を返すと、ループを継続しつつ Claude がリトライを判断できます。

```python
@tool(
    "fetch_sales_data",
    "指定月の売上データを取得する。月は YYYY-MM 形式で指定する。",
    {"month": str},
)
async def fetch_sales_data(args):
    month = args["month"]
    data = {
        "2024-01": {"total": 4480},
        "2024-02": {"total": 4460},
    }

    if month not in data:
        # ❌ raise するとループが止まる
        # raise ValueError(f"{month} のデータなし")

        # ✅ is_error で返すと Claude がリトライを判断する
        return {
            "content": [{"type": "text", "text": f"{month} のデータは存在しません"}],
            "is_error": True,
        }

    return {"content": [{"type": "text", "text": str(data[month])}]}
```

### 5. 良いツール説明文の書き方

説明文は Claude が「いつ・なぜ」このツールを呼ぶかを判断する唯一の手がかりです。

```python
# ❌ 悪い例：何をするか不明確
@tool("get_data", "データを取得する", {"id": str})

# ✅ 良い例：いつ使うか・何が返るかが明確
@tool(
    "get_customer_order",
    "顧客IDから注文履歴を取得する。"
    "「○○さんの注文を確認して」など、顧客別の注文照会タスクで使用する。"
    "戻り値は注文日・商品名・金額のリスト。",
    {"customer_id": str},
)
```

### 6. 複数ツールを組み合わせる

```python
@tool("get_sales", "指定月の売上を取得する。月は YYYY-MM 形式。", {"month": str})
async def get_sales(args): ...

@tool("calculate_growth", "2つの数値から成長率(%)を計算する。", {"base": float, "current": float})
async def calculate_growth(args):
    rate = ((args["current"] - args["base"]) / args["base"]) * 100
    return {"content": [{"type": "text", "text": f"{rate:.1f}%"}]}

analysis_server = create_sdk_mcp_server(
    name="analysis", version="1.0.0", tools=[get_sales, calculate_growth]
)

async for message in query(
    prompt="2024年1月と2月の売上を比べて成長率を出して",
    options=ClaudeAgentOptions(
        mcp_servers=[analysis_server],
        allowed_tools=[
            "mcp__analysis__get_sales",
            "mcp__analysis__calculate_growth",
        ],
    ),
): ...
```

### 7. 練習課題

`agent-practice/` の中で：

1. `add_numbers(a: float, b: float)` と `multiply_numbers(a: float, b: float)` の 2 つのカスタムツールを作り、「3 と 4 を足して、その結果に 2 を掛けて」を実行してください
2. 存在しないキーを渡したとき `is_error: True` で返し、Claude が別の値でリトライすることを確認してください
3. ツールの説明文を意図的にあいまいにして呼ばれなくなる体験をしてから、具体的な説明に直して改善を確認してください

---

## Chapter 5: セッション管理

### 1. なぜセッション管理が必要か

`query()` を呼ぶたびに新しい会話が始まります。複数ステップに分けた作業や、一度調べた内容を引き継いで別のタスクをこなす場合は**セッションの継続**が必要です。

```
【セッションなし】
1回目: 「auth モジュールを読んで構造を把握して」→ 把握完了
2回目: 「さっき把握した内容をもとにリファクタリングして」→ ❌ 1回目の記憶がない

【セッション継続】
1回目: 「auth モジュールを読んで構造を把握して」→ session_id を保存
2回目: resume=session_id で続き → ✅ 1回目の文脈を引き継いでリファクタリング
```

### 2. session_id の取得

セッション ID は `ResultMessage` に含まれます。

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage

async def main():
    session_id = None

    async for message in query(
        prompt="src/auth.py を読んで、認証フローの概要を説明して",
        options=ClaudeAgentOptions(allowed_tools=["Read", "Glob"]),
    ):
        if isinstance(message, ResultMessage):
            session_id = message.session_id   # ← ここで取得
            print(f"session_id: {session_id}")
            print(message.result)

    return session_id

asyncio.run(main())
```

### 3. セッションを再開する（resume）

取得した `session_id` を `resume` オプションに渡すと、前回の文脈が引き継がれます。

```python
async def continue_session(session_id: str):
    async for message in query(
        prompt="把握した内容をもとに、処理を関数に分割してリファクタリングして",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit"],
            resume=session_id,     # ← 前回のセッションを指定
            permission_mode="acceptEdits",
        ),
    ):
        if isinstance(message, ResultMessage):
            print(message.result)

asyncio.run(continue_session("取得した session_id"))
```

### 4. セッションを分岐させる（fork）

「別のアプローチを試したいが、元のセッションは残したい」場合は `fork_session=True` を使います。

```python
async def try_alternative(session_id: str):
    forked_id = None

    async for message in query(
        prompt="JWT の代わりに OAuth2 を使う場合の設計を提案して",
        options=ClaudeAgentOptions(
            allowed_tools=["Read"],
            resume=session_id,
            fork_session=True,    # ← 元のセッションを変えずに分岐
        ),
    ):
        if isinstance(message, ResultMessage):
            forked_id = message.session_id   # 新しい ID が発行される
            print(message.result)

    return forked_id
```

### 5. 同一プロセスでマルチターン会話をする

スクリプト内で複数回やり取りしたい場合は `ClaudeSDKClient` が便利です。

```python
import asyncio
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions, ResultMessage

async def main():
    async with ClaudeSDKClient(
        options=ClaudeAgentOptions(allowed_tools=["Read", "Edit", "Glob"])
    ) as client:
        # 1 回目
        await client.query("src/ ディレクトリの全 .py ファイルを一覧して")
        async for message in client.receive_response():
            if isinstance(message, ResultMessage):
                print("1回目:", message.result)

        # 2 回目（自動で同一セッション継続）
        await client.query("その中でテストファイルを除いた本体ファイルだけ教えて")
        async for message in client.receive_response():
            if isinstance(message, ResultMessage):
                print("2回目:", message.result)

asyncio.run(main())
```

### 6. コスト追跡

`ResultMessage` にはコストが含まれます。セッションをまたぐ場合は累計で管理しましょう。

```python
total_cost = 0.0

async for message in query(prompt="...", options=...):
    if isinstance(message, ResultMessage):
        if message.total_cost_usd is not None:   # エラー時は None になりうる
            total_cost += message.total_cost_usd
            print(f"このターン: ${message.total_cost_usd:.4f} / 累計: ${total_cost:.4f}")
        print(f"使用ターン数: {message.num_turns}")
```

### 7. 知っておくべき制限事項

| 項目 | 内容 |
|------|------|
| **コンテキストの累積** | 会話履歴はターンをまたいで積み上がる。長いセッションでは古い内容が要約（compact）される |
| **自動 compact の通知** | compact 発生時は `SystemMessage(subtype="compact_boundary")` が届く |
| **初期プロンプトの消失** | compact 後は最初の指示が消える可能性がある。重要なルールは `CLAUDE.md` に書く |
| **セッションの保存場所** | `~/.claude/projects/<cwd>/` 以下の `.jsonl` ファイル |
| **別マシンでの再開** | セッションファイルはローカル保存。別環境で再開するにはファイルの移動が必要 |

```python
# compact 境界を検知する
from claude_agent_sdk import SystemMessage

async for message in query(prompt="...", options=...):
    if isinstance(message, SystemMessage):
        if message.subtype == "compact_boundary":
            print("⚠ 会話履歴が要約されました。重要な文脈は再度伝えてください")
```

### 8. 練習課題

`agent-practice/` の中で：

1. 2 回の `query()` を連続して呼び、`session_id` で繋げる。1 回目で「`numbers.py` を読む」、2 回目で「さっき読んだファイルに新しい関数を追加して」を実行してください
2. `fork_session=True` で同じ起点から「足し算版」と「掛け算版」の 2 パターンの修正案を別々のセッションとして作成してください
3. `total_cost_usd` を累積してセッション全体のコストを表示してください

---

## Chapter 6: エラーハンドリングと堅牢化

### 1. エラーの発生場所は 3 つある

Claude Agent SDK で起きるエラーは発生場所によって対処が異なります。

```
① query() 自体の例外      … API 接続・認証失敗など
② ツール実行中のエラー     … カスタムツールが is_error を返す
③ ResultMessage の異常終了 … ターン上限・コスト上限・実行エラー
```

### 2. ① query() の例外をキャッチする

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage

async def safe_query(prompt: str):
    try:
        async for message in query(
            prompt=prompt,
            options=ClaudeAgentOptions(allowed_tools=["Read", "Bash"]),
        ):
            if isinstance(message, ResultMessage):
                return message.result

    except Exception as e:
        # ネットワーク障害・API キー不正・レート制限など
        print(f"❌ クエリ失敗: {type(e).__name__}: {e}")
        return None
```

> **よくある例外**:
> - `AuthenticationError` — API キー不正・未設定
> - `ConnectionError` — ネットワーク障害
> - `RateLimitError` — レート制限（しばらく待ってリトライ）

### 3. ③ ResultMessage の終了理由を必ず確認する

正常完了（`success`）以外の場合は `result` フィールドが空になります。

```python
async def run_task(prompt: str, max_turns: int = 20):
    async for message in query(
        prompt=prompt,
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit", "Bash"],
            max_turns=max_turns,
        ),
    ):
        if isinstance(message, ResultMessage):
            match message.subtype:
                case "success":
                    return message.result

                case "error_max_turns":
                    print(f"⚠ {max_turns} ターンで完了しませんでした。")
                    print("タスクを小さく分割するか、max_turns を増やしてください")

                case "error_max_budget_usd":
                    print("⚠ コスト上限に達しました")

                case "error_during_execution":
                    print("❌ 実行中にエラーが発生しました")

                case _:
                    print(f"⚠ 異常終了: {message.subtype}")

    return None
```

### 4. 上限を設定して暴走を防ぐ

上限を設けないと、複雑なタスクで Claude が何十ターンも動き続けることがあります。

```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Edit", "Bash"],
    max_turns=15,           # ターン上限（目安: 小タスク 5〜10、大タスク 20〜30）
    max_budget_usd=1.00,    # コスト上限 $1.00
)
```

| タスクの規模 | `max_turns` の目安 |
|-------------|-------------------|
| 単一ファイルの修正 | 5〜10 |
| 複数ファイルにまたがる変更 | 10〜20 |
| リポジトリ全体の分析 | 20〜30 |

### 5. よくある落とし穴 10 選

#### 落とし穴 1: ループを `break` で抜ける

```python
# ❌ ResultMessage の後にもシステムイベントが来る場合がある
async for message in query(...):
    if isinstance(message, ResultMessage):
        result = message.result
        break   # ← 途中で抜けると後処理が走らない

# ✅ 最後まで消費する
async for message in query(...):
    if isinstance(message, ResultMessage):
        result = message.result
        # break しない
```

#### 落とし穴 2: `total_cost_usd` が None になる

```python
# ❌ エラー終了時は None になりうる
total += message.total_cost_usd   # TypeError の可能性

# ✅ None ガードを入れる
if message.total_cost_usd is not None:
    total += message.total_cost_usd
```

#### 落とし穴 3: カスタムツールで例外を raise する

```python
# ❌ ループ全体が止まる
async def my_tool(args):
    raise ValueError("データなし")

# ✅ is_error で返してループを継続させる
async def my_tool(args):
    return {
        "content": [{"type": "text", "text": "データなし"}],
        "is_error": True,
    }
```

#### 落とし穴 4: `bypassPermissions` をローカルで使う

```python
# ❌ ローカルマシンで使うと意図しないファイル削除・コマンド実行が起きる
permission_mode="bypassPermissions"

# ✅ ローカルでは acceptEdits にとどめる
permission_mode="acceptEdits"
# bypassPermissions は Docker コンテナ・CI 環境のみ
```

#### 落とし穴 5: セッション再開後に初期指示が消える

```python
# ❌ compact 後は最初の prompt の指示が消える可能性がある
# ✅ 重要なルールは CLAUDE.md に書いておくか、resume 時の prompt で再度伝える
options = ClaudeAgentOptions(resume=session_id)
# → prompt で改めて制約を伝える
```

#### 落とし穴 6: `allowed_tools` にカスタムツールの完全名を指定し忘れる

```python
# ❌ カスタムツールはサーバー名込みの完全名が必要
allowed_tools=["get_inventory"]

# ✅ mcp__{server_name}__{tool_name} 形式で指定
allowed_tools=["mcp__inventory__get_inventory"]
```

#### 落とし穴 7: ツール説明文が短すぎる

```python
# ❌ Claude がいつ使うか判断できない
@tool("search", "検索する", {"query": str})

# ✅ 使いどころと戻り値を明記する
@tool(
    "search_products",
    "商品名・カテゴリ・価格帯で商品を検索する。"
    "「○○を探して」「××円以下の商品は？」などの照会タスクで使用。"
    "マッチした商品名と価格のリストを返す。",
    {"query": str, "max_price": float},
)
```

#### 落とし穴 8: Python 3.10 未満で動かそうとする

```bash
# エラー例
No matching distribution found for claude-agent-sdk

# 確認
python --version   # 3.10 以上が必要
```

#### 落とし穴 9: `effort` を上げすぎてコストが膨らむ

```python
# ❌ 単純なタスクに "max" を使うとコストが数倍になる
effort="max"

# ✅ タスクの複雑さに合わせる
effort="low"     # 単純な検索・確認タスク
effort="medium"  # 通常の開発タスク（デフォルト相当）
effort="high"    # 複雑な分析・設計タスク
```

#### 落とし穴 10: 同一セッションに無関係なタスクを詰め込む

```python
# ❌ 文脈が汚染されて精度が落ちる
await client.query("auth モジュールをレビューして")
await client.query("全く別のこと：README を英語に翻訳して")

# ✅ 無関係なタスクは別の query() セッションで実行する
async for m in query(prompt="auth モジュールをレビューして", ...): ...
async for m in query(prompt="README を英語に翻訳して", ...): ...   # 新しいセッション
```

### 6. 練習課題

`agent-practice/` の中で：

1. `max_turns=3` に設定して「`src/` 全体をリファクタリングして」を実行し、`error_max_turns` を発生させてみてください。次に `max_turns=20` で同じタスクを完了させてください
2. `is_error: True` を返すカスタムツールを作り、Claude が別の入力でリトライすることを確認してください
3. `try/except` で `query()` を包み、API キーを意図的に間違えた場合の例外をキャッチしてください

---

## Chapter 7: 次の一歩

ここまでで Claude Agent SDK の基本は身につきました。実務に組み込んでいくには次のトピックがあります。

### 1. 学んだことの振り返り

```
Chapter 0: Claude Agent SDK の位置づけ（CLI・Client SDK との違い）
Chapter 1: インストールと query() の初回成功体験
Chapter 2: エージェントループの 4 種類のメッセージと制御オプション
Chapter 3: 組み込みツールと permission_mode の使い分け
Chapter 4: @tool デコレーターでカスタムツールを定義する
Chapter 5: session_id で文脈を引き継ぐ・fork する
Chapter 6: ResultMessage の終了理由確認・落とし穴 10 選
```

### 2. まだ触れていない機能

本教材では取り上げなかった機能です。実務でスケールさせるときに使います。

#### サブエージェント（`Agent` ツール）

複雑なタスクを専門エージェントに分割して並列・逐次実行させる仕組みです。

```python
options = ClaudeAgentOptions(
    allowed_tools=["Agent"],
    sub_agents=[
        AgentDefinition(
            name="analyzer",
            description="コードの品質を分析する専門エージェント",
            prompt="シニアエンジニアとしてコードをレビューしてください",
            tools=["Read", "Glob", "Grep"],
        ),
    ],
)
```

#### フック（Hooks）

ツール呼び出しの前後に決定論的な処理を差し込みます。CLAUDE.md は「助言」ですが、フックは「必ず実行される」点が違います。

```
ユースケース例：
- Edit の前に必ず lint を走らせる
- 特定ディレクトリへの Write を禁止する
- Bash の実行コマンドをログに記録する
```

#### MCP サーバー統合

外部の MCP サーバー（GitHub・Slack・PostgreSQL・Notion など）を `mcp_servers` に追加して Claude に使わせられます。

```python
options = ClaudeAgentOptions(
    mcp_servers=[github_mcp_server, slack_mcp_server],
    allowed_tools=[
        "mcp__github__create_pull_request",
        "mcp__slack__send_message",
    ],
)
```

#### 構造化出力（Structured Outputs）

最終回答を JSON スキーマで受け取る機能です。後続処理でパースする手間が省けます。

```python
from pydantic import BaseModel

class ReviewResult(BaseModel):
    has_bugs: bool
    severity: str     # "low" / "medium" / "high"
    summary: str

options = ClaudeAgentOptions(
    allowed_tools=["Read"],
    output_schema=ReviewResult,
)
# ResultMessage.result が ReviewResult 型で返る
```

#### 非インタラクティブモード（CLI 経由）

SDK を使わず CLI から呼び出す場合は `-p` フラグで同等のことができます。

```bash
claude -p "src/ のバグを修正して" --allowedTools "Read,Edit,Bash" --output-format json
```

CI/CD の軽量な用途ではこちらの方が手軽です。

### 3. 実務に落とすコツ

- **最小ツールから始める**: `Read` だけで動作確認してから `Edit`・`Bash` を追加していく
- **`max_turns` は最初から設定する**: 上限なしで本番に出すと暴走するリスクがある
- **カスタムツールのエラーは `is_error` で返す**: `raise` するとループが止まる
- **セッションをまたぐ場合は文脈を再確認する**: compact 後に重要な指示が消えることがある
- **コストを定期的に確認する**: `total_cost_usd` を累計して予算管理する

### 4. 次の教材

→ **`claude-agent-sdk-best-practices.md`**（予定）に進んでください。以下が学べます：

- サブエージェントとオーケストレーターパターン
- フックで決定論的な処理を差し込む
- MCP サーバーの構築と統合
- 並列実行・バッチ処理の設計
- 本番デプロイとセキュアな運用
- チーム共有設定と失敗パターン集

---

## 関連リソース

- Agent SDK 概要: https://code.claude.com/docs/en/agent-sdk/overview
- クイックスタート: https://code.claude.com/docs/en/agent-sdk/quickstart
- カスタムツール: https://code.claude.com/docs/en/agent-sdk/custom-tools
- セッション管理: https://code.claude.com/docs/en/agent-sdk/sessions
- パーミッション: https://code.claude.com/docs/en/agent-sdk/permissions
- Python リファレンス: https://code.claude.com/docs/en/agent-sdk/python

### 関連教材

- `claude-code-tutorial.md` — Claude Code CLI の使い方（インタラクティブな開発）
- `claude-code-best-practices.md` — Claude Code の運用ノウハウ
