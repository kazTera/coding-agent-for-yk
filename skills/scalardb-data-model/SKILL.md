---
name: scalardb-data-model
description: ScalarDBの制約を考慮したデータモデル設計を行います。Partition Key・Clustering Key・Secondary Index の設計、ホットスポット評価、メタデータオーバーヘッド見積もり、バックエンドDB選定を含みます。ワークフローStep 04（データモデル設計）で使用します。
---

# ScalarDB Data Model Design Skill

## 目的

ScalarDB の制約と最適化を考慮したデータモデルを設計し、以下を生成する:

- テーブルスキーマ定義（ScalarDB形式）
- PK/CK/SI 設計と根拠
- ホットスポットリスク評価
- メタデータオーバーヘッド見積もり
- バックエンドDB選定結果

## リファレンス資料

| 資料 | パス | 参照内容 |
|------|------|---------|
| 論理データモデル | `research/03_logical_data_model.md` | 7つのデータモデルパターン |
| 物理データモデル | `research/04_physical_data_model.md` | PK/CK/SI設計、パフォーマンス |
| DB調査 | `research/05_database_investigation.md` | 対応DB一覧、特性比較 |
| ScalarDB 3.17 | `research/13_scalardb_317_deep_dive.md` | メタデータ分離機能 |
| ワークフロー04 | `workflow/04_data_model_design.md` | Step 04 手順 |
| データモデルテンプレート | `workflow/templates/data_model_template.md` | テーブル設計テンプレート |

---

## 実行フロー

### Stage 1: 入力の確認

1. Step 01（要件分析）と Step 03（適用範囲）の成果物を読み込む
2. ScalarDB管理対象テーブル一覧を確認
3. ドメインモデル（Step 02）から集約構造を確認

### Stage 2: テーブルスキーマ設計

各テーブルについて以下を定義:

**テーブル定義テンプレート**:

```markdown
### テーブル: {namespace}.{table_name}

#### 基本情報
| 項目 | 値 |
|------|-----|
| 名前空間 | {namespace} |
| テーブル名 | {table_name} |
| 所有サービス | {service} |
| ScalarDB管理 | Yes/No |
| バックエンドDB | {db_type} |

#### カラム定義
| カラム名 | ScalarDB型 | キー種別 | 説明 |
|---------|-----------|---------|------|
| {col} | TEXT/INT/BIGINT/FLOAT/DOUBLE/BOOLEAN/BLOB | PK/CK(ASC/DESC)/SI/- | {desc} |

#### キー設計

**Partition Key**: `{column}`
- 選定理由: {reason}
- カーディナリティ: High/Mid/Low
- ホットスポットリスク: High/Mid/Low

**Clustering Key**: `{column} {ASC/DESC}`
- 選定理由: {reason}
- アクセスパターン: {pattern}

**Secondary Index**: `{column}`（必要な場合のみ）
- 選定理由: {reason}
- パフォーマンス影響: {impact}
```

### Stage 3: ホットスポット評価

各テーブルの Partition Key について:

| テーブル | PK | カーディナリティ | アクセス偏り | リスク | 対策 |
|---------|-----|---------------|------------|--------|------|
| | | High/Mid/Low | 均等/偏あり | High/Mid/Low | {対策} |

**対策パターン**:
- ハッシュプレフィックス付与
- 複合キーの導入
- テーブル分割
- 時系列データのバケット化

### Stage 4: メタデータオーバーヘッド見積もり

| テーブル | 予想レコード数 | レコードサイズ | メタデータ | オーバーヘッド率 |
|---------|-------------|-------------|-----------|--------------|
| | | {bytes} | ~200 bytes | {percentage}% |

**Transaction Metadata Decoupling 適用判断**:
- レコードサイズが小さい（< 1KB）場合はオーバーヘッド率が高くなるため検討
- ただし READ レイテンシへの影響を評価すること

### Stage 5: バックエンドDB選定

| テーブル群 | 推奨DB | 選定理由 | 代替案 |
|-----------|--------|---------|--------|
| | | | |

**選定基準**:
- ScalarDB対応状況（`research/05_database_investigation.md` 参照）
- データアクセスパターン（OLTP/OLAP）
- スケーラビリティ要件
- 運用コスト
- チームの習熟度

### Stage 6: 出力

成果物を `output/phase2/04_data_model_design.md` に出力。

---

## ScalarDB 対応データ型

| ScalarDB型 | 説明 |
|-----------|------|
| BOOLEAN | 真偽値 |
| INT | 32ビット整数 |
| BIGINT | 64ビット整数 |
| FLOAT | 32ビット浮動小数点 |
| DOUBLE | 64ビット浮動小数点 |
| TEXT | 文字列 |
| BLOB | バイナリ |

## 使用例

```
/plan-step 04

Phase 1 の結果に基づいてデータモデルを設計してください。
対象サービス: 注文サービス、在庫サービス、決済サービス
バックエンドDB候補: MySQL 8.0, Amazon DynamoDB
```
