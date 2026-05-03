---
title: Cognee 公式 Claude Code プラグインと自作ツールキットを比較してみた
authors:
  - JapanNomu
emoji: "\U0001F50D"
type: tech
topics:
  - claudecode
  - cognee
  - mcp
  - ollama
  - llm
published: true
---

## はじめに

先日、[Claude Code に Cognee グラフ記憶を追加する自作ツールキット](https://zenn.dev/japannomu/articles/20260502_cognee-graph-memory-toolkit) を公開しました。前記事では「Cognee MCP は標準では Claude Code から使いにくい3つの隙間がある」という観点から、それを埋めるツールキットを紹介しました。

公開した後で、改めて Cognee 公式が **Claude Code 用のプラグイン** を出していることを整理し直す機会がありました。「公式プラグインがあるなら、なぜわざわざ自作したのか？」「自作ツールキットの存在意義は本当にあったのか？」── ここを事実ベースで検証する必要があると判断したので、本記事で両者を細かく比較します。

結論を先に書くと：

- **公式プラグイン** は Cognee Cloud + クラウド LLM API 前提の **クラウドネイティブ路線** で、Claude Code の hook を6つも使った高機能な session 連携を持つ
- **自作ツールキット** は ローカル Cognee + ローカル LLM 前提の **完全ローカル自己完結路線** で、初期投入・大量ナレッジ取り込み・動作実証データ・日本語ドキュメントを備える
- 両者は **競合ではなく補完** であり、ユースケースで使い分け、または併用することができる

具体的に何がどう違うのか、ソースを引きながら見ていきます。


## Cognee 公式が提供している Claude Code 関連の選択肢を整理する

ここがまずわかりにくいので、ソースを引きながら順に整理します。

### 1. `cognee-mcp` ── 公式 MCP サーバー本体

リポジトリ: `topoteretes/cognee` のサブディレクトリ [`cognee-mcp/`](https://github.com/topoteretes/cognee/tree/main/cognee-mcp)

これは Cognee 本体を MCP サーバーとして起動するための実装です。Claude Code に限らず、**Cursor / Cline / Claude Desktop など MCP に対応したクライアント全般** から利用できます。

公式 README には以下と明記されています：

> "exposes only `remember`, `recall`, and `forget` for agent memory workflows"

提供している MCP ツールは **`remember` / `recall` / `forget` の3つだけ**。エージェントメモリのワークフローに最低限必要な API に絞った設計です。

自作ツールキットも内部ではこの `cognee-mcp` を MCP サーバーとして使っています。

### 2. `cognee-integrations/integrations/claude-code/` ── 公式 Claude Code プラグイン

リポジトリ: [`topoteretes/cognee-integrations`](https://github.com/topoteretes/cognee-integrations) の `integrations/claude-code/` サブディレクトリ

**本記事で「公式プラグイン」と呼ぶのはこちら** です。Claude Code の Plugin Marketplace 形式で配布される構成（`.claude-plugin/` 設定 + `hooks/` + `skills/` + `agents/` + `scripts/` を含む）。

公式ドキュメント [docs.cognee.ai/cognee-mcp/integrations/claude-code](https://docs.cognee.ai/cognee-mcp/integrations/claude-code) によれば、6つのライフサイクル hook と3つの slash command を提供します（後述）。

### 3. `cognee-integration-claude` ── Claude **Agent SDK** 用ライブラリ（紛らわしい）

リポジトリ: [`topoteretes/cognee-integration-claude`](https://github.com/topoteretes/cognee-integration-claude)

名前が紛らわしいですが、こちらは **Claude Agent SDK 用** の Python ライブラリで、Claude Code（CLI）用ではありません。`pip install cognee-integration-claude` で入り、`add_tool` / `search_tool` などを `ClaudeAgentOptions` 経由で組み込む使い方です。

本記事の比較対象ではありませんが、検索すると上位に出てくるので念のため整理しました。

### 4. 自作ツールキット `JapanNomu/tools` / `JapanNomu/tools-ja`

リポジトリ:
- 🇬🇧 [JapanNomu/tools](https://github.com/JapanNomu/tools)
- 🇯🇵 [JapanNomu/tools-ja](https://github.com/JapanNomu/tools-ja)

公式 `cognee-mcp` (上記1) を MCP サーバーとして使いつつ、その上に「Claude Code から快適に使えるためのスクリプト群」を被せた連結ツールキットです。詳細は [前記事](https://zenn.dev/japannomu/articles/20260502_cognee-graph-memory-toolkit) を参照してください。


## 比較表（細かく）

両者を観点ごとに細かく見ていきます。

| 観点 | 公式プラグイン (`cognee-integrations/claude-code`) | 自作ツールキット (`JapanNomu/tools`) |
|------|---|---|
| **配布形式** | Claude Code Plugin Marketplace 形式 (`.claude-plugin/`) | 通常の git リポジトリ + 手動セットアップ |
| **インストール** | `pip install cognee` + `claude --plugin-dir <path>` で読み込み | git clone + venv + `pip install cognee-mcp "cognee[fastembed]"` + `claude mcp add` |
| **MCP 接続スクリプト** | プラグインが内部で処理（ユーザーは意識不要） | `start_cognee_mcp.py`（環境変数の取り回し・cwd 問題対処を独自実装） |
| **想定 LLM** | `LLM_API_KEY` 環境変数前提（OpenAI / Anthropic 等のクラウド API がデフォルト） | Ollama ローカル LLM がデフォルト（qwen2.5:14b）。クラウド API も `.env` 数行で切替可 |
| **Cognee の運用形態** | ローカル in-process / Cognee Cloud / 自前 API サーバーの3モード対応 | **ローカル in-process 前提**（KuzuDB + LanceDB + FastEmbed の完全ローカル構成） |
| **ローカル LLM 動作実証** | ドキュメントに明示記載なし（Ollama サポートは未明記） | qwen2.5:14b（num_ctx=8192）で `remember` / `search(CHUNKS)` / `search(GRAPH_COMPLETION)` / `recall` の全機能を実機検証済み |
| **自動キャプチャ hook 数** | **6つ**（SessionStart / UserPromptSubmit / PostToolUse / Stop / PreCompact / SessionEnd） | 2つ（UserPromptSubmit / Stop） |
| **session cache → グラフ同期** | SessionEnd で `cognee.improve()` を自動呼び出し | キューファイル + cron 5分毎の flusher |
| **context injection（プロンプト送信前の関連知識自動注入）** | **あり**（UserPromptSubmit hook で session cache を検索しコンテキストとして注入・3秒タイムアウト） | なし（CLAUDE.md ルールで AI 自身に search を呼ばせる方式） |
| **PostToolUse による tool 実行結果の自動キャプチャ** | あり | なし |
| **slash command** | **3つ**（`/cognee-memory:cognee-remember`・`/cognee-memory:cognee-search`・`/cognee-memory:cognee-sync`） | なし |
| **同梱サンプル知識** | **なし** | **4ファイル**（git push ルール / 設計判断 / 開発教訓 / よくあるエラー）。インストール直後に動作確認可 |
| **任意ディレクトリの取り込みスクリプト** | **なし**（slash command で1件ずつ remember する想定） | `import_to_graph.py`（`~/.claude/rules/` 等の任意ディレクトリを丸ごと取り込み可能） |
| **大量ナレッジの分割投入＋リトライ機構** | **なし** | `split_knowledge.py`（H2 見出しで分割）+ `import_knowledge.py`（cognify 失敗時に最大3回リトライ・`--dry-run` 対応） |
| **MCP ツールの提供範囲** | `remember` / `recall` / `forget` の3つ（`cognee-mcp` 経由） | 同左（`cognee-mcp` をそのまま利用） |
| **ドキュメント言語** | 英語のみ | **英語版・日本語版の両方**（README / SETUP.md / GETTING_STARTED.md / HARNESS_GUIDE.md） |
| **ライセンス** | （リポジトリ参照） | MIT |


## 「プラグインがあるのに、なぜツールキットを作ったのか？」

ここが本記事の本題です。比較表を踏まえて、**公式プラグインに含まれていない要素** と、それがないと使う側がどう困るのかを順に見ていきます。

### 不足1: 完全ローカル LLM での動作実証データがない

公式プラグインのドキュメントには `LLM_API_KEY` の設定が中心に書かれており、**Ollama などのローカル LLM での動作確認データが明示されていません**。Cognee 本体は `LLM_PROVIDER=ollama` で Ollama を受け入れる構造があるため、技術的には動くはずですが、**「どのモデルなら recall まで含めて完答できるのか」という実証情報が足りない** のが現実です。

これがないと使う側はどう困るかと言うと：

- ローカルで試そうとしてエラーが出たとき、「自分の設定が悪いのか / モデルが不適合なのか」 を切り分けられない
- 適当に手元の Ollama モデル（llama3.1:8b など）で試して `recall` が JSON Schema 違反で落ち、原因がわからずに諦める
- 「ローカル LLM ってどれを選べばいいの？」という最初の一歩で詰まる

自作ツールキットでは、**`docs/GETTING_STARTED.md` に「動作実証下限機種」「快適に動かすための推奨スペック」「実証済みモデル名（qwen2.5:14b・num_ctx=8192）」が明記されています**。前記事でも書いた通り、検証環境（NVIDIA GeForce RTX 4060 Laptop GPU・VRAM 8GB / RAM 32GB）で実機検証した結果として記載されているデータです。

### 不足2: 任意ディレクトリの取り込みスクリプトがない

公式プラグインの slash command `/cognee-memory:cognee-remember` は、**1件ずつテキストや内容を Cognee に登録する** ためのコマンドです。

ところが実用上は「`~/.claude/rules/` にある自分のルール集を全部 Cognee に取り込みたい」「自分の Obsidian Vault の Markdown を全部投入したい」というニーズが必ず発生します。これを slash command で1件ずつやるのは現実的ではありません。

公式プラグインには **ディレクトリ単位の取り込みスクリプトが標準で付いていません**。使う側は自前で Python スクリプトを書くか、Cognee の Python API を直接呼ぶ必要があります。

自作ツールキットでは `src/main_src/import_to_graph.py` がこの責任を引き受けます。

### 不足3: 大量ナレッジを安全に投入する仕組みがない

これも実用上の大きな差です。

数十〜数百ファイル規模のナレッジを Cognee に投入するとき、**ローカル LLM は時々 cognify に失敗します**（JSON Schema 違反・LLM 応答の Pydantic 検証失敗・タイムアウトなど）。1ファイルが大きいほど失敗率が上がります。

公式プラグインの想定では、ファイルが落ちた場合の **再開ポイント管理・リトライ・大きいファイルの分割** はユーザーが自前で実装することになります。

自作ツールキットでは：

- `src/knowledge_src/split_knowledge.py` が **H2 見出しごとに自動分割** して `knowledge/user_chunks/` に出力
- `src/knowledge_src/import_knowledge.py` が **1チャンクごとに cognify を呼び・失敗時は最大3回リトライ・`--dry-run` で対象一覧を事前確認**

夜間に流して朝結果を確認する運用ができます。

### 不足4: 同梱サンプル知識がない

公式プラグインはインストール直後の状態では **空っぽ** です。Claude Code から「Add this code to Cognee memory」と言って何かを登録するところから始まります。

これだと「インストールはできたっぽいけど、本当に動いているのか確認できない」という壁が初見ユーザーに立ちはだかります。

自作ツールキットには `knowledge/sample_knowledge/` 配下に4ファイルが同梱されており：

- `01_claude_code_tips.md` ── git push ルール・タスク管理・コミット粒度などの実用ノウハウ
- `02_software_dev_lessons.md` ── テスト設計・V字モデル・整合性チェック順序
- `03_design_decisions.md` ── KuzuDB 採用理由・MCP scope=user 採用理由など
- `04_common_errors.md` ── SearchPreconditionError・LLM フォーマットエラーなど

`load_sample.py` 一発で投入し、`docs/GETTING_STARTED.md` の体験シナリオ A〜D で動作確認できます。**動かなかったらインストールが間違っている、という切り分けができる** 設計です。

### 不足5: 日本語ドキュメントがない

公式プラグインのドキュメントは英語のみです。

技術的に問題ない人にはどうでもいいかもしれませんが、**Claude Code 利用者層には英語に抵抗のある日本語話者も多い** のが実態です。Cognee 自体が新しい技術スタック（KuzuDB / LanceDB / FastEmbed）の集合体で、ハマりどころも多いため、母語ドキュメントの存在は導入障壁を大きく下げます。

自作ツールキットには `JapanNomu/tools-ja` リポジトリで完全な日本語版（README / SETUP.md / GETTING_STARTED.md / HARNESS_GUIDE.md）を提供しています。

### 不足6: MCP 起動スクリプトの「環境変数取り回し」が見えない

これは細かい話ですが、実体験として地味に大事です。

`cognee-mcp` を Claude Code から起動するとき、`config/.env` に書いた `LLM_API_KEY` `SYSTEM_ROOT_DIRECTORY` 等が **cognee-mcp 子プロセスに渡らない** という問題が発生します。Claude Code が hook 経由で MCP サーバーを起動する関係で、cwd が配布物のディレクトリと一致しないためです。

結果として `claude mcp add cognee` した瞬間は動いて見えても、cognify を呼んだ瞬間に `LLMAPIKeyNotSetError (Status 422)` で落ちる、という罠にハマります。

公式プラグインは Plugin Marketplace 形式で配布されるためこの問題を回避しているはずですが、**手動で `claude mcp add` する場合（自作ツールキットや、独自に cognee-mcp を組み込む場合）はこの罠が立ちはだかります**。

自作ツールキットの `src/main_src/start_cognee_mcp.py` は、この問題に対処するため **`config/.env` を `os.environ` に明示的に読み込んでから execv で cognee-mcp 子プロセスに継承** する独自実装を持っています。依存ライブラリは Python 標準ライブラリのみで、追加 pip install は不要です。


## 逆に「公式プラグインにあって、ツールキットにないもの」

公平に書くと、公式プラグインの方が機能の幅は広いです。

### 公式が優位な点1: hook の数（6つ vs 2つ）

公式プラグインの 6 hook：

| Hook | 公式の機能 |
|------|-----------|
| **SessionStart** | config 読み込み・per-directory session ID 計算・Cognee Cloud 接続 |
| **UserPromptSubmit** | session cache を検索してプロンプトに関連コンテキストを注入（3秒タイムアウト） |
| **PostToolUse** | tool 名・input・output を session cache にキャプチャ |
| **Stop** | ユーザー中断時の最終アシスタント応答をキャプチャ |
| **PreCompact** | session + graph context からメモリアンカーを構築 |
| **SessionEnd** | `cognee.improve()` を呼んで session データを永続グラフに反映 |

自作ツールキットは UserPromptSubmit と Stop の2つだけで、PostToolUse でのツール実行結果の自動キャプチャはしていません。

### 公式が優位な点2: context injection（プロンプト送信前の自動注入）

公式の UserPromptSubmit hook は、**ユーザーがプロンプトを送信した瞬間に session cache を検索し、関連コンテキストを Claude のコンテキストに自動注入** します。AI が能動的に search を呼ばなくても、過去の関連情報がコンテキストに乗っているという挙動です。

自作ツールキットでは context injection はせず、CLAUDE.md ルールで「**AI が作業前に必ず search を呼ぶ**」運用を内面化させています。これは AI の自律性に依存する分、抜けが起きやすい面はあります。

### 公式が優位な点3: slash command

`/cognee-memory:cognee-remember` `/cognee-memory:cognee-search` `/cognee-memory:cognee-sync` の3つで、ユーザーが明示的に Cognee を操作するインタフェースが提供されます。

自作ツールキットには slash command がなく、AI 経由で MCP ツールを呼ぶか、ターミナルから Python スクリプトを叩くかになります。

### 公式が優位な点4: Cognee Cloud / 自前 API サーバーへの接続

公式プラグインは `service_url` 設定で Cognee Cloud や自前で立てた Cognee API サーバーに接続できる構成です。チームで共有のメモリを使いたいケースなどで有用です。

自作ツールキットは完全ローカル運用を前提にしているため、Cognee Cloud 連携はサポート対象外です。


## 結論：両者は「競合」ではなく「補完」

ここまで見てきた通り、両者は思想が異なります。

| 観点 | 公式プラグイン | 自作ツールキット |
|------|----------------|----------------|
| 思想 | **Cognee の機能をフルに使うクラウドネイティブ路線** | **完全ローカル運用に振り切った頭から尻尾まで自己完結路線** |
| 想定運用 | クラウド LLM API + Cognee Cloud またはローカル | ローカル Cognee + ローカル LLM（クラウド API 切替も可） |
| 強み | hook の充実 / slash command / context injection / Cloud 連携 | 動作実証データ / 大量ナレッジ初期投入 / 同梱サンプル / 日本語ドキュメント |
| 弱み | ローカル LLM 動作実証 / ディレクトリ単位の取り込み / サンプル知識 / 日本語docs が薄い | hook 数が少ない / slash command がない / context injection がない |

### こう使い分けるとよい

**公式プラグインを選ぶべきケース:**

- クラウド LLM API（Claude / OpenAI）を使う前提
- チームで Cognee Cloud を共有運用したい
- session cache の context injection が欲しい
- slash command で明示的にメモリを操作したい

**自作ツールキットを選ぶべきケース:**

- 完全ローカル運用したい（追加課金ゼロ・データを外に出したくない）
- ローカル LLM（Ollama）でちゃんと動くか実証データが欲しい
- 既存ナレッジ（`~/.claude/rules/` や Obsidian Vault など）を投入してから使いたい
- 日本語ドキュメントで導入したい
- 同梱サンプルでまず動作確認してから本番ナレッジを入れたい

### 併用も可能

実は公式プラグインと自作ツールキットは、内部で同じ `cognee-mcp` を使っているため、**両方を併用することも可能** です。具体的には：

1. 自作ツールキットの `import_to_graph.py` / `split_knowledge.py` / `import_knowledge.py` で **既存ナレッジを初期投入**
2. その上に公式プラグインを被せて **session cache + context injection の恩恵を受ける**

この組み合わせなら、両者の強みを取り込みつつ、互いの弱みを補えます。


## まとめ

- **Cognee 公式 Claude Code プラグイン**は、Cognee Cloud + クラウド LLM API 前提で、hook 6つ・slash command 3つ・context injection を備えた **クラウドネイティブ路線**
- **自作ツールキット (`JapanNomu/tools` / `tools-ja`)** は、ローカル Cognee + ローカル LLM 前提で、動作実証 / 任意ディレクトリ投入 / 大量ナレッジ分割投入 / 同梱サンプル / 日本語ドキュメントを備えた **完全ローカル自己完結路線**
- 機能の幅では公式が広いが、**完全ローカル運用に必要な周辺ツール群（実証データ・初期投入・分割リトライ・サンプル）は自作ツールキットが補っている**
- 両者は競合ではなく補完。ユースケースで使い分ける、または併用することができる

リポジトリ：

- 🇯🇵 自作ツールキット 日本語版: https://github.com/JapanNomu/tools-ja
- 🇬🇧 自作ツールキット 英語版: https://github.com/JapanNomu/tools
- 🇬🇧 Cognee 公式プラグイン: https://github.com/topoteretes/cognee-integrations
- 🇬🇧 Cognee 公式 MCP サーバー: https://github.com/topoteretes/cognee/tree/main/cognee-mcp
- 🇬🇧 Cognee 公式ドキュメント: https://docs.cognee.ai/

「**完全ローカルで Claude Code に記憶を持たせたい**」という方には自作ツールキットを、「**クラウドで運用してチームで共有したい**」という方には公式プラグインをおすすめします。

前記事では自作ツールキットの中身を細かく紹介していますので、興味のある方はあわせてご覧ください：

- [Claude Code に Cognee グラフ記憶を追加する実用ツールキットを公開しました](https://zenn.dev/japannomu/articles/20260502_cognee-graph-memory-toolkit)
