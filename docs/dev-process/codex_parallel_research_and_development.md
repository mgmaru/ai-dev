# Codexでの並列調査・並列開発の進め方

## 1. はじめに

Codexを使用した開発では、複数の作業を並列に進めることで、

- 技術調査の高速化
- 調査範囲の拡大
- 実装時間の短縮
- 複数観点からのレビュー
- コードベース探索の効率化

などが期待できます。

ただし、Codexにおける並列化には大きく分けて、

1. **SubAgentによる並列化**
2. **Git Worktreeによる並列化**

の2種類があります。

この2つは目的が異なります。

---


## 2. Codex・Git Worktree・Main Agent・SubAgentの全体像

Codexで並列開発や並列調査を考えるときは、まず次の関係を理解すると整理しやすくなります。

> **Git Worktree = コードを編集する作業環境**  
> **Main Agent = その作業を進める司令塔**  
> **SubAgent = Main Agentから仕事を委譲される担当者**

重要なのは、Git WorktreeそのものがAgentを管理する仕組みではないという点です。

より正確には、

> **Main Agentが特定のGit Worktreeを作業環境として利用し、そのMain Agentが必要に応じてSubAgentを起動する**

という関係になります。

### 2.1 全体構造

```text
Git Repository
│
├─ Worktree A
│    │
│    └─ Codex Chat / Main Agent A
│          │
│          ├─ SubAgent A-1
│          ├─ SubAgent A-2
│          └─ SubAgent A-3
│
├─ Worktree B
│    │
│    └─ Codex Chat / Main Agent B
│          │
│          ├─ SubAgent B-1
│          └─ SubAgent B-2
│
└─ Main working tree
     │
     └─ Codex Chat / Main Agent C
```

この図では、

- `Worktree A` と `Worktree B` が、それぞれ独立したGit上の作業場所
- 各Codex ChatのMain Agentが、そのWorktree上の作業を担当
- Main Agentが必要に応じてSubAgentへ調査・探索・レビューなどを委譲

という構成になっています。

### 2.2 役割の違い

| 要素 | 役割 | 例 |
|---|---|---|
| Git Repository | プロジェクト全体 | Webアプリのリポジトリ |
| Git Worktree | 独立したコード作業環境 | 認証機能用、検索機能用 |
| Main Agent | そのタスクの司令塔 | 要件整理、実装、統合、最終判断 |
| SubAgent | Main Agentから委譲された担当者 | 調査、探索、テスト、レビュー |

したがって、

```text
Git Worktree
=
並列開発の単位

Main Agent
=
各作業の司令塔

SubAgent
=
その作業内部での並列分担
```

と理解すると分かりやすくなります。

### 2.3 WorktreeとSubAgentは組み合わせて使える

Git WorktreeとSubAgentは、どちらか一方を選ぶ排他的な仕組みではありません。

例えば、

```text
Repository
│
├─ Worktree A：認証機能
│    │
│    └─ Main Agent
│         ├─ Explorer
│         │    └─ 既存認証コードを探索
│         │
│         └─ Reviewer
│              └─ 実装後のレビュー
│
└─ Worktree B：検索機能
     │
     └─ Main Agent
          ├─ Researcher
          │    └─ 検索ライブラリを調査
          │
          └─ Tester
               └─ テスト観点を整理
```

のように利用できます。

この場合、

1. **Worktreeによって複数の開発作業を分離する**
2. **各WorktreeのMain Agentが必要に応じてSubAgentを利用する**

という二段階の並列化になります。

### 2.4 すべてのWorktreeにSubAgentが必要なわけではない

小さなタスクであれば、

```text
Worktree
└─ Main Agent
```

だけで十分です。

一方で、調査やレビューなどを分担したい場合だけ、

```text
Worktree
└─ Main Agent
    ├─ SubAgent
    ├─ SubAgent
    └─ SubAgent
```

と増やします。

つまり、

> **Worktreeを作ったら必ずSubAgentを使う、という関係ではありません。**

SubAgentは必要な場合だけMain Agentが利用します。

### 2.5 具体例：認証機能を並列開発する場合

```text
Worktree: feature/auth
        │
        ▼
Main Agent
「認証機能を実装する」
        │
        ├─ SubAgent A
        │    └─ 既存認証コードを探索
        │
        ├─ SubAgent B
        │    └─ OAuth仕様を調査
        │
        └─ SubAgent C
             └─ テスト観点を整理
        │
        ▼
Main Agent
実装・統合・最終確認
```

この例では、

- `feature/auth` Worktreeがコードを編集する場所
- Main Agentが認証機能全体を担当
- SubAgentが調査・探索・テスト観点の整理を担当
- 最終的な実装判断と統合はMain Agentが担当

という構造になります。

---

## 3. SubAgentとGit Worktreeの基本的な違い

まず、最も重要な違いを整理します。

| 項目 | SubAgent | Git Worktree |
|---|---|---|
| 主な目的 | タスク・思考・調査の分担 | コード編集環境の分離 |
| イメージ | 作業する「人」 | 作業する「場所」 |
| 得意な用途 | 調査、探索、分析、レビュー | 複数機能の並列実装 |
| コード編集 | 可能だが競合に注意 | 独立した環境で編集可能 |
| 並列調査 | ◎ | △ |
| 並列開発 | △ | ◎ |
| 同じファイルを触る場合 | 競合しやすい | Worktree間では分離可能 |
| `/agent`による管理 | ○ | × |
| Agent数制御 | ○ | × |

簡単に表現すると、

```text
SubAgent
========

「誰に仕事を任せるか」

Main Agent
    │
    ├─ SubAgent A
    ├─ SubAgent B
    └─ SubAgent C
```

一方、Git Worktreeは、

```text
Git Worktree
============

「どこでコードを編集するか」

Repository
    │
    ├─ Worktree A
    ├─ Worktree B
    └─ Worktree C
```

という違いがあります。

---

## 4. 基本的な使い分け

大まかには次のように考えると分かりやすいです。

```text
やりたいこと
    │
    ├─ 調査・分析・レビュー
    │       │
    │       └─ SubAgent
    │
    └─ 複数のコード変更
            │
            └─ Git Worktree
```

具体的には以下のようになります。

| タスク | 推奨 |
|---|---|
| 技術調査 | SubAgent |
| ライブラリ比較 | SubAgent |
| GitHub Issue調査 | SubAgent |
| コードベース探索 | SubAgent |
| バグ原因調査 | SubAgent |
| セキュリティレビュー | SubAgent |
| テスト観点の洗い出し | SubAgent |
| 複数機能の同時実装 | Git Worktree |
| Frontend / Backend並列開発 | Git Worktree |
| 独立したBug Fix | Git Worktree |
| 同じファイルの同時修正 | 並列化を避ける |
| API仕様・DB Schema決定 | 先に直列で決める |

---

## 5. SubAgentとは

SubAgentは、Main Agentから仕事を分担される別のAgentです。

```text
Main Agent
    │
    ├─ SubAgent A
    │      └─ 調査
    │
    ├─ SubAgent B
    │      └─ 調査
    │
    └─ SubAgent C
           └─ レビュー
```

Main Agentが、

- タスクを分割する
- SubAgentに仕事を依頼する
- 結果を回収する
- 最終的な判断をする

というCoordinatorの役割を持ちます。

---

## 6. SubAgentは自然言語で起動・指示できる

Codexでは、SubAgentを利用するために複雑なAPIを書く必要はなく、自然言語で並列化を依頼できます。

```text
この技術について並列調査してください。

3つのSubAgentを使用してください。

Agent 1:
公式ドキュメントを中心に調査する。

Agent 2:
GitHubのソースコード、Issue、Changelogを中心に調査する。

Agent 3:
他のAgentとは独立して調査し、
特に反例・制約・注意点を探す。

すべてのSubAgentの終了を待ってください。

その後Main Agentが、

- 一致している点
- 食い違っている点
- 最も信頼できる結論
- 不確実な点

を整理してください。
```

この「自然言語による並列化」は、**SubAgentの機能**です。

Git Worktreeの機能を説明しているわけではありません。

---

## 7. SubAgentの役割には3種類ある

SubAgentの「役割」については、混同しやすいため整理しておく必要があります。

大きく分けて、

1. 組み込みAgent
2. Promptで一時的に与える役割
3. Custom Agent

があります。

```text
SubAgent
    │
    ├─ ① Built-in Agent
    ├─ ② Promptによる一時的な役割
    └─ ③ Custom Agent
```

### 7.1 組み込みAgent

Codexにはあらかじめ用意されているAgentがあります。

| Agent | 主な用途 |
|---|---|
| `default` | 汎用的な作業 |
| `worker` | 実装・コード変更 |
| `explorer` | コードベース探索・読み取り中心の作業 |

```text
Built-in Agents

default
    └─ 汎用

worker
    └─ 実装
       修正

explorer
    └─ 探索
       調査
       コードリーディング
```

---

## 8. Promptで一時的に役割を設定する

例えば、

```text
Agent 1:
公式ドキュメント担当

Agent 2:
GitHub担当

Agent 3:
反証担当
```

と指定した場合、`Agent 1` というAgentを新しく作成しているわけではありません。

その実行中だけ、

```text
SubAgent A
    ↓
公式ドキュメント担当

SubAgent B
    ↓
GitHub担当

SubAgent C
    ↓
反証担当
```

という役割を与えています。

つまり、**一時的な役割分担**です。

---

## 9. Custom Agent

同じ役割を何度も利用する場合は、Custom Agentとして定義できます。

```text
.codex/
└── agents/
    ├── researcher.toml
    ├── reviewer.toml
    └── tester.toml
```

| Custom Agent | 役割例 |
|---|---|
| `researcher` | 技術調査 |
| `reviewer` | コードレビュー |
| `tester` | テスト観点・テスト実行 |
| `security-reviewer` | セキュリティ確認 |
| `architect` | 設計レビュー |

なお、`researcher`、`reviewer`、`tester` などは標準搭載された名前ではなく、**自分で定義するCustom Agentの例**です。

---

## 10. 最初からCustom Agentを作る必要はない

並列調査を始める段階では、Custom Agentを作らなくても問題ありません。

まずは、

```text
3つのSubAgentを使用してください。

1. 公式情報担当
2. GitHub担当
3. 反証担当
```

程度で十分です。

その後、毎回同じ役割を指定するようになった場合にCustom Agent化すると効率的です。

```text
最初
  ↓
Promptで役割指定
  ↓
何度か運用
  ↓
よく使う役割が分かる
  ↓
Custom Agent化
```

---

## 11. `/agent` はSubAgentを管理する機能

Codex CLIでは、

```text
/agent
```

を使用して、実行中のAgentを確認できます。

これは**SubAgentの機能**です。Git Worktreeを確認するコマンドではありません。

```text
Main Agent
    │
    ├─ SubAgent A
    │   └─ 公式ドキュメント調査
    │
    ├─ SubAgent B
    │   └─ GitHub調査
    │
    └─ SubAgent C
        └─ 反証調査
```

---

## 12. SubAgentへ途中で追加指示する

実行中のSubAgentへ追加指示することもできます。

```text
GitHubを調査しているSubAgentに、
v2以降のIssueも確認するよう指示してください。
```

あるいは、

```text
公式ドキュメント担当のAgentに、
最新版だけを対象にするよう追加指示してください。
```

```text
Main Agent
    │
    ├── 指示
    ↓
SubAgent
    │
    ├── 調査中
    │
    ← 追加指示
    │
    └── 調査継続
```

---

## 13. 並列数の制御もSubAgentの機能

例えば、

```toml
[agents]
max_concurrent_threads_per_session = 4
```

という設定があります。

これは、**SubAgentの同時実行数を制御する設定**です。

Git Worktree数を制御する設定ではありません。

```text
Main Agent
    │
    ├─ SubAgent A ── Running
    ├─ SubAgent B ── Running
    ├─ SubAgent C ── Running
    │
    └─ SubAgent D ── Waiting
```

---

## 14. Git Worktreeとは

Git Worktreeは、Agentの種類ではありません。

1つのGitリポジトリから複数の作業ディレクトリを作り、それぞれ別の作業を進めるためのGitの仕組みです。

```text
Repository
    │
    ├─ Worktree A
    │      └─ Feature A
    │
    ├─ Worktree B
    │      └─ Feature B
    │
    └─ Worktree C
           └─ Bug Fix
```

Codexでは、それぞれのWorktree上で別のチャット・作業を進めることで並列開発できます。

```text
Codex Chat A
    ↓
Worktree A
    ↓
Frontend開発


Codex Chat B
    ↓
Worktree B
    ↓
Backend開発


Codex Chat C
    ↓
Worktree C
    ↓
Bug Fix
```

---

## 15. Worktreeが有効な理由

同じ作業ディレクトリで複数Agentにコードを書かせると、

```text
Agent A
    ↓
api.ts変更

Agent B
    ↓
api.ts変更

Agent C
    ↓
api.ts変更
```

となり、競合や意図しない上書きが発生しやすくなります。

Worktreeを分ければ、

```text
Worktree A
    └─ api.ts

Worktree B
    └─ api.ts

Worktree C
    └─ api.ts
```

と、それぞれ独立して編集できます。

そのため、

- **調査の並列化 → SubAgent**
- **コード編集の並列化 → Worktree**

という使い分けが有効です。

---

## 16. 並列開発では「依存関係」を先に考える

Worktreeを使用すれば何でも並列化できるわけではありません。

特に重要なのが、

> **依存する設計は先に決め、独立した実装だけを並列化する**

ことです。

例えば、

- API仕様
- DB Schema
- Type
- Interface

が決まっていない状態で、

- Backend
- Frontend
- Test

を同時に作らせると、それぞれ異なる前提で実装する可能性があります。

```text
① 要件
    ↓
② API Contract
    ↓
③ Interface / Type
    ↓
④ タスク分解
    ↓
 ┌───────┬───────┐
 ↓       ↓       ↓
Frontend Backend Test
```

---

## 17. 並列調査によって精度は上がるのか

並列調査によって、必ず正確性が上がるわけではありません。

ただし、

- 調査範囲
- 情報量
- 見落としの少なさ
- 複数観点からの検証

については改善しやすくなります。

悪い並列調査は、

```text
Agent A ─┐
Agent B ─┼─ 同じ質問・同じ情報源
Agent C ─┘
           ↓
        多数決
```

です。

同じモデル・同じ情報源を使用すれば、同じ間違いをする可能性があります。

---

## 18. 精度を上げる並列調査

おすすめなのは、各Agentに異なる観点を持たせる方法です。

```text
                 Main Agent
                      │
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
    Agent A        Agent B       Agent C
      │              │             │
  公式情報       実装・GitHub      反証
      │              │             │
        └─────────────┼─────────────┘
                      ↓
                  Main Agent
                      ↓
                比較・統合
```

### Agent A：一次情報

- 公式ドキュメント
- 公式仕様
- RFC
- 公式ブログ

### Agent B：実装情報

- GitHub Source
- Issue
- Pull Request
- Changelog
- 実際のコード

### Agent C：反証

- 制約
- 反例
- バージョン差
- Deprecated情報
- Edge Case
- 他Agentの見落とし

---

## 19. 単純多数決より「根拠」を見る

例えば、

```text
Agent A → Xが正しい
Agent B → Xが正しい
Agent C → Yが正しい
```

となっても、

```text
2対1だからX
```

と判断するべきではありません。

例えば、

```text
Agent A
ブログ記事

Agent B
Stack Overflow

Agent C
公式仕様
```

なら、Agent Cの方が信頼性が高い可能性があります。

そのためMain Agentでは、

- 人数
- 根拠
- 情報源
- 更新日時
- 一次情報か
- 実装との整合性

を確認する必要があります。

---

## 20. おすすめの並列調査構成

個人開発では、まず3Agent構成が扱いやすいです。

```text
                Main Agent
                     │
          ┌──────────┼──────────┐
          ↓          ↓          ↓
        Scout A    Scout B     Critic
          │          │          │
      公式情報    実装情報      反証
          │          │          │
          └──────────┼──────────┘
                     ↓
                 Main Agent
                     ↓
                  結論
```

### Scout A

一次情報中心。

```text
Official Docs
Specification
RFC
Official Blog
```

### Scout B

実装・現場情報中心。

```text
GitHub Source
Issue
Pull Request
Changelog
```

### Critic

他Agentとは独立して、

```text
反例
制約
バージョン差
誤解
Edge Case
```

を探します。

---

## 21. 調査用Prompt例

```text
このテーマについて並列調査してください。

3つのSubAgentを使用してください。

Agent 1:
公式ドキュメント・公式仕様など一次情報を中心に調査してください。

Agent 2:
GitHubのSource Code、Issue、Pull Request、Changelogなど、
実装側の情報を中心に調査してください。

Agent 3:
他のAgentとは独立して調査してください。
特に反例、制約、バージョン差、注意点、
一般的な説明と実際の挙動が異なる点を探してください。

各Agentは調査根拠と情報源を示してください。

すべてのAgentの終了を待ってください。

最後にMain Agentが、

- 各Agentの調査結果
- 一致点
- 相違点
- 根拠の強さ
- 不確実な点
- 最終的な結論

を整理してください。

単純な多数決ではなく、
一次情報と根拠の強さを優先して判断してください。
```

---

## 22. 将来的なCustom Agent構成

並列調査を何度も行うようになった場合は、

```text
.codex/
└── agents/
    ├── official-researcher.toml
    ├── github-researcher.toml
    ├── critic.toml
    ├── reviewer.toml
    └── tester.toml
```

のようにCustom Agent化できます。

```text
Main Agent
    │
    ├─ official-researcher
    ├─ github-researcher
    ├─ critic
    ├─ reviewer
    └─ tester
```

ただし、最初からここまで作り込む必要はありません。

---

## 23. おすすめの段階的導入

### Stage 1：自然言語だけ

```text
3つのSubAgentを使って、
公式・GitHub・反証の3方向から調査してください。
```

↓

### Stage 2：並列数を制御

```toml
[agents]
max_concurrent_threads_per_session = 3
```

↓

### Stage 3：よく使う役割を固定

毎回同じ役割を使うようになったら、

```text
researcher
critic
reviewer
tester
```

などをCustom Agent化します。

↓

### Stage 4：AGENTS.mdでルール化

さらに、

```text
調査タスクは原則3Agent
公式情報・実装情報・反証に分割
```

などをAGENTS.mdへ記述します。

---

## 24. 最終的なおすすめ構成

```text
                     Codex
                       │
                  Main Agent
                       │
       ┌───────────────┼───────────────┐
       │                               │
       ▼                               ▼
   SubAgents                     Git Worktrees
       │                               │
       │                               │
 調査・分析・レビュー              コード並列開発
       │                               │
 ┌─────┼─────┐                 ┌───────┼───────┐
 ↓     ↓     ↓                 ↓       ↓       ↓
Docs GitHub Critic         Frontend Backend Bugfix
       │                               │
       └──────────────┬────────────────┘
                      ↓
                  Main Agent
                      ↓
               統合・レビュー
                      ↓
                    Merge
```

---

## 25. 判断基準

迷った場合は、次の質問で判断できます。

### Q1. コードを書き換える必要があるか？

**No**

→ SubAgent

**Yes**

→ Q2へ

### Q2. 他の作業と同じファイルを変更する可能性があるか？

**Yes**

→ 原則として並列化しない

**No**

→ Git Worktree候補

### Q3. 目的は「別の観点から考えること」か？

**Yes**

→ SubAgent

```text
調査
レビュー
分析
探索
   ↓
SubAgent


独立した機能実装
Bug Fix
Frontend / Backend
   ↓
Git Worktree
```

---

## 26. まとめ

| ポイント | 内容 |
|---|---|
| SubAgentとは | タスクを分担するAgent |
| Worktreeとは | Git上の独立した作業場所 |
| 自然言語で並列化 | SubAgent |
| `/agent` | SubAgentの確認・切り替え |
| Agent同時実行数 | SubAgentの設定 |
| `default / worker / explorer` | 組み込みAgent |
| `researcher / reviewer`等 | Custom Agentの例 |
| `Agent 1: 公式担当` | Promptによる一時的役割 |
| 並列調査 | SubAgentが適する |
| 並列実装 | Worktreeが適する |
| 調査精度 | Agent数より役割分離・情報源・反証が重要 |

最も簡単に覚えるなら、

```text
SubAgent
=
「誰に仕事を分担するか」


Git Worktree
=
「どこでコードを編集するか」
```

です。

そして今回のような技術調査では、

```text
Main Agent
    │
    ├─ 公式情報担当
    ├─ GitHub・実装担当
    └─ 反証・制約担当
            │
            ↓
        Main Agent
            │
            ↓
       比較・統合・結論
```

という **3つ程度のSubAgentを使った並列調査**から始めるのが適しています。

最初はPromptによる役割指定だけで運用し、同じ役割を繰り返し利用するようになってからCustom AgentやAGENTS.mdへ発展させるのが、過剰に複雑化せず扱いやすい方法です。


---

## 27. 「SubAgentの役割は3種類」という整理についての補足

前述した、

1. 組み込みAgent
2. Promptで一時的に与える役割
3. Custom Agent

という3分類は、**Codex公式が定義している正式な「3種類のAgent分類」ではありません**。

理解しやすくするために、本ドキュメント内で整理した分類です。

Codex公式ドキュメント上では、主に次の仕組みが説明されています。

- Codexに標準搭載されている **Built-in Agent**
  - `default`
  - `worker`
  - `explorer`
- ユーザーがTOMLで定義する **Custom Agent**
- Promptによって「このSubAgentは公式ドキュメント担当」などと、その場で仕事を割り当てる方法

つまり、

```text
SubAgentへの役割付け
        │
        ├─ Built-in Agentを使う
        │
        ├─ Custom Agentを使う
        │
        └─ Promptでその場の仕事内容を指定する
```

と理解する方が正確です。

公式ドキュメント:
- https://learn.chatgpt.com/docs/agent-configuration/subagents

---

## 28. Main Agentには役割設定がないのか

結論としては、

> **Main Agentにも指示や設定を与えることはできる。**
>
> ただし、`default` / `worker` / `explorer` やCustom Agentという仕組みは、主に「spawnされるSubAgentをどう構成するか」という文脈の機能である。

という理解が適切です。

### 28.1 Main AgentとSubAgentの違い

```text
Main Agent
│
│  ユーザーとの会話
│  要件整理
│  判断
│  SubAgentへの委譲
│  結果統合
│
├─ SubAgent A
├─ SubAgent B
└─ SubAgent C
```

Main Agentは、現在のCodexセッションの中心となるAgentです。

一方でSubAgentは、

> Main Agentから特定のタスクを委譲されて起動するAgent

です。

Codex公式では、SubAgentを「delegated agent」、SubAgentが作業する場所を「agent thread」と説明しています。

### 28.2 Main Agentもカスタマイズできる

Main Agentに対しても、

- 通常のPrompt
- `AGENTS.md`
- `~/.codex/config.toml`
- プロジェクト固有設定
- モデル・推論設定
- 権限・Sandbox設定

などを通して、動作方針を設定できます。

特に `AGENTS.md` は、Codexが作業を始める前に読み込むプロジェクト指示です。

```text
AGENTS.md
    │
    ↓
Main Codex Session
    │
    ├─ コーディング規約
    ├─ テスト方針
    ├─ 使用ライブラリ
    ├─ 設計方針
    └─ SubAgent利用方針
```

例えば、

```markdown
# Development rules

- 変更前に既存実装を調査する
- 実装後はテストを実行する
- 技術調査ではSubAgentを利用する
- 公式情報を優先する
```

といったルールをMain Agentを含むCodexの作業方針として与えられます。

公式ドキュメント:
- https://learn.chatgpt.com/docs/agent-configuration/agents-md

### 28.3 「Custom Agent」とMain Agentは少し役割が違う

Custom Agentファイルは、公式ドキュメントでは **spawned sessionsに適用する設定レイヤー** と説明されています。

そのため、

```text
Main AgentをCustom Agentにする
```

というより、

```text
Main Agent
    │
    ├─ researcher Custom Agentをspawn
    ├─ reviewer Custom Agentをspawn
    └─ tester Custom Agentをspawn
```

と考える方が分かりやすいです。

Main Agent自身の恒常的なルールは、

```text
config.toml
+
AGENTS.md
+
Prompt
```

で設定し、

専門的なSubAgentは、

```text
.codex/agents/*.toml
```

で設定する、という役割分担が自然です。

---

## 29. Custom Agentを作成するメリット

Custom Agentの最大のメリットは、

> **特定の仕事に必要な「役割・モデル・推論量・権限・ツール」をあらかじめセットとして定義できること**

です。

公式ドキュメントではCustom Agentごとに、

- `name`
- `description`
- `developer_instructions`
- `model`
- `model_reasoning_effort`
- `sandbox_mode`
- `mcp_servers`
- `skills.config`

などを設定できます。

### 29.1 毎回同じPromptを書く必要がなくなる

Custom Agentを使わない場合、

```text
このAgentは公式ドキュメントだけを調査してください。
コードは変更しないでください。
一次情報を優先してください。
バージョン差も確認してください。
根拠となるURLを返してください。
```

と毎回指定する必要があります。

Custom Agentなら、

```text
docs-researcher
```

というAgentへ一度設定しておけば、

```text
docs-researcherにこのライブラリを調査させて
```

のように利用できます。

```text
毎回長いPrompt
        ↓
 Custom Agent化
        ↓
役割を再利用
```

となります。

### 29.2 Agentの役割がぶれにくくなる

公式ドキュメントでは、良いCustom Agentは **narrow and opinionated**、つまり役割を狭く明確にすることが推奨されています。

例えばResearch Agentに、

```toml
developer_instructions = """
公式ドキュメントと一次情報を優先する。
コードは変更しない。
不明な点は推測しない。
バージョン差を確認する。
根拠を必ず示す。
"""
```

と設定しておけば、調査途中で、

```text
ついでにコードも修正する
```

といった役割逸脱を減らせます。

### 29.3 タスクに合わせてモデルを変えられる

これはCustom Agentの大きなメリットです。

```text
Main Agent
    │
    ├─ Explorer
    │     └─ 高速・低コストモデル
    │
    ├─ Researcher
    │     └─ 中程度の推論
    │
    └─ Reviewer
          └─ 高い推論能力
```

のような構成にできます。

例えば、

```toml
name = "reviewer"
model_reasoning_effort = "high"
```

として、レビューだけ推論量を増やすことができます。

反対に、大量のファイル探索などは高速モデルに任せることで、

- 品質
- 実行速度
- Token使用量

のバランスを調整できます。

### 29.4 Agentごとに権限を制限できる

例えば調査Agentなら、

```toml
sandbox_mode = "read-only"
```

としておけば、

```text
Researcher
    │
    ├─ ファイルを読む     ○
    ├─ コードを調査する   ○
    └─ コードを書き換える ×
```

という安全な構成にできます。

これは特に並列調査で有効です。

複数のResearch Agentが誤ってソースコードを変更することを防げます。

### 29.5 AgentごとにMCPやSkillsを変えられる

Custom Agentでは、

```text
mcp_servers
skills.config
```

なども設定できます。

そのため、

```text
Main Agent
    │
    ├─ docs-researcher
    │      └─ Documentation MCP
    │
    ├─ code-reviewer
    │      └─ Repository tools
    │
    └─ tester
           └─ Test関連tools
```

のように、

> **仕事に必要なツールだけを持った専門Agent**

を作れます。

これによって不要なツール選択を減らし、Agentの役割を明確にできます。

---

## 30. Custom Agentを作ると精度は上がるのか

結論は、

> **Custom Agentを作っただけで自動的に精度が上がるわけではない。**
>
> しかし、適切に設計すれば結果の品質や再現性を上げやすくなる。

です。

### 精度向上につながる要因

```text
Custom Agent
    │
    ├─ 専門的なInstructions
    ├─ 適切なModel
    ├─ 適切なReasoning Effort
    ├─ 必要なToolだけを使用
    └─ 明確な責務
            │
            ↓
     Agentの迷い・脱線を削減
            │
            ↓
       品質・再現性向上
```

例えば、通常Agentに単純に、

```text
このライブラリを調査して
```

と依頼するよりも、

```text
公式ドキュメントを一次情報として使用する
GitHub Issueだけで結論を出さない
対象Versionを確認する
事実と推測を分ける
根拠を示す
```

と設定されたResearcherを使った方が、調査品質は安定しやすくなります。

### ただし、悪い設定では逆効果になる

例えば、

```text
「公式情報以外は絶対に使用しない」
```

と過度に制限すると、

- 実装上の問題
- GitHub Issue
- 現場で発生しているBug
- ドキュメントと実装の不一致

を見落とす可能性があります。

そのためCustom Agentは、

> **精度を自動的に上げる機能ではなく、精度を上げやすい作業条件を固定する仕組み**

と考えるのが適切です。

---

## 31. Custom Agentはセッションを共有するのか

ここは重要です。

結論としては、

> **「Main AgentとCustom Agentが同じセッションを丸ごと共有する」と考えるのは正確ではありません。**

CodexではSubAgentごとに **Agent Thread** が作られます。

```text
Main Thread
    │
    ├─ Agent Thread A
    ├─ Agent Thread B
    └─ Agent Thread C
```

各SubAgentはそれぞれのthreadで作業し、結果をMain Threadへ返します。

```text
Main Agent
    │
    ├── Task ──→ Researcher Thread
    │                 │
    │                 └─ 調査
    │
    │     ←─ Summary / Result
    │
    └─ Main Agentが統合
```

この仕組みによって、調査ログや大量の中間出力をMain Agentの会話へすべて詰め込まずに済みます。

公式ドキュメントでも、この仕組みのメリットとして、

- Main Agentを要件・判断・最終出力に集中させる
- 探索やログなどのノイズをSubAgent側へ移す
- Raw outputではなく要約した結果をMain Agentへ返す

ことが説明されています。

### 31.1 「共有」より「継承」と「結果返却」

より正確には、

```text
完全なセッション共有
        ×

設定の継承
        +
独立Threadで実行
        +
結果をMainへ返却
        ○
```

です。

SubAgentは設定によって、

- 親AgentのModel / Reasoning設定
- Sandbox policy
- Permission mode
- その他のsession設定

などを継承する場合があります。

Custom Agent側で設定した項目は上書きできます。

例えば、

```text
Main Agent
model = 高性能モデル
sandbox = workspace-write

        │ spawn
        ↓

Researcher
model = 高速モデル       ← Custom Agentで変更
sandbox = read-only      ← Custom Agentで制限
```

といった構成ができます。

### 31.2 Custom Agentは「記憶を共有する機能」ではない

Custom Agentを作ったからといって、

```text
Researcherが前回調査した内容を
次回自動的にすべて覚えている
```

という仕組みになるわけではありません。

Custom Agentファイルが保存しているのは主に、

```text
役割
設定
指示
Model
Tool
Permission
```

です。

したがって、

```text
Custom Agent
≠
共有Memory

Custom Agent
=
再利用可能なAgent設定
```

と理解すると分かりやすいです。

---

## 32. Custom Agentを使うべきタイミング

Custom Agent化が向いているのは、

> **同じ種類の仕事を何度も行う場合**

です。

### Custom Agentを作らなくてもよい場合

```text
今回だけ特殊な調査をする
        ↓
Promptで役割指定
```

### Custom Agentを作るとよい場合

```text
毎回ライブラリ調査を行う
        ↓
docs-researcher

毎回コードレビューを行う
        ↓
reviewer

毎回テスト観点を確認する
        ↓
tester
```

判断基準は、

```text
同じ役割を繰り返す？
        │
   ┌────┴────┐
   │         │
  No        Yes
   │         │
   ↓         ↓
Prompt    Custom Agent
```

で考えると簡単です。

---

## 33. 今回の用途でおすすめする構成

技術調査を頻繁に行うのであれば、最終的には次の3Agent構成が使いやすいです。

```text
                   Main Agent
                       │
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
official-researcher  repo-researcher  critic
        │              │              │
     公式情報       Source/Issue      反証
        │              │              │
        └──────────────┼──────────────┘
                       ↓
                   Main Agent
                       ↓
                  比較・統合
```

### `official-researcher`

目的:

- 公式Docs
- RFC
- 公式仕様
- Version確認

設定例:

```text
read-only
中程度のReasoning
一次情報優先
```

### `repo-researcher`

目的:

- Source Code
- GitHub Issue
- Pull Request
- Changelog

設定例:

```text
read-only
高速モデル
コード探索重視
```

### `critic`

目的:

- 他Agentの結論を疑う
- 反例を探す
- Version差を確認
- 根拠不足を指摘する

設定例:

```text
read-only
高めのReasoning
反証中心
```

この構成では、

```text
専門化
+
独立調査
+
反証
+
Main Agentによる統合
```

を組み合わせられるため、単純に同じPromptを3Agentへ送るより、調査品質を高めやすくなります。

---

## 34. 追加調査を踏まえた最終整理

| 疑問 | 結論 |
|---|---|
| 「役割は3種類」は公式分類か | **いいえ。本ドキュメントで理解のために整理した分類** |
| Built-in Agentは何か | `default` / `worker` / `explorer` |
| Custom Agentは何か | spawned SubAgent用の再利用可能な設定 |
| Promptの役割指定は何か | その実行だけの一時的な仕事分担 |
| Main Agentに役割設定はないのか | **ある。Prompt・AGENTS.md・config等で設定可能** |
| Main Agentを`worker`等にするのか | 通常はその理解ではなく、`worker`等はspawnするAgentの役割として考える |
| Custom Agentで精度は上がるか | 自動保証はないが、専門化・モデル選択・指示固定により改善しやすい |
| セッションを共有できるか | 完全共有ではない。SubAgentは独立Threadで動き、設定を一部継承し結果をMainへ返す |
| 過去の調査内容を自動記憶するか | Custom Agent自体にはその意味での共有Memory機能はない |
| 最大のメリット | **役割・指示・モデル・推論量・権限・ツールを再利用可能な形で固定できること** |

最終的には、

```text
Main Agent
=
司令塔・判断・統合
+
Prompt / AGENTS.md / configで方針設定


Custom SubAgent
=
専門家
+
専用Instructions
+
専用Model
+
専用Reasoning
+
専用Tool
+
専用Permission
```

という関係で理解すると分かりやすくなります。

---

## 35. 参考資料

- OpenAI Codex Subagents  
  https://learn.chatgpt.com/docs/agent-configuration/subagents

- OpenAI Codex AGENTS.md  
  https://learn.chatgpt.com/docs/agent-configuration/agents-md


---

## 36. Codexでトークン消費を抑える方法

SubAgentを活用すると並列化によって調査や開発を高速化できますが、Agent数が増えるほど総Token使用量も増えやすくなります。

そのため、並列化では、

> **「たくさんAgentを起動する」ことではなく、「必要なAgentだけを、必要な範囲で動かす」こと**

が重要です。

---

### 36.1 軽量モデルをCustom Agentに設定する

Custom Agentでは、役割ごとに異なるModelを設定できます。

例えば、

```text
Main Agent
    │
    ├─ Explorer
    │     └─ 軽量モデル
    │
    ├─ Researcher
    │     └─ 軽量～中程度のモデル
    │
    └─ Critic / Reviewer
          └─ 高性能モデル
```

のように、

- 単純な探索
- ファイル検索
- 情報収集
- ログ確認

などは軽量モデルへ任せ、

- 設計判断
- 複雑なレビュー
- 反証
- セキュリティ確認
- 最終判断

などだけ高性能モデルへ任せる構成が有効です。

Custom Agentでは、例えば次のようにModelを指定できます。

```toml
name = "repo-explorer"

model = "<lightweight-model>"
model_reasoning_effort = "low"

sandbox_mode = "read-only"
```

この方法は、

- コスト
- 実行速度
- 高性能モデルの利用量

を抑えるうえで有効です。

ただし、

> **軽量モデルを使うことと、処理するToken数そのものを減らすことは完全には同じではありません。**

同じ10,000 Tokenを処理した場合、Modelが軽量でもToken数そのものは10,000です。

したがって、本当に総Token量を減らすには、以下の方法も組み合わせる必要があります。

---

## 37. 不要なSubAgentを起動しない

最も基本的で効果が大きい方法です。

例えば、単純な質問なのに、

```text
Main Agent
    │
    ├─ Agent A
    ├─ Agent B
    ├─ Agent C
    └─ Agent D
```

と毎回4 Agentを起動すると、調査内容が重複し、Token消費が大きくなります。

タスクの難易度に応じて、Agent数を変える方が効率的です。

| タスク | 推奨構成 |
|---|---|
| 単純な質問・小さな修正 | Main Agentのみ |
| コード探索が必要 | Main + Explorer |
| 外部技術調査 | Main + Researcher |
| 複数観点が必要な調査 | Main + 2～3 SubAgents |
| 重要な技術選定 | Main + Researcher + Critic |
| 重大なレビュー | 必要に応じて複数Reviewer |

つまり、

```text
簡単
  ↓
Mainだけ

少し複雑
  ↓
Main + 1 Agent

重要
  ↓
Main + 2～3 Agent
```

くらいから始める方が無駄が少なくなります。

---

## 38. Agent同士の重複調査を減らす

並列調査でTokenを浪費しやすい典型例は、

```text
Agent A
「React Queryについて全部調査」

Agent B
「React Queryについて全部調査」

Agent C
「React Queryについて全部調査」
```

のように、全Agentへほぼ同じ仕事を与えることです。

これでは、

```text
Agent A  █████████████
Agent B  █████████████
Agent C  █████████████

       大量の重複
```

が発生します。

代わりに、

```text
Agent A
公式ドキュメント

Agent B
GitHub Source / Issue / Changelog

Agent C
反例・制約・Version差
```

と分けます。

```text
Agent A  ████ 公式
Agent B       ████ 実装
Agent C            ████ 反証
```

このように、

> **Agent数を増やすのではなく、調査空間を分割する**

ことが重要です。

これはToken削減だけでなく、調査の網羅性向上にもつながります。

---

## 39. Reasoning Effortを必要以上に上げない

Custom AgentではModelだけでなく、Reasoning Effortも役割ごとに調整できます。

例えば、

```text
コード検索
    ↓
Low

公式Docs確認
    ↓
Low / Medium

通常の実装
    ↓
Medium

複雑な設計レビュー
    ↓
High

重大なバグ・セキュリティ
    ↓
High
```

のように使い分けます。

例えばExplorerなら、

```toml
model_reasoning_effort = "low"
```

として、

Reviewerなら、

```toml
model_reasoning_effort = "high"
```

とする考え方です。

すべてのAgentをHigh Reasoningにするより、

```text
単純作業
    ↓
軽量Model + Low Reasoning

重要な判断
    ↓
高性能Model + High Reasoning
```

の方が効率的です。

---

## 40. 調査範囲を明確にする

Tokenを増やす原因の一つが、曖昧なPromptです。

例えば、

```text
このライブラリについて詳しく調査してください。
```

だけだと、

```text
公式Docs
   ↓
GitHub
   ↓
Issue
   ↓
Blog
   ↓
関連ライブラリ
   ↓
さらに探索……
```

と調査範囲が広がる可能性があります。

代わりに、

```text
公式ドキュメントから以下の3点だけ調査してください。

1. Cacheの有効期限
2. invalidateの挙動
3. Version 5での変更点
```

のように対象を限定します。

---

## 41. 終了条件を指定する

調査対象だけでなく、

> **「いつ調査を終了するのか」**

も指定すると効率が上がります。

例えば、

```text
次の3点について一次情報を確認できた時点で
追加調査を終了してください。
```

あるいは、

```text
公式ドキュメントとChangelogで結論が一致した場合、
追加のBlog調査は不要です。
```

と指定します。

基本形としては、

```text
Scope
  ↓
Stop Condition
  ↓
Output Format
```

の3つをPromptへ含めるのがおすすめです。

---

## 42. SubAgentからMain Agentへ返す情報を絞る

SubAgentが調査したすべての情報をMain Agentへ返す必要はありません。

例えばSubAgentが、

```text
GitHub Issue 50件
Source Code 20ファイル
Documentation 10ページ
```

を確認したとしても、

Main Agentへは、

```text
結論
+
重要な根拠
+
不確実な点
```

だけ返せば十分な場合があります。

```text
SubAgent

大量の探索
████████████████████████

          ↓ 要約

Main Agent

結論      ███
根拠      ██
不確実性  █
```

これにより、

- Main AgentのContext肥大化
- 同じ情報の再処理
- 後続推論でのToken増加

を抑えやすくなります。

例えばCustom Agentへ、

```text
Main Agentには以下だけ返すこと。

- 結論
- 根拠3件以内
- 不確実な点
- 次に確認すべき事項

探索ログそのものは返さない。
```

と設定しておく方法があります。

---

## 43. `AGENTS.md` を必要以上に巨大化させない

`AGENTS.md` はプロジェクト共通ルールとして非常に便利ですが、常に必要となる指示だけを入れる方が効率的です。

例えば、

```text
AGENTS.md
├─ React詳細ルール 200行
├─ Git詳細ルール 200行
├─ Testing詳細ルール 300行
├─ Deployment詳細ルール 200行
└─ Research詳細ルール 300行
```

のように巨大化させるより、

```markdown
# Core rules

- Existing implementation patterns take precedence.
- Keep changes minimal.
- Run tests relevant to changed code.
- Prefer official sources for technical research.
- Use SubAgents only for independent tasks.
```

など、

> **ほぼすべての作業で必要になるルールだけを残す**

方が扱いやすくなります。

専門的なルールは、

- Custom Agent
- Skill
- 必要時のPrompt
- 専用ドキュメント

などへ分離する方がよい場合があります。

---

## 44. Main Agentを「司令塔」に寄せる

Main Agent自身が、

```text
調査
+
コード探索
+
ログ解析
+
実装
+
テスト
+
レビュー
+
統合
```

をすべて行うと、Main ThreadのContextが大きくなります。

そこで、

```text
                    Main Agent
                 要件・判断・統合
                       │
       ┌───────────────┼───────────────┐
       ↓               ↓               ↓
   Explorer        Researcher        Tester
   軽量Model        軽量Model        軽量Model
     Low            Low/Medium          Low
       │               │               │
       └───────────────┼───────────────┘
                       ↓
                  簡潔な結果
                       ↓
                   Main Agent
                       ↓
                    最終判断
```

とします。

ポイントは、

> **大量の探索をMain Agentから切り離し、Main Agentには判断に必要な情報だけを戻すこと**

です。

これはToken効率だけでなく、Main Agentが重要な情報へ集中しやすくなるというメリットもあります。

---

## 45. Custom Agentは「繰り返す作業」に使う

Custom Agentを作ること自体にも設定や管理コストがあります。

したがって、

```text
今回だけの作業
        ↓
Promptで指定
```

で十分です。

一方、

```text
同じ役割を何度も使う
        ↓
Custom Agent
```

とします。

例えば、

```text
毎回コードベースを探索する
        ↓
repo-explorer

毎回公式Docsを確認する
        ↓
official-researcher

毎回実装後レビューする
        ↓
reviewer
```

という使い方です。

目安として、

> **同じ指示を2～3回コピペするようになったらCustom Agent化を検討する**

くらいで十分です。

---

## 46. 並列数制限は「Token削減」より暴走防止

例えば、

```toml
[agents]
max_concurrent_threads_per_session = 3
```

として同時実行数を制限することもできます。

ただし注意点があります。

例えば5 Agentを、

```text
同時5 Agent
```

で実行する代わりに、

```text
3 Agent
↓
終了
↓
残り2 Agent
```

と実行しても、最終的に5 Agentすべてが同じ量の処理をすれば、総Token量は大きくは変わりません。

したがって並列数制限は、

- Agentの大量spawn防止
- 意図しない並列化防止
- Resource管理
- 実行状況を把握しやすくする

ための仕組みとして考える方が適切です。

総Token削減には、

```text
Agentそのものを減らす
```

方が直接的に効きます。

---

## 47. Token効率を考えたおすすめAgent構成

個人開発では、例えば次の構成が扱いやすいです。

```text
AGENTS.md
│
│ プロジェクト共通ルール
│
▼
Main Agent
高性能Model
Medium
│
├─ repo-explorer
│    軽量Model
│    Low
│    read-only
│
├─ researcher
│    軽量～中程度Model
│    Low / Medium
│    read-only
│
└─ critic
     高性能Model
     High
     read-only
     ※重要な場合だけ起動
```

運用としては、

```text
通常
↓
Mainだけ


コード探索が必要
↓
Main + Explorer


外部技術調査が必要
↓
Main + Researcher


重要な技術選定
↓
Main
├─ Explorer
├─ Researcher
└─ Critic
```

と、必要に応じてAgentを追加します。

---

## 48. Token削減方法の優先順位

実用上は、次の順番で考えると分かりやすいです。

| 優先度 | 方法 | 主な効果 |
|---|---|---|
| ★★★★★ | 不要なSubAgentを起動しない | 総Token量を直接削減 |
| ★★★★★ | Agent間の重複作業をなくす | 無駄な調査Tokenを削減 |
| ★★★★☆ | 調査範囲を限定する | 探索量を削減 |
| ★★★★☆ | 終了条件を指定する | 過剰探索を防止 |
| ★★★★☆ | Reasoning Effortを下げる | 推論コストを削減 |
| ★★★★☆ | 軽量Modelを利用する | コスト・速度改善 |
| ★★★☆☆ | Main Agentへ要約だけ返す | Context肥大化を防止 |
| ★★★☆☆ | `AGENTS.md`を簡潔にする | 常時読み込む指示を削減 |
| ★★☆☆☆ | 並列数上限を設定する | 暴走・大量spawn防止 |

---

## 49. Token消費を考えるときの基本式

厳密な料金計算式ではありませんが、考え方としては、

```text
総Token消費
≈
Agent数
×
各Agentが読むContext
×
各Agentの探索量
×
推論量
```

と考えると分かりやすいです。

したがって、本当に重要なのは、

```text
① 何Agent起動するか

② 各Agentに何を読ませるか

③ どこまで調査させるか

④ どのReasoning Effortを使うか

⑤ どのModelを使うか
```

です。

「軽量Modelを使う」だけではなく、

```text
Agent数を減らす
+
重複を減らす
+
Scopeを狭める
+
終了条件を決める
+
出力を要約する
+
必要なAgentだけ高性能Modelを使う
```

を組み合わせることで、品質を維持しながらToken効率を高められます。

---

## 50. Token効率まで含めた最終的な役割分担

今回までの議論をまとめると、

```text
AGENTS.md
=
プロジェクト全体の共通ルール


Custom Agent TOML
=
繰り返し利用する専門家の設定
+
Model
+
Reasoning
+
権限
+
Tool


Prompt
=
今回だけの具体的な仕事


Main Agent
=
司令塔
+
必要なSubAgentだけ起動
+
結果の比較・統合
+
最終判断
```

となります。

Token効率の観点では、

> **「すべてを高性能Agentで処理する」のではなく、「単純作業は軽量Agent、重要な判断だけ高性能Agent」にする**

ことが基本です。

さらに、

> **Agent数・探索範囲・重複・Reasoning Effortを抑えることの方が、総Token量そのものを減らすうえでは重要**

です。

このため、個人開発では、

```text
通常はMain Agentだけ
        ↓
必要なときだけ専門SubAgent
        ↓
単純作業は軽量Model
        ↓
重要な判断だけ高性能Model
```

という運用が、品質・速度・Token消費のバランスを取りやすい方法です。


---

## 50. Custom Agent TOMLファイルの主要な設定項目

Custom Agentは、次のいずれかのディレクトリへ1 Agentにつき1つのTOMLファイルとして定義します。

```text
個人共通
~/.codex/agents/

プロジェクト専用
.codex/agents/
```

例えば、

```text
.codex/
└── agents/
    ├── researcher.toml
    ├── reviewer.toml
    └── repo-explorer.toml
```

という構成です。

CodexはこれらのTOMLを、**spawnされたSubAgentのセッション設定を上書きするConfiguration Layer**として読み込みます。

そのためCustom Agent TOMLは、単なる「役割名ファイル」ではなく、

```text
役割
+
振る舞い
+
Model
+
Reasoning
+
Sandbox
+
MCP
+
Skills
```

などをまとめて設定できるファイルです。

---

## 51. 必須設定は3項目

Custom Agent TOMLで必須なのは、次の3項目です。

| 設定項目 | 型 | 必須 | 役割 |
|---|---|---:|---|
| `name` | string | ○ | Agentの識別名 |
| `description` | string | ○ | Codexが「いつこのAgentを使うべきか」を判断する説明 |
| `developer_instructions` | string | ○ | Agentの具体的な振る舞い・制約・仕事の進め方 |

最小構成は次のようになります。

```toml
name = "researcher"

description = """
Technical research agent for investigating libraries,
frameworks, APIs, and specifications.
"""

developer_instructions = """
Investigate the requested technical topic.
Prefer primary and official sources.
Separate verified facts from assumptions.
Return concise findings to the parent agent.
"""
```

この3項目だけでもCustom Agentとして定義できます。

---

## 52. `name` — Agentの識別名

```toml
name = "official_researcher"
```

`name` は、CodexがそのCustom Agentを識別するときに使用する名前です。

例えばMain Agentから、

```text
official_researcherを使って公式仕様を確認してください。
```

のように参照できます。

重要なのは、

> **ファイル名ではなく `name` がAgentの正式な識別子**

という点です。

例えば、

```text
ファイル名
official-researcher.toml

name
official_researcher
```

でも動作上の識別には `official_researcher` が使われます。

ただし管理しやすくするため、

```text
official-researcher.toml
↓
name = "official_researcher"
```

のように、ファイル名とAgent名を近い名称にするのがおすすめです。

### 組み込みAgentと同名にした場合

Codexには、

```text
default
worker
explorer
```

というBuilt-in Agentがあります。

Custom Agentで同じ `name` を定義した場合は、**Custom Agent側が優先されます**。

例えば、

```toml
name = "explorer"
```

とすると、標準の`explorer`ではなく、自分で定義したCustom Agentが利用されます。

そのため、意図しない上書きを避けるなら独自名にする方が安全です。

---

## 53. `description` — 「いつ使うAgentなのか」を説明する

```toml
description = """
Read-only codebase explorer for locating relevant files,
symbols, dependencies, and execution paths.
"""
```

`description` は単なる説明文ではなく、

> **このAgentが何を担当するAgentなのかをCodexへ伝える重要な情報**

です。

例えば、

```toml
description = "Research agent."
```

よりも、

```toml
description = """
Documentation specialist for verifying APIs,
configuration options, and version-specific behavior
using primary documentation.
"""
```

の方が役割が明確になります。

Custom Agentは、

> **narrow and opinionated**

つまり「狭く、明確な専門性を持つ」ようにすることが推奨されています。

そのため `description` では、

```text
何をするか
+
どんなときに使うか
```

を明確にするとよいです。

---

## 54. `developer_instructions` — Agentの具体的な行動ルール

Custom Agentで最も重要な設定の一つです。

```toml
developer_instructions = """
Stay in research mode.

Prefer official documentation and primary sources.

Do not modify source code.

Verify version-specific behavior.

Return:
- conclusion
- evidence
- uncertainties

Do not return raw exploration logs.
"""
```

ここには、

- 何をするか
- 何をしないか
- 情報源の優先順位
- 調査方法
- 終了条件
- Main Agentへ返す形式

などを書きます。

例えばResearcherなら、

```text
調査だけ行う
コード変更禁止
一次情報優先
Version確認
結論と根拠だけ返す
```

と設定できます。

Reviewerなら、

```text
Correctness
Security
Regression
Missing Tests
```

などを見るよう指定できます。

### Token効率にも影響する

例えば、

```toml
developer_instructions = """
Prefer targeted searches over broad scans.
Stop when sufficient evidence is found.
Return only conclusions and key evidence.
"""
```

と設定すると、

- 不要な全ファイル探索
- 過剰な追加調査
- Main Agentへの大量ログ返却

を減らしやすくなります。

---

## 55. `model` — Agentが使用するModel

任意項目です。

```toml
model = "gpt-5.6-terra"
```

Custom AgentごとにModelを変えることができます。

例えば、

```text
Main Agent
高性能Model

├─ repo_explorer
│    └─ 軽量Model
│
├─ docs_researcher
│    └─ 軽量Model
│
└─ reviewer
     └─ 高性能Model
```

とすることで、

```text
単純・大量処理
↓
高速・低コストModel

複雑な判断
↓
高性能Model
```

という最適化ができます。

### 設定しなかった場合

`model` をCustom Agentに書かなければ、親Agentや `[agents]` のDefault設定から解決されます。

そのため、

```toml
name = "researcher"
description = "..."
developer_instructions = "..."
```

だけでも利用可能です。

---

## 56. `model_reasoning_effort` — 推論量

任意項目です。

```toml
model_reasoning_effort = "low"
```

Agentがどの程度の推論を行うかを設定します。

代表的な考え方は、

| 用途 | Reasoning |
|---|---|
| 単純な検索 | `low` |
| コード探索 | `low` ～ `medium` |
| 通常の技術調査 | `medium` |
| コードレビュー | `high` |
| 複雑な設計・Security | `high`以上を検討 |

利用できる値はModelによって異なります。

高いReasoning Effortは複雑な問題で品質向上が期待できますが、

- Token消費
- 応答時間

も増加します。

そのため、

```text
Explorer
↓
Low

Researcher
↓
Medium

Reviewer
↓
High
```

のような使い分けが有効です。

---

## 57. `model` と `model_reasoning_effort` の継承

この2項目には設定の優先順位があります。

概念的には、

```text
親Agentの設定
      ↓
[agents] Default
      ↓
明示的なspawn指定
      ↓
Custom Agent TOML
```

のように設定が解決され、**Custom Agentファイルに明示された `model` / `model_reasoning_effort` が最終的に優先されます**。

つまり、

```toml
model = "..."
model_reasoning_effort = "low"
```

とCustom Agentへ書けば、そのAgent専用設定として固定できます。

### `model`だけ指定する場合の注意

Custom Agentで、

```toml
model = "..."
```

だけ設定した場合、既に解決されているReasoning Effortはそのまま維持されます。

そのModelがそのReasoning Effortに対応していない場合や、明確に変更したい場合は、

```toml
model = "..."
model_reasoning_effort = "medium"
```

のように両方指定した方が安全です。

---

## 58. `sandbox_mode` — ファイル操作権限

任意項目です。

例えば、

```toml
sandbox_mode = "read-only"
```

とすると、調査・レビュー中心のAgentを読み取り専用にできます。

代表的な使い方は、

```text
Researcher
↓
read-only

Explorer
↓
read-only

Reviewer
↓
read-only

Implementation Agent
↓
workspace-write
```

です。

### Research Agentでは特に有効

例えば、

```toml
name = "official_researcher"
sandbox_mode = "read-only"
```

とすることで、

```text
ドキュメント調査      ○
コードを読む          ○
コードを書き換える    ×
```

という役割にできます。

複数SubAgentを並列実行するときに、意図しないコード変更を防止するため非常に有効です。

### 親Agentからの継承

`Custom Agent TOML`に `sandbox_mode` を設定しない場合は、親側のSandbox設定を継承します。

ただし、実行中に親セッションで変更したPermissionやSandboxなどのRuntime Overrideは、SubAgent spawn時に再適用される場合があります。

そのため、

> Custom Agent TOMLだけを見れば最終的な実行権限が完全に決まるとは限らない

点には注意が必要です。

---

## 59. `mcp_servers` — Agent専用のMCP Server

Custom AgentにはMCP Server設定も含められます。

例えば、

```toml
[mcp_servers.openaiDeveloperDocs]
url = "https://developers.openai.com/mcp"
```

とすると、そのAgentから専用MCP Serverを利用できます。

例えば、

```text
docs_researcher
       │
       └─ Documentation MCP

browser_debugger
       │
       └─ Chrome DevTools MCP
```

のように、

> **Agentの仕事に必要なToolだけを持たせる**

ことができます。

これは、

- Agentの専門化
- Tool選択の迷いを減らす
- 不要なTool利用を抑える

という意味でも有効です。

### 例

```toml
name = "docs_researcher"

description = """
Documentation specialist for API verification.
"""

model_reasoning_effort = "medium"
sandbox_mode = "read-only"

developer_instructions = """
Verify API behavior using the documentation MCP server.
Return concise references.
Do not modify code.
"""

[mcp_servers.openaiDeveloperDocs]
url = "https://developers.openai.com/mcp"
```

---

## 60. `skills.config` — Agentで使用するSkill

Custom AgentごとにSkill設定を変更することもできます。

例えば、

```toml
[[skills.config]]
path = "/path/to/SKILL.md"
enabled = false
```

のように設定できます。

これにより、

```text
Agent A
├─ Skill 1 ○
├─ Skill 2 ○
└─ Skill 3 ×

Agent B
├─ Skill 1 ×
├─ Skill 2 ○
└─ Skill 3 ○
```

のようにAgentごとに利用可能なSkill構成を変えられます。

例えば、

```text
docs_researcher
↓
Documentation関連Skill

tester
↓
Testing関連Skill

reviewer
↓
Review関連Skill
```

と役割ごとに分ける考え方です。

---

## 61. Custom Agent TOMLの主要設定まとめ

主要項目をまとめると次のようになります。

| 設定 | 必須 | 役割 | 主な用途 |
|---|---:|---|---|
| `name` | ○ | Agent識別名 | Agentを参照・spawn |
| `description` | ○ | 使用目的を説明 | Codexが役割を判断 |
| `developer_instructions` | ○ | 詳細な行動ルール | 調査・レビュー方針 |
| `model` | × | 使用Model | 品質・速度・コスト調整 |
| `model_reasoning_effort` | × | 推論量 | Token・品質調整 |
| `sandbox_mode` | × | Sandbox権限 | read-only等の安全制御 |
| `mcp_servers` | × | MCP Tool追加 | 外部Tool・Docs・Browser等 |
| `skills.config` | × | Skill設定 | Agent固有Skillの有効化/無効化 |

特に最初に理解しておくべきなのは、

```text
name
description
developer_instructions
```

の必須3項目と、

```text
model
model_reasoning_effort
sandbox_mode
```

の3項目です。

この6つだけでも、かなり実用的なCustom Agentを作れます。

---

## 62. Custom Agent TOMLとGlobal `[agents]` 設定の違い

混同しやすいため注意が必要です。

### Custom Agent TOML

```text
.codex/agents/researcher.toml
```

は、

> **特定のAgentだけの設定**

です。

例えば、

```toml
name = "researcher"
model_reasoning_effort = "medium"
sandbox_mode = "read-only"
```

などを定義します。

### Global `[agents]`

一方、

```text
.codex/config.toml
```

や、

```text
~/.codex/config.toml
```

の、

```toml
[agents]
```

は、

> **SubAgent機能全体のDefault・制御設定**

です。

主要項目は、

| 設定 | 役割 |
|---|---|
| `agents.enabled` | Multi-Agent機能の有効/無効 |
| `agents.max_concurrent_threads_per_session` | SubAgent最大同時Thread数 |
| `agents.default_subagent_model` | SubAgentのDefault Model |
| `agents.default_subagent_reasoning_effort` | Default Reasoning Effort |
| `agents.interrupt_message` | Agent中断時のMessage制御 |

です。

構造としては、

```text
.codex/config.toml
│
│ Global SubAgent Settings
│
└─ [agents]
      │
      ├─ enabled
      ├─ max_concurrent_threads_per_session
      ├─ default_subagent_model
      └─ default_subagent_reasoning_effort


.codex/agents/
│
├─ researcher.toml
├─ reviewer.toml
└─ explorer.toml
       │
       └─ Agent固有設定
```

となります。

---

## 63. おすすめのCustom Agent最小構成

最初からMCPやSkillまで設定すると複雑になりやすいため、最初は次の6項目程度がおすすめです。

```toml
name = "researcher"

description = """
Technical researcher for investigating libraries,
frameworks, APIs, and implementation approaches.
"""

model = "<appropriate-model>"
model_reasoning_effort = "medium"

sandbox_mode = "read-only"

developer_instructions = """
Research only the requested subject.

Prefer:
1. Official documentation
2. Specifications
3. Source code
4. GitHub issues and changelogs

Do not modify source code.

Stop when sufficient evidence is available.

Return:
- conclusion
- key evidence
- uncertainty

Keep the result concise.
"""
```

この構成で、

```text
Identity
    ↓
name

Role selection
    ↓
description

Behavior
    ↓
developer_instructions

Cost / Speed
    ↓
model

Reasoning
    ↓
model_reasoning_effort

Safety
    ↓
sandbox_mode
```

を一通り制御できます。

---

## 64. 調査用Custom Agentのおすすめ構成

並列技術調査を行う場合は、例えば次のように分けられます。

### `official_researcher.toml`

```toml
name = "official_researcher"

description = """
Researcher for verifying technical claims
using official documentation and specifications.
"""

model_reasoning_effort = "medium"
sandbox_mode = "read-only"

developer_instructions = """
Focus on official and primary sources.

Verify:
- current behavior
- supported versions
- configuration
- limitations

Do not modify code.

Stop when the requested points are verified.

Return concise conclusions with evidence.
"""
```

### `repo_explorer.toml`

```toml
name = "repo_explorer"

description = """
Fast read-only codebase explorer for locating
files, symbols, dependencies, and execution paths.
"""

model_reasoning_effort = "low"
sandbox_mode = "read-only"

developer_instructions = """
Stay in exploration mode.

Prefer targeted searches over broad scans.

Identify:
- relevant files
- symbols
- dependencies
- execution paths

Do not modify code.

Return only information needed by the parent agent.
"""
```

### `critic.toml`

```toml
name = "critic"

description = """
Critical reviewer for challenging technical conclusions
and finding missing constraints, edge cases, and contradictions.
"""

model_reasoning_effort = "high"
sandbox_mode = "read-only"

developer_instructions = """
Challenge the current conclusion.

Look for:
- unsupported assumptions
- contradictory evidence
- version differences
- edge cases
- missing constraints

Do not repeat research already established unless needed
to verify a suspected problem.

Return only meaningful objections and their evidence.
"""
```

このように、

```text
official_researcher
        ↓
一次情報

repo_explorer
        ↓
実装・コード

critic
        ↓
反証
```

と責務を明確にすることで、

- 重複調査削減
- Token効率向上
- 調査範囲拡大
- 結論の検証

を同時に狙えます。

---

## 65. Custom Agent設定で特に重要な設計原則

Custom Agentを作るときは、設定項目を増やすこと自体が目的ではありません。

重要なのは、

```text
1 Agent
=
1つの明確な責務
```

に近づけることです。

悪い例：

```text
researcher
=
調査
+
実装
+
テスト
+
レビュー
+
設計
```

良い例：

```text
researcher
=
調査のみ

reviewer
=
レビューのみ

tester
=
テストのみ
```

Custom Agentの効果を高めるには、

```text
明確なdescription
+
具体的なdeveloper_instructions
+
必要十分なModel
+
適切なReasoning
+
最小限の権限
+
必要なToolだけ
```

という構成にするのがおすすめです。

---

## 66. Custom Agent設定の参考資料

OpenAI公式ドキュメント:

- Codex Subagents / Custom agents  
  https://learn.chatgpt.com/docs/agent-configuration/subagents

- Codex Config Basics  
  https://learn.chatgpt.com/docs/config-file/config-basic

公式ドキュメントでは、Custom Agent TOMLは通常のCodex Session Configと同じ設定項目を上書きできるConfiguration Layerとして扱われています。

そのため、今後より高度な設定を行う場合は、Custom AgentページだけでなくCodex Config Referenceも確認するとよいです。

