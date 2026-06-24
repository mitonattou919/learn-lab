# OpenAI Agents SDK チュートリアル（Python）

> OpenAI Agents SDK を初めて触る人向けのハンズオン教材。8 章を順に進めれば「エージェントを作る・ツールを与える・複数エージェントを連携させる」までが身につきます。
> 基本を習得したら → `openai-agents-sdk-best-practices.md` へ

---

## Chapter 0: OpenAI Agents SDK とは

### 1. 「エージェント」が必要になった背景

ChatGPT や GPT-4 の API は「テキストを渡すとテキストが返ってくる」一問一答の設計です。賢い返答はできても、**自分では何もしません**。

```
【従来の LLM API】
あなた: 「明日の東京の天気を教えて」
  ↓
GPT-4: 「私はリアルタイム情報にアクセスできないため…」

【Agents SDK を使ったエージェント】
あなた: 「明日の東京の天気を教えて」
  ↓
エージェント: [天気 API ツール実行]
  ↓
「明日の東京は晴れ、最高 27℃ の見込みです」
```

**OpenAI Agents SDK** は、この「自分でツールを呼んで、考えて、答えを出す」エージェントを Python で素早く作るためのフレームワークです。

### 2. 従来の API との比較

| | ChatGPT API 直接利用 | Agents SDK |
|--|---------------------|------------|
| 動作モデル | 一問一答 | ループ（考える→ツール呼ぶ→また考える） |
| ツール呼び出し | 自前で実装が必要 | デコレータ1行で定義 |
| 複数エージェント連携 | 自前で実装が必要 | `handoffs` で宣言するだけ |
| 構造化出力 | JSON モード + パース | Pydantic モデルを渡すだけ |
| 会話ループ管理 | 自前で実装が必要 | `Runner` が自動で担当 |

### 3. 「優秀なプロジェクトチーム」だと思って付き合う

- 指示（`instructions`）が明確なら確実にこなす
- 曖昧な指示には曖昧な動作が返る
- ツールに何を与えるかで能力が決まる
- 最終的な判断と責任は人間が持つ

### 4. SDK の 4 つの核心概念

```
Agent ──── ツールと指示を持つ LLM の「役割設定」
  │
  │  Runner に渡す
  ▼
Runner ─── ループを回して最終回答まで自動実行するエンジン
  │
  │  途中でツールを呼ぶ
  ▼
Tool ───── Python 関数を @function_tool で変換したもの
  │
  │  複雑なタスクは別エージェントへ
  ▼
Handoff ── 別の専門エージェントにタスクを引き渡す仕組み
```

---

## Chapter 1: 環境構築と初回実行

### 1. 必要な環境

| 項目 | 要件 | 確認コマンド |
|------|------|-------------|
| Python | **3.10 以上**（必須） | `python --version` |
| uv | 最新推奨 | `uv --version` |
| OpenAI API キー | [platform.openai.com](https://platform.openai.com/api-keys) で発行 | — |

uv が未インストールの場合：

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 2. プロジェクトの作成

```bash
uv init my-agent && cd my-agent
uv add openai-agents
```

インストール確認：

```bash
uv run python -c "import agents; print('OK')"
```

### 3. API キーの設定

`.env` ファイルを作成します（Git にコミットしないこと）：

```bash
echo "OPENAI_API_KEY=sk-..." > .env
```

`.gitignore` に追加：

```bash
echo ".env" >> .gitignore
```

### 4. 5 分で成功体験

`main.py` を作成してください：

```python
import os
from dotenv import load_dotenv
from agents import Agent, Runner

load_dotenv()  # .env から API キーを読み込む

agent = Agent(
    name="Assistant",
    instructions="あなたは親切なアシスタントです。日本語で簡潔に答えてください。",
)

result = Runner.run_sync(agent, "焼き芋ってなんですか？")
print(result.final_output)
```

```bash
uv add python-dotenv   # .env 読み込み用
uv run python main.py
```

**期待される出力例：**

```
焼き芋とは、サツマイモを焼いて調理した日本の伝統的な食べ物です。
甘くてほくほくとした食感が特徴で、秋冬の風物詩として親しまれています。
```

> **うまくいかない時**: `OPENAI_API_KEY` が設定されているか `echo $OPENAI_API_KEY` で確認。`.env` の読み込みに失敗している場合は `load_dotenv()` より前に `print(os.getenv("OPENAI_API_KEY"))` を挿入して確認してください。

### 5. 何が起きているのか

```
Runner.run_sync(agent, "焼き芋ってなんですか？")
        ↓
  1. ユーザー入力を GPT-4o に送る
  2. ツールを呼ぶ必要がなければ → 直接回答を生成
  3. result.final_output に最終回答が入る
```

`Runner.run_sync()` は同期関数なので、`asyncio` は不要です。これが Agents SDK の入門を簡単にしているポイントです。

### 6. 練習課題

1. `instructions` を「データ分析の専門家です」に変えて「標準偏差とは何ですか？」と聞いてみてください
2. `model="gpt-4o-mini"` を `Agent` に追加して、より安いモデルで動作することを確認してください
3. 日本語以外の質問（英語など）を渡したとき、どう応答するか試してみてください

---

## Chapter 2: Agent の設計

### 1. Agent クラスのパラメータ

`Agent` に渡せる主要パラメータの一覧です。

| パラメータ | 型 | 説明 |
|-----------|----|------|
| `name` | str | エージェントの識別名（必須） |
| `instructions` | str | エージェントの役割・行動指針（system prompt 相当） |
| `model` | str | 使用するモデル名（省略時は `gpt-4o`） |
| `tools` | list | 使用できるツールのリスト |
| `handoffs` | list | 委譲先のエージェントリスト |
| `output_type` | type | 構造化出力の型（Chapter 5 で詳しく解説） |

```python
from agents import Agent

agent = Agent(
    name="SalesAnalyst",
    model="gpt-4o-mini",
    instructions="""
あなたは売上データの分析を専門とするアナリストです。
数値を示す際は単位を明記し、増減がある場合はパーセンテージも示してください。
""",
)
```

### 2. 良い instructions の書き方

instructions の質がエージェントの動作品質を直接決めます。

| 悪い例 | 良い例 |
|--------|--------|
| `「アシスタントです」` | `「カスタマーサポート担当です。丁寧な敬語で返答し、解決できない場合は担当部署に案内してください」` |
| `「データを分析して」` | `「売上データを分析し、エグゼクティブサマリー→数値ハイライト→推奨アクションの順で報告してください」` |
| `「コードを直して」` | `「Python コードのバグ修正を担当します。修正前後のコードと、なぜ修正したかの理由を必ず説明してください」` |

#### 書くべきこと、書かなくていいこと

| ✅ 書く | ❌ 書かない |
|--------|-----------|
| この役割が何をするか | LLM が常識で知っていること |
| 出力のフォーマット・構成 | 「丁寧に答えてください」（デフォルトで丁寧） |
| ドメイン固有のルール | 一般的な作法・マナー |
| エラー時の振る舞い | 「プロとして」「最善を尽くして」等の抽象表現 |

### 3. モデルの選び方

| モデル | 速度 | コスト | 用途 |
|--------|------|--------|------|
| `gpt-4o-mini` | 速い | 安い | 開発・テスト・学習 |
| `gpt-4o` | 普通 | 普通 | 本番の汎用タスク |
| `o3-mini` | 遅い | 高い | 複雑な推論・数学 |

**学習中は `gpt-4o-mini` を使うとコストを抑えられます。**

### 4. 複数の会話ターンを続ける

同じエージェントに複数の質問を続ける場合は `input` リストで会話履歴を渡します。

```python
from agents import Agent, Runner
from openai.types.responses import ResponseInputParam

agent = Agent(
    name="Assistant",
    instructions="日本語で答えるアシスタントです。",
)

# 1回目の質問
result1 = Runner.run_sync(agent, "焼き芋とは何ですか？")
print(result1.final_output)

# 2回目: 前の会話履歴を引き継ぐ
result2 = Runner.run_sync(
    agent,
    input=result1.to_input_list() + [{"role": "user", "content": "どこで買えますか？"}],
)
print(result2.final_output)
```

`result.to_input_list()` が前回の会話履歴（ユーザー発言＋エージェント応答）をリストに変換してくれます。

### 5. 練習課題

1. `instructions` を工夫して「関西弁で話す料理研究家エージェント」を作り、レシピを聞いてみてください
2. `model="gpt-4o-mini"` と `model="gpt-4o"` で同じ質問をして、回答の違いを比較してください
3. 上記の「会話履歴を引き継ぐ」コードを使って、3 ターン以上の会話を続けてみてください

---

## Chapter 3: ツールを作る

### 1. なぜツールが必要か

Chapter 1〜2 のエージェントは「知識を持つ会話相手」でした。しかしリアルタイムデータ取得・計算・外部操作が必要な場面では限界があります。

```
【ツールなし】
「今月の売上合計を計算して」
→「申し訳ありませんが、実際のデータにアクセスできません」

【ツールあり】
「今月の売上合計を計算して」
→ [get_sales_data() 実行] → 「今月の売上合計は 4,480 万円です」
```

### 2. @function_tool の基本

```python
from agents import Agent, Runner, function_tool

@function_tool
def get_current_time() -> str:
    """現在の日時を返す。時刻に関する質問に使用する。"""
    from datetime import datetime
    return datetime.now().strftime("%Y年%m月%d日 %H:%M:%S")

agent = Agent(
    name="TimeAgent",
    instructions="ユーザーの質問に日本語で答えてください。",
    tools=[get_current_time],  # ← ここに渡す
)

result = Runner.run_sync(agent, "今何時ですか？")
print(result.final_output)
```

**ポイント：**
- デコレータ `@function_tool` を付けるだけで LLM が呼べるツールになる
- **docstring（`"""〜"""`）が必須** — LLM がいつ使うかを docstring で判断する
- 型ヒントを書くと LLM へのスキーマが自動生成される

### 3. 引数を持つツール

```python
from agents import function_tool

@function_tool
def get_sales_data(month: str) -> dict:
    """
    指定した月の売上データを取得する。
    売上レポート作成や月次比較の際に使用する。

    Args:
        month: 対象月（YYYY-MM 形式、例: "2024-01"）

    Returns:
        製品別の売上データ辞書
    """
    dummy_data = {
        "2024-01": {"製品A": 1200, "製品B": 980, "製品C": 1540},
        "2024-02": {"製品A": 1350, "製品B": 1020, "製品C": 1200},
    }
    if month not in dummy_data:
        return {"error": f"{month} のデータは存在しません"}  # 辞書でエラーを返す
    return {"month": month, "sales": dummy_data[month]}
```

> **エラーは例外でなく辞書で返す**: `raise ValueError(...)` するとエージェントのループが止まります。`{"error": "..."}` の形で返すと LLM が自分で処理してくれます。

### 4. 型ヒントとスキーマ

型ヒントから LLM に渡すスキーマが自動生成されます。Pydantic も使えます。

```python
from pydantic import BaseModel, Field
from agents import function_tool

class SalesQuery(BaseModel):
    month: str = Field(description="対象月（YYYY-MM形式）")
    product: str = Field(description="製品名。全製品なら 'all' を指定")

@function_tool
def query_sales(query: SalesQuery) -> dict:
    """売上データをクエリする。月と製品を指定して照会する。"""
    # ... 実装
    return {"result": "..."}
```

### 5. ツール設計の原則

| 原則 | 悪い例 | 良い例 |
|------|--------|--------|
| 1 ツール 1 責任 | `get_and_analyze_and_format()` | `get_data()` / `calculate_stats()` / `format_report()` |
| docstring に「いつ使うか」を書く | `"""データを取得する"""` | `"""売上レポート作成前に呼ぶ。月次データを取得する。"""` |
| エラーは辞書で | `raise ValueError(...)` | `return {"error": "..."}` |
| 型ヒントを省略しない | `def get_data(month):` | `def get_data(month: str) -> dict:` |

### 6. 練習課題

1. `calculate_discount(price: int, rate: float) -> int` というツールを作り、「1000円の商品を 20% 引きにした値段は？」と聞いてみてください
2. `get_weather(city: str) -> dict` というツールを作り、ダミーデータを返すようにしてみてください（実際に API を呼ばなくてよい）
3. docstring を空にしたときと詳しく書いたときで、LLM のツール選択にどう影響するか試してみてください

---

## Chapter 4: Runner による実行制御

### 1. Runner の 3 つのモード

| メソッド | 動作 | 使いどころ |
|---------|------|-----------|
| `Runner.run_sync()` | 同期・完了まで待つ | スクリプト、CLI ツール |
| `Runner.run()` | 非同期・完了まで待つ | FastAPI 等の async フレームワーク |
| `Runner.run_streamed()` | 非同期・逐次ストリーム | チャット UI、リアルタイム表示 |

### 2. run_sync（同期実行）

最もシンプルな使い方です。

```python
from agents import Agent, Runner

agent = Agent(name="Assistant", instructions="日本語で答えてください。")

result = Runner.run_sync(agent, "Pythonの特徴を3つ教えて")
print(result.final_output)
```

### 3. run（非同期実行）

FastAPI や他の async 環境で使います。

```python
import asyncio
from agents import Agent, Runner

agent = Agent(name="Assistant", instructions="日本語で答えてください。")

async def main():
    result = await Runner.run(agent, "Pythonの特徴を3つ教えて")
    print(result.final_output)

asyncio.run(main())
```

> **Jupyter Notebook で使う場合**: `asyncio.run()` が使えない環境では `await main()` を直接実行するか、`nest_asyncio` を使ってください。

### 4. run_streamed（ストリーミング実行）

チャット UI のように「考えながら表示する」用途に使います。

```python
import asyncio
from agents import Agent, Runner

agent = Agent(name="Assistant", instructions="日本語で答えてください。")

async def main():
    result = Runner.run_streamed(agent, "Pythonの特徴を5つ教えて")
    async for event in result.stream_events():
        if event.type == "raw_response_event":
            # テキストをリアルタイムで表示
            pass
    print(result.final_output)

asyncio.run(main())
```

### 5. max_turns でループを制限する

エージェントはツールを呼ぶ→考える→またツールを呼ぶ…を繰り返します。これが想定外に長引かないよう上限を設定できます。

```python
from agents import Agent, Runner, RunConfig

agent = Agent(name="Assistant", instructions="日本語で答えてください。")

result = Runner.run_sync(
    agent,
    "今月の売上を分析して",
    run_config=RunConfig(max_turns=10),  # 最大 10 ターン
)
```

デフォルトは 10 ターンです。ツールを多用するエージェントでは増やすことがあります。

### 6. RunResult の中身

`result` オブジェクトから取り出せる主な情報：

```python
result = Runner.run_sync(agent, "...")

result.final_output        # 最終回答（str）
result.to_input_list()     # 会話履歴（次の会話継続に使う）
result.last_agent          # 最後に応答したエージェント
```

### 7. 練習課題

1. `run_sync` と `run` の両方で同じ質問をして、同じ結果が得られることを確認してください
2. `max_turns=2` に設定し、ツールを複数回呼ぶ必要があるタスクがどうなるか試してください
3. `run_streamed` を使って、返答が少しずつ表示されるチャット風の表示を実装してみてください

---

## Chapter 5: 構造化出力

### 1. なぜ構造化出力が必要か

```python
# 文字列で返ってくる場合
result = Runner.run_sync(agent, "田中太郎さんの情報を教えて")
print(result.final_output)
# → "田中太郎さんは30歳で、東京在住のエンジニアです。"
# → このあと名前・年齢・職業を取り出すのが大変

# 構造化出力なら
# → result.final_output.name == "田中太郎"
# → result.final_output.age == 30
# → result.final_output.job == "エンジニア"
```

### 2. Pydantic モデルで出力型を定義する

```python
from pydantic import BaseModel, Field
from agents import Agent, Runner

class PersonInfo(BaseModel):
    name: str = Field(description="氏名")
    age: int = Field(description="年齢")
    city: str = Field(description="居住都市")
    job: str = Field(description="職業")

agent = Agent(
    name="InfoExtractor",
    instructions="テキストから人物情報を抽出してください。",
    output_type=PersonInfo,  # ← ここで型を指定
)

result = Runner.run_sync(
    agent,
    "田中太郎（30歳）は東京在住のエンジニアです。"
)

# result.final_output は PersonInfo 型のオブジェクト
info = result.final_output
print(f"名前: {info.name}")   # → 田中太郎
print(f"年齢: {info.age}")    # → 30
print(f"都市: {info.city}")   # → 東京
print(f"職業: {info.job}")    # → エンジニア
```

### 3. リスト・ネスト構造にも対応

```python
from typing import List
from pydantic import BaseModel, Field
from agents import Agent, Runner

class Product(BaseModel):
    name: str
    sales: int
    rank: int

class SalesReport(BaseModel):
    month: str = Field(description="対象月（YYYY-MM 形式）")
    total: int = Field(description="合計売上（万円）")
    top_products: List[Product] = Field(description="売上上位製品リスト")
    summary: str = Field(description="一言サマリー")

agent = Agent(
    name="ReportGenerator",
    instructions="売上データを分析して構造化レポートを作成してください。",
    output_type=SalesReport,
)
```

### 4. 構造化出力を使うケース

| 使う場面 | 例 |
|---------|-----|
| データ抽出 | テキストから名前・日付・金額を取り出す |
| 分類 | メールをカテゴリに分類する |
| レポート生成 | 分析結果を決まったフォーマットで出力 |
| 後段処理への受け渡し | 次の処理がフィールドで参照できる |

> **注意**: `output_type` を指定すると `result.final_output` の型が変わります。文字列でなく Pydantic オブジェクトになるので、`result.final_output.field_name` でアクセスします。

### 5. 練習課題

1. `RecipeInfo` という Pydantic モデル（料理名・材料リスト・手順・調理時間）を定義して、「カレーの作り方」を構造化出力で取得してください
2. `List[str]` を `output_type` に指定して、「Python の特徴を 5 つ」を文字列リストとして取得してください
3. `Field(description=...)` を省略したときと書いたときで、出力の精度に差が出るか試してみてください

---

## Chapter 6: マルチエージェントと Handoff

### 1. なぜマルチエージェントか

1 つのエージェントにすべてのタスクを任せると、`instructions` が長くなって動作が不安定になります。役割を分離することで品質が上がり、管理も楽になります。

```
【単一エージェントの問題】
「データ取得・分析・レポート作成・品質チェックを全部やれ」
→ instructions が肥大化 → どれも中途半端

【マルチエージェントの解決策】
トリアージエージェント
   ├── データエージェント   ← DB からデータ取得
   ├── 分析エージェント     ← 統計分析・洞察抽出
   └── レポートエージェント ← 文書生成
```

### 2. Handoff の基本

エージェントを `handoffs` リストに渡すだけで委譲が設定できます。

```python
from agents import Agent, Runner

# 専門エージェントを定義
billing_agent = Agent(
    name="BillingAgent",
    instructions="請求・支払いに関する問い合わせを担当します。",
)

support_agent = Agent(
    name="SupportAgent",
    instructions="技術的なサポート問い合わせを担当します。",
)

# トリアージエージェントが適切なエージェントに振り分ける
triage_agent = Agent(
    name="TriageAgent",
    instructions="""
あなたはカスタマーサービスの受付担当です。
問い合わせ内容を判断して、適切な担当エージェントに転送してください。
""",
    handoffs=[billing_agent, support_agent],  # ← 委譲先を登録
)

result = Runner.run_sync(triage_agent, "先月の請求書の金額が間違っているようです")
print(result.final_output)
```

LLM が問い合わせ内容を判断して `billing_agent` に自動的に委譲します。

### 3. エージェントの description を書く

`description` はトリアージエージェントが「どのエージェントに委譲すべきか」を判断するために使われます。

```python
# 悪い例：曖昧すぎて判断できない
billing_agent = Agent(
    name="BillingAgent",
    description="データを処理するエージェント",  # ← 何をするのか不明
)

# 良い例：トリガーワードと担当範囲を明記
billing_agent = Agent(
    name="BillingAgent",
    description="""
請求・支払い・領収書・返金に関する問い合わせを専門とするエージェント。
「請求金額が違う」「支払い方法を変えたい」「返金したい」などを担当する。
""",
)
```

### 4. Handoff のカスタマイズ

`handoff()` 関数を使うと委譲時の動作を細かく設定できます。

```python
from agents import Agent, handoff
from agents.extensions.handoff_filters import remove_all_tools

refund_agent = Agent(
    name="RefundAgent",
    instructions="返金処理を担当します。",
)

triage_agent = Agent(
    name="TriageAgent",
    instructions="問い合わせを振り分けます。",
    handoffs=[
        handoff(
            refund_agent,
            input_filter=remove_all_tools,  # ツール呼び出し履歴を除いて渡す
        )
    ],
)
```

### 5. マルチエージェントの設計パターン

| パターン | 構造 | 向いているケース |
|---------|------|----------------|
| **トリアージ** | 受付 → 専門エージェント | カスタマーサポート、問い合わせ分類 |
| **パイプライン** | A → B → C → 最終回答 | データ取得 → 分析 → レポート生成 |
| **並列実行** | 親 → [A, B, C] 同時 → 集約 | 複数データソースの同時取得 |

### 6. どのエージェントが動いたか確認する

```python
result = Runner.run_sync(triage_agent, "...")
print(f"最後に応答したエージェント: {result.last_agent.name}")
```

### 7. 練習課題

1. 「天気担当」と「料理担当」の 2 つの専門エージェントを作り、トリアージエージェントが適切に振り分けるか確認してください
2. `description` を省略したときと詳しく書いたときで、委譲の正確さがどう変わるか試してください
3. `result.last_agent.name` を使って、どのエージェントが最終回答を出したか表示してください

---

## Chapter 7: 次の一歩

ここまでで OpenAI Agents SDK の基本は身につきました。実務に組み込んでいくには次のトピックがあります。

### Guardrails（ガードレール）

入出力を検証してリスクのある応答をブロックする仕組みです。

```python
from pydantic import BaseModel
from agents import Agent, Runner, GuardrailFunctionOutput, input_guardrail

class SafetyCheck(BaseModel):
    is_safe: bool
    reason: str

@input_guardrail
async def check_input(ctx, agent, input) -> GuardrailFunctionOutput:
    """不適切な入力をブロックする"""
    # 安全チェックのロジック
    return GuardrailFunctionOutput(output_info=SafetyCheck(is_safe=True, reason="OK"))
```

### ホスト済みツール

OpenAI が提供するすぐに使えるツール群です。

```python
from agents import Agent
from agents.tools import WebSearchTool, CodeInterpreterTool

agent = Agent(
    name="ResearchAgent",
    instructions="Web 検索を使ってユーザーの質問に答えてください。",
    tools=[WebSearchTool()],  # ← Google 検索相当
)
```

| ツール | 機能 |
|--------|------|
| `WebSearchTool` | Web 検索 |
| `FileSearchTool` | ファイル・ドキュメント検索（Vector Store） |
| `CodeInterpreterTool` | Python コードを安全な環境で実行 |
| `ImageGenerationTool` | DALL-E で画像生成 |

### 非同期・ストリーミングの本番活用

```bash
# FastAPI との組み合わせ
uv add fastapi uvicorn
```

```python
from fastapi import FastAPI
from agents import Agent, Runner

app = FastAPI()
agent = Agent(name="Assistant", instructions="...")

@app.post("/chat")
async def chat(message: str):
    result = await Runner.run(agent, message)
    return {"response": result.final_output}
```

### もっと深く学ぶ

→ **`openai-agents-sdk-best-practices.md`** に進んでください。以下が学べます：

- **RunConfig** でエージェント全体の設定を制御する
- **Context** を使ってエージェント間でデータを共有する
- **トレーシング** でデバッグ・モニタリングする
- **MCP（Model Context Protocol）** でツールを外部サーバーに切り出す
- **本番設計パターン**（リトライ・タイムアウト・コスト管理）

### 公式リソース

- 公式ドキュメント: https://openai.github.io/openai-agents-python/
- GitHub リポジトリ: https://github.com/openai/openai-agents-python
- サンプルコード: https://github.com/openai/openai-agents-python/tree/main/examples

### 関連教材

- `google-adk-lesson.md` — Google 版エージェントフレームワーク（比較参考に）

---

## 関連リソース

- **公式ドキュメント**: https://openai.github.io/openai-agents-python/
- **クイックスタート**: https://openai.github.io/openai-agents-python/quickstart/
- **Agent リファレンス**: https://openai.github.io/openai-agents-python/agents/
- **Runner リファレンス**: https://openai.github.io/openai-agents-python/running_agents/
- **Tools リファレンス**: https://openai.github.io/openai-agents-python/tools/
- **Handoffs リファレンス**: https://openai.github.io/openai-agents-python/handoffs/
- **GitHub リポジトリ**: https://github.com/openai/openai-agents-python
