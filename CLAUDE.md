# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

ScalarDB Cluster (v3.17) を活用したマイクロサービスアーキテクチャの実装計画を、4フェーズ・13ステップのワークフローで段階的に策定するプロジェクト。コードは書かない。設計ドキュメント（Markdown）を生成する。

## ディレクトリ構造

| ディレクトリ | 役割 |
|------------|------|
| `workflow/` | ワークフロー定義（13ステップ + テンプレート）。**読み取り専用** |
| `research/` | 調査フェーズの成果物（16ドキュメント）。**読み取り専用** |
| `output/` | 生成される成果物。`phase1/` 〜 `phase4/` に分類 |
| `skills/` | エージェント用リファレンス（ScalarDB 制約・パターン・テンプレート）。SKILL.md 形式 |
| `agents/` | カスタムエージェント定義（Task tool の prompt として使用） |
| `.claude/commands/` | スラッシュコマンド（オーケストレーション層） |
| `docs/plans/` | Claude Code の EnterPlanMode 用作業ディレクトリ |

## ワークフロー構造

```
Phase 1: 要件・判断  → Step 01-03 → output/phase1/
Phase 2: 設計       → Step 04-06 → output/phase2/
Phase 3: 基盤       → Step 07-10 → output/phase3/  (08-10は並列実行可能)
Phase 4: 実行       → Step 11-13 → output/phase4/
```

依存関係: `01 → 02 → 03 → 04 → 05 → 06 → 07 → 08,09,10 → 11 → 12 → 13`

各ステップの定義は `workflow/NN_*.md` に記載。参照すべき調査資料は各ステップ内に明記されている。

## オーケストレーションアーキテクチャ

スラッシュコマンド → エージェント → スキルの3層構造:

1. **スラッシュコマンド** (`.claude/commands/`): ユーザーインターフェース。ワークフロー定義・エージェント定義・調査資料を読み込み、Task tool の prompt を構成する
2. **エージェント** (`agents/`): 専門知識と実行手順を持つ。Task tool の `general-purpose` サブエージェントとして起動される
3. **スキル** (`skills/`): エージェントが設計時に参照するナレッジベース（対話型ではない）

### エージェント→ステップ対応

| エージェント | ファイル | 推奨モデル | 対応ステップ / スコープ |
|-------------|---------|-----------|----------------------|
| Requirements Analyst | `requirements-analyst.md` | sonnet | 01, 03 |
| ScalarDB Architect | `scalardb-architect.md` | opus(04,05) / sonnet(他) | 02, 04, 05, 06, 11, 12 |
| Infrastructure Designer | `infrastructure-designer.md` | sonnet | 07, 08, 09, 10, 13 |
| 整合性レビュー | `review-consistency.md` | sonnet | 全フェーズレビュー（構造的整合性） |
| ScalarDB技術レビュー | `review-scalardb.md` | sonnet | 全フェーズレビュー（ScalarDB制約） |
| 運用準備レビュー | `review-operations.md` | sonnet | 全フェーズレビュー（運用準備） |
| リスクレビュー | `review-risk.md` | opus | 全フェーズレビュー（分散システムリスク） |
| ビジネス要件レビュー | `review-business.md` | sonnet | 全フェーズレビュー（ビジネス要件） |
| レビュー統合 | `review-synthesizer.md` | sonnet | 全フェーズレビュー（統合レポート生成） |

### `/plan-step` の実行フロー

1. `workflow/NN_*.md` を読み込み（手順・チェックリスト）
2. 対応する `agents/*.md` を読み込み（専門知識）
3. ワークフロー定義が指す `research/*.md` を読み込み（調査資料）
4. 先行ステップの成果物を `output/` から読み込み（前提条件確認）
5. すべてを prompt に含めて Task tool で `general-purpose` エージェントを起動
6. エージェントがユーザーと対話しながら成果物を `output/phaseN/NN_*.md` に出力

## スラッシュコマンド

| コマンド | 説明 |
|---------|------|
| `/plan` | ワークフロー全体の管理。進捗確認と次ステップの提案 |
| `/plan-step <N>` | ステップ N（01〜13）を実行 |
| `/plan-review <phase>` | フェーズ（phase1〜phase4）の5パースペクティブ並列レビュー |
| `/plan-review final` | 全13ステップ完了後のクロスフェーズ最終レビュー |
| `/plan-status` | 全体の進捗状況をダッシュボード表示（haiku 推奨） |

## モデル選択ガイドライン

エージェント呼び出し時は、タスクの複雑さに応じてモデルを選択し、トークン消費を最適化する。

| モデル | 用途 | 選択基準 |
|-------|------|---------|
| `haiku` | ステータス確認、テンプレート生成、ファイル一覧、フォーマット整形 | 判断不要の定型処理 |
| `sonnet` | 設計分析、ドキュメント生成、要件整理、レビュー実行 | 分析・生成が必要だが定型パターンに従う処理 |
| `opus` | アーキテクチャ判断、トレードオフ評価、クロスカッティング分析 | 複数の選択肢から最適解を導く複雑な判断 |

## ScalarDB ドメイン知識

### コア概念

- **Consensus Commit**: OCC（楽観的並行性制御）+ クライアント協調型2PCによる分散トランザクションプロトコル
- **Coordinator テーブル**: トランザクション状態を管理する中央テーブル。保護必須
- **メタデータカラム**: ScalarDB が各レコードに付加する管理カラム（tx_id, tx_state, tx_version 等）
- **ScalarDB Analytics**: CDC経由でOLAPクエリを実現するコンポーネント

### v3.17 最適化機能

| 機能 | 説明 |
|------|------|
| Piggyback Begin | BEGIN を最初の操作に相乗り。ラウンドトリップ削減 |
| Write Buffering | 書き込みをクライアント側でバッファリング。ネットワーク呼出削減 |
| Batch Operations | 複数操作を一括送信 |
| Transaction Metadata Decoupling | メタデータを別テーブルに分離。READ レイテンシに影響あり |

### 制約・注意事項

- ScalarDB 管理テーブルでは DB 固有機能（トリガー、ストアドプロシージャ、DB レベル制約等）は使用不可
- 2PC 適用は最大 2-3 サービスに制限（分散モノリスリスク回避）
- OCC 競合率は 5% 未満を推奨
- Partition Key のホットスポットリスクを必ず評価する
- メタデータオーバーヘッド（約200バイト/レコード）を見積もる

## ユビキタス言語

| 日本語 | English | 定義 |
|--------|---------|------|
| 境界コンテキスト | Bounded Context | サービスの責務範囲。ユビキタス言語が一貫する領域 |
| 集約 | Aggregate | トランザクション境界となるエンティティ群 |
| 集約ルート | Aggregate Root | 集約への外部アクセスポイントとなるエンティティ |
| 適用範囲 | ScalarDB Scope | ScalarDB が管理するテーブル群の範囲 |
| 強整合性 | Strong Consistency | ACID トランザクションによるデータ整合性 |
| 結果整合性 | Eventual Consistency | Saga パターン等による遅延整合性 |
| 管理対象テーブル | Managed Table | ScalarDB Consensus Commit で管理されるテーブル |
| 非管理テーブル | Non-managed Table | ScalarDB 管理外のテーブル（DB固有機能使用可） |

## 出力フォーマット規約

- 言語: 日本語
- ファイル形式: Markdown
- 図表: Mermaid 記法を使用
- テーブル: Markdown テーブル形式
- チェックリスト: `- [ ]` 形式
- ファイル命名: `NN_snake_case.md`（NN はステップ番号）
- 出力先: `output/phaseN/` 配下
