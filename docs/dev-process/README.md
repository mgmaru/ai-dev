# 開発プロセスと設計手法

**どう作るか。** ドキュメント設計、テスト、スライス分割、そしてエージェントに開発させる環境（外部ハーネス）の作り方を扱う。

| ドキュメント | 内容 |
|---|---|
| [tdd-primer.md](./tdd-primer.md) | TDD（テスト駆動開発）入門。TDDとは何か、流派と周辺手法、メリット・デメリット、実証研究、AIエージェント時代のTDD |
| [ai-development-documentation-guide.md](./ai-development-documentation-guide.md) | AI時代の開発ドキュメント設計ガイド。ドキュメントの種類と粒度、コードとの乖離、共通言語としてのテストと型、垂直スライスとTDD |
| [harness-engineering-guide.md](./harness-engineering-guide.md) | ハーネスエンジニアリング実践ガイド。依存方向の機械的検証、カスタムリンター、構造テスト、修復手順つきエラーメッセージ、ドキュメント陳腐化対策、エージェントへの可視性設計、検証ループ |
| [codex_parallel_research_and_development.md](./codex_parallel_research_and_development.md) | Codexでの並列調査・並列開発の進め方。SubAgent と Git Worktree の使い分け、Custom Agent、並列調査で精度を上げる方法、トークン消費の抑え方 |

**読む順:** `tdd-primer.md` → `ai-development-documentation-guide.md` → `harness-engineering-guide.md`
TDD入門は設計ガイド第7章の前提知識。ハーネスガイドは設計ガイド第7章（依存グラフの可視化）の続きにあたり、「決めた設計を機械的に強制する」方法を扱う。
Codexの並列調査・並列開発は独立した話題（エージェントの運用方法）なので、どこから読んでもよい。
