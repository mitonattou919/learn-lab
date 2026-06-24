# OpenAI Agents SDK 概要

> OpenAI Agents SDK の全体像を俯瞰する overview 教材。ChatGPT API との違いから Agent / Runner / Tool / Handoff / Guardrail / Tracing の関係まで、手を動かす前の「地図」として使ってください。
> 手を動かして基礎を身につけたい方 → `openai-agents-sdk-tutorial.md` へ

---

## Chapter 0: OpenAI Agents SDK とは

### 1. 「会話するAI」から「行動するAI」へ

**OpenAI Agents SDK** は、複数の AI エージェントを組み合わせてタスクを自律的にこなすワークフローを Python で構築するためのフレームワークです。

まず、似た名前の概念を整理します。

| レイヤ | 概念 | 役割 |
|--------|------|------|
| **UI** | ChatGPT | ユーザーが会話する UI アプリ |
| **低レベル API** | OpenAI Responses API / Chat Completions API | モデルへの HTTP リクエスト（テキスト入出力） |
| **エージェントランタイム** | **OpenAI Agents SDK** ← ここ | ループ管理・ツール実行・ハンドオフを自動化するフレームワーク |

Responses API だけでエージェントを作ろうとすると、次の処理をすべて自分で実装する必要があります。

```text
【Responses API 単体でやること】
・LLM に「ツールを呼んでいいよ」と伝える
・LLM がツール呼び出しを返す → 実行 → 結果を次のリクエストに含める
・「まだ終わってない → もう一度 LLM に投げる」を繰り返す
・別のエージェントに委譲するタイミングを判断する
・入出力の安全チェックを挟む
・実行ログを記録する

【Agents SDK がやること】
上記すべてをフレームワークが自動管理。
コードに書くのは「何ができるエージェント」か、だけ。
```

### 2. Swarm から生まれた「production-ready」版

Agents SDK は OpenAI の実験的プロジェクト **Swarm** の後継です。Swarm は「マルチエージェントのシンプルな抽象化」を探る実験でしたが、本番運用には向いていませんでした。

Agents SDK はその設計思想を引き継ぎ、トレーシング・ガードレール・ストリーミングを加えて **「教育用から本番まで」** 使えるライブラリとして 2025年 3月にリリースされました。

> Python 3.10 以上が必要。最新版（v0.17.x）は `pip install openai-agents` または `uv add openai-agents` でインストール。

### 3. 3 つのプリミティブ

設計の核心は **「概念を最小限に絞る」** ことです。

```text
Agent
  ↑ 能力を与える    → Tool（ツール）
  ↑ 仕事を渡す      → Handoff（ハンドオフ）
  ↑ 安全を守る      → Guardrail（ガードレール）
  ↑ 全体を見る      → Tracing（トレーシング）
  └ これらを動かす  → Runner（ランナー）
```

**Agent** が「何ができるか」を定義し、**Runner** が「動かし方」を制御し、**Handoff** が「誰に渡すか」を決め、**Guardrail** が「許していいか」を判断します。Tool と Tracing はサポートレイヤです。

### 4. 他フレームワークとの役割分担

OpenAI Agents SDK は唯一の選択肢ではありません。用途に応じて使い分けが必要です。

| フレームワーク | 設計思想 | 向いているケース |
|--------------|---------|----------------|
| **OpenAI Agents SDK** | 3プリミティブでシンプル、OpenAI 最適化 | OpenAI モデルを使った本番エージェント |
| **LangChain / LangGraph** | マルチプロバイダー、グラフ構造 | 複雑な条件分岐 / OpenAI 以外のモデル |
| **CrewAI** | 役割ベースのチーム構成 | 人間の「チーム」を模したマルチエージェント |
| **Google ADK** | Google エコシステム統合 | Gemini / Vertex AI / GCP との連携 |
| **LlamaIndex Workflows** | RAG との組み合わせ | ドキュメント検索特化のエージェント |

「OpenAI モデルで、シンプルに素早く本番投入したい」場面が Agents SDK の最も輝くところです。

---

## Chapter 1: 全体アーキテクチャ

### 1. 構成要素の関係

OpenAI Agents SDK は 6 つの主要要素で構成されています。

```text
                ┌───────────────────────────────────┐
                │           Runner（実行エンジン）    │
                └────────────────┬──────────────────┘
                                 │ エージェントループを管理
                    ┌────────────▼────────────┐
                    │   Agent（LLM の設定単位） │
                    │  instructions / model   │
                    └───┬─────────┬───────────┘
                        │         │
          ┌─────────────┘         └──────────────────┐
          │                                           │
  ┌───────▼──────┐    ┌─────────────┐    ┌───────────▼──────┐
  │    Tool      │    │   Handoff   │    │    Guardrail     │
  │（能力の拡張）│    │（委譲先指定）│    │（入出力の安全ゲート）│
  └──────────────┘    └─────────────┘    └──────────────────┘

                                ↕（全体を記録）
                          ┌──────────┐
                          │ Tracing  │
                          │（可観測性）│
                          └──────────┘
```

| 要素 | 役割 | 例 |
|------|------|-----|
| **Runner** | エージェントループの制御、ツール実行の自動化 | `Runner.run_sync(agent, "質問")` |
| **Agent** | instructions と tools を持つ LLM の設定単位 | カスタマーサポート Bot、翻訳エージェント |
| **Tool** | エージェントが呼び出せる能力 | Web 検索、DB クエリ、コード実行 |
| **Handoff** | 別エージェントへのタスク委譲 | トリアージ Bot → 専門エージェント |
| **Guardrail** | 入出力の安全・品質チェック | 禁止ワード遮断、PII 検出 |
| **Tracing** | 実行全体の記録と外部可視化 | Langfuse、LangSmith との連携 |

### 2. エージェントループの流れ

`Runner.run()` を呼ぶと、以下のループが自動で動き続けます。

```text
ユーザー入力
    │
    ▼
[Input Guardrail]─→（問題あり）→ InputGuardrailTripwireTriggered 例外
    │（問題なし）
    ▼
[LLM へ送信]（instructions + tools + 会話履歴）
    │
    ├─ LLM がテキスト回答を返す → 終了（最終出力）
    │
    ├─ LLM がツール呼び出しを返す
    │       └─→ ツール実行 → 結果を LLM に渡す → ループ継続
    │
    └─ LLM がハンドオフを返す
            └─→ 別エージェントに切り替え → ループ継続
    │
    ▼（最終エージェントがテキスト回答）
[Output Guardrail]─→（問題あり）→ OutputGuardrailTripwireTriggered 例外
    │（問題なし）
    ▼
RunResult（最終出力 + 会話履歴 + 実行ログ）
```

このループは **`max_turns` に達する**か、**エージェントが最終回答を返す**まで繰り返します。ループ制御のコードは不要です。

### 3. 3 つの実行モード

Runner は用途に応じて 3 つの呼び出しモードを提供します。

| メソッド | 返り値 | 使いどころ |
|---------|--------|-----------|
| `Runner.run_sync()` | `RunResult` | スクリプト・バッチ処理（同期） |
| `Runner.run()` | `RunResult`（非同期） | FastAPI など `async/await` 環境 |
| `Runner.run_streamed()` | `RunResultStreaming` | チャット UI でトークンを順次表示 |

```python
# 同期（最もシンプル）
result = Runner.run_sync(agent, "東京の天気は？")
print(result.final_output)

# ストリーミング（UI 向け）
from openai.types.responses import ResponseTextDeltaEvent

result = Runner.run_streamed(agent, "レポートを書いて")
async for event in result.stream_events():
    if event.type == "raw_response_event" and isinstance(event.data, ResponseTextDeltaEvent):
        print(event.data.delta, end="", flush=True)
```

### 4. RunResult が持つ情報

実行が完了した `RunResult` には以下が含まれます。

| プロパティ | 内容 |
|-----------|------|
| `final_output` | 最終エージェントの回答 |
| `new_items` | 実行中に生じたアイテム（ツール呼び出し・ハンドオフなど） |
| `last_agent` | 最後に回答したエージェント |
| `raw_responses` | LLM の生レスポンス |
| `to_input_list()` | 次ターンの入力として渡せるリストに変換 |

`to_input_list()` を次回の入力にそのまま渡すことで、**多ターンの会話**を構築できます。

---

## Chapter 2: Agent（LLM の設定単位）

### 1. Agent とは何か

**Agent** は、「どのモデルを使い」「何をすべきかの指示を持ち」「どんな能力を持ち」「誰に仕事を渡せるか」をひとまとめにした設定オブジェクトです。

```python
from agents import Agent

agent = Agent(
    name="サポートBot",
    model="gpt-5.5",
    instructions="あなたは親切なカスタマーサポートエージェントです。",
    tools=[...],
    handoffs=[...],
    output_type=None,  # 構造化出力が必要な場合に Pydantic モデルを指定
)
```

Agent 自体はステートレスです。実行するたびに Runner が会話履歴を注入します。

### 2. 5 つのパラメータ

| パラメータ | 型 | 役割 |
|-----------|-----|------|
| `name` | `str` | エージェントの識別名。ハンドオフ時に「誰に渡したか」の記録に使われる |
| `model` | `str` / `Model` | 使用する LLM（例: `"gpt-5.5"`, `"gpt-5.4-mini"`） |
| `instructions` | `str` / 関数 | システムプロンプト。関数を渡すと実行時に動的生成できる |
| `tools` | `list[Tool]` | エージェントが呼び出せるツールのリスト |
| `handoffs` | `list[Agent]` | 仕事を委譲できる別エージェントのリスト |
| `output_type` | Pydantic モデル | 指定すると最終出力を構造化 JSON で強制する |

### 3. 動的 instructions（関数渡し）

`instructions` に Python 関数を渡すと、実行時にコンテキストを参照した動的プロンプトを生成できます。

```python
def build_instructions(ctx, agent) -> str:
    user_name = ctx.context.get("user_name", "ゲスト")
    return f"{user_name} さんを担当するサポートエージェントです。丁寧に対応してください。"

agent = Agent(
    name="パーソナルBot",
    instructions=build_instructions,  # 関数を渡す
)
```

静的な文字列では対応できない「ユーザーごとに異なるプロンプト」や「セッション状態に応じた振る舞い変化」に使います。

### 4. 構造化出力（output_type）

Pydantic モデルを `output_type` に渡すと、最終回答が **JSON として型保証されて返ってくる**ようになります。

```python
from pydantic import BaseModel

class SupportTicket(BaseModel):
    category: str       # "billing", "technical", "general"
    priority: int       # 1〜5
    summary: str
    requires_escalation: bool

agent = Agent(
    name="分類Bot",
    instructions="問い合わせを分析して分類票を出力してください。",
    output_type=SupportTicket,  # 構造化出力
)
```

`result.final_output` は `str` ではなく `SupportTicket` インスタンスになります。後続処理での `result.final_output.priority` といったアクセスが型安全にできます。

### 5. よくあるアンチパターン

- **1つの Agent に instructions を詰め込みすぎる** → LLM の動作が不安定になる。役割で Agent を分け、Handoff を使う
- **すべてのエージェントに同じ model を使う** → コストと速度のトレードオフ。トリアージ役には軽量モデル、判断が必要な役には高性能モデルを使い分ける
- **output_type を使わずに LLM 出力を `eval()` で解析する** → セキュリティリスク。output_type + Pydantic を使う

---

## Chapter 3: Tool（能力の拡張）

### 1. Tool とは何か

**Tool** は、エージェントが「テキストを生成する」以外のことをするための手段です。LLM はツールの定義（名前・説明・引数の型）を見て、「今この状況でどのツールを呼ぶべきか」を判断します。

```text
【Tool なしエージェント】
「今月の売上を教えて」→ 「学習データの範囲では...（古い情報）」

【Tool ありエージェント】
「今月の売上を教えて」→ [get_sales_data() を呼び出す] → DBの実データを元に回答
```

### 2. Tool の種類（代表的な3カテゴリ）

Agents SDK のツールは目的と実行場所で分類できます。公式には5カテゴリ以上ありますが、実務でよく使う代表的な3つを押さえておくと十分です。

| 種類 | 実行場所 | 定義方法 | 用途 |
|------|---------|---------|------|
| **Function Tool** | 自分のサーバー | `@function_tool` デコレータ | 社内 DB クエリ・計算・API 呼び出し |
| **Hosted Tool** | OpenAI のサーバー | SDK 提供のクラス | Web 検索・コード実行・ファイル検索 |
| **MCP Tool** | MCP サーバー（外部） | `MCPServerStdio` / `HostedMCPTool` | GitHub・Slack・Figma など外部ツール |

### 3. Function Tool（自作ツール）

最もよく使うのがこの形式です。**Python 関数 + `@function_tool` デコレータ** だけで定義できます。

```python
from agents import function_tool

@function_tool
def get_weather(city: str) -> str:
    """指定した都市の現在の天気を返す。
    
    天気に関する質問や天気予報が必要なとき呼び出すこと。
    """
    # 実際の実装では天気 API を叩く
    return f"{city}の天気: 晴れ、気温 25℃"
```

**docstring が LLM への説明文**になります。「いつ使うか」「何を渡すか」「何が返るか」を明確に書くと、LLM のツール選択精度が上がります。引数の型ヒントも必須です（JSON Schema に変換されて LLM に渡されます）。

### 4. Hosted Tool（OpenAI サーバー実行）

OpenAI のインフラで実行される組み込みツールです。`OpenAIResponsesModel` 利用時のみ使えます。

| ツール | 機能 |
|--------|------|
| `WebSearchTool` | Web 検索 |
| `FileSearchTool` | OpenAI Vector Store への RAG 検索 |
| `CodeInterpreterTool` | サンドボックスでの Python コード実行 |
| `ImageGenerationTool` | テキストから画像生成 |
| `HostedMCPTool` | リモート MCP サーバーのツールをそのまま公開 |

```python
from agents import Agent, WebSearchTool, FileSearchTool

agent = Agent(
    name="リサーチBot",
    instructions="Web 検索と社内文書を使って質問に答えてください。",
    tools=[
        WebSearchTool(),
        FileSearchTool(vector_store_ids=["vs_abc123"]),
    ],
)
```

### 5. MCP Tool（外部ツールサーバー連携）

**MCP（Model Context Protocol）** は、ツールを標準プロトコルで公開するための仕様です。GitHub・Slack・Figma など多くのサービスが MCP サーバーを提供しており、それらをそのまま Agent のツールとして使えます。

```text
Agent
  │ MCP プロトコルで接続
  ▼
MCP サーバー（GitHub・Slack・ファイルシステム等）
  └→ ツール一覧を返す
  └→ ツールを実行する
```

ローカル MCP サーバー（`MCPServerStdio`）とリモート MCP サーバー（`HostedMCPTool`）の 2 つの接続方式があります。

### 6. Tool の選択基準

```text
┌─ 自社インフラ・社内 API を叩きたい ──────────────→ Function Tool
│
├─ Web 検索 / コード実行が必要で、実行環境を管理したくない → Hosted Tool
│
└─ サードパーティサービス（GitHub / Slack 等）と連携したい → MCP Tool
```

### 7. よくあるアンチパターン

- **docstring を省略する / 短すぎる** → LLM がツールをいつ使えばいいかわからず、使われないか誤用される
- **エラー時に例外を投げる** → エージェントループが止まる。エラー情報は文字列か辞書で返してループを継続させる
- **1つのツールに複数の責務を持たせる** → 1ツール1責任にして LLM に判断させる

---

## Chapter 4: Handoff（マルチエージェント連携）

### 1. Handoff とは何か

**Handoff** は、あるエージェントが「この仕事は自分より別のエージェントの方が適している」と判断したとき、タスクを委譲する仕組みです。

```text
【単一エージェントの限界】
1つのエージェントがすべてを担う
→ instructions が肥大化
→ 動作が不安定（「翻訳Bot」が「コーディングBot」の振る舞いをする）
→ 特定の役割だけ改善するのが難しい

【Handoff によるマルチエージェント】
役割ごとに Agent を分けて、必要なときに委譲する
→ 各エージェントの instructions がシンプルに保てる
→ 役割単位でチューニング・テストできる
```

### 2. Handoff の仕組み

`handoffs=[agent_b]` と指定するだけで、**`transfer_to_agent_b` というツール**が自動生成されて LLM に公開されます。LLM は「今の状況ではこのエージェントに渡すべき」と判断したとき、このツールを呼び出します。

```python
billing_agent = Agent(name="請求担当", instructions="請求・支払いに関する質問を担当する。")
tech_agent    = Agent(name="技術担当", instructions="技術的な問題のトラブルシューティングを担当する。")

triage_agent = Agent(
    name="受付Bot",
    instructions="問い合わせを分析し、適切な担当エージェントに引き継ぐ。",
    handoffs=[billing_agent, tech_agent],
    # → LLM には transfer_to_請求担当 と transfer_to_技術担当 が見える
)
```

### 3. 代表的な 2 パターン

#### トリアージパターン（1入口 → 専門エージェント）

```text
ユーザー → 受付Bot（triage_agent）
                │
        ┌───────┴────────┐
        ▼                ▼
    請求担当Bot       技術担当Bot
```

カスタマーサポートや問い合わせ分類に最適です。受付 Bot はデフォルトの軽量モデルでよく、専門 Bot だけ高性能モデルを使うなどコスト設計もできます。

#### パイプラインパターン（逐次委譲）

```text
ユーザー → 調査Bot → 分析Bot → 執筆Bot → 最終回答
```

調査・分析・執筆のように前工程の出力が次工程の入力になる場合に使います。各 Bot が専門に集中できるため品質が安定します。

### 4. input_filter（引き継ぐ履歴を加工する）

委譲時にデフォルトでは**全会話履歴**が次のエージェントに引き継がれます。`input_filter` を使うと引き継ぐ履歴を加工できます。

```text
ユースケース例：
- 受付Bot の内部独り言を削除してから専門Bot に渡す
- 関係ない話題の履歴を除外してコンテキストを絞る
- 要約した形式に変換して渡す
```

### 5. Handoff vs Tool の使い分け

Handoff と Tool の違いは「会話の制御を移すかどうか」です。

| 観点 | Tool | Handoff |
|------|------|---------|
| 実行後の制御 | 元のエージェントに戻る | 新しいエージェントが引き継ぐ |
| 向いているケース | DBクエリ・計算など補助的な処理 | 別の専門家に完全に任せる |
| 会話履歴 | 変化なし | 引き継がれる（加工可能） |

### 6. よくあるアンチパターン

- **すべてのエージェントが互いに handoffs を持つ** → 無限委譲ループのリスクがある。委譲は一方向に設計する
- **受付 Bot の instructions に専門知識を詰め込む** → 受付 Bot はトリアージだけに徹し、専門知識は専門 Bot に委ねる
- **max_turns を設定しない** → ループが収束しない場合に無限実行。必ず上限を設ける

---

## Chapter 5: Guardrail（入出力の安全ゲート）

### 1. Guardrail とは何か

**Guardrail** は、エージェントの入力と出力に挟み込む**検証レイヤ**です。LLM 本体の instructions だけでは制御しきれない安全チェックを、コードレベルで強制できます。

```text
ユーザー入力
    │
[Input Guardrail] ← ここで弾く（LLM に渡す前）
    │
  Agent（LLM）
    │
[Output Guardrail] ← ここで弾く（ユーザーに返す前）
    │
最終回答
```

### 2. 2 種類の Guardrail

| 種類 | 動くタイミング | 典型的な用途 |
|------|-------------|-------------|
| **Input Guardrail** | ユーザー入力を受け取った直後 | 禁止トピック遮断、PII 検出、悪意ある入力のブロック |
| **Output Guardrail** | 最終回答を返す直前 | 品質チェック、機密情報の流出防止、フォーマット検証 |

### 3. Tripwire（発火 → 即時停止）

Guardrail 内で問題を検知したとき、`tripwire_triggered=True` を返すと **`InputGuardrailTripwireTriggered`（入力側）または `OutputGuardrailTripwireTriggered`（出力側）例外**が投げられてエージェントループが即座に停止します。

```python
from agents import Agent, input_guardrail, GuardrailFunctionOutput, RunContextWrapper

@input_guardrail
async def no_competitor_topics(ctx: RunContextWrapper, agent: Agent, input: str) -> GuardrailFunctionOutput:
    """競合他社の話題が含まれていたら弾く"""
    COMPETITORS = ["competitor_a", "competitor_b"]
    flagged = any(c in input.lower() for c in COMPETITORS)
    return GuardrailFunctionOutput(
        output_info={"flagged": flagged},
        tripwire_triggered=flagged,
    )

agent = Agent(
    name="サポートBot",
    instructions="製品サポートを担当します。",
    input_guardrails=[no_competitor_topics],
)
```

```text
入力: "competitor_a の方が良くないですか？"
  → Tripwire 発火
  → InputGuardrailTripwireTriggered 例外
  → "その話題にはお答えできません" と返す（アプリ側でハンドリング）
```

### 4. Guardrail は別の LLM で実装できる

Guardrail 自体も LLM を使って判断させることができます。

**Input Guardrail** はデフォルトでメインエージェントと**並列実行**されます（`run_in_parallel=False` で逐次実行に切り替え可能）。

```text
[Input Guardrail（小さい LLM で分類）] ─┐ デフォルトは並列実行
[メインエージェント実行]               ─┘
        ↓（どちらかが発火したら停止）
```

**Output Guardrail** は最終エージェントの回答が確定した後に実行されるため、Input と違い並列ではありません。

また、Input Guardrail はチェーンの**最初のエージェント**にだけ、Output Guardrail は**最後に回答したエージェント**にだけ適用されます。途中の委譲先エージェントでは走りません。

### 5. Guardrail の使いどころ

```text
✅ 向いているケース
・禁止トピック・競合他社の話題をコードレベルで遮断したい
・PII（個人情報）が出力に含まれないよう保証したい
・出力フォーマットの最終チェック（output_type だけでは足りない場合）
・ユーザー向けの Bot でコンプライアンス要件を満たす必要がある

❌ 向いていないケース
・ビジネスロジックの判断（Agent の instructions に書く）
・毎回実行される重い処理（Guardrail はすべてのターンで実行される）
```

### 6. よくあるアンチパターン

- **すべての検証を Guardrail に書く** → Guardrail は「絶対に通してはいけないもの」だけに限定する。通常の業務ロジックは Agent や Tool に書く
- **Guardrail 内で重い処理をする** → 毎リクエスト実行されるため、軽量な判断に留める
- **Output Guardrail だけで安全を担保しようとする** → 入力段階での遮断（Input Guardrail）が最もコスト効率がよい

---

## Chapter 6: Tracing とモデル対応

### 1. Tracing（実行の可観測性）

Agents SDK には**トレーシングが組み込まれており、デフォルト有効**です。Runner を実行するだけで、実行全体の記録が自動で収集されます。

#### トレースとスパンの関係

```text
Trace（1回のワークフロー全体）
  └─ agent_span（エージェント A の実行）
       ├─ generation_span（LLM へのリクエスト）
       ├─ function_span（ツール呼び出し）
       │    └─ generation_span（Guardrail 用 LLM）
       └─ handoff_span（エージェント B への委譲）
            └─ agent_span（エージェント B の実行）
                 └─ generation_span（LLM へのリクエスト）
```

| スパン種別 | 記録対象 |
|-----------|---------|
| `agent_span` | エージェントの実行開始・終了 |
| `generation_span` | LLM へのリクエスト・レスポンス |
| `function_span` | Function Tool の呼び出しと結果 |
| `handoff_span` | ハンドオフ操作 |
| `guardrail_span` | Guardrail の実行結果 |

#### 外部サービスへの連携

トレースは OpenAI のダッシュボードに送られますが、外部 APM ツールへの転送もサポートされています。

```text
Agents SDK
  └→（デフォルト）OpenAI Platform のトレース UI
  └→（add_trace_processor で追加）
       ├─ Langfuse
       ├─ LangSmith
       ├─ MLflow
       ├─ Weights & Biases
       ├─ Arize Phoenix
       └─ その他 25+ サービス
```

### 2. OpenAI 以外のモデルを使う

Agents SDK は OpenAI モデルに最適化されていますが、他のプロバイダーも使えます。

| 方法 | スコープ | 用途 |
|------|---------|------|
| `set_default_openai_client(client)` | グローバル | Azure OpenAI や OpenAI 互換 API |
| `ModelProvider` を Runner に渡す | 実行単位 | 実行ごとにプロバイダーを切り替え |
| `Agent.model` に `Model` インスタンスを渡す | Agent 単位 | Agent ごとに異なるモデルを使う |

**LiteLLM / any-llm アダプター** を使うと、Anthropic・Google・Mistral 等の幅広いモデルに接続できます（`pip install openai-agents[litellm]`）。

> ⚠ OpenAI 以外のプロバイダーを使う場合の注意：
> - Responses API 非対応のプロバイダーは `set_default_openai_api("chat_completions")` への切り替えが必要
> - トレースが OpenAI サーバーに送られる。無効化するには `set_tracing_disabled(True)` を呼ぶ
> - Hosted Tool（WebSearchTool 等）は使えない

### 3. モデル選択の基準

同一プロジェクト内でもエージェントの役割に応じてモデルを使い分けるのが効果的です。

> モデル名は SDK のバージョンや OpenAI のリリースで変わります。以下は v0.17.x 時点の SDK デフォルトおよびドキュメント記載モデルです。最新の利用可能モデルは公式ドキュメントを確認してください。

| 役割 | モデル例（v0.17.x 時点） | 理由 |
|------|----------|------|
| トリアージ・分類（デフォルト） | `gpt-5.4-mini` | SDK のデフォルト。高速・低コスト |
| 高品質な汎用回答 | `gpt-5.5` | 最上位品質が必要な判断タスク |
| 複雑な推論・計画 | `o3` / `o4-mini` | 多段階の推論に強い |
| リアルタイム音声 | `gpt-realtime-2` | 低レイテンシの音声対話 |

### 4. よくあるアンチパターン

- **全エージェントに最高性能モデルを使う** → コストが無駄になる。役割に応じて使い分ける
- **トレースを本番から無効化する** → 問題発生時の原因追跡ができなくなる。外部 APM に転送して保持する
- **OpenAI 以外のモデルで Hosted Tool を使おうとする** → `OpenAIResponsesModel` でのみ動作する。確認してから設計する

---

## Chapter 7: 設計判断と次の一歩

### 1. OpenAI Agents SDK を選ぶべきとき・選ばないとき

```text
✅ OpenAI Agents SDK が向いているケース

・OpenAI モデル（GPT-5 系・o3 系）を主軸に使う
・カスタマーサポート、コンテンツ生成、コード生成など典型的なエージェント用途
・「素早く動くものを作って、後から改善する」プロトタイプ優先の開発
・トリアージ→専門エージェントのようなシンプルなマルチエージェント構成
・音声エージェント（gpt-realtime-2 等のリアルタイムモデル対応）

❌ 他のフレームワークを検討すべきケース

・OpenAI 以外のモデルをメインで使う → LangChain / LiteLLM
・条件分岐が複雑なグラフ構造のワークフロー → LangGraph
・Google Cloud / Gemini エコシステムと深く統合したい → Google ADK
・RAG と組み合わせた文書検索エージェント → LlamaIndex Workflows
```

### 2. 設計上の重要な選択ポイント

| 判断ポイント | 選択肢 | 基準 |
|-----------|--------|------|
| **単一 Agent vs マルチ Agent** | 単一で始め、複雑になったら分割 | instructions が肥大化・不安定になってきたらシグナル |
| **Function Tool vs Hosted Tool** | 社内データ → Function、Web/ファイル検索 → Hosted | 実行環境をどちらが管理するか |
| **同期 vs ストリーミング** | スクリプト → `run_sync()`、UI → `run_streamed()` | ユーザーへのフィードバック速度が必要かどうか |
| **output_type あり vs なし** | 後続処理で使う → あり、会話回答だけ → なし | 出力を構造化データとして扱うかどうか |
| **Guardrail の配置** | 禁止トピック → Input、品質チェック → Output | 問題を早期に遮断するほどコスト効率が良い |

### 3. 典型的なシステム構成パターン

#### パターン A: カスタマーサポート Bot（トリアージ型）

```text
ユーザー → 受付 Bot（軽量モデル）
                │ handoffs
        ┌───────┼───────┐
        ▼       ▼       ▼
    請求担当  技術担当  一般担当
  （高品質）（高品質）（軽量モデル）
                              │
                    + Input Guardrail（競合他社 / PII チェック）
```

#### パターン B: 情報収集・分析パイプライン

```text
調査 Bot ─→ 分析 Bot ─→ 執筆 Bot
（WebSearchTool）       （output_type: Report）
```

#### パターン C: コード生成と実行

```text
コーダー Bot ─→ レビュー Bot ─→ 修正 Bot（必要なら）
               + CodeInterpreterTool（実行して結果確認）
```

### 4. アンチパターン集（設計全体）

| アンチパターン | 症状 | 対処 |
|-------------|------|------|
| **神 Agent** | 1つのエージェントが何でもする | 役割で Agent を分けて Handoff を使う |
| **循環 Handoff** | A→B→A→B の無限委譲 | 委譲方向を一方向に設計し、max_turns を設定 |
| **Guardrail 乱用** | すべての検証を Guardrail に書く | 業務ロジックは Agent / Tool に書く |
| **トレース無効化** | 本番問題の原因追跡ができない | トレースを外部 APM に転送して永続化 |
| **docstring 省略** | ツールが使われない / 誤用される | 「いつ使うか」を明確に docstring に書く |

### 5. 次の一歩

overview（この教材）は**「地図」**です。実際に手を動かして動かしてみるには tutorial へ進んでください。

```text
今いる場所:
  openai-agents-sdk-overview.md ← あなたはここ
       ↓
  openai-agents-sdk-tutorial.md
  → uv での環境構築から、Agent/Tool/Handoff/Guardrail を順に実装
  → Python 3.10以上と OpenAI API キーがあれば始められる
```

---

## 関連リソース

- **公式ドキュメント**: https://openai.github.io/openai-agents-python/
- **GitHub リポジトリ**: https://github.com/openai/openai-agents-python
- **examples/ ディレクトリ**: カスタマーサポート・金融リサーチ・音声エージェントなどの実装例が揃っている

### 関連教材

- [openai-agents-sdk-tutorial.md](openai-agents-sdk-tutorial.md) — 手を動かして学ぶハンズオン教材（uv 環境構築から Handoff まで）
