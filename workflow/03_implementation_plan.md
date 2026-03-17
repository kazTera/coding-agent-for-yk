# Step 03: 実装計画

## 目的

データベース設計に基づいて実装タスクを整理する。

## 担当エージェント

`database-designer`

## 前提条件

- Step 02 完了（`output/02_database_design.md` が存在）

## 参照リソース

- `output/01_requirements.md`
- `output/02_database_design.md`

## 実行手順

1. 設計書の読み込み
2. 実装タスクの洗い出し
3. 依存関係の整理
4. 優先順位付け
5. 実装ガイドの作成

## 出力

`output/03_implementation_plan.md`

```markdown
# 実装計画書

## 概要

| 項目 | 内容 |
|------|------|
| 総タスク数 | |
| 優先度高タスク | |

## タスク一覧

### Phase 1: データベース構築

| # | タスク | 依存 | 優先度 |
|---|--------|------|--------|
| 1 | DDL実行 | - | 高 |
| 2 | 初期データ投入 | 1 | 中 |

### Phase 2: アプリケーション実装

| # | タスク | 依存 | 優先度 |
|---|--------|------|--------|
| 3 | モデル層実装 | 1 | 高 |
| 4 | リポジトリ層実装 | 3 | 高 |

## 実装ガイド

### データベース接続設定

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/dbname
    username: user
    password: password
```

### モデル実装例

```java
@Entity
@Table(name = "table_name")
public class EntityName {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
}
```
```

## 完了条件

- [ ] 全タスクが洗い出されている
- [ ] 依存関係が整理されている
- [ ] 実装ガイドが作成されている
