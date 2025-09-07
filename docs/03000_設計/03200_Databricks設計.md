# Databricks 設計書

---

## 文書情報

- 文書名: Databricks 設計書（ソロプレナー／AIネイティブ企業基盤）
- 版数: 0.1（ドラフト）
- 作成日: 2025-09-07
- 更新日: 2025-09-07
- 作成者: システム開発チーム
- 参照: 01100_企画書.md（v1.1）, 02100_要件定義書.md（v0.10）

---

## 1. 目的/スコープ

知識・分析基盤として既存資産の索引化/検索/分析を提供し、VS Code拡張およびAIエージェントのRAG/意思決定を支える。要件対応は主に FR-009〜011, FR-033〜041, NFR-006/007/010/011/012/013/014/015/016/017/019/020/021/023 にまたがる。

対象外: VS Code拡張からの書込み操作（FR-026参照）。

---

## 2. 論理アーキテクチャ

- データレイヤ
  - Bronze/Silver/GoldのDeltaテーブル（文書/コード/ノートブック/メタデータ）
  - Vector Search インデックス（埋め込み空間）
- アクセス/実行レイヤ
  - SQL Statement Execution API（ハイブリッドモード）[FR-033]
  - SQL Queries API（保存済みクエリ）[FR-037]
  - Genie API（会話BI）[FR-038]
  - Vector Search Indexes/Endpoints API（類似検索/運用）[FR-036]
  - Real-Time Serving Endpoints（モデル推論）[FR-035]
- 統合レイヤ
  - Gateway API（社内）: Databricks APIへの1:1プロキシ（FR-039〜041）
  - 監査/メトリクス: 相関IDで集中集約（NFR-010）

---

## 3. データモデル

### 3.1 メタデータスキーマ（抜粋）

- `documents`（Delta）
  - `doc_id`(PK), `path`, `title`, `type`(md, ipynb, code), `project`, `tags[]`, `updated_at`, `source_url`
  - `task_id`, `actor`, `origin`（git/redmine/manual）, `pii_level`（none/internal/secret）
- `chunks`（Delta）
  - `chunk_id`, `doc_id`, `seq`, `text`, `embedding[]`, `hash`
- `queries_log`（Delta）
  - `ts`, `user_or_agent`, `query`, `mode`, `correlation_id`, `result_size`, `pii_masked`

### 3.2 Vector Search

- インデックス: `vx_documents_v1`
- フィールド: `embedding`, `doc_id`, `title`, `path`, `tags`, `pii_level`
- フィルタリング: `project`, `pii_level`, `tags`

---

## 4. インジェスト/索引化パイプライン

1) 収集: リポジトリ（docs/, src/）、Redmineエクスポート、ノートブック
2) 前処理: 正規化/分割（セクション/コードブロック単位）、言語検出
3) 埋め込み: 指定モデル（社内ポリシー準拠）でベクトル化
4) 書込: `documents`/`chunks`へDelta書込、Vector Searchへアップサート
5) 品質: 重複除去、PII検出と`pii_level`付与（NFR-023, 015）

スケジュール: 1日数回のインクリメンタル。トリガ: Git Push/Redmine更新のWebhook。

---

## 5. API利用設計

### 5.1 SQL Statement Execution（FR-033）

- パラメータ: `wait_timeout=10s`, `on_wait_timeout=CONTINUE`（ハイブリッド）
- 大量結果: `disposition=EXTERNAL_LINKS`、25MiB超はSAS URLから取得（FR-034）
- セキュリティ: SAS取得時はAuthorizationヘッダ送信しない（FR-034）

### 5.2 Vector Search（FR-036）

- 類似検索: クエリ埋め込み→Index検索→`doc_id`で結合→抜粋を生成
- パラメータ: `top_k`, `score_threshold`, `filters`

### 5.3 SQL Queries（FR-037）/ Genie（FR-038）/ Serving（FR-035）

- 保存済みクエリ実行、会話BI開始、リアルタイム推論をGateway経由で呼出。

### 5.4 Gateway API（FR-039〜041）

- 方針: リクエスト転送＋認証付与のみ。応答は改変しない（FR-041）。
- ルーティング: `/kb/search`, `/kb/sql-query`, `/kb/genie`, `/kb/serve` → 各Databricks APIへ1:1
- 監査: `X-Correlation-Id`必須、マスキングフラグを付与（FR-027/032）

---

## 6. セキュリティ/権限/コンプライアンス

- 認証: SSO/OIDC（NFR-007）
- 認可: RBAC/ABACでプロジェクト/データクラスに基づき制御（NFR-006/013）
- データ管理: 分類（機密/内部/公開）、保全期間1〜3年（NFR-014）
- PII: 表示前マスキング、ログはハッシュ化保存（NFR-023, FR-032）
- 境界防御: WAF/IP制限、TLS1.2+（NFR-017/016）

---

## 7. 可観測性/監査/運用

- ログ/メトリクス/トレースを集中管理（NFR-010/019）。相関IDでエンドツーエンド追跡。
- 監査レポート: 期間/案件別にPDF/CSV出力（FR-017）。
- 自己回復: 失敗ジョブのリトライ/再実行（NFR-020）。

---

## 8. 性能/容量計画

- SLO: LM Tools経由 p95≤5s、同時接続100（NFR-021, 177再掲）
- チューニング: クエリキャッシュ（FR-031）、インデックス最適化、分割戦略
- コスト: 実行プロファイルの可視化と上限通知

---

## 9. エラーハンドリング

- タイムアウト継続時: `state=RUNNING`のジョブはポーリングで結果取得（FR-033）
- EXTERNAL_LINKS取得失敗: SAS再発行 or サイズ閾値下げ
- 権限不足: エラーログとRedmineタスク起票（FR-001/015）

---

## 10. 受け入れ基準（設計反映）

- AB-03/07/11/12/13/14/15 を満たす設定/API利用/ログ運用

---

## 11. 未決事項/リスク

- 埋め込みモデル選定/更新方針
- Genie/Servingの組織内公開範囲
- 保全期間とストレージコストのバランス

---

## 12. トレーサビリティ（抜粋）

- FR-009/010/011/017/020/033〜041 → §3, §4, §5, §7
- NFR-006/007/010/012/013/014/015/016/017/019/020/021/023 → §6〜§8
