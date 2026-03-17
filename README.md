# MySQL データベース設計プロジェクト

MySQL を使用したアプリケーションのデータベース設計を、3ステップのワークフローで段階的に策定する Claude Code プロジェクト。

## 前提条件

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) がインストールされていること

## クイックスタート

```bash
# プロジェクトディレクトリで Claude Code を起動
claude

# ワークフローの進捗確認と次ステップの提案
/plan

# Step 01 から実行開始
/plan-step 01
```

## ディレクトリ構成

```
.
├── CLAUDE.md              # プロジェクトルール（Claude Code が自動読み込み）
├── README.md              # 本ファイル
├── workflow/              # ワークフロー定義（3ステップ）
├── output/                # ワークフロー実行で生成される成果物
├── skills/                # エージェント用リファレンススキル
├── agents/                # カスタムエージェント定義
├── research/              # 調査資料（必要に応じて追加）
└── .claude/commands/      # スラッシュコマンド定義
```

## ワークフロー

```mermaid
flowchart LR
    W01["01 要件分析"] --> W02["02 データベース設計"] --> W03["03 実装計画"]

    style W01 fill:#e8f5e9,stroke:#4caf50
    style W02 fill:#e3f2fd,stroke:#2196f3
    style W03 fill:#fff3e0,stroke:#ff9800
```

## スラッシュコマンド

| コマンド | 説明 |
|---------|------|
| `/plan` | ワークフロー全体の管理。進捗確認と次ステップの提案 |
| `/plan-step <N>` | ステップ N（01〜03）を実行 |
| `/plan-review` | 設計ドキュメントのレビュー |
| `/plan-status` | 進捗ダッシュボード表示 |

### 使用例

```
# 要件分析を実行
/plan-step 01

ECサイトを開発中です。
商品、注文、ユーザーの管理が必要です。
```

## ステップ詳細

| Step | 名称 | 出力 |
|------|------|------|
| 01 | 要件分析 | `output/01_requirements.md` |
| 02 | データベース設計 | `output/02_database_design.md` |
| 03 | 実装計画 | `output/03_implementation_plan.md` |

## エージェント構成

| エージェント | 役割 |
|-------------|------|
| Database Designer | データベース設計、要件分析、実装計画 |
| Design Reviewer | 設計ドキュメントのレビュー |
