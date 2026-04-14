# Google ADK 学習教材

> Google ADK 未経験者が基礎から実務活用まで学べる教材集

---

## Chapter 0: AIエージェントとは何か・ADKの概念とWhy Google ADK？

### 0-1. AIエージェントとは何か

#### 従来のAI（LLM単体）との違い

```
【従来のLLM】
ユーザー → 質問 → LLM → 回答（テキスト）
                         ↑
                    知識の範囲内でしか答えられない
                    リアルタイム情報なし
                    外部操作不可

【AIエージェント】
ユーザー → 指示 → エージェント → 計画を立てる
                       ↓
                   ツールを選んで実行
                   （Web検索・DB照会・コード実行・APIコール）
                       ↓
                   結果を評価して次のアクション決定
                       ↓
                   最終回答
```

**エージェントの本質：「考えて、行動して、結果を見て、また考える」ループ**

#### エージェントを構成する3要素

| 要素 | 説明 | 例 |
|------|------|----|
| **モデル（LLM）** | 頭脳。推論・判断を担う | Gemini, GPT-4o |
| **ツール（Tool）** | 手足。外部と接続する能力 | 検索、DB操作、メール送信 |
| **オーケストレーター（Orchestrator）** | 指揮者。複数エージェントを束ねる役割 | ADKのAgentクラス |

---

### 0-2. Google ADK（Agent Development Kit）とは

Google ADKは、Googleが提供するPythonベースのエージェント開発フレームワークです。

#### ADKの位置づけ

```
アプリケーション層
    └── Google ADK（エージェントの骨格を提供）
         ├── Geminiモデル（推論エンジン）
         ├── Tool calling（ツール呼び出し）
         ├── Session管理（会話の記憶）
         └── マルチエージェント協調
```

#### ADKが解決する問題

エージェントをゼロから作ると以下が必要になります：
- プロンプト管理
- ツール定義と呼び出しロジック
- 会話履歴の管理
- エラーハンドリング
- 複数エージェント間の通信

**ADKはこれらをすべてフレームワークとして提供します。**

---

### 0-3. Why Google ADK？

#### 主要フレームワーク比較

| フレームワーク | 特徴 | 向いているケース |
|---------------|------|-----------------|
| **LangChain** | 豊富なインテグレーション、歴史が長い | 多様なLLM・ツールを組み合わせたい |
| **AutoGen** | マルチエージェント会話が得意 | エージェント同士の対話を設計したい |
| **CrewAI** | 役割分担（Crew）が直感的 | チーム型エージェントを素早く作りたい |
| **Google ADK** | Googleエコシステムとの親和性、シンプルな設計 | Gemini・Google Cloudと統合したい |

#### ADKを選ぶ理由

1. **Geminiとのネイティブ統合** — Geminiモデルをそのまま使える
2. **シンプルなAPI設計** — `Agent`・`Tool`・`Runner`の3概念で理解できる
3. **マルチエージェント対応** — Phase 3以降で活きる協調設計が組み込み済み
4. **Nano Banana 2連携** — Geminiファミリーのモデルを統一的に扱える
5. **Google Cloud連携** — BigQuery・GCS・Vertex AIとの接続が自然

---

### 0-4. ADKの核心概念（用語整理）

#### Agent（エージェント）
```
目的を持ち、ツールを使って自律的に動くプログラム単位。
ADKでは LlmAgent クラスで表現される。
```

#### Tool（ツール）
```
エージェントが外部と接続するための「手段」。
Pythonの関数に @tool デコレータを付けるだけで定義できる。

例：
- search_web(query) → Web検索
- query_database(sql) → DB照会
- generate_image(prompt) → 画像生成
```

#### Session（セッション）
```
会話のコンテキスト（文脈）を保持する入れ物。
「誰と」「どんな目的で」「どこまで話したか」を管理する。
```

#### Runner（ランナー）
```
エージェントを実行するエンジン。
「このエージェントを、このセッションで、この入力で動かせ」と指示する。
```

#### Orchestrator（オーケストレーター）
```
複数のエージェントを束ねる指揮者エージェント。
「分析エージェントに頼んで、その結果をレポートエージェントに渡せ」
という指示を自分で判断して実行する。
```

---

### 0-5. エージェントが動く仕組み（ReActパターン）

ADKの内部では **ReAct**（Reasoning + Acting）パターンが使われています。

```
入力: 「今月の売上トップ5製品を分析して」
          ↓
[Reasoning] どのツールを使うべきか考える
「→ まずDBから売上データを取得する必要がある」
          ↓
[Acting] ツールを実行
query_database("SELECT product, sales FROM ... ORDER BY sales DESC LIMIT 5")
          ↓
[Observation] 結果を観察
「→ [製品A: 1200万, 製品B: 980万, ...]」
          ↓
[Reasoning] 次に何をすべきか考える
「→ データが取れた。分析してまとめる」
          ↓
最終回答を生成
```

このループは目標達成まで自動的に繰り返されます。

---

### 0-6. このプロジェクトで作るもの（全体像）

```
Phase 1  Hello Agent        → エージェントを動かす最小構成
Phase 2  ツールとアクション   → 外部DBや計算ツールを追加
Phase 3  マルチエージェント   → 分析役・執筆役・検証役を分けて協調
Phase 4  状態管理            → セッションをまたいだ記憶の保持
Phase 5  評価・デバッグ       → エージェントの品質を測る
Phase 6  実装課題            → データ取得→分析→レポート→インフォグラフィック
                               （Nano Banana 2 / gemini-3.1-flash-image-preview）
```

最終的に目指すシステム：

```
[データソース] → [分析エージェント] → [レポートエージェント]
                                              ↓
                               [インフォグラフィック生成エージェント]
                               （Nano Banana 2でビジュアル出力）
```

---

### 練習課題（Chapter 0）

**Q1（概念確認）**
以下のうち、「エージェントのツール」として適切でないものはどれか？

a) データベースに対してSQLを実行する関数
b) LLMが会話を記憶するための仕組み
c) 外部APIを呼び出してデータを取得する関数
d) CSVファイルを読み込んで集計する関数

**Q2（設計）**
「毎朝、売上レポートを自動生成してSlackに投稿するエージェント」を設計するとき、必要なToolを3つ挙げてください。

**Q3（考察）**
LLM単体とエージェントの違いを、「リアルタイム性」「外部操作」「自律性」の3つの軸で比較してください。

---

### まとめ（Chapter 0）

| キーワード | 一言説明 |
|-----------|---------|
| Agent | 目的を持って自律的に動くプログラム |
| Tool | エージェントの「手足」となる関数 |
| Session | 会話コンテキストを保持する入れ物 |
| Runner | エージェントを起動・実行するエンジン |
| Orchestrator | 複数エージェントを束ねる指揮者 |
| ReAct | 考える→行動する→観察するのループ |

---

## Chapter 1: 環境構築とHello Agent

### 1-1. 環境構築

#### 必要なもの

| 項目 | バージョン | 確認コマンド |
|------|-----------|-------------|
| Python | 3.10以上 | `python --version` |
| uv | 最新推奨 | `uv --version` |
| Gemini APIキー | — | Google AI Studio で取得 |

uvのインストール（未インストールの場合）：

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

#### ステップ1：プロジェクトの作成

```bash
# uv でプロジェクトを初期化（仮想環境も自動作成）
uv init my-adk-agent && cd my-adk-agent
```

#### ステップ2：ADKのインストール

```bash
uv add google-adk
```

インストール確認：

```bash
uv run python -c "import google.adk; print('ADK OK')"
```

#### ステップ3：APIキーの設定

```bash
# .env ファイルを作成（Gitにコミットしないこと）
echo "GOOGLE_API_KEY=your_api_key_here" > .env
```

APIキーの取得先：[Google AI Studio](https://aistudio.google.com/app/apikey)

#### ディレクトリ構成

```
my-adk-agent/
├── .env                  # APIキー（Git管理外）
├── .gitignore
├── .venv/                # uv が自動作成する仮想環境
├── pyproject.toml        # uv init が生成するプロジェクト設定
├── hello_agent/
│   ├── __init__.py
│   └── agent.py          # エージェント定義
└── main.py               # 実行エントリポイント
```

---

### 1-2. ADKの3つの核心クラス

Phase 0で学んだ概念がコードにどう対応するかを確認します。

```
概念              ADKクラス/関数
─────────────────────────────────
Agent        →   LlmAgent
Tool         →   @tool デコレータ付き関数
Runner       →   Runner
Session      →   InMemorySessionService
```

---

### 1-3. Hello Agent を作る

#### hello_agent/agent.py

```python
from google.adk.agents import LlmAgent

# エージェントの定義
root_agent = LlmAgent(
    name="hello_agent",
    model="gemini-2.5-flash",          # 使用するGeminiモデル
    description="挨拶と簡単な質問に答えるエージェント",
    instruction="""
あなたは親切なアシスタントです。
ユーザーに日本語で丁寧に回答してください。
""",
)
```

#### main.py

```python
import asyncio
from dotenv import load_dotenv
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai.types import Content, Part
from hello_agent.agent import root_agent

load_dotenv()  # .envからAPIキーを読み込む

async def main():
    # セッションサービスの初期化（会話の記憶を管理する）
    session_service = InMemorySessionService()

    # セッションを作成（アプリ名・ユーザーID・セッションIDを指定）
    session = await session_service.create_session(
        app_name="hello_app",
        user_id="user_001",
        session_id="session_001",
    )

    # ランナーの初期化（エージェントとセッションを紐付ける）
    runner = Runner(
        agent=root_agent,
        app_name="hello_app",
        session_service=session_service,
    )

    # ユーザーの入力を作成
    user_message = Content(
        role="user",
        parts=[Part(text="こんにちは！焼き芋って何ですか？")]
    )

    print("=== Hello Agent 起動 ===\n")

    # エージェントを実行し、イベントをストリームで受け取る
    async for event in runner.run_async(
        user_id="user_001",
        session_id="session_001",
        new_message=user_message,
    ):
        # 最終的な回答イベントのみ表示
        if event.is_final_response() and event.content and event.content.parts:
            print(f"エージェント: {event.content.parts[0].text}")

asyncio.run(main())
```

#### 実行

```bash
uv run python main.py
```

**期待される出力例：**
```
=== Hello Agent 起動 ===

エージェント: こんにちは！焼き芋（やきいも）とは、
サツマイモを焼いて調理したもののことです。...
```

---

### 1-4. コード解説

#### LlmAgent のパラメータ

```python
root_agent = LlmAgent(
    name="hello_agent",        # エージェントの識別名（一意であること）
    model="gemini-2.5-flash",  # Geminiモデル名
    description="...",         # このエージェントが何をするか（他のエージェントへの説明に使われる）
    instruction="...",         # system promptに相当。エージェントの振る舞いを規定する
)
```

#### Runner の役割

```
Runner = エージェント実行エンジン

Runner が持つ情報：
  - どのエージェントを使うか（root_agent）
  - どのアプリに属するか（app_name）
  - セッションをどこに保存するか（session_service）

Runner が行うこと：
  1. ユーザー入力をエージェントに渡す
  2. エージェントの思考・ツール実行をループ管理
  3. 各ステップのイベントをストリームで返す
```

#### InMemorySessionService

```
セッションをメモリ上に保持するサービス。
プロセスを終了するとデータは消える。

本番環境では DatabaseSessionService などを使って
会話履歴を永続化する（Phase 4で詳しく学ぶ）。
```

#### イベントストリームとは

```python
async for event in runner.run_async(...):
```

ADKはエージェントの処理を**イベントの連続**として返します。

```
イベントの種類（主なもの）：
  - ToolCallEvent      : ツールを呼び出した
  - ToolResultEvent    : ツールの結果が返ってきた
  - ModelResponseEvent : モデルが回答した
  - FinalResponseEvent : 最終回答（is_final_response() == True）
```

---

### 1-5. adk web でブラウザから試す（オプション）

ADKにはブラウザUIが内蔵されています。

```bash
# hello_agent ディレクトリの親から実行
uv run adk web
```

ブラウザで `http://localhost:8000` を開くと、チャットUIでエージェントと対話できます。

```
ディレクトリ構成の注意：
adk web は __init__.py と agent.py を持つパッケージを自動検出します。
root_agent という名前の LlmAgent が agent.py に定義されている必要があります。
```

---

### 1-6. モデル名の選び方

| モデル名 | 特徴 | 推奨用途 |
|---------|------|---------|
| `gemini-2.5-flash` | 高速・低コスト | 開発・テスト・学習 |
| `gemini-2.5-flash-thinking` | 思考プロセスを持つ | 複雑な推論タスク |
| `gemini-1.5-pro` | 高品質・大きなコンテキスト | 本番・長文処理 |

**学習中は `gemini-2.5-flash` を使うとコストを抑えられます。**

---

### 1-7. よくあるエラーと対処法

#### エラー1：APIキーが読み込まれない

```
google.auth.exceptions.DefaultCredentialsError
```

対処：
```python
# main.py の先頭で必ず load_dotenv() を呼ぶ
from dotenv import load_dotenv
load_dotenv()

# または環境変数を直接セット
import os
os.environ["GOOGLE_API_KEY"] = "your_key"
```

#### エラー2：モジュールが見つからない

```
ModuleNotFoundError: No module named 'hello_agent'
```

対処：
```bash
# hello_agent/__init__.py が存在するか確認
touch hello_agent/__init__.py

# main.py と同じ階層から実行しているか確認
pwd  # my-adk-agent/ であること

# uv run を使って実行しているか確認
uv run python main.py
```

#### エラー3：非同期関数の実行エラー

```
RuntimeError: asyncio.run() cannot be called when another event loop is running
```

対処：Jupyter Notebookで実行する場合は `asyncio.run()` の代わりに：
```python
import nest_asyncio
nest_asyncio.apply()
await main()
```

---

### 練習課題（Chapter 1）

**課題1（基本）**
`instruction` を変更して、エージェントを「データ分析の専門家」として振る舞わせてください。「平均値とは何か」と質問し、回答を確認してください。

**課題2（応用）**
会話を複数回続けてみてください（同じセッションIDで2回 `run_async` を呼ぶ）。2回目の入力で「さっきの質問を覚えていますか？」と聞いたとき、どう答えるか確認してください。

```python
# ヒント：2回目のメッセージを送る
user_message2 = Content(
    role="user",
    parts=[Part(text="さっきの質問を覚えていますか？")]
)
async for event in runner.run_async(
    user_id="user_001",
    session_id="session_001",   # 同じセッションIDを使う
    new_message=user_message2,
):
    if event.is_final_response() and event.content and event.content.parts:
        print(f"エージェント: {event.content.parts[0].text}")
```

**課題3（設計）**
Phase 6の最終目標「データ分析→レポート→インフォグラフィック」システムでは、どのような `instruction` を各エージェントに与えるべきか考えてみてください（コードは不要、文章で）。

---

### まとめ（Chapter 1）

```
Phase 1 で学んだこと：

1. ADKのインストールと環境構築
2. LlmAgent の定義（name / model / description / instruction）
3. InMemorySessionService でセッションを管理する
4. Runner でエージェントを実行する
5. イベントストリームから最終回答を取り出す
6. adk web でブラウザUIを使う
```

---

## Chapter 2: ツールとアクション

### 2-1. ツールとは何か

Phase 1のエージェントは「知識を持つ会話相手」でした。
しかし現実の業務では、**リアルタイムのデータ取得・計算・外部操作**が必要です。

それを可能にするのが **Tool（ツール）** です。

```
【ツールなしエージェント】
「今月の売上を教えて」→ 「学習データに基づくと...（古い情報）」

【ツールありエージェント】
「今月の売上を教えて」→ [DBツール実行] → 「今月の売上は1,234万円です」
```

#### ツールの定義方法（基本）

```python
def get_current_time() -> str:
    """現在の日時を返す"""
    from datetime import datetime
    return datetime.now().strftime("%Y年%m月%d日 %H:%M:%S")
```

**ポイント：**
- 普通のPython関数をそのまま `tools=[]` に渡すだけでADKがツールとして認識する
- **docstring（"""〜"""）が必須** — LLMがいつ使うかをここで判断する
- 引数と戻り値の型ヒントを書くとLLMの精度が上がる

---

### 2-2. ツールをエージェントに渡す

```python
from google.adk.agents import LlmAgent

def get_current_time() -> str:
    """現在の日時を返す"""
    from datetime import datetime
    return datetime.now().strftime("%Y年%m月%d日 %H:%M:%S")

root_agent = LlmAgent(
    name="time_agent",
    model="gemini-2.5-flash",
    description="時刻を教えてくれるエージェント",
    instruction="ユーザーの質問に日本語で答えてください。",
    tools=[get_current_time],   # ← ここにリストで渡す
)
```

---

### 2-3. 実践：データ分析ツールを作る

データ分析・レポート生成に向けて、実用的なツール群を実装します。

#### ファイル構成

```
my-adk-agent/
├── .env
├── analysis_agent/
│   ├── __init__.py
│   ├── agent.py
│   └── tools.py          # ツール定義をまとめるファイル
└── main.py
```

#### analysis_agent/tools.py

```python
from typing import Optional
import statistics


def calculate_stats(numbers: list[float]) -> dict:
    """
    数値リストの基本統計量（平均・中央値・最大・最小・標準偏差）を計算して返す。
    売上データや測定値の分析に使用する。

    Args:
        numbers: 分析対象の数値リスト（例: [100, 200, 150, 300]）

    Returns:
        平均・中央値・最大・最小・標準偏差を含む辞書
    """
    if not numbers:
        return {"error": "データが空です"}

    return {
        "count": len(numbers),
        "mean": round(statistics.mean(numbers), 2),
        "median": round(statistics.median(numbers), 2),
        "max": max(numbers),
        "min": min(numbers),
        "stdev": round(statistics.stdev(numbers), 2) if len(numbers) > 1 else 0,
    }


def get_sales_data(month: str) -> dict:
    """
    指定した月の売上データをデータベースから取得する。
    月は "2024-01" のような形式で指定する。

    Args:
        month: 対象月（YYYY-MM形式、例: "2024-01"）

    Returns:
        製品別の売上データ辞書
    """
    # 実際の開発ではここでDBクエリを実行する
    dummy_data = {
        "2024-01": {"製品A": 1200, "製品B": 980, "製品C": 1540, "製品D": 760},
        "2024-02": {"製品A": 1350, "製品B": 1020, "製品C": 1200, "製品D": 890},
        "2024-03": {"製品A": 1100, "製品B": 1150, "製品C": 1680, "製品D": 920},
    }

    if month not in dummy_data:
        return {"error": f"{month} のデータは存在しません"}

    return {
        "month": month,
        "sales": dummy_data[month],
        "total": sum(dummy_data[month].values()),
    }


def compare_months(month1: str, month2: str) -> dict:
    """
    2つの月の売上を比較して増減率を計算する。
    前月比・前年同月比などの分析に使用する。

    Args:
        month1: 比較元の月（YYYY-MM形式）
        month2: 比較先の月（YYYY-MM形式）

    Returns:
        各製品の売上増減額・増減率を含む辞書
    """
    dummy_data = {
        "2024-01": {"製品A": 1200, "製品B": 980, "製品C": 1540, "製品D": 760},
        "2024-02": {"製品A": 1350, "製品B": 1020, "製品C": 1200, "製品D": 890},
        "2024-03": {"製品A": 1100, "製品B": 1150, "製品C": 1680, "製品D": 920},
    }

    if month1 not in dummy_data or month2 not in dummy_data:
        return {"error": "指定された月のデータが存在しません"}

    d1 = dummy_data[month1]
    d2 = dummy_data[month2]

    comparison = {}
    for product in d1:
        diff = d2[product] - d1[product]
        rate = round((diff / d1[product]) * 100, 1)
        comparison[product] = {
            f"{month1}": d1[product],
            f"{month2}": d2[product],
            "増減": diff,
            "増減率(%)": rate,
        }

    return {"comparison": comparison}
```

#### analysis_agent/agent.py

```python
from google.adk.agents import LlmAgent
from .tools import calculate_stats, get_sales_data, compare_months

root_agent = LlmAgent(
    name="analysis_agent",
    model="gemini-2.5-flash",
    description="売上データの取得・統計分析・月次比較を行うデータ分析エージェント",
    instruction="""
あなたは優秀なデータアナリストです。
ユーザーの依頼に対して、適切なツールを使ってデータを取得・分析し、
わかりやすく日本語で報告してください。

数値を示す際は単位（万円）を明記し、
増減がある場合は必ずパーセンテージも示してください。
""",
    tools=[calculate_stats, get_sales_data, compare_months],
)
```

#### main.py

```python
import asyncio
from dotenv import load_dotenv
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai.types import Content, Part
from analysis_agent.agent import root_agent

load_dotenv()

async def ask(runner, user_id, session_id, text):
    """質問を送って最終回答を表示するヘルパー関数"""
    print(f"\nユーザー: {text}")
    print("-" * 40)

    message = Content(role="user", parts=[Part(text=text)])

    async for event in runner.run_async(
        user_id=user_id,
        session_id=session_id,
        new_message=message,
    ):
        if event.is_final_response():
            print(f"エージェント: {event.content.parts[0].text}")


async def main():
    session_service = InMemorySessionService()
    await session_service.create_session(
        app_name="analysis_app",
        user_id="analyst_01",
        session_id="session_001",
    )

    runner = Runner(
        agent=root_agent,
        app_name="analysis_app",
        session_service=session_service,
    )

    await ask(runner, "analyst_01", "session_001",
              "2024年1月の売上データを教えてください")

    await ask(runner, "analyst_01", "session_001",
              "[1200, 980, 1540, 760] の基本統計量を計算してください")

    await ask(runner, "analyst_01", "session_001",
              "2024年1月と2月の売上を比較して、増減をまとめてください")

asyncio.run(main())
```

---

### 2-4. ツール実行の内部フロー

```
ユーザー: 「2024年1月の売上データを教えて」
         ↓
[LLM が判断]
「get_sales_data ツールを使うべき。引数は month="2024-01"」
         ↓
[ADK が get_sales_data("2024-01") を実行]
{"month": "2024-01", "sales": {...}, "total": 4480}
         ↓
[LLM が結果を受け取り、自然言語に変換]
「2024年1月の売上は、製品A:1200万円、製品B:980万円...合計4480万円です」
         ↓
最終回答をユーザーに返す
```

このプロセスを **Tool Calling（ツール呼び出し）** と呼びます。

---

### 2-5. ツール設計のベストプラクティス

#### 1. docstring は LLM への説明文

```python
# 悪い例：docstringが短すぎてLLMが使いどころを判断できない
def get_data(month: str) -> dict:
    """データを取得する"""
    ...

# 良い例：いつ使うか・何を渡すか・何が返るかが明確
def get_sales_data(month: str) -> dict:
    """
    指定した月の製品別売上データをDBから取得する。
    売上レポート作成や前月比分析の前処理として使用する。

    Args:
        month: 対象月（YYYY-MM形式、例: "2024-01"）

    Returns:
        製品名をキー、売上金額（万円）を値とする辞書
    """
    ...
```

#### 2. エラーは例外でなく辞書で返す

```python
# 悪い例：例外を投げるとエージェントのループが止まる
def get_sales_data(month: str) -> dict:
    if month not in data:
        raise ValueError("データなし")   # NG

# 良い例：エラー情報を辞書で返してLLMに処理させる
def get_sales_data(month: str) -> dict:
    if month not in data:
        return {"error": f"{month} のデータは存在しません"}  # OK
```

#### 3. 1ツール1責任

```python
# 悪い例：1つのツールで取得・分析・フォーマットを全部やる
def get_and_analyze_and_format(month: str) -> str:
    ...

# 良い例：取得・分析・比較を分離してLLMに組み合わせさせる
def get_sales_data(month: str) -> dict: ...
def calculate_stats(numbers: list[float]) -> dict: ...
def compare_months(month1: str, month2: str) -> dict: ...
```

---

### 2-6. 組み込みツール（BuiltIn Tools）

ADKが標準で提供しているツールもあります。

```python
from google.adk.tools.built_in_tools import google_search

root_agent = LlmAgent(
    name="search_agent",
    model="gemini-2.5-flash",
    instruction="Web検索を使ってユーザーの質問に答えてください。",
    tools=[google_search],   # Google検索ツール
)
```

| 組み込みツール | 機能 |
|--------------|------|
| `google_search` | Googleでウェブ検索 |
| `code_execution` | Pythonコードを実行 |
| `vertex_ai_search` | Vertex AI Search を利用 |

---

### 2-7. イベントでツール実行を観察する

```python
async for event in runner.run_async(...):
    # ツール呼び出しイベント
    if hasattr(event, 'tool_calls') and event.tool_calls:
        for tc in event.tool_calls:
            print(f"[ツール呼び出し] {tc.name}({tc.args})")

    # ツール結果イベント
    if hasattr(event, 'tool_results') and event.tool_results:
        for tr in event.tool_results:
            print(f"[ツール結果] {tr.output}")

    # 最終回答
    if event.is_final_response():
        print(f"[最終回答] {event.content.parts[0].text}")
```

---

### 練習課題（Chapter 2）

**課題1（基本）**
以下の仕様で `format_report` ツールを追加してください。

```
機能：辞書データを日本語のマークダウンテーブルに変換する
引数：data（dict）, title（str）
戻り値：str（マークダウン形式の文字列）
```

**課題2（応用）**
`get_top_products` ツールを作成してください。

```
機能：指定月の売上データから上位N件の製品を返す
引数：month（str）, top_n（int, デフォルト3）
戻り値：製品名と売上のリスト（売上降順）
```

**課題3（設計）**
Phase 6の「データ取得→分析→レポート生成→インフォグラフィック」パイプラインで必要になりそうなツールを5つ考えて、関数名・引数・docstringの1行説明を書いてください（実装不要）。

---

### まとめ（Chapter 2）

```
Phase 2 で学んだこと：

1. 普通のPython関数を tools=[] に渡すだけでツールとして定義できる
2. docstring がLLMの「ツール選択の判断材料」になる
3. LlmAgent の tools=[] にリストで渡す
4. エラーは辞書で返してLLMに処理させる
5. 1ツール1責任の原則
6. 組み込みツール（google_search など）も使える
```

---

## Chapter 3: マルチエージェント設計

### 3-1. なぜマルチエージェントか

Phase 2では1つのエージェントに複数ツールを持たせました。
しかし複雑なタスクでは、**1エージェントにすべてを任せる設計**に限界があります。

```
【単一エージェントの問題】
・instructionが長くなり、行動が不安定になる
・「データ取得」「分析」「レポート執筆」「品質チェック」を1つで担うと責任が不明確
・一部を変更すると全体に影響する
・並列実行ができない

【マルチエージェントの解決策】
・各エージェントが1つの役割に集中 → 高品質・安定
・Orchestratorが全体を制御 → 変更に強い設計
・独立したエージェントは並列実行可能 → 高速化
```

#### 人間のチームに例えると

```
プロジェクトマネージャー（Orchestrator）
    ├── データエンジニア（DataAgent）  ← DBからデータ取得
    ├── データアナリスト（AnalysisAgent） ← 統計分析・洞察抽出
    ├── ライター（ReportAgent）         ← レポート文書生成
    └── レビュアー（ReviewAgent）       ← 内容の品質チェック
```

---

### 3-2. ADKのマルチエージェント構造

ADKでは **親エージェント（Orchestrator）が子エージェントを `sub_agents` として持つ**構造で表現します。

```python
orchestrator = LlmAgent(
    name="orchestrator",
    sub_agents=[data_agent, analysis_agent, report_agent],
    ...
)
```

#### エージェント間の委譲（ハンドオフ）

Orchestratorは `description` を読んで、**どの子エージェントに仕事を委譲するか**を自動判断します。

```
ユーザー → Orchestrator
              ↓ 「データが必要だ → DataAgentに委譲」
           DataAgent（実行）
              ↓ 結果を Orchestrator に返す
           Orchestrator
              ↓ 「分析が必要だ → AnalysisAgentに委譲」
           AnalysisAgent（実行）
              ↓ 結果を Orchestrator に返す
           Orchestrator
              ↓ 「レポートを書く → ReportAgentに委譲」
           ReportAgent（実行）
              ↓
           最終回答
```

---

### 3-3. 実践：分析レポート自動生成システム

#### ファイル構成

```
my-adk-agent/
├── .env
├── report_system/
│   ├── __init__.py
│   ├── agent.py          # Orchestrator定義
│   ├── sub_agents.py     # 各サブエージェント定義
│   └── tools.py          # ツール（Phase 2から流用・拡張）
└── main.py
```

#### report_system/sub_agents.py

```python
from google.adk.agents import LlmAgent
from .tools import get_sales_data, calculate_stats, get_top_products, compare_months, check_report_quality


data_agent = LlmAgent(
    name="data_agent",
    model="gemini-2.5-flash",
    description="""
    売上データの取得・集計を専門とするエージェント。
    「〇月のデータを取得して」「上位製品を教えて」「月比較して」
    などのデータ取得・集計タスクを担当する。
    """,
    instruction="""
    あなたはデータエンジニアです。
    ツールを使って正確にデータを取得し、
    取得したデータをそのまま構造化して報告してください。
    解釈や評価は行わず、データの提示に専念してください。
    """,
    tools=[get_sales_data, get_top_products, compare_months],
)

analysis_agent = LlmAgent(
    name="analysis_agent",
    model="gemini-2.5-flash",
    description="""
    数値データの統計分析・トレンド解釈・洞察抽出を専門とするエージェント。
    「統計量を計算して」「トレンドを分析して」「増減の理由を考察して」
    などの分析・解釈タスクを担当する。
    """,
    instruction="""
    あなたはデータアナリストです。
    提供されたデータをツールで統計分析し、
    ビジネス的な洞察（なぜ増えたか・何が重要か）を加えて報告してください。
    数値の羅列だけでなく、その意味を解説することが重要です。
    """,
    tools=[calculate_stats],
)

report_agent = LlmAgent(
    name="report_agent",
    model="gemini-2.5-flash",
    description="""
    分析結果をもとにビジネスレポートを執筆するエージェント。
    「レポートを作成して」「文書にまとめて」「報告書を書いて」
    などの文書生成タスクを担当する。
    """,
    instruction="""
    あなたはビジネスライターです。
    提供された分析データをもとに、経営層向けの月次売上レポートを作成してください。

    レポートの構成：
    1. エグゼクティブサマリー（3行以内）
    2. 数値ハイライト（箇条書き）
    3. トレンド分析（前月比・注目点）
    4. 結論と推奨アクション

    明確・簡潔・実用的なレポートを心がけてください。
    """,
    tools=[],
)

review_agent = LlmAgent(
    name="review_agent",
    model="gemini-2.5-flash",
    description="""
    生成されたレポートの品質チェック・改善提案を専門とするエージェント。
    「レポートをレビューして」「品質チェックして」「改善点を教えて」
    などのレビュータスクを担当する。
    """,
    instruction="""
    あなたは品質管理の専門家です。
    提供されたレポートをcheck_report_qualityツールで検証し、
    不足している要素があれば具体的な改善提案を行ってください。
    品質スコアが3未満の場合は、必ず改善点を明示してください。
    """,
    tools=[check_report_quality],
)
```

#### report_system/agent.py

```python
from google.adk.agents import LlmAgent
from .sub_agents import data_agent, analysis_agent, report_agent, review_agent


root_agent = LlmAgent(
    name="orchestrator",
    model="gemini-2.5-flash",
    description="売上データの取得・分析・レポート生成・品質チェックを統括するオーケストレーター",
    instruction="""
    あなたはプロジェクトマネージャーです。
    ユーザーの依頼を分析し、以下の手順で適切なエージェントに作業を委譲してください。

    【標準的なレポート生成フロー】
    1. data_agent   → 売上データの取得・集計
    2. analysis_agent → 統計分析・洞察の抽出
    3. report_agent   → レポート文書の生成
    4. review_agent   → 品質チェック・改善提案

    各エージェントの結果を次のエージェントに引き継ぎ、
    最終的にユーザーに完成したレポートを提供してください。
    """,
    sub_agents=[data_agent, analysis_agent, report_agent, review_agent],
)
```

---

### 3-4. エージェント間の通信パターン

ADKには3つの委譲パターンがあります。

#### パターン1：Sequential（逐次）— 最も一般的

```
Orchestrator → Agent_A → Agent_B → Agent_C → 結果
```

前のエージェントの出力が次のエージェントの入力になります。

#### パターン2：Parallel（並列）

```
Orchestrator → Agent_A ─┐
              → Agent_B ─┤→ 結果を集約
              → Agent_C ─┘
```

```python
from google.adk.agents import ParallelAgent

parallel_fetcher = ParallelAgent(
    name="parallel_fetcher",
    sub_agents=[fetch_jan_agent, fetch_feb_agent, fetch_mar_agent],
)
```

#### パターン3：Loop（ループ）

```
Orchestrator → Agent_A → 条件チェック → 不合格なら再実行
                                       → 合格なら次へ
```

```python
from google.adk.agents import LoopAgent

improvement_loop = LoopAgent(
    name="improvement_loop",
    sub_agents=[report_agent, review_agent],
    max_iterations=3,  # 最大3回繰り返す
)
```

---

### 3-5. description の書き方（委譲精度を上げる）

OrchestratorはサブエージェントのdescriptionだけをLLMが読んで委譲先を決めます。

```python
# 悪い例：何をするエージェントか不明確
data_agent = LlmAgent(
    name="data_agent",
    description="データを処理するエージェント",  # 曖昧
    ...
)

# 良い例：いつ委譲すべきかのトリガーワードを含める
data_agent = LlmAgent(
    name="data_agent",
    description="""
    売上データの取得・集計を専門とするエージェント。
    「〇月のデータを取得して」「上位製品を教えて」「月比較して」
    などのデータ取得・集計タスクを担当する。
    """,
    ...
)
```

---

### 3-6. デバッグ：どのエージェントが動いたか確認する

```python
async for event in runner.run_async(...):
    author = getattr(event, 'author', 'unknown')

    if hasattr(event, 'content') and event.content:
        for part in event.content.parts:
            if hasattr(part, 'function_call') and part.function_call:
                print(f"[{author}] ツール呼び出し: {part.function_call.name}")
            if hasattr(part, 'function_response') and part.function_response:
                print(f"[{author}] ツール結果受信: {part.function_response.name}")

    if event.is_final_response():
        print(f"\n[最終回答 by {author}]\n{event.content.parts[0].text}")
```

---

### 3-7. 全体アーキテクチャの整理

```
Phase 6 で目指す最終システム：

                    ┌─────────────────────────┐
User ──────────────▶│   OrchestratorAgent      │
                    │  （全体を統括・委譲判断）  │
                    └────────────┬────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                   ▼
    ┌──────────────┐  ┌──────────────────┐  ┌──────────────┐
    │  DataAgent   │  │  AnalysisAgent   │  │  ReportAgent │
    │（DB取得・集計）│  │（統計・洞察抽出） │  │（文書生成）  │
    └──────────────┘  └──────────────────┘  └──────────────┘
                                                      │
                                                      ▼
                                          ┌───────────────────┐
                                          │  InfographicAgent │
                                          │（Nano Banana 2で  │
                                          │  画像生成）       │
                                          └───────────────────┘
```

---

### 練習課題（Chapter 3）

**課題1（基本）**
`review_agent` がレポートの品質スコアを判定する部分で、スコアが2以下の場合に `report_agent` を再実行する仕組みを、`LoopAgent` を使って設計してみてください（コードの骨格だけでOK）。

**課題2（応用）**
3ヶ月分（2024-01, 02, 03）のデータを並列で取得する `ParallelAgent` を使った設計を実装してください。

**課題3（設計）**
「競合他社の価格情報をWebから取得する agent」を追加するとしたら、Orchestratorの `instruction` にどのような追記が必要か考えてください。また、そのエージェントにはどんなツールが必要かを設計してください。

---

### まとめ（Chapter 3）

```
Phase 3 で学んだこと：

1. マルチエージェントの必要性（責任分離・安定性・並列化）
2. sub_agents でOrchestratorに子エージェントを持たせる
3. descriptionがOrchestratorの委譲判断の根拠になる
4. Sequential / Parallel / Loop の3パターン
5. LoopAgent・ParallelAgent の使い方
6. イベントのauthorプロパティでどのエージェントが動いたか確認できる
```

---

## Chapter 4: 状態管理とセッション

### 4-1. セッションと状態管理が必要な理由

Phase 1〜3では `InMemorySessionService` を使いました。
これはプロセスが終わると**すべての会話履歴が消える**という問題があります。

```
【InMemorySessionService の問題】
プロセス起動 → 会話 → プロセス終了 → データ消滅
次回起動 → 「先週話した分析の続きをお願い」→ 「何のことですか？」

【永続化セッションで解決】
プロセス起動 → 会話 → プロセス終了 → DBに保存
次回起動 → 「先週話した分析の続きをお願い」→ 「先週の売上分析ですね。続きを始めます」
```

また、エージェント間でデータを共有する仕組みも必要です：

```
DataAgent が取得したデータを → AnalysisAgent が使う → ReportAgent が使う
           ↑
  これを「状態（State）」として管理する
```

---

### 4-2. ADKの状態管理の全体像

```
Session
├── id             : セッションの識別子
├── user_id        : ユーザーの識別子
├── app_name       : アプリ名
├── events         : 会話履歴（メッセージ・ツール実行記録）
└── state          : キー・バリューの辞書（エージェント間共有データ）
                     例: {"current_month": "2024-01", "report_draft": "..."}
```

#### State（状態）の3つのスコープ

| スコープ | プレフィックス | 有効範囲 |
|---------|-------------|---------|
| **Session State** | なし（デフォルト） | 現在のセッション内 |
| **User State** | `user:` | 同じユーザーの全セッション |
| **App State** | `app:` | アプリ全体（全ユーザー共通） |

```python
# 書き込み例（ToolContextを使う）
context.state["current_month"] = "2024-01"           # セッション限定
context.state["user:preferred_format"] = "markdown"  # ユーザー永続
context.state["app:company_name"] = "株式会社ABC"    # アプリ全体共通
```

---

### 4-3. ToolContext：ツール内でStateを読み書きする

ツール関数に `ToolContext` を引数として追加すると、State にアクセスできます。

```python
from google.adk.tools.tool_context import ToolContext


def save_analysis_result(result: dict, context: ToolContext) -> str:
    """
    分析結果をセッションStateに保存する。
    後続のエージェントがこのデータを参照できるようにする。

    Args:
        result: 保存する分析結果の辞書
        context: ADKが自動的に渡すセッションコンテキスト（指定不要）
    """
    context.state["latest_analysis"] = result
    return f"分析結果をStateに保存しました: {list(result.keys())}"


def get_analysis_result(context: ToolContext) -> dict:
    """
    前のエージェントが保存した分析結果をStateから取得する。
    ReportAgentやReviewAgentが分析データを参照するために使用する。
    """
    result = context.state.get("latest_analysis")
    if result is None:
        return {"error": "分析結果がStateに存在しません。先に分析を実行してください。"}
    return result
```

> **注意：** `context: ToolContext` はADKが自動注入します。ユーザーやLLMがこの引数を指定する必要はありません。

---

### 4-4. 永続化セッション：DatabaseSessionService

#### SQLiteを使った永続化

```bash
uv add sqlalchemy aiosqlite
```

```python
from google.adk.sessions import DatabaseSessionService

# SQLiteファイルにセッションを永続化
session_service = DatabaseSessionService(
    db_url="sqlite+aiosqlite:///./sessions.db"
)
```

#### PostgreSQLを使う場合（本番環境）

```bash
uv add asyncpg
```

```python
session_service = DatabaseSessionService(
    db_url="postgresql+asyncpg://user:password@localhost/mydb"
)
```

---

### 4-5. 実践：状態を引き継ぐ分析パイプライン

#### stateful_system/tools.py（抜粋）

```python
from google.adk.tools.tool_context import ToolContext
import statistics


def fetch_and_store_sales(month: str, context: ToolContext) -> dict:
    """
    指定月の売上データを取得してStateに保存する。
    以降のエージェントがこのデータを参照できるようにする。
    """
    dummy_data = {
        "2024-01": {"製品A": 1200, "製品B": 980, "製品C": 1540, "製品D": 760},
        "2024-02": {"製品A": 1350, "製品B": 1020, "製品C": 1200, "製品D": 890},
        "2024-03": {"製品A": 1100, "製品B": 1150, "製品C": 1680, "製品D": 920},
    }
    if month not in dummy_data:
        return {"error": f"{month} のデータは存在しません"}

    sales = dummy_data[month]
    result = {"month": month, "sales": sales, "total": sum(sales.values())}

    context.state["current_sales_data"] = result
    context.state["current_month"] = month
    return result


def analyze_and_store(context: ToolContext) -> dict:
    """StateのデータをStateから取得して統計分析し、分析結果をStateに保存する。"""
    data = context.state.get("current_sales_data")
    if not data:
        return {"error": "売上データがStateに存在しません"}

    sales_values = list(data["sales"].values())
    sorted_products = sorted(data["sales"].items(), key=lambda x: x[1], reverse=True)

    analysis = {
        "month": data["month"],
        "stats": {
            "mean": round(statistics.mean(sales_values), 2),
            "median": round(statistics.median(sales_values), 2),
            "max": max(sales_values),
            "min": min(sales_values),
            "total": data["total"],
        },
        "top_product": sorted_products[0][0],
        "low_product": sorted_products[-1][0],
        "sorted_products": sorted_products,
    }
    context.state["current_analysis"] = analysis
    return analysis
```

#### main.py（永続化版）

```python
import asyncio
from dotenv import load_dotenv
from google.adk.runners import Runner
from google.adk.sessions import DatabaseSessionService
from google.genai.types import Content, Part
from stateful_system.agent import root_agent

load_dotenv()

APP_NAME = "stateful_report_app"
USER_ID = "manager_01"
SESSION_ID = "persistent_session_001"


async def main():
    session_service = DatabaseSessionService(
        db_url="sqlite+aiosqlite:///./sessions.db"
    )

    session = await session_service.get_session(
        app_name=APP_NAME,
        user_id=USER_ID,
        session_id=SESSION_ID,
    )
    if session is None:
        session = await session_service.create_session(
            app_name=APP_NAME,
            user_id=USER_ID,
            session_id=SESSION_ID,
        )
        print("新規セッションを作成しました")
    else:
        print(f"既存セッションを再開します（State: {dict(session.state)}）")

    runner = Runner(
        agent=root_agent,
        app_name=APP_NAME,
        session_service=session_service,
    )

    message = Content(
        role="user",
        parts=[Part(text="2024年2月の売上データを分析して月次レポートを作成してください")]
    )

    async for event in runner.run_async(
        user_id=USER_ID,
        session_id=SESSION_ID,
        new_message=message,
    ):
        if event.is_final_response():
            print(event.content.parts[0].text)


asyncio.run(main())
```

---

### 4-6. セッションをまたいだ記憶の活用

```python
# 2回目の実行：前回のStateを活用する
message2 = Content(
    role="user",
    parts=[Part(text="前回のレポートと比較して、3月のレポートも作ってください")]
)

# 同じ SESSION_ID を使えば前回のStateが引き継がれる
async for event in runner.run_async(
    user_id=USER_ID,
    session_id=SESSION_ID,   # ← 同じセッションIDを使う
    new_message=message2,
):
    if event.is_final_response():
        print(event.content.parts[0].text)
```

---

### 4-7. State設計のベストプラクティス

#### キー命名規則

```python
# 悪い例：衝突しやすい短い名前
context.state["data"] = ...
context.state["result"] = ...

# 良い例：エージェント名+データ種別で名前空間を分ける
context.state["data_agent:sales_2024_01"] = ...
context.state["analysis_agent:stats_result"] = ...
context.state["report_agent:draft_v1"] = ...
```

#### Stateに保存すべきもの・すべきでないもの

```
✅ 保存すべき
  - エージェント間で共有するデータ（中間結果）
  - ユーザーの設定・好み（レポート形式など）
  - パイプラインの進捗状況

❌ 保存すべきでない
  - 大きなバイナリデータ（画像・PDFなど）→ ファイルシステムやGCSに保存
  - 頻繁に書き換わる揮発的なデータ → ツールの戻り値で直接渡す
  - 機密情報（APIキー・パスワード）→ 環境変数で管理
```

---

### 4-8. セッションサービスの選択指針

| サービス | 用途 | 特徴 |
|---------|------|------|
| `InMemorySessionService` | 開発・テスト | 高速・プロセス終了で消える |
| `DatabaseSessionService` (SQLite) | ローカル本番 | ファイルに永続化 |
| `DatabaseSessionService` (PostgreSQL) | クラウド本番 | スケーラブル・共有可能 |
| `VertexAiSessionService` | GCP本番 | Vertex AI管理・GCP統合 |

---

### 練習課題（Chapter 4）

**課題1（基本）**
`get_session_summary` ツールを呼ぶと、現在のStateに何が保存されているか確認できます。main.pyを改造して、各サブエージェントの実行後にStateのサマリーを表示するようにしてください。

**課題2（応用）**
`user:` プレフィックスを使って、ユーザーの「好みのレポート言語（日本語/英語）」をStateに保存し、`report_agent` がその設定を読んでレポートの言語を切り替えるように改造してください。

```python
# ヒント
context.state["user:preferred_language"] = "english"
```

**課題3（設計）**
同じユーザーが3ヶ月間（2024年1〜3月）毎月レポートを依頼するユースケースで、過去のレポートを参照しながら「前月比トレンドコメント」を自動追記する設計を考えてください。

---

### まとめ（Chapter 4）

```
Phase 4 で学んだこと：

1. InMemorySessionService → DatabaseSessionService で永続化
2. session.state でエージェント間データ共有
3. State のスコープ：Session / User（user:）/ App（app:）
4. ToolContext を引数に追加するだけでState読み書きが可能
5. 既存セッションの get_session → create_session のパターン
6. Stateキーの命名規則とアンチパターン
```

---

## Chapter 5: 評価・デバッグ・モニタリング

### 5-1. なぜ評価・デバッグが必要か

```
【評価なしの問題】
・エージェントの回答が良いのか悪いのかわからない
・プロンプトを変えたら改善したのか悪化したのかわからない
・どのエージェントがボトルネックか特定できない
・本番障害の原因を追跡できない

【評価・モニタリングありの状態】
・回答品質をスコアで定量比較できる
・プロンプト改善の効果を数値で確認できる
・遅いエージェント・失敗率の高いツールを特定できる
・本番のトレースログで原因を追跡できる
```

---

### 5-2. デバッグの基本：イベントログを読む

#### 全イベントを詳細に出力する

```python
async def run_with_debug(runner, user_id, session_id, text):
    """全イベントを詳細に表示するデバッグ実行関数"""
    message = Content(role="user", parts=[Part(text=text)])
    print(f"\n{'='*60}")
    print(f"入力: {text}")
    print(f"{'='*60}")

    async for event in runner.run_async(
        user_id=user_id,
        session_id=session_id,
        new_message=message,
    ):
        author = getattr(event, 'author', 'unknown')
        content = getattr(event, 'content', None)

        if content is None:
            continue

        for part in content.parts:
            if hasattr(part, 'text') and part.text:
                label = "[最終回答]" if event.is_final_response() else "[思考中]"
                print(f"{label} [{author}] {part.text[:100]}...")

            if hasattr(part, 'function_call') and part.function_call:
                fc = part.function_call
                print(f"[ツール呼び出し] [{author}] {fc.name}({dict(fc.args)})")

            if hasattr(part, 'function_response') and part.function_response:
                fr = part.function_response
                print(f"[ツール結果] [{author}] {fr.name} → {str(fr.response)[:80]}...")
```

#### ADKの組み込みログを有効化

```python
import logging

logging.basicConfig(level=logging.INFO)

# より詳細なデバッグ情報が必要な場合
logging.getLogger("google.adk").setLevel(logging.DEBUG)
```

---

### 5-3. ADK組み込みの評価フレームワーク

ADKには `pytest` ベースの評価フレームワークが内蔵されています。

#### テストケース定義：tests/test_data/report_eval.test.json

```json
[
  {
    "query": "2024年1月の売上データを取得してください",
    "expected_tool_use": [
      {
        "tool_name": "get_sales_data",
        "tool_input": {"month": "2024-01"}
      }
    ],
    "reference": "2024年1月の売上データ：製品A 1200万円、製品B 980万円、製品C 1540万円、製品D 760万円、合計 4480万円"
  },
  {
    "query": "売上上位3製品を教えてください（2024年1月）",
    "expected_tool_use": [
      {
        "tool_name": "get_top_products",
        "tool_input": {"month": "2024-01", "top_n": 3}
      }
    ],
    "reference": "上位3製品は製品C（1540万円）、製品A（1200万円）、製品B（980万円）です"
  }
]
```

#### テスト実行スクリプト：tests/test_report_agent.py

```python
import pytest
from google.adk.evaluation import AgentEvaluator


@pytest.mark.asyncio
async def test_tool_use_and_response():
    """ツール呼び出しと回答品質を評価するテスト"""
    await AgentEvaluator.evaluate(
        agent_module="report_system",
        eval_dataset_file_path="tests/test_data/report_eval.test.json",
        num_runs=1,
    )
```

```bash
# pytestで実行
uv run pytest tests/test_report_agent.py -v

# ADK CLIで実行（より詳細な結果が見られる）
uv run adk eval report_system tests/test_data/report_eval.test.json
```

---

### 5-4. 評価メトリクスの種類

| メトリクス | 内容 | スコア範囲 |
|-----------|------|-----------|
| `tool_trajectory_avg_score` | 正しいツールを正しい引数で呼んだか | 0.0〜1.0 |
| `response_match_score` | 期待回答との意味的な近さ | 0.0〜1.0 |
| `safety_score` | 安全でない出力がないか | 0.0〜1.0 |

---

### 5-5. カスタム評価関数を作る

```python
from dataclasses import dataclass
from typing import Callable


@dataclass
class EvalCase:
    query: str
    checker: Callable[[str], bool]
    description: str


EVAL_CASES = [
    EvalCase(
        query="2024年1月の売上データを教えてください",
        checker=lambda r: "1200" in r and "製品C" in r,
        description="売上データ取得：主要数値が含まれるか",
    ),
    EvalCase(
        query="[1200, 980, 1540, 760] の平均を計算してください",
        checker=lambda r: "1120" in r,
        description="統計計算：正しい平均値（1120）が含まれるか",
    ),
    EvalCase(
        query="2024年1月と2月の売上を比較してください",
        checker=lambda r: ("増" in r or "減" in r) and "%" in r,
        description="月次比較：増減とパーセンテージが含まれるか",
    ),
    EvalCase(
        query="2024年4月のデータを取得してください",
        checker=lambda r: "存在しません" in r or "ありません" in r or "error" in r.lower(),
        description="エラーハンドリング：存在しないデータへの適切な応答",
    ),
]
```

---

### 5-6. モニタリング：Langfuseとの連携

```bash
uv add langfuse opentelemetry-sdk opentelemetry-exporter-otlp-proto-http
```

```python
# .env に追加
# LANGFUSE_PUBLIC_KEY=pk-lf-...
# LANGFUSE_SECRET_KEY=sk-lf-...
# LANGFUSE_HOST=https://cloud.langfuse.com
```

```python
import os
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace.export import SimpleSpanProcessor
from google.adk.instrumentation import instrument_adk

exporter = OTLPSpanExporter(
    endpoint=f"{os.getenv('LANGFUSE_HOST')}/api/public/otel/v1/traces",
    headers={
        "Authorization": f"Basic {os.getenv('LANGFUSE_AUTH')}",
    },
)

provider = TracerProvider()
provider.add_span_processor(SimpleSpanProcessor(exporter))
instrument_adk(provider)
```

---

### 5-7. モニタリング：Cloud Trace（GCP本番環境）

```bash
uv add opentelemetry-exporter-gcp-trace
```

```python
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from google.adk.instrumentation import instrument_adk

exporter = CloudTraceSpanExporter(project_id="your-gcp-project-id")
provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(exporter))
instrument_adk(provider)
```

---

### 5-8. 評価ループの実践パターン

```
┌─────────────────────────────────────────────────────────┐
│                   評価改善サイクル                        │
│                                                          │
│  1. テストケース作成                                      │
│     └── 「正しい動作」を eval.json に定義する            │
│              ↓                                           │
│  2. ベースライン測定                                      │
│     └── adk eval で現状スコアを記録する                  │
│              ↓                                           │
│  3. 改善（instruction / tools / sub_agents を変更）      │
│              ↓                                           │
│  4. 再評価                                               │
│     └── スコアが上がったか確認する                       │
│              ↓                                           │
│  5. 本番デプロイ → Langfuse/Cloud Traceで監視           │
│              ↓                                           │
│  1. に戻る（新しいケースを追加）                         │
└─────────────────────────────────────────────────────────┘
```

---

### 5-9. よくある問題と診断方法

#### 問題1：エージェントが間違ったツールを呼ぶ

```
診断：tool_trajectory_avg_score が低い
原因：ツールのdocstringが不明確 / instructionが競合している

対処：
1. docstringに「いつ使うか」を明確に書く
2. 似たツールは名前で差別化する
3. instructionでツールの使い分けを明示する
```

#### 問題2：回答品質が安定しない

```python
# モデルのtemperatureを下げて確定的な回答を得る
from google.genai.types import GenerateContentConfig

agent = LlmAgent(
    ...
    generate_content_config=GenerateContentConfig(temperature=0.1),
)
```

#### 問題3：レイテンシが高い

```
診断：Langfuseのトレースでどのステップが遅いか確認

よくある原因と対処：
・Orchestratorの委譲判断が遅い → sub_agentsの数を減らす
・ツールのAPI呼び出しが遅い → キャッシュを追加する
・直列処理になっている → 独立タスクはParallelAgentを使う
```

#### 問題4：無限ループ・停止しない

```python
runner = Runner(
    agent=root_agent,
    app_name="app",
    session_service=session_service,
    max_llm_calls=20,   # 最大LLM呼び出し回数を制限
)
```

---

### 練習課題（Chapter 5）

**課題1（基本）**
Phase 2で作った `analysis_agent` に対して、以下の評価ケースを `report_eval.test.json` 形式で3件追加してください。

```
ケース1：正しいツールが呼ばれるか（calculate_stats）
ケース2：存在しない月を指定したときのエラーハンドリング
ケース3：2ヶ月比較の結果に増減率が含まれるか
```

**課題2（応用）**
`evaluate_agent()` 関数を使って、Phase 3で作った `report_system` の評価を実行してみてください。少なくとも5ケースの `EvalCase` を定義し、スコアが何%になるか確認してください。

**課題3（設計）**
Phase 6の最終システムにおいて、モニタリングで特に注視すべき指標を3つ挙げ、それぞれ「なぜ重要か」を説明してください。

---

### まとめ（Chapter 5）

```
Phase 5 で学んだこと：

1. イベントログでリアルタイムにデバッグする
2. ADK組み込み評価（AgentEvaluator + .test.json）でツール精度を測る
3. カスタムchecker関数で業務固有の品質基準を実装する
4. Langfuse / Cloud Trace でトレース・コスト・レイテンシを可視化する
5. 評価→改善→再評価のサイクルを回す
6. よくある問題（誤ったツール選択・不安定な回答・遅延・ループ）の診断と対処
```

---

## Chapter 6: データ分析・レポート＆インフォグラフィック自動生成

### 6-1. このフェーズの目標

Phase 1〜5の知識をすべて統合し、以下の一気通貫パイプラインを実装します。

```
【実装する自動化パイプライン】

  ユーザー入力:「2024年Q1の売上分析レポートとインフォグラフィックを作って」
       ↓
  ┌────────────────────────────────────────────────────────┐
  │                  OrchestratorAgent                      │
  └───────────────────────┬────────────────────────────────┘
                          │ 順次委譲
        ┌─────────────────┼─────────────────────┐
        ▼                 ▼                      ▼
  [DataAgent]      [AnalysisAgent]        [ReportAgent]
  DBからデータ取得    統計分析・洞察抽出     マークダウンレポート生成
  Stateに保存        Stateに保存            Stateに保存
        └─────────────────┴──────────────────────┘
                          ↓
                  [InfographicAgent]
                  Nano Banana 2 で画像生成
                  (gemini-3.1-flash-image-preview)
                          ↓
              レポートPDF + インフォグラフィック画像
```

---

### 6-2. Nano Banana 2（画像生成モデル）とは

**Nano Banana 2** は Gemini ファミリーの画像生成モデルです。ADK の `LlmAgent` から直接呼び出せます。

| 項目 | 内容 |
|------|------|
| モデルID | `gemini-3.1-flash-image-preview` |
| 用途 | テキストから画像を生成 |
| 特徴 | テキスト指示に基づいた高品質な画像生成 |
| 出力 | PNG/JPEG形式のバイナリデータ |

#### 画像生成の基本（ADK外・単体テスト用）

```python
import os
import base64
from pathlib import Path
from google import genai
from google.genai.types import GenerateContentConfig
from dotenv import load_dotenv

load_dotenv()

client = genai.Client(api_key=os.getenv("GOOGLE_API_KEY"))

response = client.models.generate_content(
    model="gemini-3.1-flash-image-preview",
    contents="""
    以下のデータを表すビジネスインフォグラフィックを生成してください：
    タイトル：2024年Q1 売上サマリー
    - 製品C が最高売上（1540万円）
    - 総売上 4480万円
    - 前月比 +12.5%
    スタイル：プロフェッショナル、青系カラーパレット、棒グラフあり
    """,
    config=GenerateContentConfig(response_modalities=["IMAGE", "TEXT"]),
)

for part in response.candidates[0].content.parts:
    if part.inline_data and part.inline_data.mime_type.startswith("image/"):
        image_data = base64.b64decode(part.inline_data.data)
        Path("output_infographic.png").write_bytes(image_data)
        print("画像を output_infographic.png に保存しました")
```

---

### 6-3. 完全実装：一気通貫パイプライン

#### ファイル構成

```
my-adk-agent/
├── .env
├── sessions.db
├── output/                    # 生成ファイルの出力先
├── pipeline/
│   ├── __init__.py
│   ├── agent.py               # Orchestrator
│   ├── sub_agents.py          # 各サブエージェント
│   └── tools.py               # 全ツール定義
└── main.py
```

#### pipeline/tools.py

```python
import os
import base64
import statistics
from pathlib import Path
from datetime import datetime
from google.adk.tools.tool_context import ToolContext
from google import genai
from google.genai.types import GenerateContentConfig


def fetch_quarterly_sales(year: int, quarter: int, context: ToolContext) -> dict:
    """
    指定した年・四半期（Q1〜Q4）の月別売上データを取得しStateに保存する。
    年はint（例: 2024）、四半期はint（1〜4）で指定する。
    """
    quarterly_map = {1: ["01", "02", "03"], 2: ["04", "05", "06"],
                     3: ["07", "08", "09"], 4: ["10", "11", "12"]}

    if quarter not in quarterly_map:
        return {"error": "quarterは1〜4で指定してください"}

    dummy_db = {
        "2024-01": {"製品A": 1200, "製品B": 980,  "製品C": 1540, "製品D": 760},
        "2024-02": {"製品A": 1350, "製品B": 1020, "製品C": 1200, "製品D": 890},
        "2024-03": {"製品A": 1100, "製品B": 1150, "製品C": 1680, "製品D": 920},
    }

    months = quarterly_map[quarter]
    result = {}
    for m in months:
        key = f"{year}-{m}"
        result[key] = dummy_db.get(key, {})

    payload = {
        "year": year,
        "quarter": quarter,
        "label": f"{year}年Q{quarter}",
        "monthly_data": result,
        "all_products": list(set(p for d in result.values() for p in d)),
    }
    context.state["quarterly_sales"] = payload
    return payload


def analyze_quarterly_sales(context: ToolContext) -> dict:
    """StateのQ1売上データを統計分析し、洞察をStateに保存する。"""
    data = context.state.get("quarterly_sales")
    if not data:
        return {"error": "売上データがStateに存在しません"}

    monthly = data["monthly_data"]
    products = data["all_products"]

    product_totals = {p: sum(monthly[m].get(p, 0) for m in monthly) for p in products}
    monthly_totals = {m: sum(monthly[m].values()) for m in monthly}
    monthly_values = list(monthly_totals.values())

    months_sorted = sorted(monthly_totals.keys())
    mom_growth = {}
    for i in range(1, len(months_sorted)):
        prev, curr = months_sorted[i-1], months_sorted[i]
        if monthly_totals[prev] > 0:
            rate = round((monthly_totals[curr] - monthly_totals[prev]) / monthly_totals[prev] * 100, 1)
            mom_growth[f"{prev}→{curr}"] = rate

    sorted_products = sorted(product_totals.items(), key=lambda x: x[1], reverse=True)

    analysis = {
        "label": data["label"],
        "product_totals": product_totals,
        "monthly_totals": monthly_totals,
        "quarter_total": sum(monthly_values),
        "monthly_avg": round(statistics.mean(monthly_values), 1),
        "top_product": sorted_products[0],
        "low_product": sorted_products[-1],
        "sorted_products": sorted_products,
        "mom_growth": mom_growth,
        "growth_trend": "上昇" if list(mom_growth.values()) and list(mom_growth.values())[-1] > 0 else "下降",
    }
    context.state["quarterly_analysis"] = analysis
    return analysis


def generate_markdown_report(context: ToolContext) -> str:
    """Stateの分析結果からマークダウン形式の経営レポートを生成しStateと.mdファイルに保存する。"""
    analysis = context.state.get("quarterly_analysis")
    if not analysis:
        return "エラー：分析結果がStateに存在しません"

    label = analysis["label"]
    now = datetime.now().strftime("%Y年%m月%d日")
    top_p, top_v = analysis["top_product"]
    low_p, low_v = analysis["low_product"]

    lines = [
        f"# {label} 売上分析レポート",
        f"作成日: {now}",
        "",
        "## エグゼクティブサマリー",
        f"{label}の総売上は **{analysis['quarter_total']:,}万円**。",
        f"トップ製品は **{top_p}**（{top_v:,}万円）、",
        f"月次平均は{analysis['monthly_avg']:,}万円、トレンドは**{analysis['growth_trend']}傾向**。",
        "",
        "## 月次売上推移",
        "| 月 | 売上（万円） | 前月比 |",
        "|---|---|---|",
    ]
    for month, total in sorted(analysis["monthly_totals"].items()):
        growth_key = [k for k in analysis["mom_growth"] if k.endswith(month)]
        growth_str = f"{analysis['mom_growth'][growth_key[0]]:+.1f}%" if growth_key else "—"
        lines.append(f"| {month} | {total:,} | {growth_str} |")

    lines += [
        "",
        "## 製品別売上ランキング",
        "| 順位 | 製品 | 売上（万円） |",
        "|---|---|---|",
    ]
    for i, (p, v) in enumerate(analysis["sorted_products"], 1):
        lines.append(f"| {i} | {p} | {v:,} |")

    lines += [
        "",
        "## 結論と推奨アクション",
        f"- **{top_p}** は引き続き最重要製品。在庫・プロモーション強化を優先する",
        f"- **{low_p}** は売上低迷。原因分析と改善施策の立案が必要",
        f"- トレンドが{analysis['growth_trend']}傾向のため、次四半期の予算配分を見直す",
    ]

    report = "\n".join(lines)
    context.state["markdown_report"] = report

    output_dir = Path("output")
    output_dir.mkdir(exist_ok=True)
    report_path = output_dir / f"report_{label.replace(' ', '_')}.md"
    report_path.write_text(report, encoding="utf-8")
    return report


def generate_infographic(context: ToolContext) -> str:
    """
    Stateの分析結果をもとにNano Banana 2（gemini-3.1-flash-image-preview）で
    インフォグラフィック画像を生成し output/ ディレクトリに保存する。
    generate_markdown_reportの実行後に呼び出す。
    """
    analysis = context.state.get("quarterly_analysis")
    if not analysis:
        return "エラー：分析結果がStateに存在しません"

    label = analysis["label"]
    top_p, top_v = analysis["top_product"]

    monthly_lines = "\n".join(
        f"  - {m}: {v:,}万円"
        for m, v in sorted(analysis["monthly_totals"].items())
    )
    product_lines = "\n".join(
        f"  {i+1}. {p}: {v:,}万円"
        for i, (p, v) in enumerate(analysis["sorted_products"])
    )

    prompt = f"""
以下のビジネスデータを表す、プロフェッショナルなインフォグラフィックを生成してください。

【データ】
タイトル: {label} 売上インフォグラフィック
総売上: {analysis['quarter_total']:,}万円
月次売上:
{monthly_lines}
製品別ランキング:
{product_lines}
トップ製品: {top_p}（{top_v:,}万円）
トレンド: {analysis['growth_trend']}傾向

【デザイン要件】
- 配色: ネイビー・スカイブルー・ホワイトのビジネス配色
- レイアウト: 上部にKPI数値、中央に棒グラフ（月次推移）、下部に製品ランキング
- フォント: 日本語対応、見出しは大きく太く
- スタイル: クリーン・モダン・経営層向けプレゼン品質
- サイズ感: A4横向き相当
"""

    client = genai.Client(api_key=os.getenv("GOOGLE_API_KEY"))

    try:
        response = client.models.generate_content(
            model="gemini-3.1-flash-image-preview",
            contents=prompt,
            config=GenerateContentConfig(response_modalities=["IMAGE", "TEXT"]),
        )

        output_dir = Path("output")
        output_dir.mkdir(exist_ok=True)
        image_path = output_dir / f"infographic_{label.replace(' ', '_')}.png"

        for part in response.candidates[0].content.parts:
            if part.inline_data and part.inline_data.mime_type.startswith("image/"):
                image_data = base64.b64decode(part.inline_data.data)
                image_path.write_bytes(image_data)
                context.state["infographic_path"] = str(image_path)
                return f"インフォグラフィックを保存しました: {image_path}"

        return "エラー：画像データが生成されませんでした"

    except Exception as e:
        return f"画像生成エラー: {str(e)}"


def get_pipeline_summary(context: ToolContext) -> dict:
    """パイプラインの実行状況と生成物のパスをまとめて返す。"""
    return {
        "データ取得": "✅" if "quarterly_sales" in context.state else "❌",
        "分析完了": "✅" if "quarterly_analysis" in context.state else "❌",
        "レポート生成": "✅" if "markdown_report" in context.state else "❌",
        "インフォグラフィック": "✅" if "infographic_path" in context.state else "❌",
        "画像パス": context.state.get("infographic_path", "未生成"),
    }
```

#### pipeline/sub_agents.py

```python
from google.adk.agents import LlmAgent
from .tools import (
    fetch_quarterly_sales,
    analyze_quarterly_sales,
    generate_markdown_report,
    generate_infographic,
    get_pipeline_summary,
)


data_agent = LlmAgent(
    name="data_agent",
    model="gemini-2.5-flash",
    description="""
    売上データをDBから取得してStateに保存するエージェント。
    「データを取得して」「〇年Q〇のデータが必要」などのデータ取得タスクを担当する。
    """,
    instruction="""
    fetch_quarterly_sales ツールを使って指定された年・四半期のデータを取得・保存してください。
    完了後、取得したデータ（対象期間・製品数・月数）の概要を報告してください。
    """,
    tools=[fetch_quarterly_sales],
)

analysis_agent = LlmAgent(
    name="analysis_agent",
    model="gemini-2.5-flash",
    description="""
    Stateのデータを統計分析して洞察を抽出するエージェント。
    「分析して」「統計を計算して」「トレンドを教えて」などの分析タスクを担当する。
    """,
    instruction="""
    analyze_quarterly_sales ツールを使ってStateのデータを分析・保存してください。
    完了後、主要な洞察（トップ製品・トレンド・注目点）を3点で報告してください。
    """,
    tools=[analyze_quarterly_sales],
)

report_agent = LlmAgent(
    name="report_agent",
    model="gemini-2.5-flash",
    description="""
    分析結果からマークダウンレポートを生成するエージェント。
    「レポートを作成して」「文書にまとめて」などの文書生成タスクを担当する。
    """,
    instruction="""
    generate_markdown_report ツールを使ってレポートを生成・保存してください。
    生成したレポートの構成（セクション数・主要KPI）を報告してください。
    """,
    tools=[generate_markdown_report],
)

infographic_agent = LlmAgent(
    name="infographic_agent",
    model="gemini-2.5-flash",
    description="""
    Nano Banana 2（gemini-3.1-flash-image-preview）を使ってインフォグラフィック画像を生成するエージェント。
    「インフォグラフィックを作って」「画像で見せて」「ビジュアライズして」などの画像生成タスクを担当する。
    """,
    instruction="""
    generate_infographic ツールを使ってインフォグラフィック画像を生成してください。
    完了後、保存先パスと画像の概要を報告してください。
    """,
    tools=[generate_infographic, get_pipeline_summary],
)
```

#### pipeline/agent.py

```python
from google.adk.agents import LlmAgent
from .sub_agents import data_agent, analysis_agent, report_agent, infographic_agent

root_agent = LlmAgent(
    name="orchestrator",
    model="gemini-2.5-flash",
    description="売上分析・レポート・インフォグラフィック生成パイプラインを統括するオーケストレーター",
    instruction="""
    あなたはデータ分析パイプラインのプロジェクトマネージャーです。
    ユーザーの依頼を受けたら、以下の順序でサブエージェントに作業を委譲してください。

    【実行順序（必ず守ること）】
    1. data_agent        → 指定年・四半期のデータ取得（Stateに保存）
    2. analysis_agent    → 統計分析・洞察抽出（Stateに保存）
    3. report_agent      → マークダウンレポート生成（ファイル出力 + Stateに保存）
    4. infographic_agent → Nano Banana 2でインフォグラフィック画像生成（ファイル出力）

    各エージェントの処理はStateを介して引き継がれます。
    全ステップ完了後、生成されたファイルのパスをユーザーに報告してください。
    """,
    sub_agents=[data_agent, analysis_agent, report_agent, infographic_agent],
)
```

#### main.py

```python
import asyncio
from dotenv import load_dotenv
from google.adk.runners import Runner
from google.adk.sessions import DatabaseSessionService
from google.genai.types import Content, Part
from pipeline.agent import root_agent

load_dotenv()

APP_NAME = "infographic_pipeline"
USER_ID = "analyst_01"
SESSION_ID = "pipeline_session_001"


async def main():
    session_service = DatabaseSessionService(
        db_url="sqlite+aiosqlite:///./sessions.db"
    )

    session = await session_service.get_session(
        app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID
    )
    if session is None:
        await session_service.create_session(
            app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID
        )

    runner = Runner(
        agent=root_agent,
        app_name=APP_NAME,
        session_service=session_service,
    )

    print("=" * 60)
    print("  売上分析 → レポート → インフォグラフィック 自動生成")
    print("=" * 60)

    message = Content(
        role="user",
        parts=[Part(text="2024年Q1（1〜3月）の売上データを分析して、マークダウンレポートとインフォグラフィックを生成してください")]
    )

    async for event in runner.run_async(
        user_id=USER_ID,
        session_id=SESSION_ID,
        new_message=message,
    ):
        author = getattr(event, 'author', None)
        if author and not event.is_final_response():
            content = getattr(event, 'content', None)
            if content:
                for part in content.parts:
                    if hasattr(part, 'function_call') and part.function_call:
                        print(f"\n[{author}] ▶ {part.function_call.name}() 実行中...")

        if event.is_final_response():
            print(f"\n{'='*60}")
            print("【最終報告】")
            print(event.content.parts[0].text)

    print("\n出力ファイルを確認してください: ./output/")


asyncio.run(main())
```

---

### 6-4. インフォグラフィックのプロンプト設計

#### 効果的なプロンプト構造

```python
prompt = """
【必須要素】
1. タイトルと対象期間
2. 具体的な数値データ（LLMに数値を直接渡す）
3. グラフの種類（棒グラフ・円グラフ・折れ線グラフ）
4. カラースキーム
5. ターゲット（経営層向け・営業向けなど）
"""
```

#### スタイルバリエーション

```python
STYLES = {
    "経営層向け": "ネイビー基調・シンプル・数値重視・余白多め",
    "営業向け":   "オレンジ/赤基調・ダイナミック・ランキング強調",
    "広報向け":   "カラフル・アイコン多用・ストーリー性・SNS映え",
    "社内報告":   "グレー基調・表形式・詳細数値・凡例あり",
}
```

---

### 6-5. 発展：本番環境への移行チェックリスト

```python
# Step 1: ダミーDBをリアルDBに置き換える
def fetch_quarterly_sales(year: int, quarter: int, context: ToolContext) -> dict:
    import sqlalchemy
    engine = sqlalchemy.create_engine(os.getenv("DATABASE_URL"))
    # ↑ ここをダミーデータからSQLクエリに変更するだけ

# Step 2: セッションをCloudに移行する
from google.adk.sessions import VertexAiSessionService
session_service = VertexAiSessionService(
    project="your-gcp-project",
    location="us-central1",
)

# Step 3: モニタリングを有効化する（Phase 5）
from google.adk.instrumentation import instrument_adk
instrument_adk(provider)  # Cloud Trace or Langfuse

# Step 4: 評価テストをCIに組み込む
# .github/workflows/eval.yml に pytest tests/ を追加
```

---

### 6-6. 最終実装課題

**課題1（基本）**
本教材のコードを実際に動かして、`output/` ディレクトリに以下の2ファイルが生成されることを確認してください。

```
output/
├── report_2024年Q1.md
└── infographic_2024年Q1.png
```

**課題2（応用）**
インフォグラフィックのスタイルをユーザーが選べるように改造してください。

```
入力例：「2024年Q1のレポートを営業向けスタイルで作って」
→ STYLES["営業向け"] のプロンプトが使われる
```

ヒント：`user:` スコープの State にスタイル設定を保存してください。

**課題3（発展）**
Q1とQ2の2期分を自動比較して「半期レポート」を生成するパイプラインを設計・実装してください。

```
追加が必要なもの：
- compare_quarters(q1_label, q2_label, context) ツール
- half_year_report_agent サブエージェント
- Orchestratorのinstruction更新
```

**課題4（最終課題）**
以下のエンドツーエンドシナリオを自分で実装してください。

```
シナリオ：
「毎月1日に前月の売上レポートとインフォグラフィックを
 自動生成してSlackに投稿するシステム」

必要な要素：
1. スケジューラーとの統合（APScheduler など）
2. Slack投稿ツールの追加
3. エラー時の再試行ロジック（LoopAgent）
4. 品質チェック後に投稿する条件分岐
```

---

### まとめ：全フェーズ振り返り

```
Phase 0  AIエージェントの概念・ADKとは何か・Why ADK
Phase 1  LlmAgent / Runner / Session の基本構造
Phase 2  @tool デコレータ・Tool Calling・ベストプラクティス
Phase 3  sub_agents・委譲パターン（Sequential/Parallel/Loop）
Phase 4  State管理・ToolContext・DatabaseSessionService
Phase 5  AgentEvaluator・カスタム評価・Langfuse モニタリング
Phase 6  fetch → analyze → report → infographic
         の一気通貫パイプライン実装
         Nano Banana 2（gemini-3.1-flash-image-preview）による画像生成
```

#### 習得したスキルセット

| スキル | 習得フェーズ |
|--------|------------|
| エージェント設計・実装 | Phase 1-2 |
| マルチエージェント協調 | Phase 3 |
| 状態管理・永続化 | Phase 4 |
| 品質評価・本番監視 | Phase 5 |
| E2Eパイプライン構築 | Phase 6 |
| 画像生成AI統合 | Phase 6 |
