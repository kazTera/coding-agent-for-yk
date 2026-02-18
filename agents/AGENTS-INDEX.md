# エージェント一覧

## 概要

本ディレクトリには、`/plan-step` および `/plan-review` コマンドが Task tool 経由で利用するカスタムエージェント定義を格納する。

各エージェントファイルは、Task tool の `general-purpose` エージェントに prompt として読み込まれ、専門知識と実行手順を提供する。

## エージェント一覧

| エージェント | ファイル | 推奨モデル | 対応ステップ |
|-------------|---------|-----------|------------|
| Requirements Analyst | `requirements-analyst.md` | sonnet | 01, 03 |
| ScalarDB Architect | `scalardb-architect.md` | opus/sonnet | 02, 04, 05, 06, 11, 12 |
| Infrastructure Designer | `infrastructure-designer.md` | sonnet | 07, 08, 09, 10, 13 |
| Review Agent | `review-agent.md` | sonnet | **非推奨** → 下記5エージェントに分割 |

### レビューエージェント（5パースペクティブ + 統合）

| エージェント | ファイル | 推奨モデル | 対応スコープ |
|-------------|---------|-----------|------------|
| 整合性レビュー | `review-consistency.md` | sonnet | 構造的整合性 |
| ScalarDB技術レビュー | `review-scalardb.md` | sonnet | ScalarDB制約 |
| 運用準備レビュー | `review-operations.md` | sonnet | 運用準備状況 |
| リスクレビュー | `review-risk.md` | opus | 分散システムリスク |
| ビジネス要件レビュー | `review-business.md` | sonnet | ビジネス要件整合 |
| レビュー統合 | `review-synthesizer.md` | sonnet | 統合レポート生成 |

## モデル選択ガイド

| モデル | 用途 | エージェント |
|-------|------|------------|
| haiku | ステータス確認、テンプレート生成 | (plan-status コマンド内) |
| sonnet | 設計分析、ドキュメント生成、レビュー | 全エージェント（標準）、レビュー4エージェント |
| opus | アーキテクチャ判断、トレードオフ評価 | ScalarDB Architect (Step 04, 05)、リスクレビュー |

## 呼び出し方法

Commands (`.claude/commands/`) 内から以下のように利用:

1. エージェント定義ファイルを読み込む
2. ワークフロー定義と調査資料を合わせて prompt を構成
3. Task tool の `general-purpose` エージェントとして起動（model パラメータ指定）

```
Task tool 呼び出し例:
  subagent_type: general-purpose
  model: sonnet
  prompt: [エージェント定義 + ワークフロー定義 + 調査資料の内容]
```
