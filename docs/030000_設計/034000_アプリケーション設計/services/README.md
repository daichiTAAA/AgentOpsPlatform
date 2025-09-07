---
title: サービス設計
version: "0.1"
status: draft
owner: システム開発チーム
updated: 2025-09-07
related_requirements: ["021000_要件定義書", "034020_ユースケース設計"]
---

# サービス設計

本ディレクトリには、ソロプレナー／AIネイティブ企業基盤システムの各ユースケースに対応するサービス設計ドキュメントが格納されています。

## サービス一覧

| サービス名                                                | 対応UC | 概要                 | 主要責務                             |
| --------------------------------------------------------- | ------ | -------------------- | ------------------------------------ |
| [StrategicExecutionService](./StrategicExecutionService/) | UC-01  | 戦略実行管理         | ビジョン分析・戦略計画生成・実行管理 |
| [MarketResearchService](./MarketResearchService/)         | UC-02  | 市場調査・分析       | 市場データ分析・調査レポート生成     |
| [IssueResolutionService](./IssueResolutionService/)       | UC-03  | 課題解決サポート     | バグ分析・修正案生成・自動PR作成     |
| [AuditGovernanceService](./AuditGovernanceService/)       | UC-04  | 監査・ガバナンス     | AI行動監査・コンプライアンス管理     |
| [KnowledgeSearchService](./KnowledgeSearchService/)       | UC-05  | ナレッジ検索・活用   | リアルタイム知識検索・コード支援     |
| [QualityManagementService](./QualityManagementService/)   | UC-06  | 品質管理・レビュー   | コード品質チェック・自動レビュー     |
| [ProjectManagementService](./ProjectManagementService/)   | UC-07  | プロジェクト進捗管理 | 進捗監視・リスク検出・対策提案       |
| [LearningService](./LearningService/)                     | UC-08  | 教育・学習支援       | 技術調査・学習計画・個別最適化       |
| [IntegrationService](./IntegrationService/)               | UC-09  | 外部連携・統合       | API統合・連携設計・統合テスト        |
| [SecurityAuditService](./SecurityAuditService/)           | UC-10  | セキュリティ監査     | 脆弱性スキャン・リスク評価・対策提案 |

## テンプレート使用方法

新規モジュールは `_template` をコピーして作成:

```bash
cp -R _template <ServiceName>
```

命名例: `StrategicExecutionService`, `MarketResearchService` など。各ファイルの `title`/`related_requirements` を更新する。

## 設計原則

### 人機協調フロー
すべてのサービスは以下の共通フローに従います：

1. **人間**: 依頼・指示（Redmine）
2. **AI**: 分析・実行（Databricks） 
3. **人機協調**: レビュー・判断（VS Code）
4. **AI**: 実行・記録（全システム）

### システム構成
- **Redmine**: 中枢管理（タスク・承認フロー・履歴）
- **Databricks**: AI実行環境（分析・データ処理・機械学習）
- **VS Code**: 人機協調ワークスペース（共同編集・レビュー）

## 関連ドキュメント

- [034020_ユースケース設計.md](../034020_ユースケース設計.md): 各UCの概要
- [036020_openapi.yaml](../../036000_API設計/036020_openapi.yaml): 詳細API仕様
- [021000_要件定義書.md](../../020000_要件定義/021000_要件定義書.md): システム要件

