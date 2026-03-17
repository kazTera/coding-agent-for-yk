# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

MySQL を使用したアプリケーションの設計ドキュメントを、3ステップのワークフローで段階的に策定するプロジェクト。コードは書かない。設計ドキュメント（Markdown）を生成する。

## ディレクトリ構造

| ディレクトリ | 役割 |
|------------|------|
| `workflow/` | ワークフロー定義（3ステップ） |
| `output/` | 生成される成果物 |
| `skills/` | エージェント用リファレンス（MySQL設計ベストプラクティス） |
| `agents/` | カスタムエージェント定義 |
| `.claude/commands/` | スラッシュコマンド |
| `research/` | 調査資料（必要に応じて追加） |

## ワークフロー構造

```
Step 01: 要件分析       → output/01_requirements.md
Step 02: データベース設計 → output/02_database_design.md
Step 03: 実装計画       → output/03_implementation_plan.md
```

依存関係: `01 → 02 → 03`

## スラッシュコマンド

| コマンド | 説明 |
|---------|------|
| `/plan` | ワークフロー全体の管理。進捗確認と次ステップの提案 |
| `/plan-step <N>` | ステップ N（01〜03）を実行 |
| `/plan-review` | 設計ドキュメントのレビュー |
| `/plan-status` | 進捗ダッシュボード表示 |

## エージェント

| エージェント | ファイル | 用途 |
|-------------|---------|------|
| Database Designer | `database-designer.md` | データベース設計 |
| Design Reviewer | `design-reviewer.md` | 設計レビュー |

## MySQL 設計ガイドライン

### データ型選択

| 用途 | 推奨型 |
|------|--------|
| 主キー | BIGINT UNSIGNED AUTO_INCREMENT |
| 日時 | DATETIME |
| 金額 | DECIMAL(19,4) |
| フラグ | TINYINT(1) |

### インデックス設計

- カーディナリティの高いカラムを優先
- WHERE句で頻繁に使用するカラム
- 複合インデックスは左端から使用される

### トランザクション

- デフォルト分離レベル: REPEATABLE READ
- 楽観的ロック: version カラムで競合検出

## 出力フォーマット規約

- 言語: 日本語
- ファイル形式: Markdown
- 図表: Mermaid 記法を使用
- テーブル: Markdown テーブル形式
- チェックリスト: `- [ ]` 形式
- ファイル命名: `NN_snake_case.md`
- 出力先: `output/` 配下
