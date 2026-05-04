---
title: 自作ツールキットが cognee 共同創業者から招待を受け、公式リポへ PR を出すまでの全記録（v0.2.1）
authors:
  - JapanNomu
emoji: "\U0001F680"
type: tech
topics:
  - claudecode
  - cognee
  - oss
  - pullrequest
  - github
published: true
---

## はじめに

[Claude Code に Cognee グラフ記憶を追加する自作ツールキット](https://zenn.dev/japannomu/articles/20260502_cognee-graph-memory-toolkit) と、その後に書いた [公式プラグインとの比較記事](https://zenn.dev/japannomu/articles/20260503_cognee-official-vs-toolkit) を公開したところ、思いがけず cognee 共同創業者 Vasilije Markovic 氏（@tricalt）から X 上で「機能拡張の PR を送ってきていい」との招待をいただきました。

OSS のメンテナーから直接「PR 送って」と招待されるのは、社交辞令ではなく **真剣にコードを見る用意がある** という意思表示です。せっかくの機会なので、本気で応えることにしました。

本記事は、その招待を受けてから cognee 公式リポジトリ `topoteretes/cognee-integrations` に **PR #47** を提出するまでの一連の作業を、戦略・チェックリスト・実作業手順・PR 説明文・レビュー対応まで含めて全記録したものです。

OSS への PR 提出が初めての方、または「招待を受けたけれど何をどう準備すればいいか分からない」という方の参考になれば幸いです。

リポジトリ:

- 🇬🇧 English: https://github.com/JapanNomu/tools
- 🇯🇵 日本語: https://github.com/JapanNomu/tools-ja

PR:

- https://github.com/topoteretes/cognee-integrations/pull/47


## 1. 経緯（時系列）

| 日付 | 出来事 |
|---|---|
| 2026-04-26 | 00029プロジェクト開始（cognee グラフメモリ MCP ツールキット開発） |
| 2026-04-27 | v1.0.0 として初回完成（UT 110件 + IT 6件 + ET 4件 + ST 4件 全 PASS）|
| 2026-05-02 | GitHub に 2 リポジトリをリリース（英語版 `JapanNomu/tools` / 日本語版 `JapanNomu/tools-ja`）|
| 2026-05-02 | Zenn 記事①公開（[ツールキット紹介](https://zenn.dev/japannomu/articles/20260502_cognee-graph-memory-toolkit)）+ X で宣伝 |
| 2026-05-03 | Zenn 記事②公開（[公式 vs 自作の比較](https://zenn.dev/japannomu/articles/20260503_cognee-official-vs-toolkit)）+ X で宣伝 |
| 2026-05-03 | cognee コミュニティメンバーが反応、Discord 内でもツールキットを宣伝してくれる |
| **2026-05-03 17:40** | **Vasilije Markovic 氏（@tricalt、cognee 共同創業者・X 認証バッジ付き）が X 上で返信**:<br>「@JapanNomu feel free to open up PR on our integrations repo if you'd like to extend features cognee has!」 |
| 2026-05-04 | PR 検討開始・作業手順書作成 |
| 2026-05-04 | cognee 1.0.5 検証完了 → v0.2.0 リリース（[v0.2.0 Zenn 記事](#)） |
| 2026-05-04 | lint 事前対応のため v0.2.1 として再リリース |
| 2026-05-04 | **PR #47 提出完了** |

公式メンテナーから直接「PR 送って」と招待されるのは、OSS 文化として **真剣にコードを見る用意があるという意思表示** です。社交辞令ではないので、本気で応えるべき場面でした。


## 2. 招待を受けたあとの戦略

招待を受けて即 PR を作るのではなく、まず以下を整理しました。

### 2-1. 基本方針

| 項目 | 方針 |
|---|---|
| **貢献形態** | ワンショット貢献（One-time contribution）。長期メンテナンスは引き受けない |
| **ライセンス** | MIT License（Copyright (c) 2026 JapanNomu）の著作権表示保持を希望 |
| **自リポジトリ** | `JapanNomu/tools` は独立して進化を継続。cognee 側の改変は逆輸入しない |
| **PR 構成** | 1 PR にまとめる + コミット論理分割で cherry-pick 可能にする |
| **メンテナー打診への対応** | 打診されても丁寧に断る（コード提供のみ、メンテはお任せ）|

### 2-2. なぜこの戦略か

- **法的に問題なし**: MIT は AS IS 条項によりメンテナンス義務ゼロ
- **OSS 文化的に正当**: cognee-community の過去 PR には単発貢献者多数（Spanner adapter 追加者 mdehsan873・AzureAISearch 修正者 gourangasatapathyvit 等、すべて 1 PR のみで完結）
- **メンテナー継続要請されにくい**: 過去データから「メンテナー継続を断ったから却下」事例は確認できず
- **自リポジトリの独立性確保**: 取り込まれた後も `JapanNomu/tools` は自分のペースで継続可能

### 2-3. 取り込まれた後の権利関係

| 項目 | 取り込み前 | 取り込み後 |
|---|---|---|
| コードの管理者 | JapanNomu | cognee メンテナーチーム |
| バグ修正の主担当 | JapanNomu | cognee チーム |
| cognee バージョンアップ追従 | JapanNomu | cognee チーム |
| 著作権表示 | JapanNomu | JapanNomu のまま保持（git history 含め永続記録）|
| Contributors 一覧 | — | cognee 公式に永続記録 |


## 3. PR 提出のタイミング判断

招待を受けた直後に PR を出さず、**配布物を一定の品質まで上げてから** PR を出すことにしました。具体的には:

- [x] cognee 1.0.5 検証完了（v0.2.0 として確定）
- [x] v0.2.0 を `JapanNomu/tools`（英語版）にプッシュ済み
- [x] v0.2.0 を `JapanNomu/tools-ja`（日本語版）にプッシュ済み
- [x] v0.2.0 のタグ付与・GitHub Release 作成済み

PR で参照するコードは v0.2.0 のスナップショット（後に lint 修正のため v0.2.1 に再リリース）。


## 4. PR 前チェックリスト

PR を出す前に、以下を機械的・実機検証で潰していきました。

### A. ライセンス整合性確認

JapanNomu のコードは MIT License で公開済（GitHub `JapanNomu/tools` に LICENSE 配置済・Copyright (c) 2026 JapanNomu）。MIT ライセンスは「shall be included」（=法的義務）として著作権表示の保持を要求するため、cognee 側に「断る」法的選択肢はありません。取り込み側は「保持して取り込む」または「取り込まない」の2択のみ。よって事前確認は不要・PR 本文に法的通知として記載するだけで完結します。

事実確認:

- `cognee-integrations` リポジトリレベル LICENSE: **未配置**（GitHub API: `license: null`）
- integrations 配下 7 ディレクトリ全て LICENSE/NOTICE/COPYING 未配置
- cognee 本体: Apache-2.0
- cognee-community: Apache-2.0

MIT → Apache-2.0 / ライセンス未指定への取り込みは法的に可能（MIT は最も緩いライセンスの 1 つ・著作権表示保持のみ義務）。

### B. 既存 cognee 公式 claude-code plugin との衝突確認

別ディレクトリ配置（`integrations/claude-code-cognee-graph-memory/`）を採用するため、ファイル名・plugin name の衝突は構造上発生しません。機能重複は H で正直評価し PR 説明文に明示します。

| 観点 | 公式 | 自作 |
|---|---|---|
| スクリプト命名規則 | ハイフン区切り | アンダースコア区切り |
| plugin.json | あり（`name: cognee-memory`）| 未保有 |
| hooks 数 | 6 hook | 2 hook + cron flusher |

別ディレクトリに配置するため、**衝突なし**。

### C. cognee 1.0.5 / cognee-mcp との API 互換性確認

- v0.2.0 が cognee 1.0.5 で動作することを実機検証済（qwen2.5:14b で 8 ツール × 5 回 = 40 回検証・35 件成功）
- cognee-mcp 0.5.4（v0.1.x から変更なし）
- 依存指定: `cognee>=1.0.5,<1.1.0`（cognee-integrations CI 要求）

### D. cognee-integrations のコーディング規約確認

リポジトリ root の `pyproject.toml` に Ruff 設定:

```toml
[tool.ruff]
target-version = "py310"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "W"]
```

`integrations/inventory.yml` への登録が必須。10 既存 integration 全件登録済の前例あり。

### E. DCO（Developer Certificate of Origin）対応

`cognee-integrations` は **DCO 非必須**（リポジトリに CONTRIBUTING.md・DCO チェック CI とも配置なし）。ただし安全策として `git commit -s` で署名コミットを作成しておきました。後から DCO 必須化された場合の手戻り防止が目的。

### F. 機密情報・社内情報の混入チェック

最新の `JapanNomu/tools` を実機 grep で 8 項目検証。**全件クリア**。

- 90001（社内ナレッジフォルダ）への参照: 0 件
- 個人情報（メールアドレス・実名等）: 0 件
- API キー・パスワードのハードコード: 0 件
- 日本語コメント・docstring: 0 件（英語版は 100% 英語化済）
- `__pycache__`・`.DS_Store` 等の不要ファイル: 0 件
- ハードコードされたローカルパス: 0 件

`~/.claude/rules/` への言及は 7 箇所ありますが、これは Claude Code 利用者向けハーネスセットアップガイドでの**標準パス説明**（Anthropic 公式ドキュメント記載のディレクトリ）であり、社内専用パスではありません。

### G. lint チェックと import チェック

cognee-integrations の Ruff 設定で配布物 11/12 を検証したところ、**v0.2.0 時点で lint エラー 13 件が発覚**:

| 種別 | 件数 | 内容 |
|---|---|---|
| **F401** Unused import | 3 件 | `os`、`subprocess` 未使用 import |
| **F841** Unused variable | 1 件 | `result =` 未使用変数 |
| **I001** Import un-sorted | 1 件 | `urllib.request` / `urllib.error` 順序違反 |
| **E501** Line too long (>100) | 8 件 | `logger.info/warning/error` や `argparse` の長い行 |

**コード動作には影響しないスタイル違反のみ**。バグはなし。

ただし PR 提出後に CI で落ちると印象が悪いため、**事前に v0.2.1 として lint 修正リリースを出す**ことにしました（次節で詳述）。

### H. 公式と重複している部分の正直な評価

| 機能 | 重複度 | 公式の状態 | JapanNomu の状態 |
|---|---|---|---|
| **ハーネス（hooks）** | 高 | 6 hook、10 スクリプト、idle-watcher・pre-compact・session-aware storage あり、slash commands 3 つ、plugin.json あり | 2 hook + cron flusher、3 スクリプト、plugin.json なし |
| **ナレッジ分割投入** | 低 | 機能なし | `split_knowledge.py` + `import_knowledge.py`（H2 見出し分割 + リトライ機能あり）|
| **サンプルナレッジ** | 低 | なし | 5 ファイルの md サンプル |
| **MCP サーバ起動ラッパー** | 中 | あり（公式 plugin 形式）| `start_cognee_mcp.py`（ローカル LLM 用）|
| **インポートツール** | 中 | あり | `import_to_graph.py`（任意ディレクトリから取り込み可能）|
| **ローカル LLM 動作実証** | 低 | クラウド前提 | qwen2.5:14b で 40 回検証 |
| **日本語ドキュメント** | 低 | なし | あり（英語版 PR では除外）|

**重複度「高」のものは PR 説明文で「reference only」と明記**し、cherry-pick 対象外でも構わないと表明しました。


## 5. v0.2.1 lint 修正リリース

第 4 章 G の lint チェックで 13 件の違反を発見したため、PR 提出前に **v0.2.1** として修正リリースを出しました。

### コード文字列を増減させない厳格な修正方針

修正は以下のルールで行いました（**コード動作影響ゼロを保証**するため）:

- **改行**: 引数の途中で改行を入れただけ。文字列・引数の中身は 1 文字も変えない
- **コメントアウト**: `# import os` のように `# ` を付けただけ。行は消さない
- **順序入替**: `urllib.error` と `urllib.request` を入れ替えただけ。import 対象は同じ
- **文字列短縮（許可済 2 箇所のみ）**: 改行だけでは 100 文字以内に収まらないログ文言だけ、意味を保ったまま冠詞等を削った（例: "the failure" → "failure"）

ロジック・関数呼び出し・引数値・動作はすべて元のまま。よって **v0.2.0 で取った全件 PASS の検証結果は引き続き有効**。

### Ruff 修正内容（5 ファイル）

| ファイル | 修正内容 |
|---|---|
| `auto_remember_completion.py` | F401: `import os` をコメントアウト・import ブロック外に移動 |
| `auto_remember_user_message.py` | F401: `import os`・`import subprocess` をコメントアウト・import ブロック外に移動 |
| `cognee_remember_flusher.py` | F841: `result =` をコメントアウト + 改行 / E501: argparse 改行 |
| `import_knowledge.py` | E501 ×5: logger.info/warning/error の引数改行 + 1 箇所だけ文字列短縮 |
| `import_to_graph.py` | I001: `urllib.error` ⇄ `urllib.request` 順序入替 / E501 ×4: argparse・logger 改行 + 1 箇所文字列短縮 |

### Why（なぜ事前に潰したか）

- PR 提出後に CI で落ちると印象を損ねる
- メンテナーから「直してから来て」と言われる手戻りを避ける
- 自分のリポジトリ（`JapanNomu/tools`）の品質も上がる（一石二鳥）

cognee 側の lint ルール（Ruff `E,F,I,W`、line-length 100、target-version py310）に従って手元で `ruff check` を実行し、**All checks passed!** を確認してから PR 用ブランチに反映する流れを採りました。

v0.2.1 リリース URL:

- 英語版: https://github.com/JapanNomu/tools/releases/tag/v0.2.1
- 日本語版: https://github.com/JapanNomu/tools-ja/releases/tag/v0.2.1


## 6. PR 構造設計

### 6-1. 1 PR + コミット論理分割（cherry-pick 可能）

cognee メンテナーが「気に入ったコミットだけ取り込んでよい」状態を作るため、以下のコミット構成にしました。**配布物のセットアップ・動作確認手順順** に並べることで、cognee 側でも上から順に取り込めば動く構造を確保しています。

```
PR タイトル: feat(claude-code): add cognee-graph-memory toolkit (cherry-pick friendly)

コミット構成:
  1. feat: add MCP server startup wrapper and graph importer
     - src/main_src/start_cognee_mcp.py
     - src/main_src/import_to_graph.py
     - メインツール（ローカル LLM 用 MCP 起動・グラフ投入）

  2. feat: add sample knowledge corpus
     - knowledge/sample_knowledge/*.md (5 ファイル)
     - src/sample_src/load_sample.py / delete_sample.py
     - 動作確認で使うサンプル

  3. feat: add knowledge ingestion utilities (split + retry)
     - src/knowledge_src/split_knowledge.py
     - src/knowledge_src/import_knowledge.py
     - 大量ナレッジの分割投入・リトライ補助ツール

  4. feat: add auto-accumulation harness
     - harness/hooks/*
     - harness/rules/*
     - harness/settings.example.json
     - Claude Code セッション自動蓄積ハーネス（ローカル LLM 想定）

  5. docs: add setup, usage, and harness guides
     - README.md / LICENSE / CHANGELOG.md
     - config/.env.example
     - docs/SETUP.md / GETTING_STARTED.md / HARNESS_GUIDE.md
```

### 6-2. 配置先ディレクトリ

`cognee-integrations` の慣習（`integrations/<対象アプリ名>/` で対象アプリごとに単一プラグインを横並び配置）に従い、**既存の公式プラグイン `integrations/claude-code/`（公式 `cognee-memory`）に上書き・混入せず、横並びの独立ディレクトリに配置**:

```
cognee-integrations/
└── integrations/
    ├── claude-code/                         ← 既存（公式プラグイン・触らない）
    └── claude-code-cognee-graph-memory/    ← ここに配置（横並び独立）
        ├── README.md
        ├── LICENSE
        ├── CHANGELOG.md
        ├── config/
        │   └── .env.example
        ├── docs/
        │   ├── SETUP.md
        │   ├── GETTING_STARTED.md
        │   └── HARNESS_GUIDE.md
        ├── harness/
        │   ├── CLAUDE_md_sample.md
        │   ├── settings.example.json
        │   ├── hooks/
        │   │   ├── auto_remember_completion.py
        │   │   ├── auto_remember_user_message.py
        │   │   └── cognee_remember_flusher.py
        │   └── rules/
        │       └── cognee_memory_usage.md
        ├── knowledge/
        │   ├── sample_knowledge/
        │   │   ├── 01_claude_code_tips.md
        │   │   ├── 02_software_dev_lessons.md
        │   │   ├── 03_design_decisions.md
        │   │   ├── 04_common_errors.md
        │   │   └── 05_graph_memory_operations.md
        │   ├── user_chunks/
        │   │   └── README.md
        │   └── user_knowledge/
        │       └── README.md
        └── src/
            ├── knowledge_src/
            │   ├── split_knowledge.py
            │   └── import_knowledge.py
            ├── main_src/
            │   ├── start_cognee_mcp.py
            │   └── import_to_graph.py
            └── sample_src/
                ├── load_sample.py
                └── delete_sample.py
```

「勝手に既存プラグインに混ぜる PR」はメンテナー目線で嫌われやすいため、**横並び独立配置** を必ず守ります。


## 7. 実作業手順（2 段階運用）

PR は公式メンテナーが見る公開操作で、不整合・コピペミス・構造ミスがあると印象を損ねます。本作業も初回で完璧に書ききれる保証はないため、以下の **2 段階で進めました**。

| Phase | 内容 | 公開影響 | 失敗時の影響 |
|---|---|---|---|
| **Phase 1: ローカル検証** | Fork → clone → ブランチ → コード移植 → lint/import チェック | なし（完全ローカル）| clone を破棄して再実行可能 |
| **Phase 2: 公開 PR** | push + PR 提出 | あり（cognee 側に通知）| Phase 1 で手順を確定してから実施するためほぼ問題なし |

絶対ルール:

1. Phase 1 を完走（lint・import すべてエラーなし・ファイル配置が想定通り）するまで Phase 2 に進まない
2. Phase 1 で問題が見つかったら手順書を修正してから再実行（手順書の精度を上げる）
3. Phase 2（push & PR 提出）は最終確認後にのみ実施

### 7-1. 事前準備

```bash
# Git 設定確認（DCO 署名のため）
git config --global user.name "JapanNomu"
git config --global user.email "japannomu@gmail.com"

# DCO 署名コミット用エイリアス
git config alias.cos "commit -s"
```

### 7-2. Fork & Clone

GitHub UI で `topoteretes/cognee-integrations` を Fork（`JapanNomu/cognee-integrations` になる）。Fork 完了後、ローカルに clone:

```bash
cd /tmp
git clone https://github.com/JapanNomu/cognee-integrations.git
cd cognee-integrations

# upstream を追加（同セッション内で本家の最新を取得できるようにする）
git remote add upstream https://github.com/topoteretes/cognee-integrations.git
```

### 7-3. ブランチ作成

```bash
git checkout -b feat/claude-code-cognee-graph-memory
```

### 7-4. コード移植（5 コミット）

```bash
SRC=/path/to/JapanNomu-tools/.../claude-code-cognee-graph-memory
DST=integrations/claude-code-cognee-graph-memory

# コミット1: MCP 起動ラッパー
mkdir -p $DST/src/main_src
cp $SRC/src/main_src/*.py $DST/src/main_src/
git add $DST/src/main_src/
git commit -s -m "feat(claude-code-cognee-graph-memory): add MCP server startup wrapper and graph importer"

# コミット2: サンプルナレッジ
mkdir -p $DST/knowledge/sample_knowledge $DST/src/sample_src
cp $SRC/knowledge/sample_knowledge/*.md $DST/knowledge/sample_knowledge/
cp $SRC/src/sample_src/*.py $DST/src/sample_src/
git add $DST/knowledge/sample_knowledge/ $DST/src/sample_src/
git commit -s -m "feat(claude-code-cognee-graph-memory): add sample knowledge corpus"

# コミット3〜5 も同様
```

すべて `git commit -s`（DCO 署名）で作成。

### 7-5. ローカル動作確認

```bash
# lint チェック
cd integrations/claude-code-cognee-graph-memory
ruff check --target-version py310 --line-length 100 --select E,F,I,W src/ harness/
# → All checks passed!

# import チェック
$VENV_PY -c "import sys; sys.path.insert(0, 'src/main_src'); import start_cognee_mcp; import import_to_graph; print('main_src OK')"
$VENV_PY -c "import sys; sys.path.insert(0, 'src/knowledge_src'); import split_knowledge; import import_knowledge; print('knowledge_src OK')"
$VENV_PY -c "import sys; sys.path.insert(0, 'src/sample_src'); import load_sample; import delete_sample; print('sample_src OK')"
# → main_src OK / knowledge_src OK / sample_src OK
```

両方ともエラーなしを確認してから次へ。

### 7-6. push & PR 提出

push と PR 提出は **別の操作** です:

| 操作 | 対象 | 公開影響 |
|---|---|---|
| **(1) push** | **自分の Fork** `JapanNomu/cognee-integrations`（origin）| 自分の Fork が更新されるだけ。公式（topoteretes/cognee-integrations）には影響なし |
| **(2) PR 提出** | GitHub UI で **公式 `topoteretes/cognee-integrations`** に「私の Fork のブランチを取り込んでください」と依頼 | 公式の Pull requests 一覧に新規 PR が追加される（cognee メンテナーに通知が飛ぶ）|

push しただけでは PR にはならないので、**push 後に GitHub UI で内容を最終確認できる**（diff・ファイル一覧を目視チェック）のが安心ポイントです。

```bash
# (1) 自分の Fork に push
git push origin feat/claude-code-cognee-graph-memory
```

push 後の確認:

- ブラウザで `https://github.com/JapanNomu/cognee-integrations/tree/feat/claude-code-cognee-graph-memory` を開く
- ファイルが想定通り配置されているか目視確認
- 5 コミットの粒度・コミットメッセージを確認

(2) GitHub UI で PR 提出:

- 自分の Fork（`JapanNomu/cognee-integrations`）を開くと「Compare & pull request」の黄色いバナーが出る
- クリックすると PR 提出画面に遷移
- **提出先（base）**: `topoteretes/cognee-integrations:main`（公式）
- **提出元（compare）**: `JapanNomu/cognee-integrations:feat/claude-code-cognee-graph-memory`
- PR タイトル・PR 説明文（Description）は次節のテンプレートをコピペ
- プレースホルダ（X 投稿の URL・commit SHA）を実値で埋める


## 8. PR 説明文テンプレート

cognee メンテナーが最初に読む PR Description は、以下の意図を含めて構成しました:

- **Summary**: PR の目的と Vasilije 氏の招待への参照
- **Cherry-pick friendly**: 「気に入ったコミットだけ取り込んでよい」という柔軟な姿勢
- **Per-commit assessment**: コミット単位での重複度を正直に開示
- **License**: MIT 著作権表示の保持を法的義務として通知
- **Maintenance**: ワンショット貢献を明示・長期メンテナンス義務を負わない姿勢を明記
- **Related articles**: 招待のきっかけになった Zenn 記事へのリンク
- **Source snapshot**: PR 提出時点の `JapanNomu/tools` リポジトリ・バージョン・commit SHA
- **DCO**: cognee-integrations では非必須だが安全策として記載

実際に PR #47 で提出した文面（プレースホルダを実値で埋めたもの）:

```markdown
## Summary

This PR contributes the cognee-graph-memory toolkit from
[JapanNomu/tools](https://github.com/JapanNomu/tools) (MIT License),
following the invitation by @tricalt on X
(https://x.com/tricalt/status/2050962347527749967).

## Cherry-pick friendly

The toolkit is provided as a complete unit, but each commit is
logically independent. Please feel free to cherry-pick only the
commits that fit cognee's roadmap, and close the rest.

## Per-commit assessment

| Commit | Feature | Overlap with existing official plugin |
|--------|---------|---------------------------------------|
| 1 | MCP server startup wrapper + graph importer | Different focus — official plugin targets cloud APIs, this targets local LLM (Ollama qwen2.5:14b) |
| 2 | Sample knowledge corpus + sample loaders | None — useful for learning and verification |
| 3 | Knowledge ingestion utilities (split + retry) | None — complementary feature for bulk ingestion |
| 4 | Auto-accumulation harness (hooks) | **Significant overlap** — official `integrations/claude-code/` has more hooks (6 vs 2) and slash commands. Consider this commit as reference only. |
| 5 | Documentation, config, and license | Partial — covers local-LLM setup, usage, and harness configuration |

## License

This contribution is licensed under the MIT License,
Copyright (c) 2026 JapanNomu. The original LICENSE file is included
in this PR.

Per MIT license terms, the copyright notice and license text
**must be retained** in all copies or substantial portions of
the included files. Please ensure the LICENSE file (or equivalent
attribution in file headers / NOTICE / LICENSE-3RD-PARTY file)
is preserved in this integration directory.

## Maintenance

This is a one-time contribution.
The original repository at JapanNomu/tools will continue to evolve
independently. I won't be able to commit to long-term maintenance
of this code within cognee, but feel free to take, modify, or
remove as needed.

## Related articles

A comparison between cognee's official Claude Code plugin and this
toolkit (which prompted Vasilije's invitation):
- [Zenn article (Japanese): cognee official vs toolkit](https://zenn.dev/japannomu/articles/20260503_cognee-official-vs-toolkit)
- [Zenn article (Japanese): toolkit introduction](https://zenn.dev/japannomu/articles/20260502_cognee-graph-memory-toolkit)

## Source snapshot

- Source repo: https://github.com/JapanNomu/tools
- Source version: v0.2.1 (cognee 1.0.5 verified, lint-clean)
- Source commit SHA: c6e7e036cfcc8766ea576cb84ae7ff4613e9629c (v0.2.1 lint cleanup)

## DCO

I affirm that all code in every commit of this pull request
conforms to the terms of the Topoteretes Developer Certificate
of Origin.
```

提出時の Title:

```
feat(claude-code): add cognee-graph-memory toolkit (cherry-pick friendly)
```


## 9. PR 提出結果（PR #47）

最終的な提出結果:

- **PR URL**: https://github.com/topoteretes/cognee-integrations/pull/47
- **PR 番号**: #47
- **状態**: 🟢 Open
- **規模**: 5 commits / 26 files changed / +2,839 行
- **base**: `topoteretes/cognee-integrations:main`
- **compare**: `JapanNomu/cognee-integrations:feat/claude-code-cognee-graph-memory`
- **Allow edits by maintainers**: ✅（メンテナーが修正コミットを直接追加可能）

提出後、Vasilije 氏の招待ツイートに対するリプライで PR #47 を報告しました。これにより:

- 招待を出した本人にダイレクトに通知
- やり取りの履歴が一連のスレッドとして残る
- 押し付けがましくない（招待への応答という文脈）


## 10. レビュー対応の心構え

PR 提出後、以下の状況に対応する準備をしました。

### 10-1. CI が落ちた場合

- 修正してコミット追加（`git commit -s` で署名）
- `git push` で同じブランチに追加 push（PR は自動更新される）

### 10-2. メンテナーから修正依頼が来た場合

- 議論に応答
- 必要なら修正コミット追加
- 「メンテ継続を引き受ける」プレッシャーがかかったら以下のテンプレで断る

### 10-3. 「メンテナーになって」と打診された場合の断り方

```
Thank you for the offer. I'd prefer to keep contributing as a
one-time submission. The original repository at JapanNomu/tools
will continue to evolve, but I won't be able to commit to
long-term maintenance of this code within cognee.

Feel free to take it as-is, modify, or remove as needed.
If anyone in the cognee community wants to take over maintenance
of this contribution, that would be wonderful.
```

これは OSS 文化として完全に正当な姿勢で、過去 cognee-community の単発貢献者も同様のスタンスでした。

### 10-4. 著作権表示の削除を要求された場合

- **拒否してよい**（MIT 法的要件のため）
- 「MIT License requires the copyright notice to be preserved in all copies or substantial portions of the Software.」と返答
- 万が一拒否されたら PR 自体を取り下げる権利あり

ただし MIT は最も緩いライセンスの一つで、著作権表示削除を要求されることは実務上ほぼないと予想されます。


## 11. リスクと回避策

| リスク | 発生確率 | 回避策 |
|---|---|---|
| メンテナー継続を要請される | 低（過去 PR 履歴より）| PR 説明文で one-time と明示・テンプレで断る |
| 著作権表示削除を要求される | 極低（MIT は法的要件）| 拒否・最悪 PR 取り下げ |
| ハーネス重複で全却下 | 中 | 説明文で正直に重複を明示・cherry-pick 推奨 |
| cognee 1.0.5 / 1.0.6 で API 破壊変更 | 中 | PR 提出前に最新 cognee で動作確認・bounded range で依存指定 |
| LICENSE 未配置リポへの提出 | 中 | PR 本文に MIT ライセンス著作権表示の保持を法的通知として記載 |
| 規模が大きすぎてレビュー停滞 | 低（既存 PR と同規模）| コミット分割で読みやすくする |
| 日本語コメント混入で却下 | 低 | チェックリスト F で確認 |


## 12. まとめ

本記事は、cognee 共同創業者からの招待を受けてから PR #47 を提出するまでの一連の作業を記録したものです。要点:

1. **戦略を先に決める**: ワンショット貢献・MIT 保持・cherry-pick 可能 PR の方針を最初に確定
2. **チェックリストで潰す**: ライセンス・衝突確認・API 互換性・規約・DCO・機密混入・lint/import・公式との重複評価の 8 項目を機械的に検証
3. **lint は事前に潰す**: PR 提出後に CI で落ちる前に v0.2.1 として手元で修正（コード動作影響ゼロを保証する厳格な修正方針）
4. **2 段階運用**: Phase 1 ローカル検証で手順精度を上げてから Phase 2 公開 PR
5. **PR 説明文を丁寧に書く**: 重複度の正直開示・cherry-pick 推奨・MIT 法的通知・ワンショット明示
6. **招待ツイートにリプライで報告**: 招待者にダイレクト通知 + 押し付けがましくない
7. **レビュー対応テンプレを準備**: メンテナー打診の断り方など事前準備

OSS への PR 提出が初めてでも、上記のように **「事前に潰せるものは全部潰す」** + **「2 段階運用で安全に進める」** + **「正直さ・柔軟性を PR 説明文で示す」** の組み合わせで、メンテナー目線の負担を最小化できます。

PR がマージされる/されないにかかわらず、JapanNomu の名前は cognee 公式リポジトリの git history に永続記録されます。これは OSS 文化の良いところで、貢献の足跡が残る仕組みです。

### リソース

- 英語版リポジトリ: https://github.com/JapanNomu/tools
- 日本語版リポジトリ: https://github.com/JapanNomu/tools-ja
- v0.2.1 リリース: https://github.com/JapanNomu/tools/releases/tag/v0.2.1
- **cognee 公式へ提出した PR**: https://github.com/topoteretes/cognee-integrations/pull/47
- 関連 Zenn 記事:
  - [Cognee グラフメモリ × Claude Code — 実用ツールキット公開](https://zenn.dev/japannomu/articles/20260502_cognee-graph-memory-toolkit)
  - [Cognee 公式 Claude Code プラグインと自作ツールキットを比較してみた](https://zenn.dev/japannomu/articles/20260503_cognee-official-vs-toolkit)
  - [Cognee 1.0.5 / Ladybug DB 対応（v0.2.0）](#)
