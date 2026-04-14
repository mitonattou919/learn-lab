# Azure 概要チュートリアル

> Azure を初めて触る人向けのハンズオン教材。8章を順に進めれば、アカウント作成から主要サービスの俯瞰、コスト管理までの基礎が身につきます。
> もっと深く知りたくなったら → Well-Architected Framework / Cloud Adoption Framework へ

---

## Chapter 0: Azure とは

### クラウドサービスとしての Azure

Azure は Microsoft が提供するクラウドプラットフォームです。ひとことで言うと **「サーバーを買わずに、必要な機能を借りて使う」仕組み**。

```
従来:          サーバー購入 → データセンター設置 → OS/MW インストール → 運用
Azure:         Portal や CLI で「VM ください」→ 数分で使える → 使った分だけ課金
```

### 他クラウド（AWS / GCP）との違い

| | AWS | GCP | Azure |
|--|-----|-----|-------|
| 強み | サービス数と成熟度 | データ分析と ML | Microsoft 製品連携（AD / Office / Windows Server） |
| 主要認証基盤 | IAM | Cloud Identity | Microsoft Entra ID（旧 Azure AD） |
| 企業導入 | スタートアップ〜大企業 | データ企業多め | 大企業・官公庁に厚い |

Azure のキーワードは **「既存の Microsoft 資産と繋がりやすい」**。Windows Server の社内環境をそのまま延長する感覚で使えるのが他との大きな違いです。

### ARM: 統一管理レイヤ

Azure のすべての操作は **Azure Resource Manager (ARM)** を通ります。Portal の GUI 操作も、Azure CLI の `az` コマンドも、PowerShell の `Az` モジュールも、裏では同じ ARM API を呼んでいます。

```
    ┌──────────┐  ┌──────────┐  ┌──────────────┐
    │  Portal  │  │   CLI    │  │  PowerShell  │
    └────┬─────┘  └────┬─────┘  └──────┬───────┘
         └─────────────┴───────────────┘
                       ↓
              Azure Resource Manager (ARM)
                       ↓
                 各リソースプロバイダ
```

→ 「どのツールで操作しても結果は同じ」と思ってよい。

### 「優秀な外注サーバールーム」だと思って付き合う

- 頼めば即用意してくれる
- 使わんでも借りとる間は課金される
- 片付け（削除）は自分の責任

---

## Chapter 1: アカウント作成と初回ハロー体験

### 1. 無料アカウント作成

1. [azure.microsoft.com/free](https://azure.microsoft.com/free) にアクセス
2. Microsoft アカウント（なければ作成）でサインイン
3. **クレジットカード登録が必須**（本人確認用。少額オーソリが走る）
4. 電話番号 SMS 認証

無料枠の内訳（2026年時点の目安、詳細は公式要確認）：

| 種別 | 内容 |
|------|------|
| 初回クレジット | $200（30日間有効） |
| 12ヶ月無料 | B1S VM 750h/月、Blob Storage 5GB 他 |
| 常時無料 | Functions 100万回/月、Cosmos DB 1000 RU/s + 25GB 他 |

> **詰まりポイント**
> - デビット/プリペイドカードは弾かれることが多い
> - 日本の電話番号が通らない場合は別番号を試す
> - 30日後は明示的に「従量課金へアップグレード」しないとリソースが止まる

### 2. Azure CLI のインストール

最短の入口は **Azure CLI**。コマンド1本でほぼ何でもできます。

**macOS**
```bash
brew update && brew install azure-cli
```

**Linux (Ubuntu/Debian)**
```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

**Windows**
```powershell
winget install -e --id Microsoft.AzureCLI
```

確認：

```bash
az version
```

### 3. 初回ログイン

```bash
az login
```

ブラウザが開いて認証。環境によっては MFA（Microsoft Authenticator）が要求される。SSH 越しなど GUI が無い環境では：

```bash
az login --use-device-code
```

### 4. 5分で成功体験: リソースグループを作って消す

```bash
# 作る
az group create --name rg-hello --location japaneast

# 一覧確認
az group list -o table

# 消す（課金停止）
az group delete --name rg-hello --yes
```

これが Azure の基本動作。**「作る・一覧で確認する・消す」**。どんなサービスでも同じリズムです。

### 5. サブスクリプション切り替え

複数の契約を持つようになったら：

```bash
az account list -o table
az account set --subscription "<名前かID>"
az account show
```

---

## Chapter 2: 中核概念 — 4階層スコープ

Azure を触って最初に混乱するのが **リソースの置き場所**。以下の4階層を押さえるだけで 8 割理解できます。

```
Management Group      ← 組織の最上位（ポリシー適用単位）
    └─ Subscription   ← 課金とリソース所有の境界
        └─ Resource Group  ← 同じライフサイクルのリソース束
            └─ Resource    ← VM、ストレージ、DB 等の実体
```

### 各階層の役割

| 階層 | 役割 | 具体例 |
|------|------|--------|
| Tenant | Microsoft Entra ID の組織単位 | `contoso.onmicrosoft.com` |
| Management Group | 複数サブスクを束ねてポリシー適用 | `本番系`・`開発系` |
| Subscription | 課金・使用量の単位 | `本番-01`・`検証-01` |
| Resource Group | まとめて作成/削除する単位 | `rg-web-prod`・`rg-db-dev` |
| Resource | 実際のサービス実体 | VM `vm-web01`、Storage `st12345` |

**上位の設定（RBAC、ポリシー、タグ）は下位に継承される**。これが階層化の最大の価値。

### リージョンと Availability Zone

- **Region**: データセンターの地理的拠点（例: `japaneast`=東日本、`westus2`=米国西部）
- **Availability Zone (AZ)**: リージョン内の物理的に分離されたデータセンター群。**AZ をまたいで配置すれば障害耐性が上がる**

初心者の選び方：

- 日本ユーザー向け → `japaneast`（東日本）が第一候補
- バックアップ先に `japanwest` も選べるが全サービス対応ではない点に注意

### 練習課題

```bash
# RG を japaneast に作って、中に Storage Account を作る
az group create -n rg-learn -l japaneast
az storage account create -g rg-learn -n learnstore$RANDOM -l japaneast --sku Standard_LRS
az resource list -g rg-learn -o table

# 後片付け（中身ごと消える）
az group delete -n rg-learn --yes
```

**RG ごと消せば中身も全部消える**。これが Azure の片付け鉄則です。

---

## Chapter 3: 主要サービス俯瞰

Azure には 200 以上のサービスがありますが、実務でよく触るのは次の 6 カテゴリ。最初はこれだけ知っていればOK。

### カテゴリ別代表プロダクト

| カテゴリ | 代表サービス | 用途 |
|---------|------------|------|
| **Compute** | Virtual Machines / App Service / Functions / Container Apps (ACA) / AKS | アプリ実行環境 |
| **Networking** | Virtual Network / Load Balancer / Firewall / ExpressRoute | ネット接続・保護 |
| **Storage** | Blob Storage / Azure Files / Data Lake | ファイル/オブジェクト保管 |
| **Database** | Azure SQL Database / Cosmos DB / PostgreSQL / Redis | データ永続化 |
| **AI + ML** | Microsoft Foundry / Azure OpenAI / Azure AI Search / Machine Learning | AI 機能・検索・エージェント開発 |
| **Analytics** | Synapse Analytics / Microsoft Fabric / Data Factory | データ分析基盤 |

### 選び方の目安（Compute 編）

迷うのは大体 Compute です。次の順で選ぶとハズレが少ない：

```
1. とりあえずアプリを動かしたい（Web/API）
   → App Service（PaaS、OS管理不要）

2. イベント駆動で短時間だけ動かしたい
   → Functions（サーバーレス、実行時間で課金）

3. Docker コンテナで動かしたい
   → Container Apps（手軽）または AKS（Kubernetes 本格運用）

4. OS レベルで自由に触りたい / レガシーをそのまま移す
   → Virtual Machines（IaaS）
```

> **原則**: **「抽象度の高い PaaS から検討し、合わなければ下に降りる」**。いきなり VM に手を出すと OS 管理コストが重くのしかかります。

### 練習課題

Portal にログインして、次のサービスがどのカテゴリに属するか調べてみてください：

- Azure Kubernetes Service
- Azure Cache for Redis
- Azure Cognitive Search
- Azure Data Factory

---

## Chapter 4: Portal / Cloud Shell / Azure CLI / Azure PowerShell の使い分け

Azure には主要な操作インターフェースが4つあります。**場面ごとの向き不向き**を知っておくと生産性が段違い。

### 4つの入口

| ツール | 動作環境 | 向いている場面 |
|--------|----------|---------------|
| **Portal** | ブラウザ GUI | 初期探索、料金確認、IAM 設定、全体俯瞰 |
| **Cloud Shell** | ブラウザ内シェル | 出先でちょっとコマンド叩く、環境構築不要 |
| **Azure CLI (`az`)** | ローカル / Linux 文化 | 自動化スクリプト、CI/CD、クロスプラットフォーム |
| **Azure PowerShell (`Az`)** | ローカル / Windows 文化 | Windows 管理、M365/Entra 連携、既存 PS 資産 |

### Portal

- 何がどこにあるか分からん時の強い味方
- コストビュー、監視ダッシュボード、IAM 設定は Portal が一番速い
- 一方で **操作の再現性が無い**。手順書化は困難

### Cloud Shell

Portal 上部の `>_` アイコンから起動。ブラウザだけで Bash/PowerShell が動く。

- `az` / `Az` / `kubectl` / `terraform` 等プリインストール済
- ローカル環境構築不要で即検証できる
- 初回起動時にストレージアカウント作成を要求される（少額課金あり）

### Azure CLI (`az`)

Linux / macOS 文化の人はこちら。JSON / テーブル出力、パイプ連携が得意。

```bash
az vm list --query "[].{Name:name, RG:resourceGroup, Size:hardwareProfile.vmSize}" -o table
az vm start -g rg-web -n vm-web01
```

### Azure PowerShell (`Az` モジュール)

Windows 管理、Microsoft 365 / Entra ID / Exchange Online との統合作業が多い現場ではこちらが主力。

**インストール（PowerShell 7 推奨）**
```powershell
Install-Module -Name Az -Repository PSGallery -Scope CurrentUser -Force
Connect-AzAccount
```

**基本操作**
```powershell
Get-AzSubscription
Set-AzContext -Subscription "<名前かID>"

New-AzResourceGroup -Name rg-hello -Location japaneast
Get-AzResourceGroup | Format-Table
Remove-AzResourceGroup -Name rg-hello -Force
```

**PowerShell を選ぶ場面**
- Windows Server / Active Directory 管理と地続きで書きたい
- Exchange Online や Microsoft Graph SDK と組み合わせる
- オブジェクトパイプライン（`|`）で複雑なフィルタ・集計をする
- 社内の既存 PS スクリプト資産が大量にある

```powershell
# 例: 停止状態の VM だけ抽出して一括起動
Get-AzVM -Status | Where-Object { $_.PowerState -eq "VM deallocated" } | Start-AzVM
```

この「オブジェクトを次のコマンドに流せる」のが PowerShell の真骨頂。文字列処理が中心の `az` とは思想が違います。

### 使い分けの目安

| やりたいこと | 最適解 |
|------------|-------|
| 今月の請求額を見る | Portal |
| 障害対応で 1 コマンド叩く | Cloud Shell |
| Terraform/Bicep と併用する CI | Azure CLI |
| Windows/Entra/M365 と連携する運用バッチ | Azure PowerShell |
| IaC で宣言的に管理 | Bicep / Terraform（これは次の一歩） |

---

## Chapter 5: コスト管理の基本

Azure で一番多い失敗は **「消し忘れて青ざめる」**。コスト機能は最初の週に必ず触っておきましょう。

### 1. Cost Analysis

Portal の「コスト管理 + 請求」→「コスト分析」で今月の支出を可視化。

- サブスク / RG / タグ / サービス種別で切り口を変えられる
- **日次で開く習慣** をつけるだけで大事故を防げる
- 課金データは 4 時間ごとに更新

### 2. Budget アラート

金額ベースで閾値を設定し、超過前にメール通知。

```bash
# 月 $50 を超えたら通知するバジェット例
az consumption budget create \
  --budget-name monthly-50 \
  --amount 50 \
  --category cost \
  --time-grain Monthly \
  --start-date 2026-04-01 \
  --end-date 2027-04-01
```

Portal からのほうが GUI で楽。Action Group と組み合わせれば **自動で VM を停止** するアクションも組める。

### 3. 無料枠の落とし穴

- 「12ヶ月無料」「30日 $200 クレジット」「常時無料」は **別物**
- サービス毎に上限数量・期間が異なる
- 依存する別サービスは無料枠対象外のことが多い（例: VM 本体は無料でもディスク/Public IP は課金）
- クレジット切れ後は **自動で従量課金に移行**

### 4. 片付け作法

```bash
# RG を消せば中身全部消える。これが一番確実
az group delete --name rg-learn --yes --no-wait
```

学習用のものは **RG を分ける**。検証が終わったら RG ごと消す。これを守るだけで課金事故はほぼゼロになります。

---

## Chapter 6: 初心者が踏む 5 つの罠

### 1. VM を「停止」しただけで課金が続く

OS 上の `shutdown` や Portal の「停止」では **Compute 課金が止まらない** ことがある。

→ **対策**: Portal 上で「停止（割り当て解除）」を選ぶ。CLI なら `az vm deallocate`。

```bash
az vm deallocate -g rg-web -n vm-web01
```

### 2. 孤児リソース（Orphan）が残る

VM を削除してもマネージドディスクや Public IP は残って課金継続。

→ **対策**: RG 単位で削除する。Cost Analysis で「ディスク」「Public IP」種別で残存を定期チェック。

### 3. オーバーサイズな SKU

オンプレ感覚で `D8s_v5` みたいな大きい VM を選んで月数万円溶ける。

→ **対策**: [Pricing Calculator](https://azure.microsoft.com/pricing/calculator/) で事前見積。Azure Advisor の「リサイズ提案」を毎月確認。

### 4. タグ未設定で費用が按分できない

複数プロジェクトで共有 Subscription を使うと、誰のコストか分からなくなる。

→ **対策**: 最初から `env=prod`、`owner=team-a` のような **タグ運用ルール**を決める。Azure Policy でタグ必須を強制できる。

### 5. リージョン選択ミス

気軽に `eastus` を選んで遅延で苦しむ、または法令的に国内必須なのに米国に置いてしまう。

→ **対策**: 日本ユーザー向けなら第一候補は `japaneast`。データ所在地の制約を先に確認。

---

## Chapter 7: 次の一歩

ここまでで Azure の基礎体力は身につきました。次はこの3方向のどれかへ進むのが王道。

### 1. Well-Architected Framework (WAF)

5本柱で設計品質を評価するフレームワーク。

| 柱 | 概要 |
|----|------|
| Reliability | 冗長化とレジリエンシで稼働率確保 |
| Security | 機密性・整合性・攻撃耐性 |
| Cost Optimization | 予算内で価値最大化 |
| Operational Excellence | 観測と自動化で障害低減 |
| Performance Efficiency | スケールで需要変化に追随 |

→ [learn.microsoft.com/azure/well-architected/](https://learn.microsoft.com/azure/well-architected/)

### 2. Cloud Adoption Framework (CAF)

組織として Azure を導入・運用するための方法論。Strategy → Plan → Ready → Adopt → Govern/Secure/Manage。企業導入の旗振り役になる人は必読。

→ [learn.microsoft.com/azure/cloud-adoption-framework/](https://learn.microsoft.com/azure/cloud-adoption-framework/)

### 3. IaC（Bicep / Terraform）

Portal ポチポチ卒業への道。**リソースをコードで宣言的に管理**することで、再現性・レビュー・バージョン管理が手に入る。

- **Bicep**: Azure 専用の DSL。ARM テンプレートより読みやすい。Microsoft 公式推し
- **Terraform**: マルチクラウド対応。社内で他クラウドも使うならこちら

### 直感を育てる

この教材のパターンは出発点です。使い続けると：

- いつ Portal で済ませ、いつ CLI を書くか
- どこまで無料枠で粘れるか
- どの粒度で RG を切るか

が体感的に分かってきます。**うまくいった時も詰まった時も「なぜそうなったか」を振り返る** のが上達の近道です。

---

## 関連リソース

- Microsoft Learn (Azure): https://learn.microsoft.com/azure/
- Azure Architecture Center: https://learn.microsoft.com/azure/architecture/
- Well-Architected Framework: https://learn.microsoft.com/azure/well-architected/
- Cloud Adoption Framework: https://learn.microsoft.com/azure/cloud-adoption-framework/
- Pricing Calculator: https://azure.microsoft.com/pricing/calculator/
- Azure CLI リファレンス: https://learn.microsoft.com/cli/azure/
- Azure PowerShell リファレンス: https://learn.microsoft.com/powershell/azure/
