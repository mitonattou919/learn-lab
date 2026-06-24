# OpenAI Agents SDK ベストプラクティス

> **対象読者**: [openai-agents-sdk-tutorial.md](openai-agents-sdk-tutorial.md) を一通りこなし、基本操作（Agent・Runner・ツール・Handoff）を把握している方。

---

## Chapter 0: アジェンティックループとは

### 1. 従来の API 呼び出しとの違い

「エージェント」とは単なる LLM の呼び出しではありません。LLM が次に何をするかを自分で判断し、ツール呼び出し・結果の処理・再判断を繰り返す「ループ」を持つシステムです。

| 観点 | 従来の API 呼び出し | Agents SDK |
|------|------------------|-----------|
| 制御の主体 | アプリコードが逐次呼び出す | LLM が次の行動を自律判断 |
| ツール実行 | コードが明示的に呼ぶ | LLM が「必要」と判断したときに呼ぶ |
| 終了条件 | 呼び出し1回で終わり | LLM が「完了」と判断するか max_turns に達するまで継続 |
| 会話管理 | アプリが履歴を管理 | SDK が自動でコンテキストを維持 |

```
[アジェンティックループの流れ]

ユーザー入力
    ↓
  LLM が考える
    ├─→ ツール不要 → 最終回答を返す（終了）
    └─→ ツール必要 → ツール実行 → 結果を LLM に戻す → LLM が考える（繰り返し）
```

### 2. 非決定性と向き合う 3 原則

アジェンティックループは**非決定的**です。同じ入力でも LLM の判断が変われば出力が変わります。これと向き合う 3 つの手段があります。

| 原則 | 手段 | 効果 |
|------|------|------|
| **上限を設ける** | `max_turns` | ループが無限に続くのを防ぐ |
| **出力を型で縛る** | `output_type` | 最終出力の構造を保証する |
| **見えるようにする** | Tracing | 何が起きたかを後から追える |

この 3 原則は開発の初期から意識する価値があります。後から追加しようとすると設計の見直しが必要になりがちです。

### 3. コスト・レイテンシ・品質のトレードオフ

| 最適化したいもの | 主な手段 | トレードオフ |
|----------------|---------|------------|
| **コスト削減** | 安いモデル・max_turns 削減・ツール数削減 | 品質・応答範囲が下がる |
| **レイテンシ削減** | 並列実行・軽量モデル・ストリーミング | コスト増・設計複雑化 |
| **品質向上** | 高品質モデル・詳細な instructions・output_type | コスト増・レイテンシ増 |
| **安全性向上** | Guardrail・max_turns・副作用の制限 | レイテンシ増・柔軟性低下 |

三者を同時に最大化することはできません。**何を犠牲にして何を優先するか** をユースケースごとに決めてから設計に入ります。

### 4. 「エージェントを使わない」選択肢

エージェントはすべての問題に適しているわけではありません。

| エージェントが向く | 従来の API 呼び出しが向く |
|----------------|----------------------|
| 入力が非構造化でどのツールを呼ぶか事前に決められない | 処理の順序が完全に固定されている |
| 複数ステップの推論が必要 | 1回の LLM 呼び出しで完結する |
| ユーザーとの対話的なやり取りがある | バッチ処理で予測可能な出力が必要 |
| ツールの組み合わせが動的に変わる | コストとレイテンシを厳格に管理したい |

エージェントの「柔軟性」はコストとレイテンシの増加と常にトレードオフです。シンプルなユースケースに無理にエージェントを使うと、保守コストが上がるだけです。

---

## Chapter 1: エージェント設計の原則

### 1. instructions の品質がすべてを決める

`instructions` はエージェントの system prompt 相当です。曖昧な指示は曖昧な動作を生みます。

| 観点 | 悪い例 | 良い例 |
|------|--------|--------|
| 役割 | `「アシスタントです」` | `「カスタマーサポート担当。敬語で回答し、解決できない場合は担当部署へ案内する」` |
| 出力形式 | `「分析して」` | `「エグゼクティブサマリー→数値ハイライト→推奨アクションの順で報告する」` |
| 制約 | なし | `「外部ツールの結果が空の場合は '不明' と返す。推測で埋めない」` |
| 完了条件 | なし | `「必ずツールの結果に基づいて回答する。ツールを1回も呼ばずに回答してはいけない」` |

⭐ **instructions に「完了条件」と「禁止事項」を書く**のが最重要です。LLM は明示しない限り常識的に振る舞おうとしますが、その「常識」がタスク固有の要件と合わないことがよくあります。

#### 動的 instructions

`instructions` に関数を渡すと、実行ごとに context を参照して動的に生成できます。

```python
from datetime import datetime
from agents import Agent, RunContextWrapper
from dataclasses import dataclass

@dataclass
class UserInfo:
    name: str
    role: str

def build_instructions(context: RunContextWrapper[UserInfo], agent: Agent) -> str:
    return f"""
あなたは {context.context.name} 専任のアナリストです。
ユーザーの権限レベル: {context.context.role}
本日の日付: {datetime.now().strftime('%Y-%m-%d')}
"""

agent = Agent(name="Analyst", instructions=build_instructions)
```

### 2. output_type で出力を構造化する

`output_type` を指定しない場合、`result.final_output` は `str` です。後続処理でのパース失敗・型エラーを防ぐため、Pydantic モデルを使いましょう。

```python
from pydantic import BaseModel, Field
from agents import Agent, Runner

class SalesReport(BaseModel):
    summary: str = Field(description="3行以内のエグゼクティブサマリー")
    total_sales: int = Field(description="合計売上（万円）")
    top_product: str = Field(description="売上1位の製品名")
    risk_flag: bool = Field(description="前月比 20% 以上の下落がある場合 True")

agent = Agent(
    name="SalesAnalyst",
    instructions="売上データを分析して構造化レポートを返す。",
    output_type=SalesReport,
)

result = Runner.run_sync(agent, "2024年1月の売上を分析して")
report = result.final_output   # SalesReport 型が保証される
if report.risk_flag:
    send_alert(report.summary)
```

> **注意**: `output_type` を指定したエージェントを Handoff の途中に置く場合は、そのエージェントが structured output を返すことを意図しているか明確にしてください。最終エージェントか単独実行向けとして設計するのがシンプルです。

### 3. モデル選択の実践指針

モデルはエージェント単位・RunConfig 単位で変更できます。マルチエージェントでは**役割に応じてモデルを使い分ける**のが鉄則です。

```python
# トリアージ：安い・速いモデルで振り分けだけ行う
triage_agent = Agent(name="Triage", model="gpt-5.4-mini", ...)

# 分析：複雑な推論が必要なので高品質モデル
analysis_agent = Agent(name="Analyst", model="gpt-5.5", ...)

# レポート生成：文章品質重視
report_agent = Agent(name="Reporter", model="gpt-5.5", ...)
```

| モデル | 使いどころ |
|--------|-----------|
| `gpt-5.4-mini` | トリアージ・分類・単純なツール呼び出し |
| `gpt-5.5` | 複雑な推論・コード生成・高品質な文章 |
| 推論モデル（`o3` 等） | 数学・論理的推論・段階的な問題解決 |

> **注意**: モデル名は SDK バージョンにより変わります。常に [公式ドキュメント](https://openai.github.io/openai-agents-python/models/) を確認してください。

### 4. エージェントの粒度を決める判断基準

1 エージェントに詰め込みすぎると instructions が肥大化し、動作が不安定になります。

| 分割すべき兆候 | 分割しなくてよい兆候 |
|--------------|-------------------|
| instructions が 500 字を超えている | 単一のドメイン・役割で完結する |
| 役割が「取得」「分析」「出力」など複数ある | ツールが 3 個以下 |
| 特定のステップだけ高品質モデルが必要 | マルチエージェントの調整コストが高い |
| 並列実行でレイテンシを削減したい | 要件がシンプルで変化しない |

---

## Chapter 2: ツール設計のベストプラクティス

### 1. docstring と型ヒントはツールの「仕様書」

`@function_tool` デコレータは docstring と型ヒントから LLM に渡すスキーマを自動生成します。ここが粗いと LLM が「いつ使うか」「何を渡すか」を誤判断します。

```python
from agents import function_tool

# 悪い例：LLM がいつ使えばよいか判断できない
@function_tool
def get_data(month: str) -> dict:
    """データを取得する"""
    ...

# 良い例：使用場面・引数・戻り値が明確
@function_tool
def get_sales_data(month: str) -> dict:
    """
    指定した月の製品別売上データを取得する。
    売上レポート作成・前月比分析・上位製品特定の前処理として使用する。

    Args:
        month: 対象月（YYYY-MM 形式、例: "2024-01"）

    Returns:
        製品名をキー、売上金額（万円）を値とする辞書と合計額
    """
    ...
```

⭐ **「いつ使うか」「使わない場面はどこか」を docstring に書く**のが最重要です。類似ツールが複数ある場合は特に、使い分けを明記してください。

### 2. エラーを例外でなく値で返す

ツールが例外を投げると、デフォルトではエージェントのループが止まります。エラー情報を戻り値に含めると、LLM が自律的にリカバリーできます。

```python
# 悪い例：ループが止まる
@function_tool
def get_sales_data(month: str) -> dict:
    if month not in data:
        raise ValueError(f"{month} のデータがありません")  # NG
    return data[month]

# 良い例：LLM がエラーを処理できる
@function_tool
def get_sales_data(month: str) -> dict:
    if month not in data:
        return {"error": f"{month} のデータは存在しません。利用可能な月: {list(data.keys())}"}
    return data[month]
```

例外をあえて投げたい場合（致命的なエラーでループを強制終了させたい場合）は、`Runner.run()` の `error_handlers` 引数で適切に処理します（Chapter 3 参照）。

### 3. Pydantic Field で入力バリデーション

型ヒントだけでなく `Annotated` + `Field` で入力制約を付けると、不正な引数でツールが呼ばれることを防げます。

```python
from typing import Annotated
from pydantic import Field
from agents import function_tool

@function_tool
def update_inventory(
    product_id: Annotated[str, Field(pattern=r"^[A-Z]{2}\d{4}$", description="製品ID（例: AB1234）")],
    quantity: Annotated[int, Field(ge=0, le=10000, description="在庫数（0〜10000）")],
) -> dict:
    """在庫数を更新する。製品ID と更新後の数量を指定する。"""
    ...
```

### 4. 冪等性と副作用の管理

**冪等性**（何度呼んでも同じ結果になること）はループ内のツールにとって重要な性質です。LLM が同じツールを複数回呼ぶ可能性があります。

| ツールの種類 | 冪等性 | 推奨対策 |
|------------|--------|---------|
| データ取得 | ✅ 自然に冪等 | 特になし（キャッシュでパフォーマンス改善可） |
| データ更新 | ❌ 冪等でない | 操作 ID・べき等キーを引数に追加 |
| メール送信 | ❌ 冪等でない | 送信済みフラグ管理・MCP ツールの `needs_approval` 設定 |
| ファイル作成 | △ 上書きなら可 | ファイル名に操作 ID を含める |

副作用の大きいツール（外部送信・DB 書き込み等）への対策は目的で使い分けます。**人間による承認が必要**なら MCP ツールの `needs_approval` 設定を使い、**入力の自動検査・ブロック**が目的なら `input_guardrail` を使います。

### 5. 1 ツール 1 責任

```python
# 悪い例：取得・分析・フォーマットを 1 ツールに詰め込む
@function_tool
def get_and_analyze_and_format_sales(month: str) -> str:
    data = fetch_data(month)
    analysis = calculate_stats(data)
    return format_report(analysis)

# 良い例：責任を分離して LLM に組み合わせさせる
@function_tool
def get_sales_data(month: str) -> dict:
    """指定月の売上データを取得する。"""
    ...

@function_tool
def calculate_stats(numbers: list[float]) -> dict:
    """数値リストの統計量（平均・中央値・最大・最小）を計算する。"""
    ...

@function_tool
def format_as_markdown_table(data: dict, title: str) -> str:
    """辞書データをマークダウンテーブル形式に変換する。"""
    ...
```

分離すると LLM が必要なツールだけを選んで呼べるようになり、不要なツール実行のコストを削減できます。

---

## Chapter 3: RunConfig による実行制御

### 1. RunConfig の全体像

`RunConfig` はランの動作をグローバルに制御するオブジェクトです。エージェント側の設定を上書き・補完します。

```python
from agents import Agent, Runner, RunConfig, ModelSettings

result = Runner.run_sync(
    agent,
    "売上を分析して",
    max_turns=10,
    run_config=RunConfig(
        model="gpt-5.5",                # エージェントのモデルを上書き
        model_settings=ModelSettings(
            temperature=0.2,            # 出力を安定させる
            parallel_tool_calls=True,   # ツールを並列呼び出し
        ),
        workflow_name="sales_analysis", # Tracing でのグループ名
        trace_include_sensitive_data=False,  # 機密データをトレースから除外
    ),
)
```

### 2. max_turns とエラーハンドラ

`max_turns` を超えると `MaxTurnsExceeded` 例外が送出されます。

```python
from agents import MaxTurnsExceeded

# 例外をキャッチする場合
try:
    result = Runner.run_sync(agent, "...", max_turns=5)
except MaxTurnsExceeded:
    print("ターン上限に達しました")
```

### 3. ModelSettings で出力を安定させる

```python
from agents import ModelSettings

careful_settings = ModelSettings(
    temperature=0.1,           # 出力の一貫性向上（0=決定的、1=多様）
    parallel_tool_calls=False, # ツールを逐次呼び出し（デバッグ・依存関係がある場合）
    truncation="auto",         # コンテキストが長すぎる場合の自動切り詰め
)
```

| パラメータ | 推奨値 | 効果 |
|-----------|--------|------|
| `temperature` | 0.0〜0.3 | 構造化タスク・分析で安定した出力 |
| `temperature` | 0.7〜1.0 | 創作・要約で多様な出力 |
| `parallel_tool_calls` | `False` | ツール間に依存関係がある場合の安全策 |
| `truncation` | `"auto"` | 長い会話でのエラーを防ぐ |

### 4. 会話履歴管理の 4 戦略

複数ターンにまたがる会話を管理する方法は 4 つあります。

| 戦略 | 方法 | 向いている場面 |
|------|------|--------------|
| **手動管理** | `result.to_input_list()` を次の入力に追加 | 小規模・ステートレスな設計 |
| **セッション** | `SQLiteSession` 等でストレージに保存 | 複数セッションをまたぐ継続会話 |
| **サーバー管理** | `conversation_id` を指定 | OpenAI のサーバーに履歴を任せる |
| **軽量チェーン** | `previous_response_id` を渡す | 直前の応答だけ引き継ぐシンプルな設計 |

```python
# 手動管理の例
result1 = Runner.run_sync(agent, "1月の売上を教えて")
result2 = Runner.run_sync(
    agent,
    input=result1.to_input_list() + [{"role": "user", "content": "2月も教えて"}],
)
```

> **注意**: `input_filter` を使ってハンドオフ時に履歴を絞ることが重要です。すべての履歴を引き継ぐとコンテキストが膨れ上がりトークンコストが増大します（Chapter 5 参照）。

### 5. RunResult から取り出せる情報

```python
result = Runner.run_sync(agent, "...")

result.final_output          # 最終回答（str または output_type のオブジェクト）
result.last_agent            # 最後に応答したエージェント
result.new_items             # ターン中の全イベント（ToolCallItem, HandoffCallItem 等）
result.raw_responses         # 各 LLM 呼び出しの生レスポンス（診断用）
result.to_input_list()       # 次ターンの入力用リスト
```

`new_items` を走査すると「どのツールが何回呼ばれたか」「ハンドオフが発生したか」などを検証できます。

```python
from agents.items import ToolCallItem, HandoffCallItem

for item in result.new_items:
    if isinstance(item, ToolCallItem):
        print(f"ツール呼び出し: {item.tool_name} (call_id: {item.call_id})")
    if isinstance(item, HandoffCallItem):
        print(f"ハンドオフ発生")

# ハンドオフ先の確認は result.last_agent を使う
print(f"最終エージェント: {result.last_agent.name}")
```

---

## Chapter 4: Context でエージェント間データを共有する

### 1. Context とは何か

ツール間・エージェント間でデータを受け渡すには、引数の連鎖（ツール→LLM→次のツール）に頼るか、**Context オブジェクト**を使うかの 2 択があります。

| 手法 | 仕組み | 向いている場面 |
|------|--------|--------------|
| 引数連鎖 | LLM がツール出力を次の入力に組み込む | 公開して問題ないデータ |
| **Context** | LLM を介さずに直接オブジェクトを共有 | 機密情報・設定値・認証情報 |

Context は LLM に**送られません**。アプリが管理するローカル専用のコンテナです。

### 2. Context の定義と渡し方

dataclass か Pydantic モデルを定義して `Runner.run()` の `context` 引数に渡します。

```python
from dataclasses import dataclass, field
from agents import Agent, Runner, RunContextWrapper, function_tool

@dataclass
class AppContext:
    user_id: str
    role: str                   # "admin" | "user"
    db_connection: object       # DB 接続オブジェクト（LLM に見せたくない）
    audit_log: list = field(default_factory=list)

@function_tool
async def get_user_data(wrapper: RunContextWrapper[AppContext], target_id: str) -> dict:
    """指定ユーザーの情報を取得する。"""
    ctx = wrapper.context
    if ctx.role != "admin" and target_id != ctx.user_id:
        return {"error": "権限がありません"}
    return fetch_from_db(ctx.db_connection, target_id)

agent = Agent(name="DataAgent", tools=[get_user_data])

context = AppContext(user_id="u123", role="admin", db_connection=conn)
result = Runner.run_sync(agent, "ユーザー u456 の情報を取得して", context=context)
```

### 3. Context をミュータブルに使う

Context オブジェクトはミュータブルです。ツールが書き込んだデータを後続のツールが読める「共有メモ帳」として活用できます。

```python
@dataclass
class PipelineContext:
    raw_data: dict | None = None
    analysis: dict | None = None

@function_tool
def fetch_data(wrapper: RunContextWrapper[PipelineContext], month: str) -> dict:
    """データを取得してコンテキストに保存する。"""
    data = query_db(month)
    wrapper.context.raw_data = data   # 後続ツールが参照できる
    return data

@function_tool
def analyze_data(wrapper: RunContextWrapper[PipelineContext]) -> dict:
    """コンテキストのデータを分析する。前に fetch_data を呼んでいること。"""
    if wrapper.context.raw_data is None:
        return {"error": "先に fetch_data を呼んでください"}
    analysis = run_analysis(wrapper.context.raw_data)
    wrapper.context.analysis = analysis
    return analysis
```

> ⚠ LLM はコンテキストの内容を知らないため、ツールの docstring に「先に XXX を呼ぶこと」と明記して誘導してください。

### 4. ToolContext で詳細情報にアクセスする

```python
from agents import ToolContext

@function_tool
async def my_tool(ctx: ToolContext[AppContext]) -> str:
    """サンプルツール。"""
    print(f"ツール名: {ctx.tool_name}")
    print(f"呼び出し ID: {ctx.tool_call_id}")
    print(f"使用トークン: {ctx.usage.total_tokens}")
    print(f"ユーザー ID: {ctx.context.user_id}")
    return "OK"
```

### 5. 実行後に Context を参照する

```python
context = PipelineContext()
result = Runner.run_sync(agent, "分析して", context=context)

print(context.raw_data)      # fetch_data が書いた値
print(context.analysis)      # analyze_data が書いた値
print(result.final_output)   # エージェントの最終回答
```

---

## Chapter 5: マルチエージェント設計パターン

### 1. 4 つのパターン概観

| パターン | 制御の主体 | 向いている場面 |
|---------|----------|--------------|
| **トリアージ型（Handoff）** | LLM が委譲先を判断 | 入力に応じて異なる専門家にルーティングしたい |
| **パイプライン型** | コードが順序を制御 | ステップの順序が固定されている |
| **並列型** | コードが並列実行 | 独立したタスクを同時に処理したい |
| **Agents-as-Tools** | 親エージェントの LLM が判断 | サブタスクをブラックボックスとして扱いたい |

### 2. トリアージ型（Handoff）

最もシンプルなパターンです。受付エージェントがユーザーの意図を判断して専門エージェントに委譲します。

```python
from agents import Agent, Runner, handoff

billing_agent = Agent(
    name="BillingAgent",
    handoff_description="""
    請求・支払い・領収書・返金に関する問い合わせを担当する。
    「請求金額が違う」「支払い方法を変えたい」「返金したい」などを処理する。
    """,
    instructions="請求担当。必ず金額と日付を確認してから回答する。",
)

support_agent = Agent(
    name="SupportAgent",
    handoff_description="""
    技術的な問題・エラー・使い方に関する問い合わせを担当する。
    「ログインできない」「エラーが出る」「使い方がわからない」などを処理する。
    """,
    instructions="技術サポート担当。再現手順を確認してから原因を特定する。",
)

triage_agent = Agent(
    name="TriageAgent",
    instructions="""
    カスタマーサービスの受付。問い合わせを判断して適切な担当者に転送する。
    転送後は余計なコメントを追加しない。
    """,
    handoffs=[billing_agent, support_agent],
)
```

⭐ **`handoff_description` はトリアージ LLM がルーティングに使う主要な判断材料です。** トリガーワードを含め、誤ルーティングを防ぎます。なお、呼び出し元の `instructions` や Handoff ツールの名前も判断に影響します。

### 3. パイプライン型（コードオーケストレーション）

処理の順序をコードで制御します。Handoff 型との違いは**コードが制御の主体である**点です。

```python
from agents import Agent, Runner

data_agent = Agent(name="DataAgent", output_type=SalesData, ...)
analysis_agent = Agent(name="AnalysisAgent", output_type=Analysis, ...)
report_agent = Agent(name="ReportAgent", output_type=Report, ...)

async def run_pipeline(month: str) -> Report:
    data_result = await Runner.run(data_agent, f"{month}のデータを取得して")
    sales_data: SalesData = data_result.final_output

    analysis_result = await Runner.run(
        analysis_agent,
        f"以下のデータを分析して:\n{sales_data.model_dump_json()}"
    )
    analysis: Analysis = analysis_result.final_output

    report_result = await Runner.run(
        report_agent,
        f"以下の分析からレポートを生成して:\n{analysis.model_dump_json()}"
    )
    return report_result.final_output
```

### 4. 並列型（asyncio.gather）

独立したタスクを同時実行してレイテンシを削減します。

```python
import asyncio
from agents import Agent, Runner

async def run_parallel_research(topic: str):
    results = await asyncio.gather(
        Runner.run(market_agent, f"市場動向: {topic}"),
        Runner.run(competitor_agent, f"競合分析: {topic}"),
        Runner.run(internal_agent, f"社内データ: {topic}"),
    )
    market, competitor, internal = [r.final_output for r in results]

    summary = await Runner.run(
        summary_agent,
        f"市場: {market}\n競合: {competitor}\n社内: {internal}\nを統合してまとめて"
    )
    return summary.final_output
```

### 5. Agents-as-Tools パターン

エージェントをツールとして別のエージェントから呼び出します。**会話が親エージェントに戻る**点が Handoff との違いです。

```python
translation_agent = Agent(
    name="TranslationAgent",
    instructions="テキストを指定された言語に翻訳する。",
)

research_agent = Agent(
    name="ResearchAgent",
    instructions="調査結果を日本語でまとめる。必要に応じて翻訳ツールを使う。",
    tools=[
        translation_agent.as_tool(
            tool_name="translate_text",
            tool_description="テキストを別の言語に翻訳する。言語コード（ja/en/zh等）を指定する。",
        )
    ],
)
```

### 6. input_filter でコンテキストを最適化する

Handoff 時にすべての会話履歴を渡すとコンテキストが膨れ、トークンコストが増大します。

```python
from agents import handoff
from agents.extensions.handoff_filters import remove_all_tools

triage_agent = Agent(
    name="Triage",
    handoffs=[
        handoff(
            specialist_agent,
            input_filter=remove_all_tools,  # ツール呼び出し履歴を除いて渡す
        )
    ],
)
```

---

## Chapter 6: Guardrail による安全設計

### 1. Guardrail が必要な場面

| リスク | 対策 |
|--------|------|
| ユーザーが不適切な入力を送る可能性がある | InputGuardrail |
| エージェントが機密情報・不適切な内容を出力する可能性がある | OutputGuardrail |
| 特定のツールを呼ぶ前に確認が必要 | ツールレベルガードレール |
| 完全に信頼されたシステム内でのみ動作する | ガードレール不要の場合も |

### 2. InputGuardrail の実装

ユーザーからの入力をチェックします。デフォルトでメインエージェントと**並列実行**されるため、レイテンシ増加を最小限に抑えられます。

```python
from pydantic import BaseModel
from agents import (
    Agent, Runner, GuardrailFunctionOutput,
    input_guardrail, RunContextWrapper, InputGuardrailTripwireTriggered
)

class SafetyResult(BaseModel):
    is_safe: bool
    reason: str

safety_checker = Agent(
    name="SafetyChecker",
    instructions="""
    入力が以下に該当するか判断して JSON で返す:
    - 個人情報の不正取得を試みる内容
    - 有害なコンテンツの生成を要求する内容
    is_safe: 問題なければ true、該当すれば false
    """,
    output_type=SafetyResult,
)

@input_guardrail
async def safety_guardrail(
    ctx: RunContextWrapper, agent: Agent, input: str
) -> GuardrailFunctionOutput:
    result = await Runner.run(safety_checker, input, context=ctx.context)
    check: SafetyResult = result.final_output
    return GuardrailFunctionOutput(
        output_info=check,
        tripwire_triggered=not check.is_safe,
    )

main_agent = Agent(
    name="MainAgent",
    instructions="ユーザーの依頼を処理する。",
    input_guardrails=[safety_guardrail],
)

try:
    result = Runner.run_sync(main_agent, user_input)
except InputGuardrailTripwireTriggered as e:
    print(f"入力がブロックされました: {e.guardrail_result.output.output_info}")
```

### 3. OutputGuardrail の実装

エージェントの最終出力をチェックします。入力チェックと違い**逐次実行**（出力が出てからチェック）です。

```python
import re
from agents import output_guardrail, OutputGuardrailTripwireTriggered

@output_guardrail
async def pii_output_guardrail(
    ctx: RunContextWrapper, agent: Agent, output: str
) -> GuardrailFunctionOutput:
    """出力に個人情報（メールアドレス・電話番号）が含まれていないか確認する。"""
    has_email = bool(re.search(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', output))
    has_phone = bool(re.search(r'\d{2,4}-\d{2,4}-\d{4}', output))

    return GuardrailFunctionOutput(
        output_info={"has_pii": has_email or has_phone},
        tripwire_triggered=has_email or has_phone,
    )
```

### 4. Guardrail の適用スコープと注意点

⚠ **重要な制約：**

- `input_guardrails` は**最初のエージェント**にのみ適用されます。Handoff 先への入力は検査されません
- `output_guardrails` は**最後に応答したエージェント**にのみ適用されます
- `RunConfig.input_guardrails` はランの**初期入力**に対する共通チェックとして有効です。ただし Handoff 後の入力や個別ツールの直前を検査するものではありません。Handoff 後の入力を検査したい場合は `input_filter` で渡す内容を制限するか、`@tool_input_guardrail` でツール直前に検査するか、Handoff 先を別の `Runner.run()` として明示的に実行する設計に切り替えます

```python
result = Runner.run_sync(
    triage_agent,
    user_input,
    run_config=RunConfig(input_guardrails=[safety_guardrail]),  # 初期入力のみに適用
)
```

### 5. 並列実行の制御

InputGuardrail はデフォルトでメインエージェントと並列実行されます。ガードレールの結果を待ってからエージェントを起動したい場合は `run_in_parallel=False` を指定します。

```python
@input_guardrail(run_in_parallel=False)  # ブロッキング実行
async def blocking_guardrail(ctx, agent, input) -> GuardrailFunctionOutput:
    """このガードレールが通過するまでエージェントは起動しない。"""
    ...
```

---

## Chapter 7: Tracing・可観測性

### 1. デフォルトのトレース動作

Agents SDK はデフォルトでトレースが有効です。各実行は自動的に記録されます。

| スパン種別 | 記録内容 |
|-----------|---------|
| `agent_span` | エージェントの開始・終了 |
| `generation_span` | LLM 呼び出しの入出力 |
| `function_span` | ツール実行の引数と戻り値 |
| `handoff_span` | ハンドオフの発生 |
| `guardrail_span` | ガードレールの実行と結果 |

OpenAI ダッシュボード（platform.openai.com → Tracing）でリアルタイムに確認できます。

### 2. トレースの設定と RunConfig

```python
from agents import RunConfig

result = Runner.run_sync(
    agent,
    "売上を分析して",
    run_config=RunConfig(
        workflow_name="monthly_sales_report",  # ダッシュボードでの識別名
        group_id="session_2024_01",            # 複数ランをグルーピング
        trace_include_sensitive_data=False,    # 入出力の内容をトレースから除外
    ),
)
```

本番環境では `trace_include_sensitive_data=False` を強く推奨します。

```bash
# 環境変数での設定も可能
OPENAI_AGENTS_TRACE_INCLUDE_SENSITIVE_DATA=0
OPENAI_AGENTS_DISABLE_TRACING=1  # トレース完全無効
```

### 3. カスタムスパンで独自の計測を追加

```python
from agents.tracing import trace, custom_span

async def run_full_pipeline(month: str):
    with trace("monthly_report_pipeline"):
        with custom_span("data_validation"):
            validate_input(month)

        result = await Runner.run(agent, f"{month}のレポートを作成して")

        with custom_span("report_storage"):
            save_report(result.final_output)

    return result.final_output
```

### 4. 外部プラットフォームへの連携

```python
from agents.tracing import add_trace_processor, set_trace_processors

# OpenAI バックエンドに加えて追加する場合
add_trace_processor(your_processor)

# OpenAI バックエンドを置き換える場合
set_trace_processors([your_processor])
```

Langfuse・LangSmith・Datadog・Braintrust など主要なプラットフォームにコミュニティ製プロセッサが存在します。

### 5. ローカルデバッグの手段

開発中に詳細な実行状況を確認する主な方法は 2 つです。

**① Python 標準 logging**

```python
import logging
logging.basicConfig(level=logging.DEBUG)
# agents モジュールのログが stdout に出力される
```

**② OpenAI Tracing UI**

`Runner.run()` の実行後、[platform.openai.com → Tracing](https://platform.openai.com/traces) で各スパンの入出力・所要時間を視覚的に確認できます。ローカルだけで完結させたい場合はカスタムプロセッサで `print` するのが最も手軽です（Section 4 参照）。

### 6. RunResult でのデバッグ

```python
from agents.items import ToolCallItem

result = Runner.run_sync(agent, "...")

tool_calls = [item for item in result.new_items if isinstance(item, ToolCallItem)]
assert len(tool_calls) >= 1, "ツールが1回以上呼ばれていること"
assert tool_calls[0].tool_name == "get_sales_data"
```

これにより「正しいツールが正しい順序で呼ばれたか」を自動テストで検証できます。

---

## Chapter 8: よくある失敗パターン

### 1. instructions が長すぎて無視される

**症状**: エージェントが instructions に書いた条件を守らない。出力フォーマットが崩れる。

**対策**: 役割・制約・フォーマットを構造化して短く保ちます。

```python
# 良い例：構造化して短く
agent = Agent(
    name="AnalysisAgent",
    instructions="""
## 役割
売上データの統計分析と洞察抽出。

## 出力フォーマット（必ず守る）
1. エグゼクティブサマリー（3行以内）
2. 数値ハイライト（箇条書き、単位明記）
3. 推奨アクション（1〜3件）

## 禁止事項
- ツールの結果なしに数値を推測しない
- 前月比を計算ツールなしに述べない
    """,
)
```

### 2. ツールが例外を投げてループが止まる

**症状**: ツール実行中にエラーが出て予期しない終了。

**対策**: エラーを辞書で返す（Chapter 2 参照）。

```python
@function_tool(failure_error_function=None)  # 例外を再 raise したい場合
def strict_tool(id: str) -> dict:
    """ID が無効な場合は例外を投げ、ループを止める。"""
    if not is_valid(id):
        raise ValueError(f"無効な ID: {id}")
    return fetch(id)
```

### 3. Guardrail の適用漏れ

**症状**: ハンドオフ先のエージェントに不正な入力が届く。

**原因**: `RunConfig.input_guardrails` はランの初期入力にのみ適用され、Handoff 後の入力は検査されません。

**対策**: 用途に応じて組み合わせます。

| 検査したい入力 | 推奨手段 |
|-------------|---------|
| ランの初期入力（共通チェック） | `RunConfig(input_guardrails=[...])` |
| Handoff 後にサブエージェントへ渡す内容を制限 | `input_filter` で不要な履歴を除去 |
| 特定ツールを呼ぶ直前 | `@tool_input_guardrail` |
| Handoff 先を厳密に検査したい | Handoff 先を別の `Runner.run()` として明示的に実行 |

### 4. max_turns の設定を忘れて無限ループ

**症状**: エージェントがツールを呼び続けて終わらない・課金が跳ね上がる。

**対策**: タスクの複雑さに応じた値を設定し、`MaxTurnsExceeded` をハンドリングします。

```python
from agents import MaxTurnsExceeded

try:
    result = Runner.run_sync(agent, user_input, max_turns=5)
except MaxTurnsExceeded:
    result = Runner.run_sync(agent, simplify(user_input), max_turns=3)
```

### 5. マルチエージェントでコンテキストが肥大化

**症状**: 後半のエージェントが前半の指示を「忘れる」。トークンコストが急増する。

**対策**: `input_filter` で不要な履歴を削除します（Chapter 5 参照）。

### 6. output_type 指定エージェントを Handoff 途中に置いたとき意図が曖昧になる

**症状**: `output_type` を指定したエージェントを Handoff の途中に置いたとき、structured output を返す意図が不明確になり動作が不安定になる。

**対策**: Handoff の途中に置く場合は、そのエージェントが何を返すべきかを `instructions` と `handoff_description` で明示します。シンプルにしたい場合は、`output_type` 指定エージェントを最終エージェントとして扱うか、パイプライン型に切り替えます。

### 7. handoff_description が曖昧でルーティングが不安定

**症状**: トリアージエージェントが間違った専門エージェントに委譲する。

**対策**: 「どんな言葉・状況でこのエージェントに委譲すべきか」を具体的なトリガーワードで記述します（Chapter 5 参照）。

---

## Chapter 9: 直感を育てる

### 1. 「どこまでエージェントに任せるか」の感覚

| LLM に委ねる | 人間・コードが制御する |
|------------|-------------------|
| 意図の解釈・自然言語の理解 | 処理の順序（パイプライン） |
| ツールを呼ぶ判断 | エラーハンドリングの境界 |
| 出力のフォーマット調整 | 機密データの受け渡し（Context） |
| 曖昧な入力への対処 | 副作用の実行タイミング |

迷ったら「LLM が間違えたとき、被害はどこまで及ぶか」を考えます。**被害が小さい判断は LLM に、大きい判断はコードで制御する**のが原則です。

### 2. デバッグの思考法

エージェントが期待通りに動かないとき、問題は 3 層のどこかにあります。

```
Layer 3: 設計の問題
    └── マルチエージェントの役割分担・ハンドオフ設計が適切か
Layer 2: 指示の問題
    └── instructions・docstring・handoff_description が LLM に伝わっているか
Layer 1: 実装の問題
    └── API の使い方・型・引数が正しいか（Tracing・RunResult で確認）
```

Layer 1（実装）は Tracing UI と `result.new_items` でほぼ解決できます。Layer 2（指示）はプロンプトの書き直しで解決します。Layer 3（設計）の問題は「パターンを変える」決断が必要です。

### 3. 「エージェントらしさ」を見極める

| 得意 | 苦手 |
|------|------|
| 非構造化入力を構造化データに変換 | 長い数値計算（ツールで補う） |
| 条件分岐をフレキシブルに処理 | 複雑なビジネスロジックの記憶 |
| 複数のツールを組み合わせた問題解決 | 一貫した状態管理（コードで補う） |
| 曖昧な要件からの仕様の補完 | 高精度な引用・出典の保証 |

**エージェントは LLM を中心に据えながら、苦手を周辺コードで補うアーキテクチャです**。

### 4. 本番移行のチェックリスト

```
□ max_turns が適切な値に設定されている
□ MaxTurnsExceeded / InputGuardrailTripwireTriggered / OutputGuardrailTripwireTriggered がハンドリングされている
□ RunConfig(trace_include_sensitive_data=False) が設定されている
□ エラーを例外でなく値で返すツール設計になっている
□ InputGuardrail が必要なすべてのエントリポイントに適用されている
□ output_type で最終出力の型が保証されている
□ Handoff の input_filter でコンテキストを最適化している
□ 本番モデルに切り替えている（gpt-5.5 等）
□ トレースの workflow_name が設定されている
□ 単体テストで new_items を使ったツール呼び出し検証がある
```

### 5. 次の一歩

この教材で扱った内容をすべて実装に組み込む必要はありません。段階的に導入することを推奨します。

| フェーズ | 取り組むこと |
|---------|------------|
| 1. 動かす | 単一エージェント + 1〜2 ツール + `output_type` |
| 2. 安定させる | `max_turns` + Guardrail + Tracing |
| 3. スケールする | マルチエージェント + Context + `input_filter` |
| 4. 最適化する | モデル選択の最適化 + コスト計測 + 並列化 |

まず Phase 1 を動かしてトレースを眺めるだけで、エージェントの動作感覚が大きく変わります。

---

## 関連リソース

- [OpenAI Agents SDK 公式ドキュメント](https://openai.github.io/openai-agents-python/)
- [openai/openai-agents-python（GitHub）](https://github.com/openai/openai-agents-python)
- [openai-agents-sdk-overview.md](openai-agents-sdk-overview.md) — 全体像の俯瞰マップ
- [openai-agents-sdk-tutorial.md](openai-agents-sdk-tutorial.md) — 手を動かして基礎を習得
