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

## 1. 表の読み方 — 5つの分類軸

### 軸1. 実装方式

入門編 §10 で挙げた2方式に、1つ足して3つです。

| 方式 | 何をするか | 決定論の制約 | 代表 |
|---|---|---|---|
| **ジャーナルリプレイ** | 完了ステップを履歴に記録し、クラッシュ時にコードを最初から流し直して記録済み分をスキップ | **強い**（Workflow コードは決定論的でなければならない） | Temporal、Restate、AWS Lambda Durable Functions、Azure Durable Functions、Cloudflare Workflows |
| **DB チェックポイント** | ステップ／ノードごとに状態を DB に書き、途中から再開 | **弱い**（最初から流し直さないので制約が緩い） | DBOS、LangGraph、Hatchet |
| **透過的スナップショット** | ランタイム（WASM）がホスト呼び出しをすべて記録するので、開発者が step で包む必要がない | **無い**（ランタイムが吸収する） | Golem |

**この軸が最も効きます。** 入門編 §5.2「Workflow の中で禁止されるもの」の表が適用されるのは、**ジャーナルリプレイ方式だけ**です。DB チェックポイント方式では `time.Now()` を直接呼んでも壊れません。

代わりに失うものもあります。リプレイ方式は「コードそのものが状態」なので状態管理コードが一行も要らないのに対し、チェックポイント方式は「どこまで進んだか」を明示的に持つ必要があり、ステップ境界がフレームワークの形（グラフのノードなど）に縛られがちです。

### 軸2. コードがどこで動くか — pull か push か

**これが serverless 適性と待機コストを決めます。**

| 型 | 動き | 帰結 |
|---|---|---|
| **pull（常駐ワーカー）** | ワーカーが常駐してサーバをポーリングし、仕事を取りに行く | serverless に載らない。アイドル中もワーカーは動いている |
| **push（ステートレス関数）** | エンジンが HTTP でアプリを呼び出す。ステップごとに別リクエスト | Next.js の API ルートや Lambda にそのまま載る。アイドル中の計算コストが0 |

Temporal は pull、Restate / Inngest / Upstash / Vercel は push です。**入門編 §9.3 の「待機中のコストが60〜80%下がる」は push 型でより極端に効きます。**

### 軸3. 状態をどこに置くか

| 置き場所 | 運用負荷 | 代表 |
|---|---|---|
| 専用クラスタ＋外部 DB | **高** | Temporal（DB ＋ Elasticsearch） |
| 単一バイナリ（自己完結） | 低 | Restate、Resonate |
| **既存の Postgres だけ** | **最小** | DBOS、Hatchet |
| ベンダのマネージド | ゼロ（ただしロックイン） | Cloudflare、AWS、Azure、Upstash |

### 軸4. ライセンス

「OSS」と書いてあっても中身が違います。

- **OSI 準拠**（MIT / Apache 2.0）— Temporal、DBOS、Hatchet、Trigger.dev、Conductor、Resonate
- **fair source / BSL**（一定期間後に OSS 化）— Restate（runtime は BSL 1.1）、Inngest（SSPL → 3年後に Apache 2.0）、Golem（BUSL-1.1 → Apache 2）
- **source-available**（OSS ではない）— LittleHorse サーバ、n8n

商用配布や SaaS 提供を考えるなら BSL / SSPL は必ず原文を読んでください。

### 軸5. 「durable execution なのか」

最後に、**そもそもこのカテゴリに入らないもの**を分けます（→ §6）。

---

## 2. 全体マップ

| # | 名前 | 種別 | 実装方式 | 状態の置き場所 | 主な言語 | ライセンス / 自前ホスト |
|---|---|---|---|---|---|---|
| 1 | **Temporal** | 汎用エンジン | ジャーナルリプレイ（pull） | 専用クラスタ＋DB | Go, Java, Python, TS, .NET, PHP, Ruby | MIT ／ 可 |
| 2 | **Restate** | 汎用エンジン | ジャーナルリプレイ（push） | 単一バイナリ | TS, Java/Kotlin, Python, Go, Rust | BSL 1.1（SDK は MIT）／ 可 |
| 3 | **DBOS Transact** | ライブラリ | DB チェックポイント | 既存 Postgres | Python, TS, Go, Java, Kotlin | MIT ／ 可（サーバ不要） |
| 4 | **Hatchet** | エンジン＋キュー | 耐久イベントログ | Postgres | Python, TS, Go, Ruby | MIT ／ 可 |
| 5 | **Inngest** | エンジン | ジャーナル（push / イベント駆動） | Inngest（or 自前 Redis+PG） | TS, Python, Go | SSPL→Apache 2.0 ／ 可 |
| 6 | **Trigger.dev** | マネージド寄り | チェックポイント／再開 | Trigger.dev（or 自前 PG） | TypeScript（Python は補助） | Apache 2.0 ／ 可 |
| 7 | **Resonate** | 汎用エンジン | ジャーナル（distributed async await） | 単一バイナリ（SQLite / PG） | TS, Python, Go, Rust, Java | Apache 2.0 ／ 可 |
| 8 | **Golem** | エージェントランタイム | **透過的スナップショット（WASM）** | Golem ランタイム | WASM に載る言語（TS 等） | BUSL-1.1→Apache 2 ／ 可 |
| 9 | **Conductor (OSS/Orkes)** | オーケストレータ | タスク＋ワーカー、全履歴永続化 | Redis / PG / MySQL / Cassandra / SQLite | Java, Python, Go, JS, C#, Ruby, Rust | Apache 2.0 ／ 可 |
| 10 | **LittleHorse** | エンジン | Kafka を write-ahead log に | Kafka | Java, Go, Python, C# | source-available（SDK は Apache 2.0）／ 可 |
| 11 | **Dapr Workflows / Agents** | サイドカー | ジャーナルリプレイ（Durable Task 系） | 差し替え可能な state store | Python, JS, .NET, Java, Go | Apache 2.0（CNCF）／ 可 |
| 12 | **AWS Lambda Durable Functions** | マネージド | ジャーナルリプレイ | AWS 管理 | Node.js, Python, Java, .NET | 商用 ／ 不可 |
| 13 | **AWS Step Functions** | マネージド | 状態機械（JSON/ASL） | AWS 管理 | 言語非依存（JSON 定義） | 商用 ／ 不可 |
| 14 | **Azure Durable Functions / Durable Task SDKs** | マネージド | イベントソーシング＋リプレイ | Durable Task Scheduler 等 | .NET, Python, Java, JS, PowerShell | 商用（SDK は OSS）／ 一部可 |
| 15 | **Cloudflare Workflows** | マネージド | ジャーナルリプレイ | Cloudflare 管理 | TypeScript（Python ベータ） | 商用 ／ 不可 |
| 16 | **Vercel Workflows / Workflow DevKit** | SDK＋マネージド | ジャーナル（`"use workflow"`） | 「World」アダプタ次第 | TypeScript, Python（ベータ） | OSS SDK ／ 可（PG 実装あり） |
| 17 | **Upstash Workflow** | マネージド | ステップメモ化（push / HTTP） | Upstash（QStash） | TypeScript, Python | 商用 ／ 不可 |
| 18 | **Convex Workflow** | コンポーネント | DB チェックポイント | Convex DB | TypeScript | OSS コンポーネント ／ Convex 前提 |
| 19 | **LangGraph** | エージェント FW | DB チェックポイント | Postgres / Redis / DynamoDB 等 | Python, JS/TS | OSS ／ 可 |

---

## 3. カテゴリA — 自前ホストできる汎用エンジン

ここが本命です。**言語もインフラも自分で選べる代わりに、運用は自分の仕事になります。**

### 3.1 Temporal

| 項目 | 内容 |
|---|---|
| ライセンス | サーバは **MIT**。Temporal Cloud は $100/月 〜（Action 課金） |
| SDK | **7言語**（Go, Java, Python, TypeScript, .NET, PHP, Ruby）。**Go 製で Go SDK が第一級** |
| 構成 | frontend / history / matching / worker の複数コンポーネント ＋ 外部 DB（Cassandra / PostgreSQL / MySQL）＋ Elasticsearch |
| 実行モデル | **pull**。長寿命ワーカーがタスクキューをポーリング |
| 主な制限 | ペイロード 2MB、履歴 51,200 イベント または 50MB |
| 待機 | `workflow.Sleep` で年単位。Signal / Update で人間の承認 |

**特徴**

- **成熟度が突出しています。** 2026年2月17日に50億ドル評価で3億ドル調達、クラウド上の累計アクション実行9.1兆（うち1.86兆が AI ネイティブ企業）。顧客に OpenAI、Snap、Netflix、JPMorgan Chase。
- **AI エコシステムのデファクトの耐久層になりつつあります。** OpenAI Agents SDK 連携は2026年3月23日に GA。Pydantic AI、Vercel AI SDK、Amazon Bedrock Strands との統合もあり、**既存のエージェントコードを書き換えずに Activity で包める**のが売り。
- 代償は運用です。**この一覧で最も重いインフラ**で、しかも pull 型なのでワーカーが常駐します。アイドル中も計算資源は消えません。
- 入門編 §11.1 のバージョニング問題が最も先鋭に出るのもここです。裏を返せば、**リプレイテストや versioning API の道具立てが最も揃っている**のもここ。

> **「durable execution を検討する」= まず Temporal を評価軸に置く、で当面は間違いになりません。** 他を選ぶ理由は、たいてい「Temporal ほどの重さが要らない」です。

### 3.2 Restate

| 項目 | 内容 |
|---|---|
| ライセンス | runtime は **BSL 1.1**（期限後に Apache 2.0）、**SDK は MIT** |
| SDK | TypeScript, Java/Kotlin, Python, Go, Rust |
| 構成 | **単一の自己完結バイナリ**（Rust 製）。別 DB も Elasticsearch も不要。HA は複数インスタンス＋オブジェクトストレージへのスナップショット |
| 実行モデル | **push**。アプリ側はただのステートレス HTTP サービス（serverless 関数でも可） |
| 独自機能 | Virtual Object（エンティティ単位の K/V 状態）、exactly-once メッセージング、耐久タイマー／プロミス、OpenTelemetry 自動生成 |

**特徴**

- **運用フットプリントが Temporal と桁で違います。** バイナリ1つ。これが最大の訴求。
- `ctx.run()` ごとにジャーナルするので、**アプリコード側に冪等キーを書かなくてよい**（入門編 §7.3 の「冪等性は設計事項」の負担が一部エンジン側に寄る）。
- 設計思想の差がよく引用されます — **「Temporal は個々のワークフローを durable にする。Restate はシステム全体を durable にする」**。ワークフローだけでなく、サービス間通信そのものに exactly-once を持ち込む発想です。
- push 型なので **serverless にそのまま載ります。** ワーカー常駐が要らない。
- 成熟度と実績は Temporal に劣ります。BSL も要確認事項です。

### 3.3 DBOS Transact

| 項目 | 内容 |
|---|---|
| ライセンス | **MIT**。DBOS Pro（Conductor UI 等）が有償 |
| SDK | Python, TypeScript, Go, Java, Kotlin |
| 構成 | **サーバなし。アプリ内のライブラリ。** 状態は既存の Postgres |
| 実行モデル | in-process |
| 性能 | **1ステップのチェックポイント = Postgres への1書き込み（1〜2ms）** |

**特徴**

- **新規インフラがゼロ。** 「Postgres はもう信用して使っている」チームにとって、追加で運用するものが何もない。デコレータで既存関数に注釈を付けるだけ。
- **レイテンシに効きます。** 外部オーケストレータへの往復が無いので、対話的・低レイテンシなワークフローに向く。Temporal 系がミリ秒〜十数ミリ秒の往復を払うところを、ローカル DB への1書き込みで済ませる。
- **workflow ID がそのまま冪等キーになる**設計。
- Pydantic AI が公式に統合をドキュメント化しています（Temporal、Prefect と並んで）。
- 弱点は、**ライブラリなのでプロセス外の監督者がいない**こと。アプリが全滅したら誰も再開させません（別プロセスの recovery 機構は用意されているが、Temporal のような独立クラスタとは前提が違う）。また Postgres がそのままスケール上限になります。

### 3.4 Hatchet

| 項目 | 内容 |
|---|---|
| ライセンス | **MIT** |
| SDK | Python, TypeScript, Go, Ruby |
| 構成 | **Postgres がタスクランタイムと可観測性の両方の耐久層** |
| 位置づけ | 「タスクキュー」と「durable execution」の中間 |

**特徴**

- 出自がタスクキューなので、**キュー側の機能が厚い**。優先度、レート制限（動的レート制限を含む）、concurrency ポリシーによる fair scheduling、ワーカーラベルによるルーティング、マルチテナンシ。
- durable task と DAG の両方を持ち、**耐久 sleep とイベント待ちを組み合わせた複雑な pause/resume 条件**が書ける。
- 入門編 §10 の指摘どおり **通常タスクは at-least-once** なので、冪等性は自分の仕事です。
- Postgres だけで動くので自前ホストが容易。Cloud は10万タスク実行まで無料、以降 $10 / 100万実行。

### 3.5 Inngest

| 項目 | 内容 |
|---|---|
| ライセンス | **fair source**（SSPL、3年後に自動で Apache 2.0） |
| SDK | TypeScript, Python, Go |
| 構成 | 自前ホスト可（1コマンド。SQLite 同梱、本番は外部 Redis + Postgres） |
| 実行モデル | **push / イベント駆動** |
| 主な制限 | ステップ出力 4 MiB、1,000ステップ、sleep は無料30日 / Pro 366日 |

**特徴**

- **イベント駆動が第一級。** Webhook やキューを起点にする設計で、`step.run()` / `step.sleep()` / `waitForEvent()` が基本語彙。CEL 式でイベントをマッチングできる。
- **アプリの HTTP ルートの中でステップが走る**ので、Next.js on Vercel のような構成にそのまま載ります。
- **sleep 中は concurrency 枠を消費しません。** アイドルが安い。
- 注意: 「Inngest は自前ホストできない」と書いた比較記事が複数流通していますが、**Inngest 1.0 で自前ホストは公式にサポートされています。** 二次情報のライセンス／自前ホスト可否は当てになりません。

### 3.6 Resonate

| 項目 | 内容 |
|---|---|
| ライセンス | **Apache 2.0** |
| SDK | TypeScript, Python, Go, Rust, Java |
| 構成 | **単一バイナリ**。デフォルト SQLite、Postgres も可 |
| モデル | **Distributed Async Await** — 言語・トランスポート非依存のプロトコル |

**特徴**

- **「普通の async/await がそのまま分散・耐久になる」**という一点に振り切った設計。新しいワークフロー DSL を覚えさせない方向。
- サーバはワーカーの **supervisor 兼 orchestrator** として振る舞う。
- 規模と実績は小さいので、本番採用は評価前提で。

### 3.7 Golem

| 項目 | 内容 |
|---|---|
| ライセンス | **BUSL-1.1**（Apache 2 へ移行予定） |
| 実行単位 | **WebAssembly コンポーネント**（専用実行エンジン） |
| 現況 | Cloud は Developer Preview、有償 GA は2026年Q3予定 |

**特徴**

- **この一覧で唯一、開発者が step で包まなくてよい方式です。** ランタイムが WASM の実行そのものを耐久化するので、入門編 §5.2 の禁止事項リストが原理的に不要になる。**「透過的な durable execution」**を主張しているのはここだけ。
- 2025年5月以降、**「durable agent runtime」へ明確にリポジション**。エージェントごとに WASM サンドボックス＋専用ファイルシステム＋専用 SQLite を持たせ、ミリ秒起動。
- ツール／ネットワーク／ファイルへのアクセスは**明示的な grant** で、プロンプトインジェクションで権限昇格できず、quota はランタイムが強制し、**すべての grant がジャーナルされる**。durable execution とサンドボックス隔離を同じ機構で解こうとしている。
- 代償は、**WASM に載せられるものしか動かない**こと。既存の Python 資産をそのまま持ち込むような使い方には向きません。

### 3.8 Conductor（OSS / Orkes）

| 項目 | 内容 |
|---|---|
| ライセンス | **Apache 2.0**（Orkes が主要メンテナ、SaaS も提供） |
| SDK | Java, Python, Go, JavaScript, C#, Ruby, Rust ＋ HTTP が話せれば任意言語 |
| 永続化 | **Redis / PostgreSQL / MySQL / Cassandra / SQLite から選択** |
| 定義 | **JSON ベース**のワークフロー定義（コードではない） |

**特徴**

- Netflix 発。**タスクとワーカーを明示的に分離する**古典的オーケストレーション。実績規模は月10億ワークフロー級。
- **実行履歴を無期限に保持し、「最初からやり直す／任意のタスクから再実行する／失敗したステップだけリトライする」を数か月後でも実行できる。** 入門編 §7.3 の「補償処理を書けるようになる」が、運用 UI のレベルで提供されている。
- AI 向けが手厚い。LLM タスク（chat/text completion）、MCP のツール呼び出し、human-in-the-loop 承認、**エージェントが実行時に生成する動的ワークフロー**。プロバイダは Anthropic / OpenAI / Azure OpenAI / Gemini / Bedrock / Mistral / Cohere / HuggingFace / Ollama など14以上をネイティブ統合。
- コードではなく JSON 定義なので、**入門編 §12.2 の「git diff でレビューできる」という利点は半減**します。iPaaS と durable execution の中間に位置すると考えるのが正確です。

### 3.9 LittleHorse

| 項目 | 内容 |
|---|---|
| ライセンス | **サーバは source-available**（AGPLv3 / SSPL — 版によって変遷しているため要確認）、**SDK は Apache 2.0** |
| SDK | Java, Go, Python, C#/.NET |
| 構成 | Kernel は Java 製、**Apache Kafka を durable write-ahead log として使用** |

**特徴**

- **Kafka を既に運用している組織にとっては、耐久層が既存資産で済む**という一点が効きます。
- User Task（人間の作業）が組み込み。リアルタイム可観測性とリトライによる自動復旧。
- ライセンスがサーバ側で OSI 準拠でない点は、選定時の明確な検討事項です。

### 3.10 Dapr Workflows / Dapr Agents

| 項目 | 内容 |
|---|---|
| ライセンス | **Apache 2.0（CNCF プロジェクト）** |
| SDK | Python, JavaScript, .NET, Java, Go |
| 基盤 | **Durable Task Framework**（Azure Durable Functions と同系統）ベース |
| 構成 | **サイドカー**。state store は差し替え可能 |

**特徴**

- **Kubernetes / マイクロサービス基盤に Dapr を既に入れているなら、追加コストがほぼゼロ**で durable workflow が手に入る。
- **Dapr Agents v1.0 が本番対応**として出ています。LLM とツールとのやり取りを毎回 durable state store に永続化し、エージェント再起動後も継続。
- Pub/Sub による**イベント駆動のマルチエージェント協調**が組み込み。durable execution とメッセージングが同じ基盤に乗るのが Dapr らしい点。

---

## 4. カテゴリB — マネージド / プラットフォーム組み込み

**運用が消える代わりに、プラットフォームに乗ります。**

### 4.1 比較表

| | AWS Lambda Durable Functions | AWS Step Functions | Azure Durable Functions | Cloudflare Workflows | Vercel Workflows | Upstash Workflow |
|---|---|---|---|---|---|---|
| 定義の書き方 | **コード** | JSON（ASL） | **コード** | **コード** | **コード**（`"use workflow"`） | **コード** |
| 言語 | Node.js / Python / Java / .NET | 非依存 | .NET / Python / Java / JS / PowerShell | TypeScript（Python ベータ） | TypeScript / Python（ベータ） | TypeScript / Python |
| 最長待機 | **1年** | Standard 1年 / Express 5分 | 実質無制限 | 365日 | — | 1年 |
| ステップ上限 | 3,000 durable operations | 25,000 履歴イベント | — | 1,024（無料）/ 25,000（有料） | — | 1,000（引き上げ可） |
| ペイロード | 耐久状態 100MB | 256 KiB / state | — | 1 MiB / step | — | 1〜50MB |
| 自前ホスト | 不可 | 不可 | Durable Task SDK は任意基盤で可 | 不可 | **可**（Postgres 実装） | 不可 |

### 4.2 AWS Lambda Durable Functions

2025年 re:Invent で発表。**入門編 §4.3 で引用されている「step で包む方式」がこれです。**

- プリミティブは4つ: **step**（チェックポイント＋自動リトライ）、**wait**（最大1年、その間の計算課金なし）、**callback**（外部システムからの応答を待つ）、**invoke**（他 Lambda を呼んで結果をチェックポイント）
- **決定論契約が明示的**: 「毎回の invocation で、同じ順序・同じ名前・同じ型の durable operation 呼び出しを行わなければならない」。`Date.now()`、UUID 生成、`Math.random()`、HTTP、DB、AWS SDK 呼び出しはハンドラ直下では禁止。
- **最も嫌らしい落とし穴**: step の外での状態変更は**黙って失敗する**。初回は正しく見え、リプレイ時にその変更だけがリセットされる。入門編 §4.4 の「黙って壊れる」の実例。
- ランタイム: Node.js 22/24、Python 3.13/3.14、Java 17/21/25、.NET。OCI コンテナも可。Java は2026年4月 GA、.NET は2026年7月 GA。2026年6月時点で約31リージョン。
- 実行タイムアウトは60秒〜1年（既定24時間）。ただし**個々の invocation は Lambda の15分制限のまま**。
- **Step Functions の代替ではありません。** 大規模な fan-out（数百万件）は Distributed Map の領分で、durable functions は苦手。AWS 自身が「補完関係」と位置づけており、ハイブリッド構成（Step Functions がサービス間グラフ、durable functions がコード中心の区間）が推奨。

### 4.3 Azure Durable Functions / Durable Task SDKs

- **orchestrator / activity の分離**は Temporal の Workflow / Activity と同型。イベントソーシング＋決定論的リプレイ。Temporal の系譜そのもの（入門編でも触れたとおり、Temporal の作者は Azure Durable Functions の設計にも関わっている）。
- 注目すべきは **Durable Task SDKs** の分離です。**ポータブルな OSS ライブラリとして切り出され、Azure Container Apps / Kubernetes / VM など任意の基盤で動く。** Azure Functions に縛られなくなった。
- **Durable Task Scheduler** が推奨の状態プロバイダ。専用ストレージアカウント不要でレイテンシも改善。Dedicated SKU が2025年11月 GA、**Consumption SKU が2026年3月 GA**。

### 4.4 Cloudflare Workflows

- `step.do` / `step.sleep` / `step.sleepUntil` / `step.waitForEvent`。既定リトライ5回（指数バックオフ）。
- **2026年5月に Dynamic Workflows** を追加。ワークフロー定義を事前デプロイせず Dynamic Worker 内でオンデマンドにロードする — **テナントごとに異なるワークフローを動かす**用途に効きます。
- **Agents SDK と第一級で統合**されている唯一のエンジン。
- **課金が2026年8月10日以降にステップ単位へ変更**（有料プランは月50万ステップ込み、超過分 $0.80 / 10万ステップ。無料は日3,000ステップ）。**sleep もイベント待ちも1ステップとして数えられます。** 長時間待機を多用する設計はコスト再計算が必要。
- 代償は Cloudflare への集中。ステップ出力 1 MiB は他より小さい。

### 4.5 Vercel Workflows / Workflow DevKit

**プログラミングモデルが異質で、注目に値します。**

```ts
"use workflow"   // ← このトップレベル関数が段取り
"use step"       // ← この単位が step。リトライ・永続化・可観測性が自動で付く
```

- **外部オーケストレータが存在しません。** 調整はすべてアプリコード内で完結し、Fluid compute の上で走る。「step が実際に走っている計算分だけ払う」モデル。
- **「Worlds」というアダプタ機構**で、イベントログ・計算・キューの3要素を差し替えられる。マネージド（Vercel）／自前ホスト（Postgres 参照実装）／組み込み、の3形態。コミュニティが MongoDB、Redis、Turso、Cloudflare 向けアダプタを作っている。
- SDK は OSS。TypeScript が本命、Python はベータ（デコレータ `@wf.workflow` / `@wf.step`）。
- **ランタイムに縛られない durable execution SDK** という立ち位置は、この一覧では他にありません。ただし新しく、実績はこれから。

### 4.6 Upstash Workflow

- QStash の上に構築。**ステップごとに自分のアプリへ HTTP 呼び出しが飛び、結果を Upstash が保持する。** ワーカーもサーバも Postgres も要らない。
- **30日 sleep が計算0で、課金上は1ステップ。** アイドルの安さは随一。$1 / 10万ステップ、月額固定費なし。
- 失敗が尽きると **DLQ に入り、「失敗したステップから再開（完了分は結果を保持）／最初からやり直し／失敗コールバックの再発火」を選べる。**
- `context.waitForEvent` / `context.notify`。イベントを待機点到達前に保存してレースを防ぐ設計。
- 自前ホスト不可。Upstash 前提。

### 4.7 Trigger.dev

- **Apache 2.0。Docker + Postgres で全機能を自前ホストでき、機能制限も実行数制限もない。** OSI 準拠ライセンスのマネージド系としては強い。
- TypeScript ファースト。タスクは Trigger.dev のマシンにデプロイされ、アプリからトリガーする（**アプリと別デプロイ**になる点は好みが分かれる）。
- プラットフォームのタイムアウトが無い。Cloud では5秒超の待機をチェックポイントして計算課金を避ける。

---

## 5. カテゴリC — エージェントフレームワーク側の耐久機構

**入門編 §9.6 の「もはやオプションではなくベースライン要件」がここに現れています。**

| フレームワーク | 耐久化の方式 | 備考 |
|---|---|---|
| **LangGraph** | **DB チェックポイント**（PostgresSaver / RedisSaver / DynamoDBSaver 等） | superstep ごとに state を保存。thread 単位、time travel、HITL interrupt |
| **Pydantic AI** | **外部エンジンに委譲** | Temporal / DBOS / Prefect との統合を公式ドキュメント化 |
| **OpenAI Agents SDK** | **外部エンジンに委譲** | Temporal 連携が2026年3月23日 GA。サンドボックス統合も |
| **Dapr Agents** | **Dapr Workflows** | v1.0 で本番対応。Pub/Sub でマルチエージェント協調 |

### 5.1 LangGraph の checkpointer に関する重要な注意

checkpointer を設定すれば「durable execution が有効」と説明されますが、**入門編の定義に照らすと、これは半分です。**

> **checkpointer は状態を保存する。しかし障害を検知して再開させる主体がいない。**

プロセスがクラッシュしても、それに気づいて再開をキックする supervisor / watchdog が存在しません。**「どこまで進んだか」は残るが、「必ず最後まで到達させる」（入門編 §1 の目的1）は満たさない。**

これは LangGraph の欠陥というより**層が違う**話です。実務では **LangGraph を Temporal / Dapr の Activity の中で走らせる**構成が定番になっています。入門編 §8.3 のループを人間が書き、その中の1ステップとして LangGraph のグラフを回すイメージです。

同じ注意は CrewAI や Google ADK など、checkpoint だけを持つフレームワーク全般に当てはまります。

---

## 6. カテゴリD — 隣接だが別物（混同注意）

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

## 7. 実装方式で並べ直す

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

## 8. 選び方

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
| **人間の承認・業務プロセスの可視化が主目的** | **Conductor** / **Camunda** | UI と human task が第一級 |
| **エージェントのサンドボックス隔離も同時に解きたい** | **Golem** | grant がジャーナルされる WASM サンドボックス |
| **既存の LangGraph 資産がある** | **LangGraph ＋ 外部エンジン** | checkpointer だけでは §5.1 の穴が残る |

### 決めきれないときの現実解

1. **まず「本当に durable execution が要るのか」を確認する。** 単発の API 呼び出しのリトライだけなら、指数バックオフのライブラリで足ります。**要るのは「複数の副作用にまたがる中途半端な成功」が起きうるとき**です（入門編 §0）。
2. **要るなら、既存インフラから逆算する。** Postgres だけなら DBOS / Hatchet、AWS 全振りなら Lambda Durable Functions、Vercel なら Inngest / Vercel Workflows。**新しいインフラを増やす判断は最後**でいい。
3. **Temporal は「重すぎる」から外すのであって、「機能が足りない」から外すことはまずない。** 迷ったら Temporal を基準にして、削れるものを探す。

---

## 9. 入門編 §10 との対応

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

- **Conductor** — 全履歴を無期限保持し、任意タスクからの再実行を UI で提供。AI プロバイダ14種をネイティブ統合
- **Vercel Workflow DevKit** — `"use workflow"` によるランタイム非依存の SDK。方向性が新しい
- **Golem** — 唯一の透過的方式（step で包まなくてよい）
- **Dapr Workflows / Agents** — CNCF。K8s に既に居るなら追加コストほぼゼロ
- **Cloudflare Workflows の課金変更**（2026年8月10日〜、sleep もステップとして課金）

---

## 10. この調査の限界

**この分野は月単位で動いています。**

- Cloudflare Workflows は「GA は2025年、2026年に再アーキテクチャ、制限と API は毎月動く」と評されている状態です。
- AWS Lambda Durable Functions は2025年12月に単一リージョンで登場し、2026年6月時点で約31リージョンまで広がりました。言語 SDK の GA も Java（4月）、.NET（7月）と順次です。
- **二次情報のライセンス・自前ホスト可否は特に信用できません。** 本書の調査中にも「Inngest は自前ホスト不可」という誤った記述が複数の比較記事で見つかりました（公式は1.0 で自前ホストを提供）。LittleHorse のサーバライセンスも AGPLv3 / SSPL の両方の記述が見つかり、確定できていません。

> **採用判断の直前には、必ず公式ドキュメントと LICENSE ファイルを直接確認してください。** 本書は候補を絞り込むための地図であって、契約の根拠ではありません。

---

## 11. 次に調べるとよいキーワード

- 各エンジンの **versioning / determinism テスト** の仕組み（入門編 §11.1）
- Temporal **Nexus**（ワークフロー間の疎結合な呼び出し）
- Restate の **Virtual Object** と、アクターモデルとの関係
- **Durable Task SDKs**（Azure）のポータビリティ — Azure 外での運用
- Vercel **Workflow DevKit の「World」アダプタ**を自前実装する場合の要件
- Golem の **WASM component model** と既存言語資産の移行コスト
- LangGraph checkpointer を Temporal Activity で包む実装パターン

---

## 出典

- [Temporal](https://temporal.io/) / [temporalio/temporal (MIT)](https://github.com/temporalio/temporal) / [Temporal Docs](https://docs.temporal.io/) / [OpenAI Agents SDK 統合](https://temporal.io/blog/announcing-openai-agents-sdk-integration) / [Pydantic AI との統合](https://temporal.io/blog/build-durable-ai-agents-pydantic-ai-and-temporal)
- [Restate](https://restate.dev/) / [restatedev/restate](https://github.com/restatedev/restate) / [Restate vs Temporal](https://restate.dev/vs/temporal) / [What is Durable Execution?](https://restate.dev/what-is-durable-execution)
- [DBOS Transact](https://www.dbos.dev/dbos-transact) / [dbos-transact-py (MIT)](https://github.com/dbos-inc/dbos-transact-py) / [DBOS vs Temporal](https://www.dbos.dev/compare/dbos-vs-temporal) / [Postgres is all you need](https://www.dbos.dev/blog/postgres-is-all-you-need-for-durable-execution)
- [hatchet-dev/hatchet (MIT)](https://github.com/hatchet-dev/hatchet) / [Hatchet](https://hatchet.run/)
- [Inngest 自前ホストの発表](https://www.inngest.com/blog/inngest-1-0-announcing-self-hosting-support)
- [resonatehq/resonate (Apache 2.0)](https://github.com/resonatehq/resonate) / [Resonate Docs](https://docs.resonatehq.io/)
- [Golem](https://golem.cloud/) / [Golem Goes Open Source](https://golem.cloud/blog/golem-goes-open-source/) / [The Rise of the Agent Runtime](https://golem.cloud/blog/the-rise-of-the-agent-runtime/)
- [Conductor OSS FAQ](https://conductor-oss.github.io/conductor/devguide/faq.html) / [Conductor Architecture](https://conductor-oss.github.io/conductor/devguide/architecture/index.html) / [Orkes](https://orkes.io/what-is-conductor)
- [littlehorse-enterprises/littlehorse](https://github.com/littlehorse-enterprises/littlehorse) / [LittleHorse Docs](https://littlehorse.io/docs)
- [Dapr Workflow](https://docs.dapr.io/developing-applications/building-blocks/workflow/workflow-architecture/) / [Dapr Agents](https://docs.dapr.io/developing-ai/dapr-agents/) / [dapr/dapr-agents](https://github.com/dapr/dapr-agents)
- [AWS Lambda Durable Functions 実践ガイド](https://hidekazu-konishi.com/entry/aws_lambda_durable_functions_practical_guide.html) / [AWS 発表（2025年12月）](https://aws.amazon.com/about-aws/whats-new/2025/12/lambda-durable-multi-step-applications-ai-workflows/) / [.NET SDK GA](https://aws.amazon.com/about-aws/whats-new/2026/07/lambdadf-dotnet/)
- [Azure Durable Task 概要](https://learn.microsoft.com/en-us/azure/durable-task/common/what-is-durable-task) / [Durable Task Scheduler](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-task-scheduler/durable-task-scheduler) / [Consumption SKU GA](https://techcommunity.microsoft.com/blog/appsonazureblog/the-durable-task-scheduler-consumption-sku-is-now-generally-available/4506682)
- [Cloudflare Workflows GA](https://blog.cloudflare.com/workflows-ga-production-ready-durable-execution/) / [Dynamic Workflows](https://blog.cloudflare.com/dynamic-workflows/) / [Workflows 製品ページ](https://www.cloudflare.com/solutions/workflows/)
- [Vercel: A new programming model for durable execution](https://vercel.com/blog/a-new-programming-model-for-durable-execution) / [Vercel Workflows Docs](https://vercel.com/docs/workflows)
- [Upstash: Durable Workflow Engines in 2026](https://upstash.com/blog/durable-workflow-engines-compared-every-major-option-in-2026) / [upstash/workflow-js](https://github.com/upstash/workflow-js)
- [Convex Workflow component](https://www.convex.dev/components/workflow) / [get-convex/workflow](https://github.com/get-convex/workflow)
- [LangGraph Durable execution](https://docs.langchain.com/oss/python/langgraph/durable-execution) / [Why Checkpoints Aren't Durable Execution (Diagrid)](https://www.diagrid.io/blog/checkpoints-are-not-durable-execution-why-langgraph-crewai-google-adk-and-others-fall-short-for-production-agent-workflows)
- [Durable Execution: How Temporal, Restate, and DBOS Are Rethinking Distributed State](https://devstarsj.github.io/2026/04/03/durable-execution-temporal-restate-dbos-distributed-workflows-2026/)
- [9 Best Temporal Alternatives (ZenML)](https://www.zenml.io/blog/temporal-alternatives) / [10 Best Inngest Alternatives (Diagrid)](https://www.diagrid.io/infrastructure/10-best-inngest-alternatives-2026)
- [Kestra vs n8n vs Windmill](https://ossalt.com/guides/kestra-vs-n8n-vs-windmill-2026) / [awesome-workflow-engines](https://github.com/meirwah/awesome-workflow-engines)
