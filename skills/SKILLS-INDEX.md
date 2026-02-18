# スキル一覧

## 概要

本ディレクトリには、エージェントが参照するリファレンス資料（ScalarDB固有の制約・パターン・テンプレート）を格納する。

スキルは対話型テンプレートではなく、エージェントが設計時に参照するナレッジベースとして機能する。

## スキル一覧

### ScalarDB 固有リファレンス

| スキル | ディレクトリ | 内容 |
|--------|------------|------|
| ScalarDB Data Model | `scalardb-data-model/` | PK/CK/SI設計制約、ホットスポット評価、メタデータオーバーヘッド、DB選定基準 |
| ScalarDB Transaction | `scalardb-transaction/` | トランザクションパターン、OCC競合率評価、リトライ戦略、v3.17最適化 |

### 設計リファレンス（既存スキルベース + ScalarDB適応）

| スキル | ディレクトリ | 元スキル | ScalarDB追加内容 |
|--------|------------|---------|-----------------|
| Domain Modeling | `domain-modeling/` | design-and-refactoring-agent | 集約境界=トランザクション境界、2PC制限考慮 |
| API Design | `api-design/` | design-and-refactoring-agent | トランザクション例外のHTTPマッピング、CDCフィルタリング |
| Database Design | `database-design/` | design-and-refactoring-agent | v3.17メタデータ分離、マルチストレージ構成 |
| Infrastructure Design | `infrastructure-design/` | design-and-refactoring-agent | v3.17設定、PDB、Coordinator保護 |
| Implementation Plan | `implementation-plan/` | design-and-refactoring-agent | ScalarDB固有タスクテンプレート |

## ステップ → リファレンス対応表

| Step | 名称 | 参照リファレンス | エージェント |
|------|------|----------------|------------|
| 01 | 要件分析・適用判断 | (ワークフロー + 調査資料で対応) | requirements-analyst |
| 02 | ドメインモデリング | `domain-modeling` | scalardb-architect |
| 03 | ScalarDB適用範囲決定 | (ワークフロー + 調査資料で対応) | requirements-analyst |
| 04 | データモデル設計 | `scalardb-data-model` + `database-design` | scalardb-architect |
| 05 | トランザクション設計 | `scalardb-transaction` | scalardb-architect |
| 06 | API・インターフェース設計 | `api-design` | scalardb-architect |
| 07 | インフラ設計 | `infrastructure-design` | infrastructure-designer |
| 08 | セキュリティ設計 | (ワークフロー + 調査資料で対応) | infrastructure-designer |
| 09 | オブザーバビリティ設計 | (ワークフロー + 調査資料で対応) | infrastructure-designer |
| 10 | 障害復旧設計 | (ワークフロー + 調査資料で対応) | infrastructure-designer |
| 11 | 実装ガイド | `implementation-plan` | scalardb-architect |
| 12 | テスト戦略 | (ワークフロー + 調査資料で対応) | scalardb-architect |
| 13 | デプロイ・ロールアウト | (ワークフロー + 調査資料で対応) | infrastructure-designer |

## 利用方法

スキルは `/plan-step` コマンドから自動的に参照される。エージェントがワークフロー定義と調査資料に加えて、該当するリファレンスの知見を設計に適用する。
