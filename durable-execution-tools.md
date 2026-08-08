# durable execution 実行環境カタログ — OSS・ツール調査

> 対象: [durable execution 入門](./durable-execution-nyumon.md) を読み終えた人
> ゴール: §10「製品の地図」を、実際に選定できる粒度まで展開する
> 調査時点: **2026年8月**

---

## 0. この文書の位置づけ

入門編の §10 では6製品を1行ずつ紹介しました。本書はそれを拡張し、**「durable execution を実行するための環境」を網羅的に並べます。**

ただし最初に断っておくべきことがあります。

> **「durable execution 製品」という単一のカテゴリは、実際には存在しません。**

同じ「ワークフロー」という言葉を使っていても、解いている問題が違うものが同居しています。本書は、それらを**混ぜずに分類すること**を主目的にしています。分類を誤ると、Airflow を検討して「Temporal より安い」という無意味な比較に行き着きます。

---

## 1. 表の読み方 — 6つの分類軸

### 軸1. 種別 — 実行基盤をどこに持つか

**初版ではこの列の定義を書いていませんでした。しかも値の粒度が揃っていませんでした。** 3つの異なる観点が混ざっていたためです。

| 混ざっていた観点 | 初版で使っていた値 |
|---|---|
| デプロイ形態 | 汎用エンジン、ライブラリ、サイドカー、マネージド |
| 出自・立ち位置 | オーケストレータ、エージェントランタイム、エージェント FW |
| 提供形態 | マネージド寄り、SDK＋マネージド、コンポーネント |

これを **「実行基盤をどこに持つか」という単一の軸**に整理し直します。値は4つです。

| 種別 | 定義 | あなたが立てて運用するもの | 判別の質問 |
|---|---|---|---|
| **セルフホスト型エンジン** | ワークフローの進捗を管理する**専用のサーバプロセス**が存在し、それを自分で立てる | エンジン本体（＋必要なら DB / Kafka） | 「`docker compose up` する対象があるか？」 |
| **埋め込みライブラリ型** | **専用サーバが存在しない。** アプリのプロセス内でライブラリとして動き、状態は既存の DB に書く | 何も（既存 DB のみ） | 「`pip install` / `npm install` だけで完結するか？」 |
| **サイドカー型** | アプリの隣に**汎用ランタイム**が並走し、その機能の1つとしてワークフローが提供される | サイドカープロセス | 「ワークフロー専用ではない基盤の一機能か？」 |
| **マネージド型** | ベンダのサービス。**自前ホストという選択肢がそもそもない** | 何も | 「自分で動かす手段があるか？ 無ければ該当」 |

**注意点が2つあります。**

1. **種別と「OSS か商用か」は独立した軸です。** セルフホスト型でもライセンスが OSI 準拠でないもの（LittleHorse）があり、マネージド型でも SDK は OSS のもの（Azure Durable Task SDKs）があります。混ぜると誤ります。
2. **多くの製品が「セルフホスト型エンジン ＋ 公式マネージド」の両方を提供しています。** 表では**自前ホストできるか**を基準に分類しました。マネージドしか使わないつもりでも、この区別は**「将来引き剥がせるか」**を決めます。

### 軸2. 実装方式

入門編 §10 で挙げた2方式に、1つ足して3つです。

| 方式 | 何をするか | 決定論の制約 | 代表 |
|---|---|---|---|
| **ジャーナルリプレイ** | 完了ステップを履歴に記録し、クラッシュ時にコードを最初から流し直して記録済み分をスキップ | **強い**（Workflow コードは決定論的でなければならない） | Temporal、Restate、AWS Lambda Durable Functions、Azure Durable Functions、Cloudflare Workflows |
| **DB チェックポイント** | ステップ／ノードごとに状態を DB に書き、途中から再開 | **弱い**（最初から流し直さないので制約が緩い） | DBOS、LangGraph、Hatchet |
| **透過的スナップショット** | ランタイム（WASM）がホスト呼び出しをすべて記録するので、開発者が step で包む必要がない | **無い**（ランタイムが吸収する） | Golem |

**この軸が最も効きます。** 入門編 §5.2「Workflow の中で禁止されるもの」の表が適用されるのは、**ジャーナルリプレイ方式だけ**です。DB チェックポイント方式では `time.Now()` を直接呼んでも壊れません。

代わりに失うものもあります。リプレイ方式は「コードそのものが状態」なので状態管理コードが一行も要らないのに対し、チェックポイント方式は「どこまで進んだか」を明示的に持つ必要があり、ステップ境界がフレームワークの形（グラフのノードなど）に縛られがちです。

### 軸3. コードがどこで動くか — pull か push か

**これが serverless 適性と待機コストを決めます。**

| 型 | 動き | 帰結 |
|---|---|---|
| **pull（常駐ワーカー）** | ワーカーが常駐してサーバをポーリングし、仕事を取りに行く | serverless に載らない。アイドル中もワーカーは動いている |
| **push（ステートレス関数）** | エンジンが HTTP でアプリを呼び出す。ステップごとに別リクエスト | Next.js の API ルートや Lambda にそのまま載る。アイドル中の計算コストが0 |

Temporal は pull、Restate / Inngest / Upstash / Vercel は push です。**入門編 §9.3 の「待機中のコストが60〜80%下がる」は push 型でより極端に効きます。**

### 軸4. 状態をどこに置くか

| 置き場所 | 運用負荷 | 代表 |
|---|---|---|
| 専用クラスタ＋外部 DB | **高** | Temporal（DB ＋ Elasticsearch） |
| 単一バイナリ（自己完結） | 低 | Restate、Resonate |
| **既存の Postgres だけ** | **最小** | DBOS、Hatchet |
| ベンダのマネージド | ゼロ（ただしロックイン） | Cloudflare、AWS、Azure、Upstash |

### 軸5. ライセンス

「OSS」と書いてあっても中身が違います。

- **OSI 準拠**（MIT / Apache 2.0）— Temporal、DBOS、Hatchet、Trigger.dev、Conductor、Resonate
- **fair source / BSL**（一定期間後に OSS 化）— Restate（runtime は BSL 1.1）、Inngest（SSPL → 3年後に Apache 2.0）、Golem（BUSL-1.1 → Apache 2）
- **source-available**（OSS ではない）— LittleHorse サーバ、n8n

商用配布や SaaS 提供を考えるなら BSL / SSPL は必ず原文を読んでください。

### 軸6. 「durable execution なのか」

最後に、**そもそもこのカテゴリに入らないもの**を分けます（→ §8）。

**このほか、操作インタフェース（→ §3）と費用（→ §4）を独立した章として扱います。** どちらも「二択で見ると判断を誤る」性質があるためです。

---

## 2. 全体マップ

### 2.1 基本情報

種別・UI・料金・機能を1つの表に詰めると読めなくなるため、**主要機能と適用場面は §2.2、UI は §3、料金は §4 に分離**しています。**行番号（#）は §2.2 と共通**なので、2つの表は左右に並べて読めます。

| # | 名前 | 種別 | 実装方式 | 状態の置き場所 | 主な言語 | ライセンス |
|---|---|---|---|---|---|---|
| 1 | **Temporal** | セルフホスト型エンジン（＋公式マネージド） | ジャーナルリプレイ（pull） | 専用クラスタ＋DB | Go, Java, Python, TS, .NET, PHP, Ruby | MIT |
| 2 | **Restate** | セルフホスト型エンジン（＋公式マネージド） | ジャーナルリプレイ（push） | 単一バイナリ | TS, Java/Kotlin, Python, Go, Rust | BSL 1.1（SDK は MIT） |
| 3 | **DBOS Transact** | **埋め込みライブラリ型** | DB チェックポイント | 既存 Postgres | Python, TS, Go, Java, Kotlin | MIT |
| 4 | **Hatchet** | セルフホスト型エンジン（＋公式マネージド） | 耐久イベントログ | Postgres | Python, TS, Go, Ruby | MIT |
| 5 | **Inngest** | セルフホスト型エンジン（＋公式マネージド） | ジャーナル（push / イベント駆動） | Inngest（or 自前 Redis+PG） | TS, Python, Go | SSPL→Apache 2.0 |
| 6 | **Trigger.dev** | セルフホスト型エンジン（＋公式マネージド） | チェックポイント／再開 | Trigger.dev（or 自前 PG） | TypeScript（Python は補助） | Apache 2.0 |
| 7 | **Resonate** | セルフホスト型エンジン | ジャーナル（distributed async await） | 単一バイナリ（SQLite / PG） | TS, Python, Go, Rust, Java | Apache 2.0 |
| 8 | **Golem** | セルフホスト型エンジン（＋マネージド:Preview） | **透過的スナップショット（WASM）** | Golem ランタイム | WASM に載る言語（TS 等） | BUSL-1.1→Apache 2 |
| 9 | **Conductor (OSS/Orkes)** | セルフホスト型エンジン（＋Orkes マネージド） | タスク＋ワーカー、全履歴永続化 | Redis / PG / MySQL / Cassandra / SQLite | Java, Python, Go, JS, C#, Ruby, Rust | Apache 2.0 |
| 10 | **LittleHorse** | セルフホスト型エンジン（＋Cloud） | Kafka を write-ahead log に | Kafka | Java, Go, Python, C# | source-available（SDK は Apache 2.0） |
| 11 | **Dapr Workflows / Agents** | **サイドカー型** | ジャーナルリプレイ（Durable Task 系） | 差し替え可能な state store | Python, JS, .NET, Java, Go | Apache 2.0（CNCF） |
| 12 | **AWS Lambda Durable Functions** | マネージド型 | ジャーナルリプレイ | AWS 管理 | Node.js, Python, Java, .NET | 商用 |
| 13 | **AWS Step Functions** | マネージド型 | 状態機械（JSON/ASL） | AWS 管理 | 言語非依存（JSON 定義） | 商用 |
| 14 | **Azure Durable Functions / Durable Task SDKs** | マネージド型（SDK は埋め込み可、バックエンドは Azure） | イベントソーシング＋リプレイ | Durable Task Scheduler 等 | .NET, Python, Java, JS, PowerShell | 商用（SDK は OSS） |
| 15 | **Cloudflare Workflows** | マネージド型 | ジャーナルリプレイ | Cloudflare 管理 | TypeScript（Python ベータ） | 商用 |
| 16 | **Vercel Workflows / Workflow DevKit** | **埋め込みライブラリ型 ＋ マネージド型** | ジャーナル（`"use workflow"`） | 「World」アダプタ次第 | TypeScript, Python（ベータ） | OSS SDK |
| 17 | **Upstash Workflow** | マネージド型 | ステップメモ化（push / HTTP） | Upstash（QStash） | TypeScript, Python | 商用 |
| 18 | **Convex Workflow** | マネージド型（Convex 基盤前提） | DB チェックポイント | Convex DB | TypeScript | OSS コンポーネント |
| 19 | **LangGraph** | **埋め込みライブラリ型** | DB チェックポイント | Postgres / Redis / DynamoDB 等 | Python, JS/TS | OSS |

### 2.2 主要機能と適用場面

**「向かない場面」を併記しています。** 選定では、こちらのほうが決定的に効くことが多いためです。

| # | 名前 | 主要機能 | 有効な場面 | 向かない場面 |
|---|---|---|---|---|
| 1 | **Temporal** | Signal / Query / Update（外部からの介入）、Saga・補償、Continue-As-New、**Workflow Versioning とリプレイテスト**、Nexus、7言語 SDK | **数日〜数年の長時間ワークフロー**、注文処理、耐久台帳、Saga、CI/CD、多言語チーム、**AI エージェントの耐久層** | 運用要員を割けない小規模チーム／ミリ秒応答が要る同期パス |
| 2 | **Restate** | **Virtual Object**（キー単位の状態＋直列化。ロックサービス不要）、exactly-once メッセージング、durable promise、耐久タイマー | **マイクロサービス間の整合性・冪等性**、イベントの exactly-once 処理、serverless、運用を軽くしたい | BSL が通らない組織／大規模な本番実績が必須の場面 |
| 3 | **DBOS Transact** | デコレータ注釈、**耐久キュー**（並行数制御）、cron スケジューラ内蔵、workflow ID = 冪等キー、Conductor UI、MCP サーバ統合 | **既に Postgres 中心**、対話的・低レイテンシ（1〜2ms）、AI エージェント、データパイプライン、cron 基盤 | Postgres のスケール上限を超える規模／プロセス外の監督者が必須の構成 |
| 4 | **Hatchet** | **優先度・静的/動的レート制限・fair scheduling**、ワーカースロット、ラベルルーティング、DAG、耐久 sleep / イベント待ち、TUI、OTel / Prometheus | **バックグラウンドジョブ基盤**、大量並列、マルチテナントの公平性、外部 API のレート制限に合わせる | 厳密な決定論的リプレイ保証が要る場面（通常タスクは at-least-once） |
| 5 | **Inngest** | イベント駆動（**CEL でマッチ**）、`step.run/sleep/waitForEvent`、flow control（レート制限・テナント別 concurrency・throttle）、ステップ単位トレース | **Webhook / イベント起点**、Next.js on Vercel などの serverless、**テナント別の流量制御** | 366日超の待機／ステップ出力 4 MiB 超 |
| 6 | **Trigger.dev** | **Realtime API（LLM 応答をフロントへストリーム）**、waitpoints、キュー・並行数制御、React hooks、ダッシュボードからのリプレイ | **メディア処理**（動画・音声）、ブラウザ自動化、**LLM 応答をユーザーに流す UI**、タイムアウトのない長時間処理 | アプリと同一デプロイにしたい場合（別デプロイになる） |
| 7 | **Resonate** | durable promise、distributed async await、`resonate tree` で呼び出しグラフ、単一バイナリ、**オープンプロトコル** | 新しい DSL を覚えたくない、**ベンダロックインを避けたい**、小さく始めたい | 大規模実績・エンタープライズサポートが要る場面 |
| 8 | **Golem** | **step 不要の透過的耐久**、エージェントごとの WASM サンドボックス＋専用 FS ＋専用 SQLite、exactly-once、**明示的 grant とその journal 化**、MCP 自動、quota 強制 | **マルチテナントのエージェント実行基盤**、カスタマーサポート自動化、コーディングエージェント、**隔離と耐久を同時に解きたい** | 既存の Python / Java 資産をそのまま載せたい／本番 SLA が今すぐ必要（Cloud は Preview） |
| 9 | **Conductor (OSS/Orkes)** | **ビジュアルビルダー**、全履歴を無期限保持、**任意タスクからの再実行**、LLM タスク（14+ プロバイダ）、MCP ツール呼び出し、ベクタ DB / RAG、human-in-the-loop 承認、動的ワークフロー | **非エンジニアも定義に関わる**、運用者が UI で復旧操作をする、ポリグロット（7言語＋HTTP）、RAG パイプライン | **git diff でのコードレビューを重視する場合**（JSON 定義のため） |
| 10 | **LittleHorse** | Kafka を write-ahead log に、**User Task（人間の作業）が組み込み**、Dashboard から実行可、`lhctl`、モジュラー構成 | **Kafka を既に運用している**、マイクロサービス構成、GPU オーケストレーション、顧客オンボーディング | OSI 準拠ライセンスが必須の組織／Kafka を持ち込みたくない場合 |
| 11 | **Dapr Workflows / Agents** | **6つのワークフローパターン**（task chaining / fan-out-fan-in / async HTTP / monitor / 外部システム連携 / Saga）、state store 差し替え、Pub/Sub マルチエージェント | **K8s に Dapr 導入済み**、ポリグロットなマイクロサービス、**マルチエージェント協調** | Dapr を入れていない環境（サイドカー導入コストが先に来る） |
| 12 | **AWS Lambda Durable Functions** | step / **wait（最大1年）** / callback / invoke、明示的な決定論契約 | **ロジックが Lambda に閉じている**、AWS 全振り、コード中心で書きたい | **大規模 fan-out**（Distributed Map の領分）／3,000 durable operation 超／非エンジニアと可視化を共有したい |
| 13 | **AWS Step Functions** | **Workflow Studio（ビジュアル）**、220+ サービス / 14,000+ API 統合、**Distributed Map（S3 から最大1万並列子実行）**、task token、Standard / Express | **数百万件の大規模並列 ETL / ML 前処理 / ログ解析**、AWS サービス間の連携、ステークホルダーとの視覚的共有、Express は高頻度短時間 | **コードでロジックを書きたい場合**／256 KiB 超のペイロード |
| 14 | **Azure Durable Functions** | orchestrator / activity、**Durable Entities**、fan-out/fan-in、eternal orchestration、**任意基盤で動く Durable Task SDKs** | **.NET / Azure 中心**、Azure Functions の既存資産、ACA / K8s へ持ち出したい | Azure 外を主戦場にする場合（バックエンドは結局 Azure） |
| 15 | **Cloudflare Workflows** | `step.do/sleep/sleepUntil/waitForEvent`、**インスタンスごとのローカル状態（DB 不要）**、**Dynamic Workflows（テナント別に定義をロード）**、Agents SDK 統合 | **既に Workers 上にいる**、AI エージェント、UGC の後処理（推論・検証）、課金ジョブ、**テナントごとに異なるワークフロー** | **長時間 sleep を多用する設計**（2026/8/10〜 sleep もステップ課金）／1 MiB 超のステップ出力 |
| 16 | **Vercel Workflows** | `"use workflow"` / `"use step"`、**sleep 無制限**、hooks、**Streams**、**マルチリージョン（run をリージョンに固定）**、Skew Protection、World アダプタ | **Next.js / Vercel 上のアプリ**、AI エージェント、**LLM 応答のストリーミング**、ステートフルな Slack bot | 1 run が25,000イベント / 10,000ステップを超える場合（子ワークフローに分割が必要） |
| 17 | **Upstash Workflow** | HTTP ステップ、`waitForEvent` / `notify`、cron、並列処理、**DLQ から失敗ステップだけ再開**、flow control | **月額固定費ゼロで始めたい**、顧客オンボーディング、EC の注文処理、決済リトライ、画像処理の並列化、大規模データの分割処理 | 自前ホストが要件の場合／1,000ステップ超（引き上げ可） |
| 18 | **Convex Workflow** | DB チェックポイント、**durable agents（無期限実行・非同期ツール呼び出し）**、Mastra 連携 | **既に Convex を使っている**、エージェントのオーケストレーション（調査→分析→レポート）、マルチエージェント協調 | Convex 以外の基盤 |
| 19 | **LangGraph** | checkpointer（Postgres / Redis / DynamoDB）、thread、**time travel**、HITL interrupt、グラフ状態機械 | **エージェントのグラフ構造を明示的に設計したい**、human-in-the-loop、state の巻き戻し | **単体では「完遂」を保証しない**（→ §7.1）。外部エンジンとの併用が前提 |

### 2.3 その前に — durable execution 自体が向かない場面

**個別製品の前に、カテゴリ全体の適用限界を押さえておくべきです。**

> **durable execution は、ステップ間にレイテンシを足す仕組みです。これは避けられない「信頼性の税金」です。**

ステップごとに状態を永続化する以上、書き込みの確認を待つ時間が必ず入ります。実測値として報告されているのは次のあたりです。

| 構成 | ステップ間のレイテンシ |
|---|---|
| 標準的な HTTP serve | **50〜250ms** |
| 常時接続（Connect）のみ | 20〜100ms |
| チェックポイントのみ | < 10ms |
| チェックポイント＋常時接続 | < 5ms |

状態ストアへの書き込み自体は5〜50ms 程度ですが、オーケストレーション全体では**1ステップあたり100〜500ms に達することがあります。** 5ステップの処理で約600ms（全体の14%）がオーケストレーションのオーバーヘッドだった、という例も報告されています。10ステップのエージェントなら1〜5秒が積み上がる計算です。

**設計思想として、durable execution は「障害はまれである」という前提に立っています。「ユーザーがミリ秒単位で待っている」という前提ではありません。**

したがって、次のような場面では **durable execution そのものを使うべきではありません。**

| 向かない場面 | 理由 | 代わりに |
|---|---|---|
| **同期リクエストパスでミリ秒応答が要る** | ステップごとの永続化がそのまま応答時間に乗る | 通常のアプリコード＋リトライ |
| **高スループットのストリーム処理** | 1件ごとの永続化がボトルネックになる | Kafka / Flink 等のストリーム処理基盤 |
| **単発の API 呼び出しをリトライしたいだけ** | 副作用が1つなら「中途半端な成功」が起きない | 指数バックオフのライブラリ |
| **スケジュール実行される DAG が主目的** | 解いている問題が違う（→ §8） | Airflow / Dagster |

> **入門編 §0 の判定基準がそのまま使えます。要るのは「複数の副作用にまたがる中途半端な成功」が起きうるときだけです。**

なお、レイテンシは製品選択で緩和できます。**DBOS のチェックポイントは1〜2ms**（外部オーケストレータへの往復が無いため）で、この点では埋め込みライブラリ型が有利です。Inngest も常時接続とチェックポイントの組み合わせで5ms 未満を出しています。

---

## 3. 操作インタフェース — 定義方法 / 監視 UI / CLI

### 3.1 まず、軸の取り方について

> **「GUI か CLI か」は、この分野では二択になりません。**

理由は単純で、**ほぼすべての製品が両方持っているからです。** 実際に調べたところ、監視用の Web UI を持たないのは **Resonate だけ**（意図的にダッシュボードを作らない方針）、CLI が主たる操作手段でないのは一部のマネージドサービスだけでした。**この問いでは製品が絞れません。**

知りたいことは、おそらく「**どうやって触るのか／学習コストはどこにかかるのか**」だと思います。それなら**3つの層に分ける**のが正しい切り方です。

| 層 | 問い | 製品間の差 |
|---|---|---|
| **① 定義** | ワークフローを**何で書くか** | **決定的。ここだけが選定を左右する** |
| **② 監視・運用 UI** | 走っているワークフローを**どう見るか** | ほぼ全製品にある。差は小さい |
| **③ CLI** | 開発・デプロイ・復旧を**どう操作するか** | ほぼ全製品にある。差は小さい |

### 3.2 一覧

| 製品 | ① 定義方法 | ② 監視・運用 UI | ③ CLI |
|---|---|---|---|
| **Temporal** | コード | **Temporal Web UI**（OSS に同梱） | `temporal` |
| **Restate** | コード | **内蔵 UI**（管理 API :9070） | `restate` |
| **DBOS** | コード（デコレータ注釈） | **Conductor**（有償／自前ホスト可）。OSS では第三者製 Argus（read-only） | `dbos workflow list / cancel / resume` |
| **Hatchet** | コード | Web UI（自前ホストに同梱、:8080） | `hatchet`（**`hatchet tui` でターミナル UI**、`worker dev`、`profile`） |
| **Inngest** | コード | Dev Server UI（ローカル）＋ Cloud ダッシュボード | `inngest-cli dev` |
| **Trigger.dev** | コード | ダッシュボード（実行のリプレイ操作あり） | `trigger.dev`（別デプロイ） |
| **Resonate** | コード | **なし（意図的にダッシュボードを作らない方針）** | `resonate`（`tree` で呼び出しグラフ、`promises get`、`serve`） |
| **Golem** | コード（WASM コンポーネント） | Golem Console | `golem` |
| **Conductor / Orkes** | **ビジュアルビルダー（ドラッグ&ドロップ）／ JSON ／ SDK** | UI（実行の可視化・再実行） | `@conductor-oss/conductor-cli` |
| **LittleHorse** | コード | **Dashboard**（有向グラフを可視化、UI から実行も可） | `lhctl` |
| **Dapr** | コード | Dapr Dashboard | `dapr` |
| **AWS Lambda Durable Functions** | コード | AWS Console / CloudWatch | AWS CLI / SAM / CDK |
| **AWS Step Functions** | **Workflow Studio（ビジュアル）／ ASL(JSON)** | AWS Console | AWS CLI |
| **Azure Durable Functions** | コード | **Durable Task Scheduler ダッシュボード** | Azure CLI / `func` / VS Code 拡張 |
| **Cloudflare Workflows** | コード | Cloudflare ダッシュボード | `wrangler` |
| **Vercel Workflows** | コード（`"use workflow"` ディレクティブ） | Vercel ダッシュボードの Workflows ページ | `vercel` |
| **Upstash Workflow** | コード | Upstash コンソール（実行の可視化） | （コンソール中心） |
| **Convex Workflow** | コード | Convex ダッシュボード | `convex` |
| **LangGraph** | コード（グラフ定義） | **LangGraph Studio** ＋ LangSmith | `langgraph` |

### 3.3 結論 — 差がつくのは「定義方法」だけ

**19製品のうち、GUI でワークフローを定義できるのは2つだけです。**

| 製品 | GUI での定義 | 裏側の表現 |
|---|---|---|
| **Orkes Conductor** | ドラッグ&ドロップのビジュアルビルダー | JSON |
| **AWS Step Functions** | Workflow Studio | ASL（JSON） |

**そしてこの2つは、どちらも出自が iPaaS / サービスオーケストレーション寄りです。**

> **入門編 §12.2 の「iPaaS は GUI、durable execution はコード」という線引きは、調査してみるとほぼ例外なく成立していました。**

裏を返すと、**durable execution を導入するとは「ワークフローをコードで書く」を受け入れることです。** GUI で組みたいなら、それは iPaaS を選ぶべき場面かもしれません（入門編 §12.2 の使い分け表）。

### 3.4 例外的に面白い2つ

- **Resonate は、意図的にダッシュボードを持ちません。** 「ワークフローが何をしているか知りたければ CLI に聞く」という方針で、`resonate tree` が呼び出しグラフ全体を、`resonate promises get` が個別ステップを出します。立ち上げるダッシュボードも、専有コンソールも無い、という設計判断です。
- **Hatchet には TUI があります。** `hatchet tui` でターミナル内にタスク・ワークフロー・ワーカーのリアルタイム可観測性が出ます。Web UI を開かずに済ませたい人向け。

---

## 4. 料金 — 3費目に分解する

### 4.1 まず、「無料か有料か」で見ると誤ります

durable execution の費用は、**独立した3費目**に分解しないと比較できません。

| 費目 | 内容 | OSS を自前ホストした場合 |
|---|---|---|
| **① ライセンス料** | ソフトウェアそのもの | **ゼロ** |
| **② マネージド利用料** | ベンダのサービス | **ゼロ**（使わないので） |
| **③ インフラ費 ＋ 運用工数** | サーバ、DB、監視、アップグレード、障害対応 | **ここに全部乗る** |

**「OSS だから無料」は ① だけを見た話です。**

具体的に言うと、Temporal を自前ホストするとライセンス料は0円ですが、**Cassandra または PostgreSQL、Elasticsearch、frontend / history / matching / worker の4コンポーネント**を運用することになります。一方 DBOS なら、追加で運用するものは**何もありません**（既存の Postgres に書くだけ）。

> **同じ「無料」でも、③ が桁で違います。** これが §1 軸1（種別）が費用に直結する理由です。**種別を見れば、③ のおおよそが分かります。**

### 4.2 一覧

| 製品 | ① 自前ホスト | ② マネージドの無料枠 | ② 最低月額 | ② 従量課金 |
|---|---|---|---|---|
| **Temporal** | 無料（MIT。Service / SDK / CLI / UI 込み） | $1,000 クレジット | **$100**（Essentials: 100万 Action、1GB Active、40GB Retained）／ Business $500 | $50 / 100万 Action（〜5M。2億超で $25）。Active $0.042/GBh、Retained $0.00105/GBh |
| **Restate** | 無料（runtime BSL、SDK MIT） | **5万 Action / 月** | 要問合せ | Action 従量 |
| **DBOS** | 無料（MIT） | ライブラリ自体が無料 | **Pro $99**（2席 / 3アプリ / 100万チェックポイント）／ Teams $499（10席 / 10アプリ / 1000万） | 追加チェックポイント $50 / 100万（Pro）、$40 / 100万（Teams） |
| **Hatchet** | 無料（MIT） | **10万タスク実行** | Team $500 ／ Scale $1,000 | **$10 / 100万タスク実行** |
| **Inngest** | 無料（SSPL→Apache 2.0） | Hobby: **5万 executions / 月**、5並列 | **Pro $99**（100万 executions、100+並列） | $25 / 25並列追加、$0.50 / 100万イベント、$3/GB span |
| **Trigger.dev** | **無料（Apache 2.0。機能制限・実行数制限なし）** | Free $0（$5 クレジット、20並列） | Hobby $10 ／ Pro $50 | 計算 $0.0000169〜0.00068 / 秒（マシン別）＋ 起動 $0.000025 / run |
| **Resonate** | 無料（Apache 2.0） | — | — | — |
| **Golem** | 無料（BUSL-1.1） | **Developer Preview 全体が無料**（SLA・データ保持保証なし） | 未定（有償 GA は2026年Q3予定） | GCU 等4次元の従量。月額基本料なし |
| **Conductor / Orkes** | 無料（Apache 2.0） | Orkes Developer Edition が無料（**本番非推奨**、SLA なし） | 要問合せ | — |
| **LittleHorse** | サーバは source-available（商用条件を要確認） | — | 商用 | — |
| **Dapr** | 無料（CNCF / Apache 2.0） | — | — | — |
| **AWS Lambda Durable Functions** | 不可 | Lambda 無料枠のみ | 月額基本料なし | **durable operation $8.00 / 100万**、書き込み **$0.25/GB**、保持 **$0.15/GB月** ＋ 通常の Lambda 課金 |
| **AWS Step Functions** | 不可 | 4,000 state transitions / 月 | 月額基本料なし | **$25 / 100万 state transitions** |
| **Azure Durable Functions** | SDK は任意基盤で可（バックエンドは Azure） | — | Consumption: 最低コミットなし | Consumption は dispatch された action 従量。Dedicated は CU 単位（1CU = 最大2,000 action/秒、50GB） |
| **Cloudflare Workflows** | 不可 | 無料プラン **3,000ステップ / 日** | Workers Paid **$5** | **2026年8月10日〜: 有料プランは月50万ステップ込み、超過 $0.80 / 10万ステップ** |
| **Vercel Workflows** | **可（Postgres 参照実装）** | Hobby: **5万イベント / 月** ＋ 書き込み 1GB（保持は不可） | Pro $20 / 人 | **$0.02 / 1,000イベント**、書き込み $0.50/GB、保持 $0.50/GB月 ＋ Functions / Queues 課金 |
| **Upstash Workflow** | 不可 | 無料枠あり | **月額基本料なし** | **$1 / 10万ステップ** |
| **Convex Workflow** | Convex 基盤前提 | Convex 無料枠 | Convex 料金に準拠 | — |
| **LangGraph** | 無料（OSS） | Platform Developer 無料 | **Plus $39 / 席**（LangSmith Plus 込み） | $0.005 / run、本番稼働 $0.0036 / 分、開発 $0.0007 / 分 |

### 4.3 課金単位の罠 — 同じワークフローでも桁が変わる

**「1回いくら」を比べても意味がありません。何を1と数えるかが製品ごとに違うからです。**

10ステップのワークフローを1回動かした場合:

| 製品 | 課金単位 | 数え方 | 概算 |
|---|---|---|---|
| **Vercel Workflows** | **イベント** | 1ステップ = 3イベント（`step_created` / `step_started` / `step_completed`）。リトライすると `step_retrying` が追加 | **約30イベント** |
| **Inngest** | **execution** | run 1 ＋ 各 step | **11 executions** |
| **Cloudflare Workflows** | **ステップ** | `step.do` はもちろん、**sleep もイベント待ちも1ステップ** | **10ステップ＋待機分** |
| **AWS Lambda Durable Functions** | **durable operation** | step / wait / checkpoint | **10前後** ＋ 通常の Lambda 課金 |
| **Temporal** | **Action** | ワークフロー開始、Activity 実行、タイマー等 | — |

### 4.4 入門編 §6.1 と直結する話

> **入門編 §6.1 の「ステップの粒度は開発者が決める」が、そのまま請求額に直結します。**

入門編ではこう書きました。

> 細かくすれば失う作業は少なくなりますが、履歴への書き込みが増えます。

**マネージドを使うなら、この「履歴への書き込みが増えます」は直接お金の話です。** クラッシュ時に失う作業量と請求額のトレードオフになります。

特に注意すべきは **Cloudflare Workflows で、2026年8月10日以降は sleep も1ステップとして課金されます。** 入門編 §5.2 で「3日待つ」がコスト0で書けると説明した設計と、コストが衝突しうる点に留意してください（計算リソースは確かに0ですが、ステップ課金は発生します）。

### 4.5 実務的な目安

- **評価・PoC 段階なら、ほぼ全製品が無料枠で足ります。** Temporal Cloud は$1,000クレジット、Restate Cloud は5万アクション/月、Inngest は5万executions/月、Hatchet は10万タスク実行、Vercel Workflows は5万イベント/月。
- **月額固定費ゼロで始められるのは Upstash Workflow（$1/10万ステップ、基本料なし）と Trigger.dev（Free $0）**、および OSS の自前ホスト全般。
- **チーム開発に入ると席課金が効いてきます。** DBOS Pro は2席まで、LangGraph Platform Plus は $39/席、Trigger.dev Pro は25席込みで以降 $20/席。**製品単価より席数で決まることがあります。**
- **自前ホストの「無料」で最も強いのは Trigger.dev です。** Apache 2.0 で、Docker + Postgres により**機能制限も実行数制限もなく**全機能を自前ホストできます。マネージド系のなかでは異例。

---

## 5. カテゴリA — 自前ホストできる汎用エンジン

ここが本命です。**言語もインフラも自分で選べる代わりに、運用は自分の仕事になります。**

### 5.1 Temporal

| 項目 | 内容 |
|---|---|
| ライセンス | サーバは **MIT**。Temporal Cloud は $100/月 〜（Action 課金） |
| SDK | **7言語**（Go, Java, Python, TypeScript, .NET, PHP, Ruby）。**Go 製で Go SDK が第一級** |
| 構成 | frontend / history / matching / worker の複数コンポーネント ＋ 外部 DB（Cassandra / PostgreSQL / MySQL）＋ Elasticsearch |
| 実行モデル | **pull**。長寿命ワーカーがタスクキューをポーリング |
| 主な制限 | ペイロード 2MB、履歴 51,200 イベント または 50MB |
| 待機 | `workflow.Sleep` で年単位。Signal / Update で人間の承認 |
| **主要機能** | Signal / Query / Update（外部からの介入）、Saga・補償、**Continue-As-New**（履歴を作り直す）、**Workflow Versioning とリプレイテスト**、Nexus |
| **公式ユースケース** | エージェント・MCP・AI パイプライン／human-in-the-loop ／Saga ／長時間ワークフロー／注文処理／耐久台帳／CI/CD ／顧客獲得 |
| **採用事例** | OpenAI、Salesforce（モノリス移行）、Twilio（自社実装の置き換え）、NVIDIA（GPU フリート管理）、Snap、Netflix、JPMorgan Chase |

**特徴**

- **成熟度が突出しています。** 2026年2月17日に50億ドル評価で3億ドル調達、クラウド上の累計アクション実行9.1兆（うち1.86兆が AI ネイティブ企業）。顧客に OpenAI、Snap、Netflix、JPMorgan Chase。
- **AI エコシステムのデファクトの耐久層になりつつあります。** OpenAI Agents SDK 連携は2026年3月23日に GA。Pydantic AI、Vercel AI SDK、Amazon Bedrock Strands との統合もあり、**既存のエージェントコードを書き換えずに Activity で包める**のが売り。
- 代償は運用です。**この一覧で最も重いインフラ**で、しかも pull 型なのでワーカーが常駐します。アイドル中も計算資源は消えません。
- 入門編 §11.1 のバージョニング問題が最も先鋭に出るのもここです。裏を返せば、**リプレイテストや versioning API の道具立てが最も揃っている**のもここ。

> **「durable execution を検討する」= まず Temporal を評価軸に置く、で当面は間違いになりません。** 他を選ぶ理由は、たいてい「Temporal ほどの重さが要らない」です。

**公式が挙げているアンチパターン**（導入時に踏みやすい順）

| アンチパターン | 何が起きるか | 対処 |
|---|---|---|
| **イベント履歴を無制限に伸ばす** | 50,000 イベントを超えると実行が強制終了する | 大きなデータは外部に置き、**Continue-As-New** で履歴を作り直す |
| **バージョニングせずに本番のコードを変える** | 決定論が壊れ、リプレイもリセットも効かなくなる | Workflow Versioning / Patching を使う（入門編 §11.1） |
| **Local Activity を冪等にしない** | 想定外のリトライで送金が二重になるなどの副作用 | 冪等キーを必ず持たせ、既定では通常の Activity を使う |
| **SDK を過剰にラップする** | アップグレードが困難になり、機能が隠れる | ラッパは薄く保ち、SDK に直接触れる余地を残す |
| **Signal / Query / Update を使わない** | 外部との連携を自前で作り込むことになる | 組み込みのメッセージパッシングを先に調べる |

**向く**: 長時間・分岐の多い基幹処理、多言語チーム、AI エージェントの耐久層
**向かない**: 運用要員を割けない小規模チーム、ミリ秒応答が要る同期パス

### 5.2 Restate

| 項目 | 内容 |
|---|---|
| ライセンス | runtime は **BSL 1.1**（期限後に Apache 2.0）、**SDK は MIT** |
| SDK | TypeScript, Java/Kotlin, Python, Go, Rust |
| 構成 | **単一の自己完結バイナリ**（Rust 製）。別 DB も Elasticsearch も不要。HA は複数インスタンス＋オブジェクトストレージへのスナップショット |
| 実行モデル | **push**。アプリ側はただのステートレス HTTP サービス（serverless 関数でも可） |
| 独自機能 | Virtual Object（エンティティ単位の K/V 状態）、exactly-once メッセージング、耐久タイマー／プロミス、OpenTelemetry 自動生成 |
| **公式ユースケース** | 耐久ワークフロー（オンボーディング／プロビジョニング／返金／承認）／マイクロサービスのオーケストレーション／非同期タスク・バックグラウンドジョブ／**イベントの exactly-once 処理**／AI エージェント・LLM ワークフロー |

**特徴**

- **運用フットプリントが Temporal と桁で違います。** バイナリ1つ。これが最大の訴求。
- `ctx.run()` ごとにジャーナルするので、**アプリコード側に冪等キーを書かなくてよい**（入門編 §7.3 の「冪等性は設計事項」の負担が一部エンジン側に寄る）。
- 設計思想の差がよく引用されます — **「Temporal は個々のワークフローを durable にする。Restate はシステム全体を durable にする」**。ワークフローだけでなく、サービス間通信そのものに exactly-once を持ち込む発想です。
- push 型なので **serverless にそのまま載ります。** ワーカー常駐が要らない。
- **Virtual Object が実務的に効きます。** キー単位の状態を保持し、同一キーへの並行更新を直列化するので、**別途ロックサービスを立てる必要がなくなります。** 「実行・通信・状態」という分散システムが壊れる3つの次元すべてに耐久性を持ち込む、という整理を公式が掲げています。
- 成熟度と実績は Temporal に劣ります。BSL も要確認事項です。

**向く**: マイクロサービス間の整合性・冪等性、イベントの exactly-once 処理、serverless、運用を軽くしたい
**向かない**: BSL が通らない組織、大規模な本番実績が選定要件になる場面

### 5.3 DBOS Transact

| 項目 | 内容 |
|---|---|
| ライセンス | **MIT**。DBOS Pro（$99/月〜、Conductor UI 等）が有償 |
| SDK | Python, TypeScript, Go, Java, Kotlin |
| 構成 | **サーバなし。アプリ内のライブラリ。** 状態は既存の Postgres |
| 実行モデル | in-process |
| 性能 | **1ステップのチェックポイント = Postgres への1書き込み（1〜2ms）** |
| **主要機能** | デコレータ注釈、**耐久キュー**（並行数制御つき）、**cron スケジューラ内蔵**、workflow ID = 冪等キー、Conductor UI、MCP サーバ統合（コーディングエージェントから監視できる） |
| **公式ユースケース** | AI エージェント（Pydantic AI / LlamaIndex / OpenAI Agents SDK）／耐久ワークフロー／耐久キュー／**cron ジョブ基盤**／データパイプライン／human-in-the-loop |
| **採用事例** | Yutori（大規模エージェント）、Dosu.dev、Supabase、Ontologize（**Google Workflows / Temporal / Airflow / Celery を評価した上で採用**） |

**特徴**

- **新規インフラがゼロ。** 「Postgres はもう信用して使っている」チームにとって、追加で運用するものが何もない。デコレータで既存関数に注釈を付けるだけ。
- **レイテンシに効きます。** 外部オーケストレータへの往復が無いので、対話的・低レイテンシなワークフローに向く。Temporal 系がミリ秒〜十数ミリ秒の往復を払うところを、ローカル DB への1書き込みで済ませる。
- **workflow ID がそのまま冪等キーになる**設計。
- Pydantic AI が公式に統合をドキュメント化しています（Temporal、Prefect と並んで）。
- **cron スケジューラと耐久キューを内蔵している**ので、Celery + Beat のような構成をまるごと置き換えられます。「durable execution が欲しい」より「ジョブ基盤を1つ減らしたい」という動機で選ばれることがあります。
- 弱点は、**ライブラリなのでプロセス外の監督者がいない**こと。アプリが全滅したら誰も再開させません（別プロセスの recovery 機構は用意されているが、Temporal のような独立クラスタとは前提が違う）。また Postgres がそのままスケール上限になります。

**向く**: 既に Postgres 中心、対話的・低レイテンシな処理、AI エージェント、cron / キュー基盤の統合
**向かない**: Postgres のスケール上限を超える規模、プロセス外の監督者が必須の構成

### 5.4 Hatchet

| 項目 | 内容 |
|---|---|
| ライセンス | **MIT** |
| SDK | Python, TypeScript, Go, Ruby |
| 構成 | **Postgres がタスクランタイムと可観測性の両方の耐久層** |
| 位置づけ | 「タスクキュー」と「durable execution」の中間 |
| **主要機能** | 優先度、**静的・動的レート制限**、concurrency ポリシーによる fair scheduling、ワーカースロット、ラベル／affinity ルーティング、DAG、耐久 sleep とイベント待ち、cron、リプレイ、OTel / Prometheus |
| **公式ユースケース** | AI エージェント／durable execution ／**大量並列ワークロード** |

**特徴**

- 出自がタスクキューなので、**キュー側の機能が厚い**。優先度、レート制限（動的レート制限を含む）、concurrency ポリシーによる fair scheduling、ワーカーラベルによるルーティング、マルチテナンシ。
- durable task と DAG の両方を持ち、**耐久 sleep とイベント待ちを組み合わせた複雑な pause/resume 条件**が書ける。
- 入門編 §10 の指摘どおり **通常タスクは at-least-once** なので、冪等性は自分の仕事です。
- Postgres だけで動くので自前ホストが容易。**`hatchet tui` によるターミナル UI** があるのも地味に効きます。
- **「durable execution が欲しい」より「Celery / Sidekiq を卒業したい」という動機で刺さります。** 外部 API のレート制限に合わせた動的スロットリングや、テナント間の公平性といった、キュー運用で実際に困る部分が最初から入っています。

**向く**: バックグラウンドジョブ基盤、大量並列、マルチテナントの公平性、外部 API のレート制限対応
**向かない**: 厳密な決定論的リプレイ保証が要る場面（通常タスクは at-least-once）

### 5.5 Inngest

| 項目 | 内容 |
|---|---|
| ライセンス | **fair source**（SSPL、3年後に自動で Apache 2.0） |
| SDK | TypeScript, Python, Go |
| 構成 | 自前ホスト可（1コマンド。SQLite 同梱、本番は外部 Redis + Postgres） |
| 実行モデル | **push / イベント駆動** |
| 主な制限 | ステップ出力 4 MiB、1,000ステップ、sleep は無料30日 / Pro 366日 |
| **主要機能** | イベント駆動（**CEL 式でマッチング**）、`step.run` / `step.sleep` / `waitForEvent`、**flow control**（レート制限・テナント別 concurrency・throttle）、ステップ単位トレース、リプレイ |
| **公式ユースケース** | AI エージェント（中断して入力を待ち、再開）／ワークフロー／バックグラウンドジョブ／スケジュールタスク |

**特徴**

- **イベント駆動が第一級。** Webhook やキューを起点にする設計で、`step.run()` / `step.sleep()` / `waitForEvent()` が基本語彙。CEL 式でイベントをマッチングできる。
- **アプリの HTTP ルートの中でステップが走る**ので、Next.js on Vercel のような構成にそのまま載ります。
- **sleep 中は concurrency 枠を消費しません。** アイドルが安い。
- **テナント別の concurrency 制御が第一級**なのは、SaaS を作るときに効きます。「特定の顧客が全体を詰まらせない」を設定で書ける。
- 注意: 「Inngest は自前ホストできない」と書いた比較記事が複数流通していますが、**Inngest 1.0 で自前ホストは公式にサポートされています。** 二次情報のライセンス／自前ホスト可否は当てになりません。

**向く**: Webhook / イベント起点、Next.js on Vercel などの serverless、テナント別の流量制御
**向かない**: 366日を超える待機、ステップ出力 4 MiB 超

### 5.6 Resonate

| 項目 | 内容 |
|---|---|
| ライセンス | **Apache 2.0** |
| SDK | TypeScript, Python, Go, Rust, Java |
| 構成 | **単一バイナリ**。デフォルト SQLite、Postgres も可 |
| モデル | **Distributed Async Await** — 言語・トランスポート非依存のプロトコル |
| UI | **なし。CLI 優先の方針**（§3.4） |
| **主要機能** | **durable promise**（即座に書き込まれ、耐久的に解決される約束）、`resonate tree` で呼び出しグラフ全体を表示、単一バイナリ、既存の Postgres と既存トランスポート（HTTP / NATS）を利用 |
| **公式ユースケース** | 長時間の非同期ワークフロー・バックグラウンドジョブ／注文処理・決済／fan-out・fan-in ／プロセスクラッシュをまたぐ再開 |

**特徴**

- **「普通の async/await がそのまま分散・耐久になる」**という一点に振り切った設計。新しいワークフロー DSL を覚えさせない方向。公式は **「Async Await *が* ワークフローである」**という言い方をしています。
- サーバはワーカーの **supervisor 兼 orchestrator** として振る舞う。
- **プロプライエタリな製品ではなく Apache 2.0 のオープンプロトコル**（distributed-async-await.io 仕様）として実装されている点を前面に出しています。ロックインを避けたいチーム向けの位置づけ。
- 規模と実績は小さいので、本番採用は評価前提で。

**向く**: 新しい DSL を覚えたくない、ベンダロックインを避けたい、小さく始めたい
**向かない**: 大規模実績やエンタープライズサポートが選定要件になる場面

### 5.7 Golem

| 項目 | 内容 |
|---|---|
| ライセンス | **BUSL-1.1**（Apache 2 へ移行予定） |
| 実行単位 | **WebAssembly コンポーネント**（専用実行エンジン） |
| 現況 | Cloud は Developer Preview（**無料、SLA・データ保持保証なし**）、有償 GA は2026年Q3予定 |
| **主要機能** | **シリアライズ不要の自動状態永続化**、exactly-once（ツール呼び出しの重複を防ぐ）、WASM サンドボックス隔離、OTel、**MCP サーバを自動提供**、モデル非依存、耐久 cron、human-in-the-loop 用 Webhook、**デプロイをまたいで生き残るストリーミング**、レート制限・quota、完全な監査ログとリプレイ |
| **公式ユースケース** | カスタマーサポート自動化／コーディング・開発エージェント／社内データ copilot ／音声・チャットエージェント |
| 言語 | TypeScript, Rust, Scala, MoonBit（いずれも同一の動作保証） |

**特徴**

- **この一覧で唯一、開発者が step で包まなくてよい方式です。** ランタイムが WASM の実行そのものを耐久化するので、入門編 §5.2 の禁止事項リストが原理的に不要になる。**「透過的な durable execution」**を主張しているのはここだけ。
- 2025年5月以降、**「durable agent runtime」へ明確にリポジション**。エージェントごとに WASM サンドボックス＋専用ファイルシステム＋専用 SQLite を持たせ、ミリ秒起動。
- ツール／ネットワーク／ファイルへのアクセスは**明示的な grant** で、プロンプトインジェクションで権限昇格できず、quota はランタイムが強制し、**すべての grant がジャーナルされる**。durable execution とサンドボックス隔離を同じ機構で解こうとしている。
- 代償は、**WASM に載せられるものしか動かない**こと。既存の Python 資産をそのまま持ち込むような使い方には向きません。

**向く**: マルチテナントのエージェント実行基盤、隔離と耐久を同時に解きたい場面
**向かない**: 既存の Python / Java 資産をそのまま載せたい、本番 SLA が今すぐ必要（Cloud は Preview）

### 5.8 Conductor（OSS / Orkes）

| 項目 | 内容 |
|---|---|
| ライセンス | **Apache 2.0**（Orkes が主要メンテナ、SaaS も提供） |
| SDK | Java, Python, Go, JavaScript, C#, Ruby, Rust ＋ HTTP が話せれば任意言語 |
| 永続化 | **Redis / PostgreSQL / MySQL / Cassandra / SQLite から選択** |
| 定義 | **ビジュアルビルダー／ JSON ／ SDK**（→ §3.3） |
| **主要機能** | 全履歴の無期限保持、**任意タスクからの再実行**、LLM タスク（14+ プロバイダ）、MCP ツール呼び出し、**ベクタ DB 連携（RAG）**、human-in-the-loop 承認、エージェントが実行時に生成する動的ワークフロー、イベントバス連携 |
| **公式ユースケース** | ワークフローオーケストレーション／マイクロサービスオーケストレーション／AI エージェント／イベント駆動システム／スケジュール実行 |
| **採用事例** | Netflix、Tesla、LinkedIn、JP Morgan（月10億ワークフロー級） |

**特徴**

- Netflix 発。**タスクとワーカーを明示的に分離する**古典的オーケストレーション。実績規模は月10億ワークフロー級。
- **実行履歴を無期限に保持し、「最初からやり直す／任意のタスクから再実行する／失敗したステップだけリトライする」を数か月後でも実行できる。** 入門編 §7.3 の「補償処理を書けるようになる」が、運用 UI のレベルで提供されている。
- AI 向けが手厚い。LLM タスク（chat/text completion）、MCP のツール呼び出し、human-in-the-loop 承認、**エージェントが実行時に生成する動的ワークフロー**。プロバイダは Anthropic / OpenAI / Azure OpenAI / Gemini / Bedrock / Mistral / Cohere / HuggingFace / Ollama など14以上をネイティブ統合。
- **19製品で唯一（Step Functions と並んで）GUI でワークフローを定義できます。** ただしコードではなく JSON 定義なので、**入門編 §12.2 の「git diff でレビューできる」という利点は半減**します。iPaaS と durable execution の中間に位置すると考えるのが正確です。
- **ワーカーは HTTP でポーリングと結果更新ができれば任意の言語で書けます。** 公式 SDK が7言語ある上に、この抜け道があるので、レガシー言語が混ざった組織でも詰みにくい。

**向く**: 非エンジニアも定義に関わる、運用者が UI で復旧操作をする、ポリグロット、RAG パイプライン
**向かない**: git diff でのコードレビューを重視する場合（JSON 定義のため）

### 5.9 LittleHorse

| 項目 | 内容 |
|---|---|
| ライセンス | **サーバは source-available**（AGPLv3 / SSPL — 版によって変遷しているため要確認）、**SDK は Apache 2.0** |
| SDK | Java, Go, Python, C#/.NET |
| 構成 | Kernel は Java 製、**Apache Kafka を durable write-ahead log として使用** |
| **主要機能** | **User Task（人間の作業）が組み込み**、リアルタイム可観測性、リトライによる自動復旧、Dashboard から実行可、`lhctl`、**モジュラー構成**（Kernel / Connect / identity を個別に採用できる） |
| **公式ユースケース** | マイクロサービス管理／イベント駆動アーキテクチャ／エージェント型ワークフロー自動化／SaaS 連携／GPU オーケストレーション・顧客オンボーディング・フリート管理 |

**特徴**

- **Kafka を既に運用している組織にとっては、耐久層が既存資産で済む**という一点が効きます。
- User Task（人間の作業）が組み込み。Dashboard は有向グラフを可視化し、UI から実行も可能。CLI は `lhctl`。
- 創業者は **「分散した JVM」** という比喩でこの製品を説明しています。数十のサービスと人間をまとめて1つの実行環境として扱う、という発想。
- ライセンスがサーバ側で OSI 準拠でない点は、選定時の明確な検討事項です。

**向く**: Kafka を既に運用している、人間の作業が処理に組み込まれている
**向かない**: OSI 準拠ライセンスが必須の組織、Kafka を持ち込みたくない場合

### 5.10 Dapr Workflows / Dapr Agents

| 項目 | 内容 |
|---|---|
| ライセンス | **Apache 2.0（CNCF プロジェクト）** |
| SDK | Python, JavaScript, .NET, Java, Go |
| 基盤 | **Durable Task Framework**（Azure Durable Functions と同系統）ベース |
| 構成 | **サイドカー**。state store は差し替え可能 |
| **主要機能** | **6つの公式ワークフローパターン**（下表）、state store の差し替え、Pub/Sub によるマルチエージェント協調、リトライ・タイムアウト・サーキットブレーカ |

**公式が定義する6パターン** — 「何が書けるのか」が最も具体的に分かる資料です。

| パターン | 用途 |
|---|---|
| **Task Chaining** | 前段の出力を次段の入力にする逐次処理。データ変換パイプライン |
| **Fan-out / Fan-in** | 並列実行して結果を集約。並行数の制御つき |
| **Async HTTP APIs** | 長時間処理を HTTP で公開。即座に instance ID を返しポーリングさせる |
| **Monitor** | `continue-as-new` による永続ワークフロー。定期的な健全性チェックと、状態に応じた間隔調整 |
| **External System Interaction** | **人間の承認や決済確認を待つ。** タイムアウトで自動的に補償へ |
| **Compensation (Saga)** | 失敗時に補償処理を逆順で実行。分散トランザクションの代替 |

**特徴**

- **Kubernetes / マイクロサービス基盤に Dapr を既に入れているなら、追加コストがほぼゼロ**で durable workflow が手に入る。
- **Dapr Agents v1.0 が本番対応**として出ています。LLM とツールとのやり取りを毎回 durable state store に永続化し、エージェント再起動後も継続。
- Pub/Sub による**イベント駆動のマルチエージェント協調**が組み込み。durable execution とメッセージングが同じ基盤に乗るのが Dapr らしい点。

**向く**: K8s に Dapr 導入済み、ポリグロットなマイクロサービス、マルチエージェント協調
**向かない**: Dapr を入れていない環境（サイドカー導入コストが先に来る）

---

## 6. カテゴリB — マネージド / プラットフォーム組み込み

**運用が消える代わりに、プラットフォームに乗ります。**

### 6.1 比較表

| | AWS Lambda Durable Functions | AWS Step Functions | Azure Durable Functions | Cloudflare Workflows | Vercel Workflows | Upstash Workflow |
|---|---|---|---|---|---|---|
| 定義の書き方 | **コード** | **GUI or JSON（ASL）** | **コード** | **コード** | **コード**（`"use workflow"`） | **コード** |
| 言語 | Node.js / Python / Java / .NET | 非依存 | .NET / Python / Java / JS / PowerShell | TypeScript（Python ベータ） | TypeScript / Python（ベータ） | TypeScript / Python |
| 最長待機 | **1年** | Standard 1年 / Express 5分 | 実質無制限 | 365日 | **無制限** | 1年 |
| ステップ上限 | 3,000 durable operations | 25,000 履歴イベント | — | 1,024（無料）/ 25,000（有料） | 10,000（イベントは25,000） | 1,000（引き上げ可） |
| ペイロード | 耐久状態 100MB | 256 KiB / state | — | 1 MiB / step | 50MB（1run 合計 2GB） | 1〜50MB |
| 自前ホスト | 不可 | 不可 | Durable Task SDK は任意基盤で可 | 不可 | **可**（Postgres 実装） | 不可 |
| **主要機能** | step / wait / callback / invoke | **Workflow Studio**、220+ サービス統合、**Distributed Map** | Durable Entities、eternal orchestration、**任意基盤で動く SDK** | ローカル状態（DB 不要）、**Dynamic Workflows**、Agents SDK 統合 | **Streams**、hooks、マルチリージョン、Skew Protection | **DLQ から失敗ステップだけ再開**、flow control |
| **有効な場面** | ロジックが Lambda に閉じている | **数百万件の大規模並列 ETL / ML / ログ解析** | .NET / Azure 中心、Azure 外へも持ち出したい | 既に Workers 上、UGC の後処理、**テナント別ワークフロー** | Next.js アプリ、**LLM 応答のストリーミング** | **月額固定費ゼロで始めたい**、EC の注文処理・決済リトライ |
| **向かない場面** | **大規模 fan-out**、3,000 operation 超 | コードで書きたい、256 KiB 超 | Azure 外が主戦場 | **長時間 sleep の多用**（ステップ課金） | 25,000イベント / 10,000ステップ超 | 自前ホストが要件 |

### 6.2 AWS Lambda Durable Functions

2025年 re:Invent で発表。**入門編 §4.3 で引用されている「step で包む方式」がこれです。**

- プリミティブは4つ: **step**（チェックポイント＋自動リトライ）、**wait**（最大1年、その間の計算課金なし）、**callback**（外部システムからの応答を待つ）、**invoke**（他 Lambda を呼んで結果をチェックポイント）
- **決定論契約が明示的**: 「毎回の invocation で、同じ順序・同じ名前・同じ型の durable operation 呼び出しを行わなければならない」。`Date.now()`、UUID 生成、`Math.random()`、HTTP、DB、AWS SDK 呼び出しはハンドラ直下では禁止。
- **最も嫌らしい落とし穴**: step の外での状態変更は**黙って失敗する**。初回は正しく見え、リプレイ時にその変更だけがリセットされる。入門編 §4.4 の「黙って壊れる」の実例。
- ランタイム: Node.js 22/24、Python 3.13/3.14、Java 17/21/25、.NET。OCI コンテナも可。Java は2026年4月 GA、.NET は2026年7月 GA。2026年6月時点で約31リージョン。
- 実行タイムアウトは60秒〜1年（既定24時間）。ただし**個々の invocation は Lambda の15分制限のまま**。
- 課金は**3次元**（durable operation $8.00/100万、書き込み $0.25/GB、保持 $0.15/GB月）＋ 通常の Lambda 課金。
- **Step Functions の代替ではありません。** 大規模な fan-out（数百万件）は Distributed Map の領分で、durable functions は苦手。AWS 自身が「補完関係」と位置づけており、ハイブリッド構成（Step Functions がサービス間グラフ、durable functions がコード中心の区間）が推奨。

**向く**: ロジックが Lambda に閉じている、AWS 全振り、コード中心で書きたい
**向かない**: 大規模 fan-out、3,000 durable operation 超、非エンジニアと可視化を共有したい

### 6.3 AWS Step Functions

**durable execution の文脈では「Lambda Durable Functions の兄弟」として理解するのが正確です。** 同じ問題を、逆側から解いています。

| 項目 | 内容 |
|---|---|
| 定義 | **Workflow Studio（ビジュアル）または ASL（JSON）** |
| 統合 | **220+ の AWS サービス、14,000+ の API アクション** |
| 種類 | **Standard**（最長1年、25,000履歴イベント）／ **Express**（最長5分、**毎秒10万起動**） |
| 目玉機能 | **Distributed Map** |
| 人間の承認 | task token（Standard のみ） |

**Distributed Map が決定的な差です。**

S3 上のオブジェクト一覧・CSV・JSON・JSONL・S3 インベントリレポートを読み、**最大10,000の並列子実行を起動**します。通常のペイロード制限とイベント履歴制限を超えて大規模データを処理できる、専用モードです。

| 用途 | 内容 |
|---|---|
| ETL パイプライン | 大規模データセットの抽出・変換・ロード |
| 機械学習 | 学習用データセットの前処理 |
| ログ解析 | 大量のログからの洞察抽出 |
| データ検証 | 定義済みルールに対する大規模検証 |

**入門編の観点から見ると:** Lambda Durable Functions は「コードで書く durable execution」、Step Functions は「宣言で書くオーケストレーション」です。AWS 自身が補完関係と位置づけており、**Step Functions がサービス間グラフを持ち、その中のコード中心の区間を durable functions が担う**ハイブリッドが推奨されています。

**Standard と Express の使い分け**は明快です。**1回の実行が5分を超えるなら Standard**（ETL、人間の承認待ち）、**5分未満で実行回数が膨大なら Express**。

**向く**: 大規模並列 ETL / ML 前処理 / ログ解析、AWS サービス間連携、ステークホルダーとの視覚的共有
**向かない**: コードでロジックを書きたい、256 KiB を超えるペイロード

### 6.4 Azure Durable Functions / Durable Task SDKs

- **orchestrator / activity の分離**は Temporal の Workflow / Activity と同型。イベントソーシング＋決定論的リプレイ。Temporal の系譜そのもの（入門編でも触れたとおり、Temporal の作者は Azure Durable Functions の設計にも関わっている）。
- 注目すべきは **Durable Task SDKs** の分離です。**ポータブルな OSS ライブラリとして切り出され、Azure Container Apps / Kubernetes / VM など任意の基盤で動く。** Azure Functions に縛られなくなった。
- **Durable Task Scheduler** が推奨の状態プロバイダ。専用ストレージアカウント不要でレイテンシも改善。Dedicated SKU が2025年11月 GA、**Consumption SKU が2026年3月 GA**。
- 課金は Consumption が **dispatch された action 従量（最低コミット・アイドルコストなし、最大500 action/秒、保持30日）**、Dedicated が **CU 単位（1CU = 最大2,000 action/秒・50GB、1デプロイ最大3CU）**。
- 独自機能として **Durable Entities**（状態を持つ小さなアクター）があり、Restate の Virtual Object に近い役割を果たします。

**向く**: .NET / Azure 中心、Azure Functions の既存資産、ACA / K8s へ持ち出したい
**向かない**: Azure 外を主戦場にする場合（バックエンドは結局 Azure）

### 6.5 Cloudflare Workflows

- `step.do` / `step.sleep` / `step.sleepUntil` / `step.waitForEvent`。既定リトライ5回（指数バックオフ）。
- **2026年5月に Dynamic Workflows** を追加。ワークフロー定義を事前デプロイせず Dynamic Worker 内でオンデマンドにロードする — **テナントごとに異なるワークフローを動かす**用途に効きます。
- **Agents SDK と第一級で統合**されている唯一のエンジン。
- **課金が2026年8月10日以降にステップ単位へ変更**（有料プランは月50万ステップ込み、超過分 $0.80 / 10万ステップ。無料は日3,000ステップ）。**sleep もイベント待ちも1ステップとして数えられます**（→ §4.4）。
- **インスタンスごとにローカル状態を持つので、DB を別途用意する必要がありません。**
- 公式ユースケースは **AI エージェント**（コードレビュー、コンテキストの圧縮、データ処理）、**非同期タスク**（ライフサイクルメール、課金ジョブ）、**ユーザー生成コンテンツの後処理**（推論の実行、アップロードの検証）。
- 代償は Cloudflare への集中。ステップ出力 1 MiB は他より小さい。

**向く**: 既に Workers 上にいる、UGC の後処理、テナントごとに異なるワークフロー
**向かない**: 長時間 sleep を多用する設計、1 MiB 超のステップ出力

### 6.6 Vercel Workflows / Workflow DevKit

**プログラミングモデルが異質で、注目に値します。**

```ts
"use workflow"   // ← このトップレベル関数が段取り
"use step"       // ← この単位が step。リトライ・永続化・可観測性が自動で付く
```

- **外部オーケストレータが存在しません。** 調整はすべてアプリコード内で完結し、Fluid compute の上で走る。「step が実際に走っている計算分だけ払う」モデル。
- **「Worlds」というアダプタ機構**で、イベントログ・計算・キューの3要素を差し替えられる。マネージド（Vercel）／自前ホスト（Postgres 参照実装）／組み込み、の3形態。コミュニティが MongoDB、Redis、Turso、Cloudflare 向けアダプタを作っている。
- SDK は OSS。TypeScript が本命、Python はベータ（デコレータ `@wf.workflow` / `@wf.step`）。
- **実行時間と sleep に上限がありません。** 一方でイベント数（25,000/run）とステップ数（10,000/run）に上限があり、2,000イベントまたは1GB を超えるとリプレイが遅くなるため、子ワークフローへの分割が推奨されています。
- **AI 用途に効く機能が揃っています。** `Streams` でワークフローの内外にデータをストリームでき（LLM 応答をユーザーに流せる）、`hooks` で外部イベントを待てます。公式ガイドにも「ステートフルな Slack bot」「Claude を使った管理エージェント」が並びます。
- **マルチリージョンの扱いが独特です。** run は開始時に1つのリージョンに固定され、状態・キュー・ストリームがそこに留まります。ホットパスでのリージョン跨ぎを避け、リージョン障害の影響範囲を封じ込める設計。
- **Skew Protection** があり、デプロイ中のバージョン不整合から守られます。入門編 §11.1 のバージョニング問題に対する、この製品なりの回答です。
- **ランタイムに縛られない durable execution SDK** という立ち位置は、この一覧では他にありません。ただし新しく、実績はこれから。

**向く**: Next.js / Vercel 上のアプリ、LLM 応答のストリーミング、ステートフルな bot
**向かない**: 1 run が25,000イベント / 10,000ステップを超える場合

### 6.7 Upstash Workflow

- QStash の上に構築。**ステップごとに自分のアプリへ HTTP 呼び出しが飛び、結果を Upstash が保持する。** ワーカーもサーバも Postgres も要らない。
- **30日 sleep が計算0で、課金上は1ステップ。** アイドルの安さは随一。$1 / 10万ステップ、月額固定費なし。
- 失敗が尽きると **DLQ に入り、「失敗したステップから再開（完了分は結果を保持）／最初からやり直し／失敗コールバックの再発火」を選べる。**
- `context.waitForEvent` / `context.notify`。イベントを待機点到達前に保存してレースを防ぐ設計。
- **公式ユースケースが具体的で、そのままテンプレートとして使えます。** エージェント（カスタムツール付き LLM）、AI データ処理（大規模データを分割して処理しレポート生成）、認可 Webhook（ユーザー作成・トライアル管理・リマインドメール）、顧客オンボーディング、EC の注文処理（在庫確認→決済→出荷）、画像処理の並列化、決済リトライ（遅延つき再試行→通知→アカウント停止）。
- 自前ホスト不可。Upstash 前提。

**向く**: 月額固定費ゼロで始めたい、EC の注文処理・決済リトライ、大規模データの分割処理
**向かない**: 自前ホストが要件の場合

### 6.8 Trigger.dev

- **Apache 2.0。Docker + Postgres で全機能を自前ホストでき、機能制限も実行数制限もない。** OSI 準拠ライセンスのマネージド系としては強い（→ §4.5）。
- TypeScript ファースト。タスクは Trigger.dev のマシンにデプロイされ、アプリからトリガーする（**アプリと別デプロイ**になる点は好みが分かれる）。
- プラットフォームのタイムアウトが無い。Cloud では5秒超の待機をチェックポイントして計算課金を避ける。
- **Realtime API が差別化点です。** ワークフロー内から LLM の応答をフロントエンドへストリームでき、React hooks でタスクの状態をリアルタイム表示できます。**「処理中です」を出すだけでなく、途中経過を見せたい UI** に向きます。
- 公式ユースケースは AI エージェント、**メディア処理（動画・音声）**、ブラウザ自動化（Puppeteer）、メールシーケンス、タイムアウトのない cron、セマンティック検索。

**向く**: メディア処理、ブラウザ自動化、LLM 応答をユーザーに流す UI、長時間処理
**向かない**: アプリと同一デプロイにしたい場合（別デプロイになる）

---

## 7. カテゴリC — エージェントフレームワーク側の耐久機構

**入門編 §9.6 の「もはやオプションではなくベースライン要件」がここに現れています。**

| フレームワーク | 耐久化の方式 | 主要機能 | 有効な場面 | 向かない場面 |
|---|---|---|---|---|
| **LangGraph** | **DB チェックポイント**（PostgresSaver / RedisSaver / DynamoDBSaver 等） | superstep ごとの state 保存、thread 単位の管理、**time travel**、HITL interrupt | エージェントのグラフ構造を明示的に設計したい、状態の巻き戻し、規制業界での実績重視 | **単体では完遂を保証しない**（→ §7.1） |
| **Pydantic AI** | **外部エンジンに委譲** | Temporal / DBOS / Prefect との統合を公式ドキュメント化 | 型付きの出力を重視、耐久層を後から選びたい | 耐久層を自前で持ちたい場合 |
| **OpenAI Agents SDK** | **外部エンジンに委譲** | Temporal 連携が2026年3月23日 GA。サンドボックス統合（Modal / Daytona / Docker / E2B） | OpenAI 中心の構成、コード実行を伴うエージェント | 耐久層を単体で完結させたい場合 |
| **Dapr Agents** | **Dapr Workflows** | v1.0 で本番対応。Pub/Sub でマルチエージェント協調、リトライ・タイムアウト・サーキットブレーカ | K8s に Dapr 導入済み、**複数エージェントの協調** | Dapr を入れていない環境 |

### 7.1 LangGraph の checkpointer に関する重要な注意

checkpointer を設定すれば「durable execution が有効」と説明されますが、**入門編の定義に照らすと、これは半分です。**

> **checkpointer は状態を保存する。しかし障害を検知して再開させる主体がいない。**

プロセスがクラッシュしても、それに気づいて再開をキックする supervisor / watchdog が存在しません。**「どこまで進んだか」は残るが、「必ず最後まで到達させる」（入門編 §1 の目的1）は満たさない。**

これは LangGraph の欠陥というより**層が違う**話です。実務では **LangGraph を Temporal / Dapr の Activity の中で走らせる**構成が定番になっています。入門編 §8.3 のループを人間が書き、その中の1ステップとして LangGraph のグラフを回すイメージです。

同じ注意は CrewAI や Google ADK など、checkpoint だけを持つフレームワーク全般に当てはまります。

---

## 8. カテゴリD — 隣接だが別物（混同注意）

**ここを混ぜると選定が壊れます。**

| ツール | 実際のカテゴリ | durable execution と何が違うか |
|---|---|---|
| **Airflow / Prefect / Dagster** | データオーケストレーション | スケジュール実行される DAG が主。**アプリケーションのビジネスロジックを耐久化する道具ではない**（Prefect は近づいてきてはいる） |
| **Argo Workflows** | Kubernetes 上のコンテナ DAG | ステップ = コンテナ。**関数呼び出しの粒度ではない**。K8s 前提 |
| **Camunda / Zeebe** | BPMN プロセスオーケストレーション | durable execution 的な性質は持つが、**主眼は業務プロセスの可視化と人間タスク**。BPMN のモデリングが中心 |
| **n8n / Windmill / Activepieces / Kestra** | 自動化 / iPaaS 寄り | コネクタと GUI（または YAML）が主。→ 入門編 §12.2 の比較表がそのまま当てはまる |
| **LLM レスポンスキャッシュ** | キャッシュ | 入門編 §1「よくある誤解 ①」の表を参照。**「まだ実行していないステップ」に関知しない** |
| **OpenTelemetry / トレーシング** | 可観測性 | 入門編 §9.5。**説明のためのもので、復旧のためのものではない** |

Windmill（Python / TypeScript / Go / Bash をワークフロー化、Git バージョニング、OSS 自前ホスト可）や Kestra（YAML 宣言、イベント駆動）は良いツールですが、**解いている問題が違います。** 「エージェントの47番目のステップで落ちたときに48番目から再開する」ためのものではありません。

---

## 9. 実装方式で並べ直す

入門編 §10 の2分類を、調査対象全体に適用したものです。

### ジャーナルベースのリプレイ

**決定論制約あり（入門編 §5.2 の禁止事項が適用される）**

Temporal、Restate、Resonate、AWS Lambda Durable Functions、Azure Durable Functions、Cloudflare Workflows、Vercel Workflows、Inngest、Upstash Workflow、Dapr Workflows

### データベースへのチェックポイント

**決定論制約が緩い / 無い**

DBOS、LangGraph、Hatchet、Convex、Trigger.dev

### 透過的スナップショット

**開発者が step で包む必要がない**

Golem

---

## 10. 選び方

決め手になりやすい順に並べます。

| 状況 | 第一候補 | 理由 |
|---|---|---|
| **Postgres は既にある。新しいインフラを増やしたくない** | **DBOS** | サーバ不要のライブラリ。1〜2ms のチェックポイント |
| **成熟度と実績を最優先。多言語チーム** | **Temporal** | 7言語、圧倒的な運用実績、AI 連携の中心 |
| **運用は軽くしたいが、Temporal 級の機能は欲しい** | **Restate** | 単一バイナリ。push 型で serverless 可 |
| **serverless / Next.js にそのまま載せたい** | **Inngest** / **Upstash Workflow** / **Vercel Workflows** | アプリの HTTP ルート内でステップが走る |
| **既に AWS / Azure / Cloudflare に全振りしている** | 各プラットフォームの durable functions | 運用がゼロ。ただしロックイン |
| **バックグラウンドジョブが主で、キュー機能が要る** | **Hatchet** | レート制限・優先度・fair scheduling が厚い |
| **Kubernetes に Dapr が入っている** | **Dapr Workflows / Agents** | 追加コストほぼゼロ |
| **Kafka を運用している** | **LittleHorse** | 耐久層が既存資産 |
| **人間の承認・業務プロセスの可視化が主目的** | **Conductor** / **Camunda** | UI と human task が第一級。**GUI で定義できる数少ない選択肢** |
| **エージェントのサンドボックス隔離も同時に解きたい** | **Golem** | grant がジャーナルされる WASM サンドボックス |
| **既存の LangGraph 資産がある** | **LangGraph ＋ 外部エンジン** | checkpointer だけでは §7.1 の穴が残る |
| **月額固定費をゼロにしたい** | **Upstash Workflow** / **Trigger.dev 自前ホスト** | 前者は $1/10万ステップのみ、後者は Apache 2.0 で無制限 |
| **大規模な fan-out（数百万件）を処理したい** | **AWS Step Functions（Distributed Map）** | S3 から最大1万並列。**durable execution 系は軒並み苦手な領域** |
| **ステップ間のレイテンシを削りたい** | **DBOS** / **Inngest（Connect + checkpointing）** | 前者は1〜2ms、後者は5ms 未満。→ §2.3 |
| **LLM の応答をユーザーにストリームしたい** | **Trigger.dev** / **Vercel Workflows** | Realtime API / Streams が第一級 |

### 決めきれないときの現実解

1. **まず「本当に durable execution が要るのか」を確認する。** 単発の API 呼び出しのリトライだけなら、指数バックオフのライブラリで足ります。**要るのは「複数の副作用にまたがる中途半端な成功」が起きうるとき**です（入門編 §0）。
2. **要るなら、既存インフラから逆算する。** Postgres だけなら DBOS / Hatchet、AWS 全振りなら Lambda Durable Functions、Vercel なら Inngest / Vercel Workflows。**新しいインフラを増やす判断は最後**でいい。
3. **Temporal は「重すぎる」から外すのであって、「機能が足りない」から外すことはまずない。** 迷ったら Temporal を基準にして、削れるものを探す。

---

## 11. 入門編 §10 との対応

入門編の表を、本書の内容で補正すると次のようになります。

| 製品 | 入門編の記述 | 補正・追加 |
|---|---|---|
| Temporal | 最も成熟。Go 製で Go SDK が第一級 | **そのとおり。** 加えて7言語 SDK、AI エコシステムの事実上の標準耐久層 |
| Restate | 軽量。`ctx.run()` ごとにジャーナル、冪等キー不要 | **そのとおり。** 加えて単一バイナリ（別 DB 不要）、push 型で serverless 可、BSL |
| Inngest | イベント駆動寄り。Webhook / キュー起点 | **そのとおり。** 加えて**自前ホスト可**（1.0 以降。誤情報が流通しているので注意） |
| DBOS | Postgres に状態を持つ。軽量 | **そのとおり。** 加えて「サーバなしのライブラリ」であることが本質。MIT |
| Hatchet | 通常タスクは at-least-once なので冪等性が必要 | **そのとおり。** 加えてキュー系機能（優先度・レート制限・fair scheduling）が強み |
| AWS Durable Functions | マネージド。step でラップする方式 | **そのとおり。** 加えて **step の外の状態変更が黙って壊れる**点が最大の落とし穴 |

**入門編には無かったが押さえるべきもの:**

- **Conductor** — 全履歴を無期限保持し、任意タスクからの再実行を UI で提供。AI プロバイダ14種をネイティブ統合。**GUI で定義できる2製品のうちの1つ**
- **Vercel Workflow DevKit** — `"use workflow"` によるランタイム非依存の SDK。方向性が新しい
- **Golem** — 唯一の透過的方式（step で包まなくてよい）
- **Dapr Workflows / Agents** — CNCF。K8s に既に居るなら追加コストほぼゼロ
- **Cloudflare Workflows の課金変更**（2026年8月10日〜、sleep もステップとして課金）

---

## 12. 質問への対応履歴 — 意図の確認と補足

本書 §1・§2.2・§2.3・§3・§4 は、以下の質問への回答として追記したものです。質問そのものへの評価を記しておきます。

| 質問 | 評価 | 対応 |
|---|---|---|
| **1. 種別の定義が分からない** | **正当な指摘。初版の不備です** | 3つの観点が混在していた。単一軸に整理し §1 軸1 に定義を明記 |
| **2. GUI か CLI か** | **意図は妥当。ただし軸が二択にならない** | 3層に分解が必要（§3）。実質「定義方法」だけが選定に効く |
| **3. 無料か有料か** | **正当。ただし二択で見ると必ず誤る** | 3費目に分解が必要（§4）。OSS の「無料」はインフラ費と運用工数を含まない |
| **4. 主要機能と有効な場面を表に** | **正当。カタログとして当然あるべき列でした** | §2.2 に「主要機能／有効な場面／**向かない場面**」を追加。各製品節にも反映 |

**質問4について1点だけ補足します。** 依頼は「主要機能」と「有効な場面」でしたが、**「向かない場面」を独断で追加しました。** カタログを実際に使う場面では、候補を**消す**根拠のほうが決定的に効くためです。さらに、製品を選ぶ前に確認すべきこととして、**durable execution というカテゴリ自体が向かない場面**を §2.3 に置きました。ステップごとの永続化が同期パスのレイテンシに直接乗る、という定量データが見つかったためです。

### まと外れな質問はありませんでした

ただし、**この3つだけでは製品を選べない**という点は指摘しておきます。選定を実際に左右するのは、優先度順に:

1. **実装方式（§1 軸2）** — ジャーナルリプレイなら決定論制約を受け入れる必要がある。**コードの書き方そのものが変わる話で、後から変更が効きません**
2. **既存インフラ（§10）** — Postgres しかないのか、AWS 全振りなのか、Kubernetes に Dapr が居るのか
3. **チームの言語（§2）** — SDK が無ければ検討対象になりません

**GUI/CLI と料金は、候補が2〜3に絞れた後の決着要因です。** 逆に言えば、初版で候補は絞れた状態なので、質問のタイミングとしては妥当です。

### 次に聞くとよいこと

- **「実行中のワークフローがある状態で、どうデプロイするか」** — 入門編 §11.1 のバージョニング問題。**導入後に最も痛む箇所**で、製品ごとに道具立てが大きく違います。ここを比較していないのが本書の最大の欠落です
- **「データがどこに置かれるか」** — マネージドを使う場合のリージョンと監査要件
- **「障害時に人間がどう介入するか」** — 失敗したワークフローの再開・スキップ・補償を、UI で押せるのか API を叩くのか。Conductor と Upstash はここが手厚く、他は差があります

---

## 13. この調査の限界

**この分野は月単位で動いています。**

- Cloudflare Workflows は「GA は2025年、2026年に再アーキテクチャ、制限と API は毎月動く」と評されている状態です。**課金体系そのものが2026年8月10日に変わります。**
- AWS Lambda Durable Functions は2025年12月に単一リージョンで登場し、2026年6月時点で約31リージョンまで広がりました。言語 SDK の GA も Java（4月）、.NET（7月）と順次です。
- **料金は特に陳腐化が早い項目です。** §4 の数値は2026年8月時点のもので、プラン構成ごと変わることがあります。
- **§2.2 の「主要機能」は、各ベンダの公式サイトが自ら掲げているものです。** つまり**マーケティングの主張であり、実測値ではありません。** 「有効な場面」も同様に、公式のユースケース記述を土台にしています。実際に要件を満たすかは PoC で確認してください。一方 **「向かない場面」は、公式が明示している制限値と、第三者による実測・批判記事を根拠にしています**（レイテンシは Inngest 自身の計測、Temporal のアンチパターンは Temporal 公式、LangGraph の checkpointer の限界は Diagrid の指摘）。
- **二次情報のライセンス・自前ホスト可否は特に信用できません。** 本書の調査中にも「Inngest は自前ホスト不可」という誤った記述が複数の比較記事で見つかりました（公式は1.0 で自前ホストを提供）。LittleHorse のサーバライセンスも AGPLv3 / SSPL の両方の記述が見つかり、確定できていません。Restate Cloud の有償プラン価格も公式ページから確定できませんでした。

> **採用判断の直前には、必ず公式ドキュメントと LICENSE ファイル、そして価格ページを直接確認してください。** 本書は候補を絞り込むための地図であって、契約の根拠ではありません。

---

## 14. 次に調べるとよいキーワード

- 各エンジンの **versioning / determinism テスト** の仕組み（入門編 §11.1、→ §12「次に聞くとよいこと」）
- Temporal **Nexus**（ワークフロー間の疎結合な呼び出し）
- Restate の **Virtual Object** と、アクターモデルとの関係
- **Durable Task SDKs**（Azure）のポータビリティ — Azure 外での運用
- Vercel **Workflow DevKit の「World」アダプタ**を自前実装する場合の要件
- Golem の **WASM component model** と既存言語資産の移行コスト
- LangGraph checkpointer を Temporal Activity で包む実装パターン

---

## 出典

**エンジン本体**

- [Temporal](https://temporal.io/) / [temporalio/temporal (MIT)](https://github.com/temporalio/temporal) / [Temporal Docs](https://docs.temporal.io/) / [Temporal Pricing](https://temporal.io/pricing) / [OpenAI Agents SDK 統合](https://temporal.io/blog/announcing-openai-agents-sdk-integration) / [Pydantic AI との統合](https://temporal.io/blog/build-durable-ai-agents-pydantic-ai-and-temporal)
- [Restate](https://restate.dev/) / [restatedev/restate](https://github.com/restatedev/restate) / [Restate vs Temporal](https://restate.dev/vs/temporal) / [Restate Cloud 一般公開](https://www.restate.dev/blog/announcing-restate-cloud-public) / [Restate CLI 設定](https://docs.restate.dev/references/cli-config)
- [DBOS Transact](https://www.dbos.dev/dbos-transact) / [dbos-transact-py (MIT)](https://github.com/dbos-inc/dbos-transact-py) / [DBOS Pricing](https://www.dbos.dev/pricing) / [DBOS CLI](https://docs.dbos.dev/python/reference/cli) / [DBOS Conductor](https://www.dbos.dev/dbos-conductor) / [DBOS vs Temporal](https://www.dbos.dev/compare/dbos-vs-temporal)
- [hatchet-dev/hatchet (MIT)](https://github.com/hatchet-dev/hatchet) / [Hatchet Pricing](https://hatchet.run/pricing) / [Hatchet CLI](https://docs.hatchet.run/cli) / [Hatchet Self-Hosting](https://docs.hatchet.run/self-hosting)
- [Inngest 自前ホストの発表](https://www.inngest.com/blog/inngest-1-0-announcing-self-hosting-support) / [Inngest Pricing](https://www.inngest.com/pricing)
- [Trigger.dev Pricing](https://trigger.dev/pricing)
- [resonatehq/resonate (Apache 2.0)](https://github.com/resonatehq/resonate) / [Resonate Docs](https://docs.resonatehq.io/) / [Resonate CLI ガイド](https://docs.resonatehq.io/get-started/cli-guide)
- [Golem](https://golem.cloud/) / [Golem Cloud 価格・商用オプション](https://golem.cloud/cloud/) / [Golem Goes Open Source](https://golem.cloud/blog/golem-goes-open-source/) / [The Rise of the Agent Runtime](https://golem.cloud/blog/the-rise-of-the-agent-runtime/)
- [Conductor OSS FAQ](https://conductor-oss.github.io/conductor/devguide/faq.html) / [Conductor UI でのワークフロー構築](https://orkes.io/content/developer-guides/build-workflows-using-ui) / [conductor-oss/conductor-cli](https://github.com/conductor-oss/conductor-cli) / [Orkes](https://orkes.io/what-is-conductor)
- [littlehorse-enterprises/littlehorse](https://github.com/littlehorse-enterprises/littlehorse) / [lhctl](https://littlehorse.io/docs/server/developer-guide/lhctl) / [LittleHorse Docs](https://littlehorse.io/docs)
- [Dapr Workflow](https://docs.dapr.io/developing-applications/building-blocks/workflow/workflow-architecture/) / [Dapr Agents](https://docs.dapr.io/developing-ai/dapr-agents/) / [dapr/dapr-agents](https://github.com/dapr/dapr-agents)

**マネージド / プラットフォーム**

- [AWS Lambda Durable Functions 実践ガイド](https://hidekazu-konishi.com/entry/aws_lambda_durable_functions_practical_guide.html) / [AWS 発表（2025年12月）](https://aws.amazon.com/about-aws/whats-new/2025/12/lambda-durable-multi-step-applications-ai-workflows/) / [.NET SDK GA](https://aws.amazon.com/about-aws/whats-new/2026/07/lambdadf-dotnet/) / [AWS Lambda Pricing](https://aws.amazon.com/lambda/pricing/)
- [Azure Durable Task 概要](https://learn.microsoft.com/en-us/azure/durable-task/common/what-is-durable-task) / [Durable Task Scheduler](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-task-scheduler/durable-task-scheduler) / [Durable Task Scheduler 課金](https://learn.microsoft.com/en-us/azure/durable-task/scheduler/durable-task-scheduler-billing) / [Consumption SKU GA](https://techcommunity.microsoft.com/blog/appsonazureblog/the-durable-task-scheduler-consumption-sku-is-now-generally-available/4506682)
- [Cloudflare Workflows GA](https://blog.cloudflare.com/workflows-ga-production-ready-durable-execution/) / [Dynamic Workflows](https://blog.cloudflare.com/dynamic-workflows/) / [Workers Pricing](https://developers.cloudflare.com/workers/platform/pricing/)
- [Vercel: A new programming model for durable execution](https://vercel.com/blog/a-new-programming-model-for-durable-execution) / [Vercel Workflows Docs](https://vercel.com/docs/workflows) / [Workflow Pricing and Limits](https://vercel.com/docs/workflows/pricing)
- [Upstash: Durable Workflow Engines in 2026](https://upstash.com/blog/durable-workflow-engines-compared-every-major-option-in-2026) / [upstash/workflow-js](https://github.com/upstash/workflow-js)
- [Convex Workflow component](https://www.convex.dev/components/workflow) / [get-convex/workflow](https://github.com/get-convex/workflow)

**エージェント / 比較記事**

- [LangGraph Durable execution](https://docs.langchain.com/oss/python/langgraph/durable-execution) / [LangGraph Pricing の解説](https://www.truefoundry.com/blog/langgraph-pricing) / [Why Checkpoints Aren't Durable Execution (Diagrid)](https://www.diagrid.io/blog/checkpoints-are-not-durable-execution-why-langgraph-crewai-google-adk-and-others-fall-short-for-production-agent-workflows)
- [Durable Execution: How Temporal, Restate, and DBOS Are Rethinking Distributed State](https://devstarsj.github.io/2026/04/03/durable-execution-temporal-restate-dbos-distributed-workflows-2026/)
- [9 Best Temporal Alternatives (ZenML)](https://www.zenml.io/blog/temporal-alternatives) / [10 Best Inngest Alternatives (Diagrid)](https://www.diagrid.io/infrastructure/10-best-inngest-alternatives-2026)
- [Kestra vs n8n vs Windmill](https://ossalt.com/guides/kestra-vs-n8n-vs-windmill-2026) / [awesome-workflow-engines](https://github.com/meirwah/awesome-workflow-engines)

**主要機能・ユースケース・アンチパターン（§2.2 / §2.3 の根拠）**

- [Temporal Use Cases（トップページ）](https://temporal.io/) / [Temporal アンチパターン](https://temporal.io/blog/spooky-stories-chilling-temporal-anti-patterns-part-1)
- [Restate: What is Durable Execution?](https://restate.dev/what-is-durable-execution)
- [DBOS（トップページ・事例）](https://www.dbos.dev/)
- [Hatchet（トップページ）](https://hatchet.run/)
- [Inngest（トップページ）](https://www.inngest.com/) / [Inngest: Eliminating latency in AI workflows](https://www.inngest.com/blog/eliminating-latency-ai-workflows)（**§2.3 のレイテンシ実測値**）
- [Trigger.dev（トップページ）](https://trigger.dev/)
- [Resonate（トップページ）](https://www.resonatehq.io/)
- [Golem（トップページ）](https://golem.cloud/)
- [Conductor（トップページ）](https://conductor-oss.github.io/conductor/index.html)
- [LittleHorse Docs](https://littlehorse.io/docs)
- [Dapr Workflow パターン](https://docs.dapr.io/developing-applications/building-blocks/workflow/workflow-patterns/)
- [AWS Step Functions Use Cases](https://aws.amazon.com/step-functions/use-cases/) / [Distributed Map の実践パターンと落とし穴](https://hidekazu-konishi.com/entry/aws_step_functions_distributed_map_guide.html)
- [Cloudflare Workflows ソリューション](https://www.cloudflare.com/solutions/workflows/)
- [Vercel Workflows Docs](https://vercel.com/docs/workflows)
- [Upstash Workflow Getting Started](https://upstash.com/docs/workflow/getstarted)
- [Convex Durable Agents](https://www.convex.dev/components/durable-agents) / [Agents Need Durable Workflows and Strong Guarantees](https://stack.convex.dev/durable-workflows-and-strong-guarantees)
