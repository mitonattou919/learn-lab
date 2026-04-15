# learn-lab

技術テーマごとのステップバイステップ学習教材集。Chapter 0 で概念と Why から入り、ハンズオンと練習課題を通じて未経験から実務活用までを段階的にカバーします。

## docs/ 一覧

### Claude Code

| ファイル | 対象 | 内容 |
|---------|------|------|
| [claude-code-tutorial.md](docs/claude-code-tutorial.md) | 初心者 | インストールから基本操作まで、5分で成功体験を得られるハンズオン教材 |
| [claude-code-best-practices.md](docs/claude-code-best-practices.md) | 中級者 | 公式ベストプラクティス準拠。Skills/Subagents/Hooks/MCP、並列実行、失敗パターン集 |

### その他のテーマ

| ファイル | 対象 | 内容 |
|---------|------|------|
| [pester.md](docs/pester.md) | PowerShell経験者 | テスト未経験者向け。Pester による PowerShell テスト入門 |
| [google-adk-lesson.md](docs/google-adk-lesson.md) | AI開発者 | Google ADK でのAIエージェント開発、基礎から実務活用まで |
| [azure-overview.md](docs/azure-overview.md) | クラウド初心者 | Azure の全体像をアカウント作成から主要サービス、コスト管理まで俯瞰 |
| [azure-waf-overview.md](docs/azure-waf-overview.md) | Azure設計者 | Well-Architected Framework の 5 本柱・トレードオフ・評価ツールをコンパクトに俯瞰 |
| [git-github-overview.md](docs/git-github-overview.md) | Git/GitHub入門者 | 内部構造・ブランチ戦略・コミット作法・GitHub 主要機能を全体俯瞰 |

## 教材のスタイル

全教材で共通する構造：

- **Chapter 0**: 概念と Why（従来手法との比較）
- **中盤**: 基本操作 → 良い使い方の原則 → 段階的な進め方
- **終盤**: 失敗パターン、運用、次の一歩
- 各章に比較表・Before/After・ハンズオン・練習課題を配置

## 新しい教材を作る

`.claude/skills/generate-learning-material/` に、上記のスタイルで新テーマの教材を対話的に生成するスキルを用意しています。

```
/generate-learning-material <テーマ> [tutorial|best-practices]
```

ワークフロー: ヒアリング → 一次情報リサーチ（並列サブエージェント）→ 章立て承認 → 1章ずつ執筆 → 仕上げ。
