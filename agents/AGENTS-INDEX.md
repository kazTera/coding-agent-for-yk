# エージェント一覧

## 概要

本ディレクトリには、ワークフロー実行時に使用するエージェント定義を格納します。

## エージェント一覧

| エージェント | ファイル | 推奨モデル | 用途 |
|-------------|---------|-----------|------|
| Database Designer | `database-designer.md` | sonnet | MySQL データベース設計 |
| Design Reviewer | `design-reviewer.md` | sonnet | 設計レビュー |

## 利用方法

`/plan-step` コマンドから Task tool の prompt として読み込まれ、`general-purpose` サブエージェントとして起動されます。
