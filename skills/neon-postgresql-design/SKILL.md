---
name: neon-postgresql-design
description: Neon (PostgreSQL) データベース設計のリファレンス。スキーマ設計、インデックス戦略、クエリ最適化、Neon固有機能のベストプラクティスを提供します。
---

# Neon PostgreSQL Design Skill

## 目的

Neon（PostgreSQL）を使用したデータベース設計のリファレンスを提供します。

## Neon 固有機能

| 機能 | 説明 | 活用場面 |
|------|------|---------|
| ブランチ | 本番DBのゼロコピーブランチを作成 | 開発・ステージング環境 |
| サーバーレスドライバ | HTTP経由の接続（@neondatabase/serverless） | Vercel Edge Functions |
| Autoscaling | 負荷に応じたコンピュート自動スケール | 本番運用 |
| Vercel Integration | 環境変数の自動設定、プレビューブランチ連携 | デプロイ |

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
| 主キー | BIGSERIAL | 自動採番（PostgreSQLにUNSIGNEDなし） |
| UUID | UUID | `gen_random_uuid()` で生成 |
| 日時 | TIMESTAMP | タイムゾーン考慮時は TIMESTAMPTZ |
| 金額 | NUMERIC(19,4) | 浮動小数点は使用しない |
| フラグ | BOOLEAN | true/false |
| 短いテキスト | VARCHAR(255) | 長さに応じて調整 |
| 長いテキスト | TEXT | PostgreSQLでは INDEX 可能 |
| JSON | JSONB | バイナリJSON、インデックス・検索対応 |
| 配列 | INTEGER[], TEXT[] | PostgreSQL固有の配列型 |

## インデックス戦略

### インデックス種類

| 種類 | 用途 | 作成例 |
|------|------|--------|
| PRIMARY KEY | 一意識別 | `PRIMARY KEY (id)` |
| UNIQUE | 重複防止 | `CREATE UNIQUE INDEX ON tbl (email)` |
| B-tree | 検索高速化（デフォルト） | `CREATE INDEX ON tbl (status)` |
| GIN | JSONB・配列・全文検索 | `CREATE INDEX ON tbl USING GIN (data)` |
| GiST | 地理・範囲データ | `CREATE INDEX ON tbl USING GiST (location)` |
| 複合インデックス | 複数カラム検索 | `CREATE INDEX ON tbl (user_id, created_at)` |
| 部分インデックス | 条件付きインデックス | `CREATE INDEX ON tbl (status) WHERE status = 'active'` |

### インデックス設計のベストプラクティス

1. **カーディナリティの高いカラムを優先**
2. **WHERE句で頻繁に使用するカラム**
3. **JOIN条件のカラム**
4. **ORDER BY、GROUP BYのカラム**
5. **複合インデックスは左端から使用される**
6. **部分インデックスで不要なデータを除外**

## テーブル設計テンプレート

```sql
CREATE TABLE table_name (
    id BIGSERIAL PRIMARY KEY,
    -- ビジネスカラム
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- updated_at 自動更新トリガー
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_table_name_updated_at
    BEFORE UPDATE ON table_name
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

## トランザクション

### 分離レベル

| レベル | 説明 | 用途 |
|--------|------|------|
| READ UNCOMMITTED | PostgreSQLではREAD COMMITTEDとして扱われる | - |
| READ COMMITTED | PostgreSQL デフォルト | 推奨 |
| REPEATABLE READ | スナップショット分離 | 集計処理 |
| SERIALIZABLE | 完全直列化 | 厳密な整合性が必要な場合 |

### ロック戦略

| ロック | 説明 | 用途 |
|--------|------|------|
| 楽観的ロック | version カラムで競合検出 | 競合が少ない場合 |
| 悲観的ロック | SELECT FOR UPDATE | 競合が多い場合 |
| Advisory Lock | アプリケーションレベルのロック | バッチ処理の排他制御 |

## パフォーマンス最適化

### クエリ最適化

1. **EXPLAIN ANALYZE で実行計画を確認**
2. **必要なカラムのみ SELECT**
3. **LIMIT を適切に使用**
4. **サブクエリより JOIN を優先**
5. **CTEを活用して複雑なクエリを分解**
6. **インデックスが使用されているか確認**

### PostgreSQL 固有の最適化

| 機能 | 用途 | 例 |
|------|------|-----|
| CTE (WITH句) | 複雑なクエリの分解 | `WITH cte AS (...) SELECT ...` |
| Window関数 | ランキング・累計 | `ROW_NUMBER() OVER (...)` |
| UPSERT | 挿入/更新の一括処理 | `ON CONFLICT ... DO UPDATE` |
| LATERAL JOIN | 行ごとのサブクエリ | `LEFT JOIN LATERAL ...` |

### Neon 固有の最適化

| 項目 | 推奨 | 理由 |
|------|------|------|
| コネクション | @neondatabase/serverless を使用 | サーバーレス環境での接続効率化 |
| ブランチ | 開発はブランチで作業 | 本番データに影響なし |
| Autosuspend | 開発DBは5分でサスペンド | コスト削減 |

## チェックリスト

- [ ] 適切なデータ型を選択したか
- [ ] 主キーを定義したか
- [ ] 外部キー制約を設定したか（必要な場合）
- [ ] インデックスを適切に設計したか
- [ ] 正規化レベルを検討したか
- [ ] updated_at トリガーを設定したか
- [ ] Neon ブランチ戦略を検討したか
