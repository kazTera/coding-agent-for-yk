---
name: api-design
description: RESTful API の設計を行います。エンドポイント定義、リクエスト/レスポンス仕様、エラーハンドリング、ScalarDBトランザクション例外のマッピングを含みます。
---

# API Design Skill

## 目的

システムのRESTful APIを設計し、以下を定義します：

1. **エンドポイント設計**: URL設計、HTTPメソッド、パス構造
2. **リクエスト/レスポンス仕様**: DTO定義、バリデーション
3. **エラーハンドリング**: エラーコード、エラーレスポンス形式、ScalarDBトランザクション例外のマッピング
4. **OpenAPI仕様書**: Swagger/OpenAPI 3.0形式
5. **CDCメタデータのフィルタリング**: tx_stateなどの内部カラムの除外
6. **サービス間通信設計**: ScalarDB 2PCを利用したマイクロサービス間通信パターン

## 参照ドキュメント

| ドキュメント | パス | 説明 |
|-------------|------|------|
| API設計手順 | `workflow/06_api_interface_design.md` | API設計の詳細ステップ |
| データアクセスパターン | `research/08_transparent_data_access.md` | ScalarDBの透過的データアクセス設計 |
| マイクロサービスアーキテクチャ | `research/01_microservice_architecture.md` | MSAパターンとサービス分割 |

## 入力パラメータ

| パラメータ | 必須 | 説明 | デフォルト |
|-----------|------|------|-----------|
| projectName | Yes | プロジェクト名 | - |
| apiVersion | No | APIバージョン | v1 |
| baseUrl | No | ベースURL | /api/v1 |
| authType | No | 認証方式 | Bearer |

## API設計原則

### RESTful設計原則

| 原則 | 説明 | 例 |
|------|------|-----|
| リソース指向 | 名詞でリソースを表現 | /users, /audit-sets |
| HTTPメソッド | 操作をメソッドで表現 | GET=取得, POST=作成 |
| ステートレス | セッション状態を持たない | トークン認証 |
| 統一インターフェース | 一貫したURL設計 | /resources/{id} |

### HTTPメソッドマッピング

| メソッド | 操作 | 成功コード | べき等性 |
|---------|------|-----------|---------|
| GET | 取得 | 200 | Yes |
| POST | 作成 | 201 | No |
| PUT | 全体更新 | 200 | Yes |
| PATCH | 部分更新 | 200 | No |
| DELETE | 削除 | 204 | Yes |

## ScalarDB トランザクション例外のAPIマッピング

ScalarDBのトランザクション例外をHTTPステータスコードに適切にマッピングし、クライアントに対して適切なリトライ戦略を提供します。

### リトライ可能な競合エラー (409 Conflict)

| ScalarDB例外 | HTTPステータス | エラーコード | リトライ戦略 |
|-------------|---------------|-------------|------------|
| `CrudConflictException` | 409 Conflict | `SCALARDB_001` | Exponential backoff でリトライ推奨 |
| `CommitConflictException` | 409 Conflict | `SCALARDB_002` | Exponential backoff でリトライ推奨 |

**レスポンス例**:
```json
{
  "error": {
    "code": "SCALARDB_001",
    "message": "データ競合が発生しました。リトライしてください",
    "details": {
      "exception": "CrudConflictException",
      "retryable": true,
      "retryStrategy": "exponential_backoff",
      "recommendedWaitMs": 1000
    },
    "timestamp": "2026-02-17T10:00:00Z",
    "traceId": "abc-123-def"
  }
}
```

### 不明なトランザクション状態 (500 + リトライガイダンス)

| ScalarDB例外 | HTTPステータス | エラーコード | リトライ戦略 |
|-------------|---------------|-------------|------------|
| `UnknownTransactionStatusException` | 500 Internal Server Error | `SCALARDB_003` | べき等性を確認してからリトライ |

**レスポンス例**:
```json
{
  "error": {
    "code": "SCALARDB_003",
    "message": "トランザクションの状態が不明です",
    "details": {
      "exception": "UnknownTransactionStatusException",
      "retryable": true,
      "retryStrategy": "idempotent_retry",
      "guidance": "操作がべき等であることを確認してからリトライしてください",
      "checkEndpoint": "/api/v1/transactions/{transactionId}/status"
    },
    "timestamp": "2026-02-17T10:00:00Z",
    "traceId": "abc-123-def"
  }
}
```

### リトライ不可能なエラー

| ScalarDB例外 | HTTPステータス | エラーコード | 説明 |
|-------------|---------------|-------------|------|
| `CommitException` | 500 Internal Server Error | `SCALARDB_004` | トランザクションのコミット失敗 |
| `UnsatisfiedConditionException` | 422 Unprocessable Entity | `SCALARDB_005` | 条件付き更新の条件不一致 |

**UnsatisfiedConditionException のレスポンス例**:
```json
{
  "error": {
    "code": "SCALARDB_005",
    "message": "更新条件が満たされていません",
    "details": {
      "exception": "UnsatisfiedConditionException",
      "retryable": false,
      "reason": "Expected version does not match current version",
      "expectedCondition": {
        "field": "version",
        "expectedValue": 5,
        "actualValue": 6
      }
    },
    "timestamp": "2026-02-17T10:00:00Z",
    "traceId": "abc-123-def"
  }
}
```

### エラーハンドリングのベストプラクティス

1. **クライアント側のリトライ実装**
   - 409 Conflictの場合: Exponential backoffでリトライ
   - 500 + UnknownTransactionStatus: べき等性確認後にリトライ
   - 422 UnsatisfiedCondition: リトライせずエラー処理

2. **サーバー側の実装**
   - トランザクションIDをログに記録
   - 分散トレーシングでトランザクション追跡
   - メトリクスで競合率を監視

3. **モニタリング**
   - 409エラーの頻度を監視し、ホットスポットを検出
   - 500エラー（UnknownTransactionStatus）の発生を監視

## CDCメタデータのフィルタリング

ScalarDB のChange Data Capture (CDC)機能で使用される内部メタデータカラムをAPIレスポンスから除外します。

### フィルタリング対象カラム

| カラム名 | 説明 | フィルタリング理由 |
|---------|------|-------------------|
| `tx_state` | トランザクション状態 | 内部管理用、クライアントに不要 |
| `tx_id` | トランザクションID | 内部管理用、クライアントに不要 |
| `tx_prepared_at` | トランザクション準備時刻 | 内部管理用、クライアントに不要 |
| `tx_committed_at` | トランザクションコミット時刻 | 内部管理用、クライアントに不要 |
| `tx_version` | トランザクションバージョン | 内部管理用、クライアントに不要 |
| `before_tx_id` | CDC用の前トランザクションID | CDC内部用 |
| `before_state` | CDC用の前状態 | CDC内部用 |
| `before_version` | CDC用の前バージョン | CDC内部用 |
| `before_prepared_at` | CDC用の前準備時刻 | CDC内部用 |
| `before_committed_at` | CDC用の前コミット時刻 | CDC内部用 |

### フィルタリング実装パターン

**1. DTOレイヤーでのフィルタリング**

```java
@JsonIgnoreProperties(value = {
    "tx_state", "tx_id", "tx_prepared_at", "tx_committed_at", "tx_version",
    "before_tx_id", "before_state", "before_version",
    "before_prepared_at", "before_committed_at"
})
public class UserResponse {
    private String id;
    private String email;
    private String name;
    // ビジネスフィールドのみ
}
```

**2. マッパーでの明示的フィルタリング**

```java
public class UserMapper {
    public UserResponse toResponse(Result result) {
        return UserResponse.builder()
            .id(result.getValue("id").get().getAsString())
            .email(result.getValue("email").get().getAsString())
            .name(result.getValue("name").get().getAsString())
            // CDCメタデータは意図的に除外
            .build();
    }
}
```

**3. 共通フィルタークラス**

```java
public class CdcMetadataFilter {
    private static final Set<String> CDC_METADATA_COLUMNS = Set.of(
        "tx_state", "tx_id", "tx_prepared_at", "tx_committed_at", "tx_version",
        "before_tx_id", "before_state", "before_version",
        "before_prepared_at", "before_committed_at"
    );

    public static Map<String, Object> filterCdcMetadata(Map<String, Object> data) {
        return data.entrySet().stream()
            .filter(entry -> !CDC_METADATA_COLUMNS.contains(entry.getKey()))
            .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
    }
}
```

### APIレスポンス例

**フィルタリング前（NG）**:
```json
{
  "id": "user-001",
  "email": "user@example.com",
  "name": "山田太郎",
  "tx_state": "COMMITTED",
  "tx_id": "tx-12345",
  "tx_version": 3,
  "before_tx_id": "tx-12344"
}
```

**フィルタリング後（OK）**:
```json
{
  "id": "user-001",
  "email": "user@example.com",
  "name": "山田太郎",
  "createdAt": "2026-02-17T10:00:00Z",
  "updatedAt": "2026-02-17T12:00:00Z"
}
```

## サービス間通信設計

ScalarDB 2PC（Two-Phase Commit）を利用したマイクロサービス間の分散トランザクション設計。

### サービス間通信パターン

#### 1. Orchestration パターン（推奨）

サービス間の分散トランザクションをオーケストレーターが調整。

```
[Order Service] (Orchestrator)
    |
    ├─> [Inventory Service] (Participant)
    ├─> [Payment Service] (Participant)
    └─> [Shipping Service] (Participant)
```

**実装例**:

```java
// Order Service (Orchestrator)
public class OrderOrchestrator {
    @POST
    @Path("/orders")
    public Response createOrder(OrderRequest request) {
        TwoPhaseCommitTransaction tx = manager.start();
        try {
            // 1. 在庫予約（Participant 1）
            inventoryClient.reserve(tx.getId(), request.getItems());

            // 2. 支払い処理（Participant 2）
            paymentClient.process(tx.getId(), request.getPayment());

            // 3. 配送手配（Participant 3）
            shippingClient.arrange(tx.getId(), request.getAddress());

            // 4. 注文作成（Coordinator）
            Order order = createOrderRecord(tx, request);

            tx.prepare();
            tx.commit();

            return Response.status(201).entity(order).build();
        } catch (CrudConflictException | CommitConflictException e) {
            tx.rollback();
            return Response.status(409).entity(new ConflictError(e)).build();
        } catch (Exception e) {
            tx.rollback();
            return Response.status(500).entity(new ServerError(e)).build();
        }
    }
}
```

#### 2. Saga パターン（代替案）

各サービスがローカルトランザクションを実行し、補償トランザクションで整合性を保つ。

```
Order Created → Reserve Inventory → Process Payment → Arrange Shipping
     ↓              ↓                    ↓                  ↓
  Rollback ← Cancel Reservation ← Refund ← Cancel Shipping
```

### API設計ガイドライン

#### トランザクションIDの伝播

**ヘッダーでの伝播**:

```http
POST /api/v1/inventory/reserve
Authorization: Bearer {token}
X-Transaction-Id: tx-12345-67890
X-Request-Id: req-abc-def
Content-Type: application/json

{
  "items": [
    {"productId": "prod-001", "quantity": 2}
  ]
}
```

#### べき等性の保証

分散トランザクションでのリトライに備えてべき等性を実装。

```java
@POST
@Path("/inventory/reserve")
@Idempotent
public Response reserveInventory(
    @HeaderParam("X-Transaction-Id") String txId,
    ReservationRequest request) {

    // 既に処理済みかチェック
    if (reservationRepository.exists(txId, request.getProductId())) {
        return Response.status(200).build(); // べき等性
    }

    // 予約処理
    reservation = reservationService.reserve(txId, request);
    return Response.status(201).entity(reservation).build();
}
```

#### タイムアウト設計

| 操作タイプ | タイムアウト | 理由 |
|----------|------------|------|
| Prepare | 30秒 | 各サービスの準備完了待ち |
| Commit | 60秒 | 全サービスのコミット完了待ち |
| Rollback | 30秒 | ロールバックは速やかに完了すべき |
| サービス間HTTP | 10秒 | ネットワーク遅延を考慮 |

#### エラーハンドリング

**Participant側のエラーレスポンス**:

```json
{
  "error": {
    "code": "INVENTORY_INSUFFICIENT",
    "message": "在庫が不足しています",
    "details": {
      "productId": "prod-001",
      "requested": 10,
      "available": 5,
      "transactionId": "tx-12345"
    },
    "retryable": false
  }
}
```

**Orchestrator側のエラーハンドリング**:

```java
try {
    inventoryClient.reserve(txId, items);
} catch (InsufficientInventoryException e) {
    // ビジネスエラー: リトライせずロールバック
    tx.rollback();
    return Response.status(422).entity(toErrorResponse(e)).build();
} catch (ServiceUnavailableException e) {
    // 一時的エラー: リトライ可能
    tx.rollback();
    return Response.status(503).entity(toRetryableError(e)).build();
}
```

### サービス間通信のベストプラクティス

1. **Circuit Breaker の実装**
   - サービス障害時の連鎖防止
   - タイムアウトとリトライの制御

2. **分散トレーシング**
   - トランザクションIDとリクエストIDの伝播
   - OpenTelemetry/Zipkinでの追跡

3. **非同期処理の活用**
   - 長時間処理はイベント駆動で分離
   - ScalarDB 2PCは同期的な短時間処理に限定

4. **部分的な失敗への対処**
   - 適切なロールバック処理
   - 補償トランザクションの実装

## 実行フロー

### Stage 1: 既存API分析

```
1.1 コントローラーの調査
    - エンドポイント一覧
    - リクエスト/レスポンス型
    - 認証・認可設定

1.2 問題点の特定
    - REST原則違反
    - 命名規則の不一致
    - バージョニングの欠如
    - ScalarDB例外の不適切なマッピング
    - CDCメタデータの漏洩
```

### Stage 2: リソース設計

```
2.1 リソースの特定
    - ドメインモデルからの抽出
    - 集約ルートの特定
    - サブリソースの定義

2.2 URL設計
    - 階層構造の設計
    - クエリパラメータの設計
    - フィルタリング・ソート
```

### Stage 3: エンドポイント設計

```
3.1 CRUD操作
    - 一覧取得 (GET /resources)
    - 詳細取得 (GET /resources/{id})
    - 作成 (POST /resources)
    - 更新 (PUT/PATCH /resources/{id})
    - 削除 (DELETE /resources/{id})

3.2 カスタムアクション
    - アクション系 (POST /resources/{id}/actions)
    - バッチ操作 (POST /resources/batch)

3.3 サービス間通信エンドポイント
    - 2PC参加エンドポイント
    - トランザクション状態確認
```

### Stage 4: リクエスト/レスポンス設計

```
4.1 リクエスト設計
    - パスパラメータ
    - クエリパラメータ
    - リクエストボディ
    - ヘッダー（X-Transaction-Id等）

4.2 レスポンス設計
    - 成功レスポンス
    - エラーレスポンス（ScalarDB例外含む）
    - ページネーション
    - CDCメタデータのフィルタリング
```

### Stage 5: エラーハンドリング設計

```
5.1 HTTPステータスコード
    - 2xx: 成功
    - 4xx: クライアントエラー
    - 5xx: サーバーエラー

5.2 ScalarDB例外のマッピング
    - CrudConflictException → 409
    - CommitConflictException → 409
    - UnknownTransactionStatusException → 500
    - CommitException → 500
    - UnsatisfiedConditionException → 422

5.3 エラーレスポンス形式
    - エラーコード体系
    - リトライ戦略の提示
    - 詳細情報
```

### Stage 6: OpenAPI仕様書生成

## 出力

```
output/phase2/06_api_interface_design.md
```

### 出力フォーマット

```markdown
# API設計書

## 概要

| 項目 | 値 |
|------|-----|
| APIバージョン | v1 |
| ベースURL | /api/v1 |
| 認証方式 | Bearer Token (JWT) |
| コンテンツタイプ | application/json |

## 設計原則

### URL設計規則

- リソース名は複数形（kebab-case）
- IDはUUID形式
- ネストは2階層まで
- クエリパラメータはcamelCase

### 認証・認可

- すべてのエンドポイントで認証必須（一部除く）
- JWT Bearer Token をAuthorizationヘッダーで送信
- ロールベースアクセス制御（RBAC）

### ScalarDB固有の設計

- トランザクション例外の適切なHTTPマッピング
- CDCメタデータのフィルタリング
- 分散トランザクションのためのトランザクションID伝播

## リソース一覧

| # | リソース | ベースパス | 説明 |
|---|---------|-----------|------|
| 1 | Users | /api/v1/users | ユーザー管理 |
| 2 | AuditSets | /api/v1/audit-sets | 監査セット |
| 3 | EventLogs | /api/v1/event-logs | イベントログ |

## 共通仕様

### リクエストヘッダー

| ヘッダー | 必須 | 説明 |
|---------|------|------|
| Authorization | Yes | Bearer {token} |
| Content-Type | Yes | application/json |
| Accept-Language | No | レスポンス言語 |
| X-Request-ID | No | リクエスト追跡ID |
| X-Transaction-Id | No* | 分散トランザクションID（2PC使用時必須） |

### ページネーション

```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 100,
    "totalPages": 5
  }
}
```

### クエリパラメータ

| パラメータ | 説明 | 例 |
|-----------|------|-----|
| page | ページ番号 | ?page=1 |
| pageSize | 1ページの件数 | ?pageSize=20 |
| sort | ソート項目 | ?sort=createdAt:desc |
| filter | フィルタ条件 | ?filter[status]=active |

## ScalarDB トランザクション例外のエラーコード

| エラーコード | ScalarDB例外 | HTTPステータス | リトライ |
|------------|-------------|---------------|---------|
| SCALARDB_001 | CrudConflictException | 409 | Yes (Exponential backoff) |
| SCALARDB_002 | CommitConflictException | 409 | Yes (Exponential backoff) |
| SCALARDB_003 | UnknownTransactionStatusException | 500 | Yes (べき等確認後) |
| SCALARDB_004 | CommitException | 500 | No |
| SCALARDB_005 | UnsatisfiedConditionException | 422 | No |

## CDCメタデータのフィルタリング

以下のカラムはAPIレスポンスから除外されます：

- `tx_state`, `tx_id`, `tx_prepared_at`, `tx_committed_at`, `tx_version`
- `before_tx_id`, `before_state`, `before_version`, `before_prepared_at`, `before_committed_at`

## サービス間通信設計

### 分散トランザクションエンドポイント

#### POST /api/v1/{resource}/prepare
トランザクションの準備フェーズ

#### POST /api/v1/{resource}/commit
トランザクションのコミット

#### POST /api/v1/{resource}/rollback
トランザクションのロールバック

### トランザクションIDの伝播

すべてのサービス間通信で`X-Transaction-Id`ヘッダーを使用してトランザクションIDを伝播します。

## エンドポイント詳細

### GET /api/v1/users

**説明**: ユーザー一覧を取得

**認可**: ADMIN, MANAGER

**クエリパラメータ**:

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| page | integer | No | ページ番号 (default: 1) |
| pageSize | integer | No | 件数 (default: 20, max: 100) |
| status | string | No | ステータスフィルタ |

**レスポンス (200)**:

```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "email": "user@example.com",
      "name": "山田太郎",
      "status": "active",
      "createdAt": "2026-01-15T09:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 45,
    "totalPages": 3
  }
}
```

注意: `tx_state`などのCDCメタデータは除外されています。

## エラーレスポンス例

### 409 Conflict（リトライ可能）

```json
{
  "error": {
    "code": "SCALARDB_001",
    "message": "データ競合が発生しました。リトライしてください",
    "details": {
      "exception": "CrudConflictException",
      "retryable": true,
      "retryStrategy": "exponential_backoff",
      "recommendedWaitMs": 1000
    },
    "timestamp": "2026-02-17T10:00:00Z",
    "traceId": "abc-123-def"
  }
}
```

### 422 Unprocessable Entity（リトライ不可）

```json
{
  "error": {
    "code": "SCALARDB_005",
    "message": "更新条件が満たされていません",
    "details": {
      "exception": "UnsatisfiedConditionException",
      "retryable": false,
      "expectedCondition": {
        "field": "version",
        "expectedValue": 5,
        "actualValue": 6
      }
    },
    "timestamp": "2026-02-17T10:00:00Z",
    "traceId": "abc-123-def"
  }
}
```

### 500 Internal Server Error（べき等確認後リトライ）

```json
{
  "error": {
    "code": "SCALARDB_003",
    "message": "トランザクションの状態が不明です",
    "details": {
      "exception": "UnknownTransactionStatusException",
      "retryable": true,
      "retryStrategy": "idempotent_retry",
      "guidance": "操作がべき等であることを確認してからリトライしてください",
      "checkEndpoint": "/api/v1/transactions/{transactionId}/status"
    },
    "timestamp": "2026-02-17T10:00:00Z",
    "traceId": "abc-123-def"
  }
}
```
```

## API設計チェックリスト

### URL設計
- [ ] リソース名が複数形になっている
- [ ] URLにアクション動詞が含まれていない
- [ ] 一貫したケース規則（kebab-case）
- [ ] 適切な階層構造

### HTTPメソッド
- [ ] 適切なメソッドが使用されている
- [ ] べき等性が考慮されている
- [ ] 適切なステータスコードを返す

### リクエスト/レスポンス
- [ ] 一貫したJSON構造
- [ ] 適切なバリデーション
- [ ] ページネーションの実装
- [ ] 適切なエラーハンドリング
- [ ] CDCメタデータのフィルタリング

### ScalarDB固有
- [ ] トランザクション例外の適切なマッピング
- [ ] リトライ戦略の明示
- [ ] トランザクションIDの伝播（2PC使用時）
- [ ] べき等性の実装（分散トランザクション）

### セキュリティ
- [ ] 認証が必要なエンドポイントの保護
- [ ] 認可の実装
- [ ] 入力値のサニタイズ

## 使用例

```
Skill: api-design

プロジェクト名: scalar-auditor-for-box
APIバージョン: v1
ベースURL: /api/v1
認証方式: Bearer (JWT)
```

## 注意事項

- 既存のAPIとの後方互換性を考慮する
- バージョニング戦略を事前に決定する
- APIドキュメントは常に最新に保つ
- セキュリティレビューを実施する
- ScalarDBのトランザクション例外を適切にHTTPステータスコードにマッピングする
- CDCメタデータは必ずAPIレスポンスから除外する
- 分散トランザクションではトランザクションIDを確実に伝播する
- べき等性を実装してリトライに対応する
