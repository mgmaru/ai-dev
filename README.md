# ai-dev

AI 時代のソフトウェア開発について調査・議論した結果をまとめたドキュメント置き場。

## ドキュメント一覧

### [docs/dev-process/](./docs/dev-process/) — 開発プロセスと設計手法
どう作るか。ドキュメント設計、テスト、スライス分割といった開発の進め方。

| ドキュメント | 内容 |
|---|---|
| [AI時代の開発ドキュメント設計ガイド](./docs/dev-process/ai-development-documentation-guide.md) | エージェントに必要なドキュメントの種類・粒度、コードとの乖離の防ぎ方、共通言語としてのテストと型、垂直スライスとTDD |
| [TDD（テスト駆動開発）入門](./docs/dev-process/tdd-primer.md) | TDDとは何か、流派と周辺手法、メリット・デメリット、実証研究、AIエージェント時代のTDD |
| [ハーネスエンジニアリング実践ガイド](./docs/dev-process/harness-engineering-guide.md) | 依存方向の機械的検証、カスタムリンター、構造テスト、修復手順つきエラーメッセージ、ドキュメント陳腐化対策、エージェントへの可視性設計、検証ループ |
| [Codexでの並列調査・並列開発の進め方](./docs/dev-process/codex-parallel-research-and-development.md) | SubAgent と Git Worktree の使い分け、Custom Agent、並列調査で精度を上げる方法、トークン消費の抑え方 |

読む順: TDD入門 → 開発ドキュメント設計ガイド → ハーネスエンジニアリング実践ガイド

### [docs/ai-platform/](./docs/ai-platform/) — AI開発基盤とガバナンス
社内に何を用意するか。API・MCP・Skills の管理体制と、その安全性の検証。

| ドキュメント | 内容 |
|---|---|
| [AI時代の開発基盤の内製化](./docs/ai-platform/ai-dev-platform-summary.md) | MCP・Skills の共有と公開範囲、社内API管理、何を買い何を作るか、個人でできること |
| [AIエージェントのAPI・MCP・Skills利用をどうテストするか](./docs/ai-platform/ai-agent-tool-governance-testing.md) | エージェントの権限・公開範囲・ネットワーク境界のテスト設計、既存の手法とツール、評価方法とリリース基準 |

読む順: 内製化のまとめ → ガバナンステスト（後者は前者から派生した詳細設計）

### [docs/execution-infra/](./docs/execution-infra/) — 連携基盤と実行基盤
何の上で動かすか。システム間のデータ連携と、途中で落ちない実行の仕組み。

| ドキュメント | 内容 |
|---|---|
| [iPaaS 入門](./docs/execution-infra/ipaas-nyumon.md) | iPaaS の位置づけ、データ連携の4層、プロセス/Atom/コネクタ、AI時代のデータ連携 |
| [durable execution 入門](./docs/execution-infra/durable-execution-nyumon.md) | リプレイの仕組み、Workflow と Activity の分離、保証されること・されないこと、AIエージェントへの適用 |
| [durable execution 実行環境カタログ](./docs/execution-infra/durable-execution-tools.md) | OSS・ツールの分類軸、カテゴリ別の比較表、操作インタフェース、料金、選び方 |

読む順: iPaaS入門 §15 → durable execution 入門 → 実行環境カタログ

## その他

| ファイル | 内容 |
|---|---|
| [notes.md](./notes.md) | 日々の調査・反省をメモ粒度で残す作業ログ |
| `prompts/` | 各ドキュメントを作成した際のプロンプト履歴（Git管理外） |
