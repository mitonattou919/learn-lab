# Azure Well-Architected Framework 概要

> Azure 上でワークロードを設計・運用する人向けの WAF 入門教材。5 本柱の全体像と、評価ツールの使い方までをコンパクトに俯瞰します。
> Azure そのものが初めての方 → `azure-overview.md` を先に
> 各柱を実装レベルで深掘りしたい方 → Microsoft Learn 公式ページへ

---

## Chapter 0: Azure WAF とは

### 1. ワークロード品質のチェックリスト

**Well-Architected Framework (WAF)** は、Azure 上に構築するワークロードが「信頼できて・安全で・無駄なく・運用しやすく・速い」状態になっているかを評価するための設計指針集です。Microsoft が公式に提供しており、クラウド設計レビューの共通言語になっています。

```
要件定義 → 設計 → 実装 → 運用
              ↑
        WAF を物差しにして
        「この設計、5本柱的にどうなん？」を問う
```

### 2. 3 つのフレームワークの役割分担

Azure の設計系フレームワークは目的別に 3 つあります。混同しやすいので整理を。

| フレームワーク | 対象レイヤ | 問い |
|---------------|------------|------|
| **Cloud Adoption Framework (CAF)** | 組織・ポートフォリオ | 会社としてどうクラウドに移るか |
| **Well-Architected Framework (WAF)** | 個別ワークロード | このシステム、ちゃんと設計できてるか |
| **Azure Architecture Center (AAC)** | 実装パターン | 具体的にどう組むか（リファレンス集） |

→ CAF で方針を決め、WAF でワークロードを評価し、AAC で実装パターンを探す、という流れです。

### 3. 「健康診断」だと思って付き合う

- 一度合格すれば終わりではない（継続的に再評価）
- 5 本柱すべて 100 点は現実的でない（トレードオフがある）
- 結果よりも **改善アクションの優先順位付け** が本質

---

## Chapter 1: 5 本柱の全体像

WAF の中核は 5 本柱です。それぞれに **設計原則 (Design principles)** と **チェックリスト (Checklist)**、**推奨事項 (Recommendations)** が用意されています。

### 1. 柱の一覧

| # | 柱 | 一言で | 代表的な Azure サービス |
|---|-----|--------|------------------------|
| 1 | **Reliability** | 落ちない・戻せる | Availability Zones, Site Recovery, Backup |
| 2 | **Security** | 侵害に強い | Entra ID, Defender for Cloud, Key Vault |
| 3 | **Cost Optimization** | 無駄を削る | Cost Management, Advisor, Reservations |
| 4 | **Operational Excellence** | 運用しやすい | Monitor, Bicep, DevOps/GitHub Actions |
| 5 | **Performance Efficiency** | 需要に応える | Load Testing, Autoscale, Cache for Redis |

### 2. 各柱の共通構造

どの柱も同じ型で書かれているので、読み方を覚えると楽です。

```
Design principles  … その柱の思想（5 原則前後）
Design review checklist … 設計時に見るべき観点
Recommendation guides … 具体的な推奨事項と実装例
Tradeoffs … 他の柱と衝突するケース
Cloud design patterns … 関連する実装パターン
Azure services guide … 柱別の主要サービス解説
```

### 3. 優先順位は状況で変わる

すべての柱を同じ重さで扱うわけではありません。業務要件によって重み付けが変わります。

| 業務特性 | 特に重視される柱 |
|---------|----------------|
| 金融・医療・公共 | Reliability, Security |
| スタートアップ MVP | Cost Optimization, Operational Excellence |
| ゲーム・配信 | Performance Efficiency, Reliability |
| 社内業務システム | Operational Excellence, Security |

→ Chapter 7 のトレードオフに繋がる考え方です。

---

## Chapter 2: Reliability（信頼性）

### 1. 何を守るのか

**「ワークロードが期待どおり動き続け、障害が起きても復旧できる状態」** を維持するための柱です。ユーザーから見て SLA / SLO / RTO / RPO が守られるかが評価軸になります。

| 指標 | 意味 |
|------|------|
| **SLA** | 提供者が約束する稼働率（例: 99.95%） |
| **SLO** | 自チームが目標に置く稼働率 |
| **RTO** | 障害発生から復旧までの許容時間 |
| **RPO** | 許容できるデータ損失時間 |

### 2. 設計原則（5 つ）

1. **ビジネス要件で信頼性目標を決める**（技術先行でなく業務から逆算）
2. **冗長化で単一障害点を消す**（ゾーン冗長・リージョン冗長）
3. **自己復旧できる設計**（ヘルスチェック・自動フェイルオーバ・リトライ）
4. **障害モードを事前に洗い出す**（FMA: Failure Mode Analysis）
5. **復旧手順を検証する**（DR 訓練・Chaos Engineering）

### 3. 設計レビューの主な観点

| 観点 | 例 |
|------|-----|
| 可用性設計 | Availability Zones の活用、リージョンペア |
| データ保護 | Geo-Redundant Storage、定期バックアップと復旧テスト |
| フェイルオーバ | Traffic Manager / Front Door での自動切替 |
| 観測性 | ヘルスモデルの定義、Application Insights |
| テスト | Azure Chaos Studio による障害注入 |

### 4. よくあるアンチパターン

- **シングルゾーン構成** のまま本番投入
- **バックアップはしてるが復旧テスト未実施**（いざというとき戻せない）
- **ヘルスチェックが HTTP 200 だけ**（依存先の DB 障害を検知できない）

---

## Chapter 3: Security（セキュリティ）

### 1. 何を守るのか

データとシステムの **機密性・完全性・可用性 (CIA)** を、脅威・誤操作・内部不正から守る柱です。クラウドでは境界が曖昧になるため、**ゼロトラスト** が前提になります。

### 2. 設計原則（5 つ）

1. **ゼロトラスト**（常に検証、暗黙の信頼を置かない）
2. **多層防御 (Defense in depth)**（ID・ネットワーク・データ・アプリを各層で守る）
3. **最小権限**（必要な人に必要な権限だけ、期限付きで）
4. **前提侵害 (Assume breach)**（侵害されている前提で検知・封じ込めを設計）
5. **シフトレフト**（開発の早い段階でセキュリティを織り込む）

### 3. 設計レビューの主な観点

| 層 | チェック項目 | 代表サービス |
|----|-------------|-------------|
| **ID** | MFA / 条件付きアクセス / PIM | Microsoft Entra ID |
| **ネットワーク** | Private Endpoint / NSG / WAF | Azure Firewall, Front Door |
| **データ** | 保存/転送時暗号化、キー管理 | Key Vault, Managed HSM |
| **脅威検知** | 異常検知・SIEM・自動対応 | Defender for Cloud, Sentinel |
| **シークレット** | コード埋め込み禁止、自動ローテーション | Managed Identity, Key Vault |

### 4. よくあるアンチパターン

- **接続文字列をソースに直書き**（Managed Identity 未使用）
- **全社員が Owner ロール**（最小権限違反）
- **パブリック IP 直公開の PaaS**（Private Endpoint 未設定）
- **ログを集約していない**（インシデント時に追跡不能）

---

## Chapter 4: Cost Optimization（コスト最適化）

### 1. 何を守るのか

クラウドの強みは **「使った分だけ課金」** ですが、裏を返せば **「使わんのに借りっぱなしも課金」** です。ビジネス価値に対してコストを最適化し、無駄を継続的に削ることが目的です。

> コスト最適化は「安くすること」ではなく **「支払うお金に見合う価値が出てるか」** を問う柱です。

### 2. 設計原則（5 つ）

1. **コスト意識の文化**（全員が課金モデルを理解する）
2. **使用量モデル化**（需要予測とキャパシティ計画）
3. **適正サイジング**（過剰 SKU を避ける）
4. **従量制・割引制度の活用**（Reservations / Savings Plan / Spot）
5. **継続的最適化**（Advisor の推奨を定例でレビュー）

### 3. 設計レビューの主な観点

| 観点 | 代表サービス・機能 |
|------|-------------------|
| 可視化 | Cost Management, タグ付け, コスト配分 |
| 割引活用 | Reserved Instances, Savings Plan, Hybrid Benefit |
| スケール制御 | Autoscale, 自動停止スケジュール（Dev/Test） |
| 未使用資産 | Advisor による未使用 IP / ディスク / VM の特定 |
| アーキテクチャ | サーバレス（Functions / Container Apps）の活用 |

### 4. よくあるアンチパターン

- **Dev 環境を 24/7 起動**（夜間停止スクリプト未導入）
- **タグ運用が崩壊**（どの部署のコストか不明）
- **本番と同スペックの検証環境**（適正サイジング未検討）
- **Reservations 未購入のまま定常稼働**（30〜60%の割引機会損失）

---

## Chapter 5: Operational Excellence（オペレーショナルエクセレンス）

### 1. 何を守るのか

ワークロードを **安全に・予測可能に・素早く** デプロイし、運用し続けるための柱です。DevOps プラクティスを土台に、属人化を排して自動化で品質を担保します。

### 2. 設計原則（5 つ）

1. **DevOps 文化**（開発と運用を分断しない）
2. **IaC (Infrastructure as Code)**（手動操作を排除）
3. **CI/CD と自動化**（ビルド・テスト・デプロイをパイプライン化）
4. **可観測性**（ログ・メトリクス・トレースの統合）
5. **安全なデプロイ実践 (SDP)**（段階的リリースとロールバック）

### 3. 設計レビューの主な観点

| 観点 | 代表サービス・手法 |
|------|-------------------|
| IaC | Bicep, Terraform, ARM Template |
| CI/CD | GitHub Actions, Azure DevOps Pipelines |
| デプロイ戦略 | Blue/Green, Canary, Feature Flags |
| 監視 | Azure Monitor, Log Analytics, Application Insights |
| インシデント対応 | ランブック, オンコール体制, ポストモーテム |
| ガバナンス | Azure Policy, Landing Zone |

### 4. よくあるアンチパターン

- **Portal 手動操作で本番構成**（再現不可・ドリフト発生）
- **ログを VM 内に溜めたまま**（横断分析できない）
- **本番直デプロイ**（Canary / Staging slot 未使用）
- **ランブック未整備**（障害時に担当者の記憶頼り）

---

## Chapter 6: Performance Efficiency（パフォーマンス効率）

### 1. 何を守るのか

需要の変化に対してリソースを **効率的にスケール** させ、ユーザーが期待する応答性能を維持する柱です。「速ければ良い」ではなく **「要件を満たしつつ過不足なく」** がポイントです。

### 2. 設計原則（5 つ）

1. **パフォーマンスターゲットを定義する**（応答時間・スループット・同時接続数）
2. **容量計画を行う**（需要予測とスケール戦略）
3. **継続的にチューニングする**（負荷テストと計測が前提）
4. **ボトルネックを特定して解消する**（推測ではなく計測）
5. **データとコードを最適化する**（クエリ・キャッシュ・非同期化）

### 3. 設計レビューの主な観点

| 観点 | 代表サービス・手法 |
|------|-------------------|
| スケール | Autoscale（水平）, SKU 変更（垂直） |
| 負荷テスト | Azure Load Testing |
| キャッシュ | Azure Cache for Redis, Front Door / CDN |
| データ層 | インデックス最適化, パーティショニング, 読み取りレプリカ |
| 非同期処理 | Service Bus, Event Grid, Queue Storage |
| 観測 | Application Insights の依存関係マップ、APM |

### 4. よくあるアンチパターン

- **本番で初めて負荷をかける**（事前の負荷テスト未実施）
- **N+1 クエリ放置**（DB 往復でレイテンシ悪化）
- **キャッシュ未活用**（静的コンテンツを毎回オリジンから配信）
- **手動スケール固定**（需要変動に追従できずコストも性能も悪化）

---

## Chapter 7: トレードオフとワークロード別ガイド

### 1. 柱はぶつかる

5 本柱はすべて 100 点を取れる設計にはなりません。どこかを強化すると別のどこかが犠牲になります。WAF の推奨事項ページには各項目に **"Tradeoffs"** セクションが明記されており、**意思決定の透明化** が求められます。

### 2. 代表的なトレードオフ

| トレードオフ | 典型例 | 緩和策 |
|-------------|--------|--------|
| **Cost ↔ Reliability** | マルチリージョン冗長は料金倍増 | RTO/RPO に応じて Active-Passive か Active-Active か選択 |
| **Security ↔ Performance** | Private Endpoint / WAF / 暗号化は遅延増 | キャッシュ併用・SKU 選定で緩和 |
| **Security ↔ Operational Excellence** | JIT アクセスは運用手数増 | PIM で承認フローを自動化 |
| **Performance ↔ Cost** | プレミアム SKU / 上限高い Autoscale は過剰プロビジョニング | 負荷テスト結果で適正値を決める |
| **Reliability ↔ Operational Excellence** | 高冗長構成はデプロイ複雑化 | IaC で標準化、デプロイスタンプ |

### 3. ワークロード別ガイダンス

WAF は汎用ガイダンスに加え、特定のワークロード種別に特化したガイドを提供しています。業務特性が明確なら、まず該当ガイドから入るのが近道です。

| ガイド | 対象 | 重視される柱 |
|--------|------|------------|
| **Mission-critical** | 99.99%+ SLO の基幹業務 | Reliability, Operational Excellence |
| **SaaS** | マルチテナント SaaS | Security, Cost Optimization |
| **AI workloads** | 生成 AI・RAG・MLOps | Performance, Cost, Security |
| **SAP / Oracle** | エンタープライズ DB 移行 | Reliability, Performance |
| **Azure Virtual Desktop** | VDI | Performance, Cost |
| **Sustainability** | カーボン効率 | Cost, Operational Excellence |

→ 各ガイドには **Design principles / Design areas / Assessment / リファレンスアーキテクチャ** がセットで用意されています。

---

## Chapter 8: Well-Architected Review と次の一歩

### 1. Well-Architected Review とは

Microsoft Assessments 上で提供される **無料のセルフアセスメントツール** です。ワークロード単位で 5 本柱に沿った設問に答えると、スコアと優先度付き推奨事項が得られます。

```
ワークロード選択 → 設問に回答 → スコアと推奨事項 → Advisor 連携
     ↑                                                  ↓
     └──────── 改善を入れて再評価でトラッキング ────────┘
```

### 2. 使いどころ

| タイミング | 目的 |
|-----------|------|
| **設計時** | 要件定義で 5 本柱チェックリストを照らし、抜け漏れを発見 |
| **リリース前** | 本番投入前のセルフレビューでスコアをベースライン化 |
| **定期レビュー** | 四半期ごとに再評価して改善の進捗を可視化 |
| **インシデント後** | ポストモーテムで該当柱の設計原則を振り返る |

### 3. 継続的改善サイクル

WAF は一度の評価で終わるものではありません。**Design → Review → Operate** のループを回し続けるのが本来の使い方です。

- **Design**: 設計原則とチェックリストを要件定義に組み込む
- **Review**: Well-Architected Review で点数化、推奨事項を優先度で並べる
- **Operate**: Azure Advisor / Defender for Cloud / Cost Management で日次監視、アラート対応

### 4. 次の一歩

- 具体的なアーキテクチャパターンを探す → **Azure Architecture Center** (`learn.microsoft.com/azure/architecture/`)
- 組織レベルのクラウド戦略を立てる → **Cloud Adoption Framework** (`learn.microsoft.com/azure/cloud-adoption-framework/`)
- Landing Zone で WAF 準拠の基盤を構築する → **Azure Landing Zones**
- 特定ワークロードを深掘りする → Mission-critical / SaaS / AI ガイド

### 5. 実務に落とすコツ

- **5 本柱すべて満点を狙わない**: トレードオフを明示化し、業務要件に沿って優先順位をつける
- **ツールの数字を盲信しない**: スコアは現状把握の参考値。最終判断は人間が持つ
- **ポストモーテムで柱に紐付ける**: 「今回の障害は Reliability の FMA 不足が原因」と整理すると再発防止が具体化する

---

## 関連リソース

- **WAF トップ**: https://learn.microsoft.com/azure/well-architected/
- **5 本柱の概要**: https://learn.microsoft.com/azure/well-architected/pillars
- **Well-Architected Review**: https://learn.microsoft.com/assessments/azure-architecture-review/
- **Tradeoffs ガイド**: https://learn.microsoft.com/azure/well-architected/cross-cutting-guides/tradeoffs
- **Mission-critical workloads**: https://learn.microsoft.com/azure/well-architected/mission-critical/
- **AI workloads**: https://learn.microsoft.com/azure/well-architected/ai/
- **Azure Advisor**: https://learn.microsoft.com/azure/advisor/advisor-overview
- **Cloud Adoption Framework**: https://learn.microsoft.com/azure/cloud-adoption-framework/
- **Azure Architecture Center**: https://learn.microsoft.com/azure/architecture/

### 関連教材

- `azure-overview.md` — Azure そのものの入門（アカウント作成・主要サービス俯瞰）
