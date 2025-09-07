# VS Code拡張機能 設計書

---

## 文書情報

- 文書名: VS Code拡張機能 設計書（ソロプレナー／AIネイティブ企業基盤）
- 版数: 0.1（ドラフト）
- 作成日: 2025-09-07
- 更新日: 2025-09-07
- 作成者: システム開発チーム
- 参照: docs/01000_企画/01100_企画書.md（v1.1, 2025-09-07）, docs/02000_要件定義/02100_要件定義書.md（v0.10, 2025-09-07）

---

## 1. 目的とスコープ

本拡張は、作業文脈（エディタ）と管理・知識基盤（Redmine/Databricks）を統合し、AI/人の協働を加速する。要件定義の以下に主として対応する。

- FR-012/013/014（ドラフト・差分・PR、タスク連携、品質チェック）
- FR-018/019/020（API/Webhook連携、通知、メタデータ自動付与）
- FR-021〜032, 024〜031（Copilot Agent連携とLM Tools、レート制御/キャッシュ/マスキング/確認ダイアログ）
- NFR-006/010/011/021/022/023（権限、可観測性、保守性、性能・同時接続、PII配慮）

対象外: Databricksへの書込み操作（FR-026に基づき検索専用）。

---

## 2. 全体アーキテクチャ

- VS Code拡張（TypeScript）
  - 機能: タスク連携、ドラフト生成/差分、品質チェック、Copilot Agent用ツール群（Databricks検索）
  - 認証: OIDC/SSO（デバイスコード/PKCE）、トークンはVS Code Secret Storageに保管
  - 監査: すべての操作に`X-Correlation-Id`を付与しRedmine/Gatewayへ伝播（FR-027, NFR-010）
- Gateway API（社内サービス、詳細は03200参照）
  - 役割: Databricks APIへのプロキシ（1:1転送、認証付与のみ、レスポンス非改変）[FR-039〜041]
- Redmine
  - Webhook/REST連携でタスク/監査/承認と往復（FR-001〜004, 015, 018, 020）
- Databricks
  - 検索・類似・クエリ・Genie・Serving（FR-033〜038）

データフロー（要旨）:
1) 拡張がユーザー操作/Agent要求を受領 → 2) 確認ダイアログ表示（FR-028）→ 3) Gateway APIを通じDatabricks検索（FR-021, 026, 039）→ 4) 結果をエディタ/パネルへ表示、PIIマスキング（FR-032）→ 5) タスク/成果物メタ情報をRedmineへ送信（FR-020）。

---

## 2.1 アクティベーション/パッケージ構成

- 拡張ID: `com.agentops.vscode-ext`
- `activationEvents`
  - `onStartupFinished`, `onCommand:agentops.*`, `workspaceContains:**/*`
- 主要依存: `vscode`, `axios`（Gateway呼出）, `zod`（設定/レスポンス検証）
- フォルダ構成（案）
  - `src/commands/*` コマンド実装
  - `src/tools/*` Copilotツール実装
  - `src/services/*` Gateway/Redmine/Telemetry
  - `src/ui/*` Webview/パネル
  - `src/policies/*` マスキング/レート/キャッシュ

---

## 3. 機能設計

### 3.1 コマンド/貢献ポイント

- `agentops.openTask` Redmineチケットを開く/紐付け（FR-001/004/013）
- `agentops.createDraft` ドラフト作成（ドキュメント/コード）（FR-012/020）
- `agentops.showDiff` 差分/PR下書き（FR-012/014）
- `agentops.runChecks` リント/テスト/文書チェックを実行（タスクに結果連携）（FR-014/019）
- `agentops.searchKnowledge` Databricks検索（Copilotツール/手動）（FR-021/024/025/026）
- `agentops.linkArtifacts` 成果物にタスクID/主体/出典を付与（FR-020）
- `agentops.settings` 拡張設定UI（許可ツール/レート/キャッシュ/マスキング）（FR-029〜032）

### 3.2 Copilot Agent ツール

- ツール名: `kb.search`, `kb.similar`, `kb.sqlQuery`, `kb.genie`, `kb.serve`
- 入力: `query`, `mode`（semantic/sql/vector/genie/serve）, `top_k`, `filters` ほか
- 出力: タイトル/要約/URL/抜粋/スコア（FR-022）、引用情報付き回答生成をCopilotへ誘導（FR-023）
- 実行前確認: 重要操作は確認ダイアログを強制（FR-028）。
- 自動/明示: Agentモードで自動選択、手動で明示選択も可（FR-024/025）。

### 3.3 タスク連携とメタデータ

- ファイル保存/PR作成時に以下を自動付与（FR-020）:
  - `Task-Id`, `Actor`（User/Agent）, `Timestamp`, `Sources[]`（URL/クエリ）
- Redmine APIでチケットコメントへ実行ログ/添付を送信（FR-003/004/018）。

### 3.4 品質チェックパイプライン

- ローカル/CI互換のジョブ実行（ESLint/Markdown Lint/ユニットテスト等）
- 結果を拡張のパネル表示＋Redmineに送信（FR-014/019）

---

## 3.5 設定スキーマ（JSON）

```json
{
  "agentops.gateway.baseUrl": {"type": "string"},
  "agentops.allowedTools": {"type": "array", "items": {"type": "string"}},
  "agentops.rateLimit.rpm": {"type": "number", "default": 30},
  "agentops.cache.ttlSec": {"type": "number", "default": 300},
  "agentops.masking.enabled": {"type": "boolean", "default": true},
  "agentops.masking.patterns": {"type": "array", "items": {"type": "string"}},
  "agentops.telemetry.enabled": {"type": "boolean", "default": true},
  "agentops.readonly": {"type": "boolean", "default": true}
}
```

---

## 4. UI/UX設計

- アクティビティバー「AgentOps」ビュー
  - ツリービュー: 現在のタスク、関連資料、最近の検索
- パネル（Webview）
  - 検索結果、引用挿入、監査ログ、チェック結果
- ステータスバー
  - 接続状態、レート制御状況、相関ID表示
- 確認ダイアログ
  - ツール呼出時/外部転送時に説明/出典/データ範囲を明示（FR-028）

---

## 4.1 UXフロー（例）

- 知識検索（手動）
  1. `agentops.searchKnowledge` 実行
  2. 入力パネルでクエリ/モード/フィルタを設定
  3. 重要操作は確認ダイアログで許諾（FR-028）
  4. 結果一覧→エディタへ引用挿入（出典つき, FR-022/023）

- PR作成
  1. `agentops.showDiff` 実行→差分確認
  2. `agentops.linkArtifacts` でメタ付与（FR-020）
  3. `agentops.runChecks` → 結果OKならPR作成→Redmineにリンク投稿（FR-014/019）

---

## 5. 設定/ポリシー

- `agentops.gateway.baseUrl` Gateway APIベースURL（必須）
- `agentops.allowedTools[]` 許可ツール（管理者が配布/ロック可能）（FR-029）
- `agentops.rateLimit` リクエスト/分（FR-030）
- `agentops.cache.ttlSec` クエリキャッシュTTL（FR-031）
- `agentops.masking.enabled` 画面表示前マスキング（FR-032, NFR-023）
- `agentops.readonly` Databricksは常時読み取りのみ（固定true, FR-026）

---

## 6. API連携設計（概要）

### 6.1 Gateway API呼出

- 共通ヘッダ: `Authorization: Bearer <access_token>`, `X-Correlation-Id`, `X-Client: vscode-ext`
- エンドポイント例（詳細は03200参照）:
  - `POST /kb/search`（内部で Statement Execution/Vector Search を呼出）
  - `POST /kb/sql-query`（SQL Queries API 実行）
  - `POST /kb/genie`（Genie API）
  - `POST /kb/serve`（Serving Endpoints 推論）
  - 全てレスポンス透過返却（FR-041）

#### 6.1.1 リクエスト/レスポンス例

```http
POST /kb/search HTTP/1.1
Authorization: Bearer <token>
X-Correlation-Id: <uuid-v4>
Content-Type: application/json

{"query":"RAG 設計","mode":"vector","top_k":8,"filters":{"project":"DOCS"}}
```

```json
{
  "results": [
    {"title":"RAG設計ガイド","url":"https://...","snippet":"...","score":0.82,
     "doc_id":"doc_123","source":"databricks"}
  ],
  "latency_ms": 820
}
```

### 6.2 Redmine REST/Webhook

- `POST /issues/:id/notes` 実行ログ/出典/相関IDの登録
- `POST /uploads` 添付ファイル（チェック結果、差分パッチ）
- Webhook受領: チケット更新通知→エディタ内ビューを更新

---

## 7. セキュリティ/権限

- 認証: OIDC（SSO）でログイン、短期アクセストークン＋リフレッシュ（NFR-007）
- 認可: RBAC/ABACに基づく機能露出（NFR-006）。Agentは権限制御下で動作。
- データ: VS Code側に機密データを保存しない、PII表示前マスキング（NFR-023）。
- 通信: TLS1.2+必須（NFR-016）。

---

## 8. 可観測性/監査

- 相関IDを全リクエストに付与（NFR-010）
- 監査イベント: `command_executed`, `tool_invoked`, `query_sent`, `result_shown`, `artifact_linked`
- ログ出力: 拡張→Gateway→Redmineに集約（FR-003/027/015）

相関ID形式: UUID v4（例: `c55a2dfe-0a9c-4b1c-9e8e-0a6b9f7d2a41`）。

---

## 9. 性能/スケーラビリティ

- SLO: LM Tools経由 p95≤5s、同時接続100ユーザー（NFR-021, 177再掲）
- レート制御/キャッシュにより冪等・負荷低減（FR-030/031）

### 9.1 レート制御/キャッシュ設計

- アルゴリズム: Token Bucket（初期バースト=10, 充填= `rpm/60`/秒）
- キャッシュ: キー=`hash(mode, query, filters)`, TTL=`cache.ttlSec`（ヒット時に`X-Cache: HIT`）

---

## 10. エラーハンドリング

- 入力検証エラー: 400系（再入力促し）
- 認証失効: 再ログイン誘導
- Gateway/Databricks障害: リトライ＋フォールバック表示、相関ID提示（NFR-020）

エラーコード対応（例）
- 400 入力不正 → ダイアログで再入力
- 401/403 認証/権限 → SSO再ログイン/管理者連絡誘導
- 429 レート超過 → 待機UI＋バックオフ
- 5xx Gateway/Databricks → リトライ（指数バックオフ最大3回）

---

## 11. 受け入れ基準（設計反映）

- AB-04/07/08/09/10/13 を満たすUI/ツール/ポリシーを実装
- VS Codeからのすべての検索は読み取り専用（FR-026）

---

## 12. 未決事項/リスク

- 具体的なOIDCプロバイダ/スコープ
- 許可ツール配布の組織管理（ポリシー同期）
- Genie/Serving Endpointsの利用上限/コスト監視

---

## 13. トレーサビリティ（抜粋）

- FR-012/013/014/018/019/020/021〜032/024〜031/026 → 本設計 §3, §5, §6, §7
- NFR-006/007/010/016/020/021/022/023 → 本設計 §7, §8, §9, §10

---

## 付録A: テレメトリ/監査イベント

- `command_executed` {`command`, `actor`, `task_id`, `correlation_id`}
- `tool_invoked` {`tool`, `mode`, `params_hash`, `allowed`, `confirmed`}
- `query_sent` {`mode`, `latency_ms`, `result_size`, `cache_hit`}
- `result_shown` {`pii_masked`, `top_k`}
- `artifact_linked` {`files[]`, `task_id`}

## 付録B: テスト計画（抜粋）

- ユニット: 設定検証、レート/キャッシュ、マスキング
- 統合: GatewayモックでAPI成功/失敗、相関ID伝播
- E2E: UC-01〜04をシナリオテスト、AB-04/07/08/09/10/13 満たすこと

## 付録C: リリース計画

- MVP: 検索/引用/手動リンク
- Beta: Copilotツール自動選択、品質チェック、Webhook連携
- GA: ポリシー配布・監査ダッシュボード・i18n仕上げ
