---
name: mysql-design
description: MySQL データベース設計のリファレンス。スキーマ設計、インデックス戦略、クエリ最適化のベストプラクティスを提供します。
---

# MySQL Design Skill

## 目的

MySQLを使用したデータベース設計のリファレンスを提供します。

## スキーマ設計の原則

### 正規化

| 正規形 | 説明 | 適用場面 |
|--------|------|---------|
| 第1正規形 | 原子値のみ、繰り返しグループなし | 基本 |
| 第2正規形 | 部分関数従属の排除 | 一般的なアプリケーション |
| 第3正規形 | 推移的関数従属の排除 | 更新が多いシステム |

### 非正規化の判断

| 状況 | 非正規化を検討 | 理由 |
|------|--------------|------|
| 読み取り重視 | Yes | JOIN削減によるパフォーマンス向上 |
| 更新重視 | No | データ整合性維持 |
| レポーティング | Yes | 集計済みデータの保持 |

## データ型選択

| 用途 | 推奨型 | 備考 |
|------|--------|------|
| 主キー | BIGINT UNSIGNED | AUTO_INCREMENT |
| UUID | CHAR(36) または BINARY(16) | BINARY(16)の方が効率的 |
| 日時 | DATETIME | タイムゾーン考慮時はTIMESTAMP |
| 金額 | DECIMAL(19,4) | 浮動小数点は使用しない |
| フラグ | TINYINT(1) | BOOLEAN |
| 短いテキスト | VARCHAR(255) | 長さに応じて調整 |
| 長いテキスト | TEXT | インデックス不可に注意 |
| JSON | JSON | MySQL 5.7+ |

## インデックス戦略

### インデックス種類

| 種類 | 用途 | 作成例 |
|------|------|--------|
| PRIMARY KEY | 一意識別 | `PRIMARY KEY (id)` |
| UNIQUE | 重複防止 | `UNIQUE KEY (email)` |
| INDEX | 検索高速化 | `INDEX idx_status (status)` |
| FULLTEXT | 全文検索 | `FULLTEXT (description)` |
| 複合インデックス | 複数カラム検索 | `INDEX idx_user_date (user_id, created_at)` |

### インデックス設計のベストプラクティス

1. **カーディナリティの高いカラムを優先**
2. **WHERE句で頻繁に使用するカラム**
3. **JOIN条件のカラム**
4. **ORDER BY、GROUP BYのカラム**
5. **複合インデックスは左端から使用される**

## テーブル設計テンプレート

```sql
CREATE TABLE `table_name` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    -- ビジネスカラム
    PRIMARY KEY (`id`),
    -- インデックス
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## トランザクション

### 分離レベル

| レベル | 説明 | 用途 |
|--------|------|------|
| READ UNCOMMITTED | ダーティリード可能 | 滅多に使用しない |
| READ COMMITTED | コミット済みのみ読取 | 一般的な用途 |
| REPEATABLE READ | MySQL デフォルト | 推奨 |
| SERIALIZABLE | 完全直列化 | 厳密な整合性が必要な場合 |

### ロック戦略

| ロック | 説明 | 用途 |
|--------|------|------|
| 楽観的ロック | version カラムで競合検出 | 競合が少ない場合 |
| 悲観的ロック | SELECT FOR UPDATE | 競合が多い場合 |

## パフォーマンス最適化

### クエリ最適化

1. **EXPLAIN で実行計画を確認**
2. **必要なカラムのみ SELECT**
3. **LIMIT を適切に使用**
4. **サブクエリより JOIN を優先**
5. **インデックスが使用されているか確認**

### 設定の最適化

| 設定 | 推奨値 | 説明 |
|------|--------|------|
| innodb_buffer_pool_size | 物理メモリの50-80% | バッファプールサイズ |
| max_connections | 適切な値 | 接続数上限 |
| slow_query_log | ON | スロークエリ監視 |

## チェックリスト

- [ ] 適切なデータ型を選択したか
- [ ] 主キーを定義したか
- [ ] 外部キー制約を設定したか（必要な場合）
- [ ] インデックスを適切に設計したか
- [ ] 正規化レベルを検討したか
- [ ] 文字コードを utf8mb4 に設定したか
