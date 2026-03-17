# ワークフロー概要

## 目的

Neon（PostgreSQL）を使用したアプリケーションの設計ドキュメントを生成します。

## ワークフロー構成

```
Step 01: 要件分析       → output/01_requirements.md
Step 02: データベース設計 → output/02_database_design.md
Step 03: 実装計画       → output/03_implementation_plan.md
```

## ステップ詳細

| Step | 名称 | エージェント | 出力 |
|------|------|-------------|------|
| 01 | 要件分析 | database-designer | 要件定義書 |
| 02 | データベース設計 | database-designer | DB設計書、DDL |
| 03 | 実装計画 | database-designer | 実装タスク一覧 |

## 依存関係

```
01 → 02 → 03
```

## 実行方法

```
/plan-step 01  # 要件分析
/plan-step 02  # データベース設計
/plan-step 03  # 実装計画
/plan-review   # レビュー
```
