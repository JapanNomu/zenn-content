---
title: Cognee グラフ記憶ツールキット v0.3.0 — 読者報告のロック競合を、確率対処ではなくアーキテクチャで根本対処した話
authors:
  - JapanNomu
emoji: "\U0001F510"
type: tech
topics:
  - claudecode
  - cognee
  - mcp
  - ollama
  - ladybug
published: true
---

## はじめに

[Cognee 1.0.5 / Ladybug DB 対応版 v0.2.0](https://zenn.dev/japannomu/articles/20260504_20260504_cognee-1-0-5-ladybug-v0-2-0) のリリース直後、本ツールキットの v0.2.1 公開記事に **uzuchi 様** からコメントで「Ladybug DB が `Could not set lock on file` を踏んでいる」という報告をいただきました。

最終的にこの報告は、「flusher のリトライ実装で確率を下げる」ではなく **「そもそも 2 つ目の `cognee-mcp` プロセスが起動しない設計に作り変える」** というアーキテクチャ刷新につながりました。本記事は v0.3.0（および同日のドキュメント整理 v0.3.1 / v0.3.2）でやったことをまとめます。

リポジトリ:

- 🇬🇧 English: https://github.com/JapanNomu/tools
- 🇯🇵 日本語: https://github.com/JapanNomu/tools-ja


## 報告の中身

uzuchi 様から共有いただいた再現条件は以下のとおりでした。

- Windows ネイティブ Claude Code → `wsl.exe -d Ubuntu-24.04 -- python3 .../start_cognee_mcp.py` を stdio transport で MCP サーバとして登録
- `shared_ladybug_lock` 未設定 (= cognee デフォルト False)
- Redis 未起動
- 当時、Could not set lock on file を踏んでいた（その後ご自身で SubAgent 構成に切り替えて回避済み）

「shared_ladybug_lock=True / Redis 構成と SubAgent でのワークアラウンドは見つけたが、当時のロック発火そのものを再現する経路が知りたい」というニュアンスの報告でした。


## 原因の切り分け

手元 (Linux/WSL2 環境・Claude Code を Linux 側で起動する構成) で次の検証を実施しました。**uzuchi 様の構成そのものは再現できておらず、原因として疑われる「2 プロセスが同一 .lbug を奪い合う」状況を別構成で再現した**、という整理です。

### 1. ladybug 単体での発火条件

ladybug は kuzu の MIT フォーク。`/tmp/ladybug-src` に clone してロック実装を読みました。

- `src/storage/storage_manager.cpp:51-78` (initDataFileHandle): `read_only=False` のとき `O_LOCKED_PERSISTENT_FILE` を指定
- `src/common/file_system/local_file_system.cpp:139-172`: Linux 側は `fcntl(F_SETLK, F_WRLCK)` でファイル全体に **non-blocking 排他ロック**
- `read_only=True` なら NO_LOCK
- SQLite と異なり busy_timeout なし → **競合 = 即エラー throw**

専用テスト .lbug を新規作成し、

- プロセス A: `read_only=False` で .lbug を 30 秒保持 → `lslocks` で `POSIX 1M WRITE` ロック確認
- プロセス B: `read_only=False` で同じ .lbug を open 試行 → **`Could not set lock on file` 再現成功**

これで **「同じ .lbug を 2 プロセスが排他ロック付きで開きにいくと、後発側が即エラーになる」** という最小再現条件が確定しました。

### 2. ツールキット経由での再現

v0.2.x の構成 (= MCP cognee サーバが Claude Code 配下で 1 個起動している前提) に対して、

- Claude Code 起動中 (= `cognee-mcp` を 1 つ保持中)
- 別プロセスとして CLI ヘルパー `src/sample_src/delete_sample.py` / `load_sample.py` / `src/knowledge_src/import_knowledge.py` を実行
- → CLI が **新しい `cognee-mcp` プロセスを spawn** (StdioTransport)
- → 同じ .lbug を握りに行ってロック競合 → 上記 1 と同じ `Could not set lock on file`

ここで **「flusher 単体の並列度ではなく、Claude Code 常駐 MCP + 別プロセス CLI の競合」** が本物の発火条件であると判明しました。

加えて v0.2.x の `harness/hooks/cognee_remember_flusher.py` (cron / nohup daemon で動かすことを想定していた flusher) も **MCP サーバを呼ぶたびに `fastmcp.StdioTransport` で `cognee-mcp` を spawn する** 実装になっていたため、cron が走るたびに同じ条件で発火する潜在性を持っていました。

### 3. MCP 経由ならロック競合が起きないこと

対比検証として、Claude Code 内から `mcp__cognee__delete_dataset` を呼んだ場合（= 既存の `cognee-mcp` を共有する経路）は、Ladybug DB ロック競合が発生しないことを実機で確認しました。これが v0.3.0 設計の根拠です。


## 確率対処は本質的解決にならない

「リトライで確率を下げる」案も検討しましたが、JapanNomu の判断としては不採用にしました。理由:

- ladybug の `fcntl(F_SETLK, F_WRLCK)` は **busy_timeout なしの即時失敗** なので、リトライしても同時に握りに行った瞬間の挙動は変わらない
- v0.2.x 時代に「flusher 単体の並列度を 3 / 5 / 10 並列にしてもエラー再現せず」と記録していたのは、本物の発火条件 (= Claude Code 常駐 MCP との競合) を見ていなかったから
- そもそも「2 つ目の `cognee-mcp` を起動するか否か」を設計で握れる場所にいる以上、握る側が「起動しない設計」を選ぶのが筋

つまり問題は実装の確率ではなく **「v0.2.x の harness は別プロセスを spawn する前提で設計されていた」** こと。設計レイヤーで作り直すしか根本解決の道はない、と整理しました。


## v0.3.0 — アーキテクチャ刷新

### before / after

| 観点 | v0.2.x | v0.3.0 |
|---|---|---|
| キュー drain 主体 | OS レベル別プロセス flusher (`harness/hooks/cognee_remember_flusher.py`) | Claude Code セッション内 skill (`harness/skills/cognee-queue-flush/SKILL.md`) |
| MCP サーバ | drain のたびに `fastmcp.StdioTransport` で新規 spawn | 既存の MCP cognee サーバを共有 |
| `cognee-mcp` プロセス数 | 起動中 1 個 + drain ごとに +1 | 常に 1 個 (= Claude Code セッション開始時に起動された 1 個のみ) |
| ロック競合 | 起きうる (構造的) | **構造上発生しない** |
| スケジューラ | OS の `crontab -e` または `nohup --daemon` | Claude Code 内蔵 `/loop 5m cognee-queue-flush` または `CronCreate(...)` |

### Claude Code の skill が drain 主体になる

新規追加した `harness/skills/cognee-queue-flush/SKILL.md` は Markdown 形式の skill です。Claude Code は `/loop` または `CronCreate` でスケジュールされたタイミングで自動的にこの skill を発火し、AI 側がその内容に従って:

1. `~/.claude/cognee_pending_remembers.jsonl` を読む
2. 各エントリに対して `mcp__cognee__remember(data, dataset_name)` を呼ぶ (= 既に走っている MCP cognee サーバへのリクエスト)
3. 失敗判定 (3 重) → 失敗エントリは `~/.claude/cognee_failed_remembers.jsonl` に退避してキューに残す
4. 成功エントリだけ書き戻す
5. `(succeeded, failed, remaining)` をレポート

を実行します。Anthropic 公式ドキュメントが述べる「scheduled task は `live in the current process`」「MCP server は session start で 1 セッション 1 プロセス起動」をそのまま利用する設計です。

### サイレントデータロストも同時修正

調査の過程で `cognee-mcp` が失敗時に `is_error=False` を返しつつ本文に `Error:` 始まりテキストを入れる upstream 仕様に気付き、v0.2.x の flusher が戻り値を破棄していたためにキューから処理済みとして消されていたことを確認しました。

新 skill 内では 3 重失敗判定を必須化:

1. `is_error=True` → 失敗
2. `content[*].text` のいずれかが `Error:` 始まり → 失敗 (upstream 欠陥対策)
3. 例外発生 → 失敗

3 つすべてパスしたエントリだけを成功とみなし、それ以外は `~/.claude/cognee_failed_remembers.jsonl` に書きつつキューに残す。これで「次の発火で再試行される」状態が保たれます。

### CLI ヘルパーの運用制約

`src/sample_src/load_sample.py` / `src/sample_src/delete_sample.py` / `src/knowledge_src/import_knowledge.py` は依然として `cognee-mcp` を新規 spawn する作りです (これらは長時間処理かつ大量投入用途なので、スキル化せず CLI のままにしました)。

そのため v0.3.0 では運用ルールとして **「これらの CLI ツールは Claude Code を起動していない状態でのみ実行する」** を明文化し、各スクリプトの先頭にも警告を追加しました。削除に関しては Claude Code 起動中でも安全に呼べる代替手段として `mcp__cognee__delete_dataset` を案内しています。

### 1 起動あたりの drain 件数を環境変数で指定可能に

`mcp__cognee__remember` は LLM や PC スペックによって 1 件あたり数秒〜数十秒かかります。1 回の drain がスケジュール間隔 (デフォルト 5 分) を超えると次回発火と重なってしまうため、

- 環境変数 `COGNEE_QUEUE_FLUSH_MAX_PER_RUN` で 1 起動あたりの上限を指定可能 (デフォルト **3**)
- 残りはキューに残して次回発火で処理

という設計にしました。`docs/HARNESS_GUIDE.md` Step 4 と `docs/SETUP.md` §2-4 に PC スペック別の目安 (cloud LLM 10〜20 / qwen2.5:14b GPU 3〜5 / CPU のみ 1〜2) を併記しています。

### セッション再起動後も schedule を維持するには

`/loop 5m cognee-queue-flush` は手軽ですがセッション内のみ有効で、Claude Code を終了すると消えます。再起動後も schedule を維持するには `CronCreate(... durable=true)` を使う必要があり、これは AI 側の Tool でしか指定できません (スラッシュコマンド `/loop` には `durable` オプションがありません)。

`docs/HARNESS_GUIDE.md` と `docs/SETUP.md` には次の依頼文例を載せました:

> Claude Code のチャットで AI に依頼:
> 「`CronCreate` を `cron="*/5 * * * *"`、`prompt="cognee-queue-flush"`、`recurring=true`、`durable=true` で呼んでください。再起動後もキューの drain が続くように。」

AI が一度この Tool を呼べば登録は `~/.claude/scheduled_tasks.json` に保存され、以降は依頼不要です。


## 動作確認

### 新アーキテクチャ用テスト 21 件

v0.2.x の FLUSHER 系テストは「別プロセス flusher の挙動」を検証するもので、v0.3.0 の skill アーキテクチャでは検証対象がそもそも違うため、新規に BATCH 系テストを 21 件追加しました。

| テスト工程 | 件数 | 主な観点 |
|---|---|---|
| UT (単体テスト) | 12 | キュー追記・3 重失敗判定・MCP 共有呼び出し |
| IT (内部結合テスト) | 4 | キューファイル I/O・混在エントリ・malformed JSON・フェイルセーフ |
| ET (外部結合テスト) | 3 | 既存 MCP 共有での remember 成功 / 報告事象の再現条件下での MCP 共有成功 / 対比検証 (旧 CLI 経路で発火) |
| ST (システムテスト) | 2 | ユーザー視点 E2E フロー / データ完全性とロック競合発生しない設計の実機実証 |

cognee 1.0.8 環境で全件 PASSED。

### SETUP 検証ゲート

リリース前必須の SETUP 検証ゲートで、報告事象と同等の再現条件下で MCP 経由の `remember` / `search` / `delete_dataset` を累計 50 回以上呼び出して **ロック競合 0 回** を確認しました。skill 配置 → スケジュール登録 → 即時起動 → 5 分後の自動発火 → 上限制御による段階的 drain → 0 件発火 → スケジュール解除、まで一連を実機で実証しました。


## v0.3.1 / v0.3.2 — ドキュメントクリーンアップ

v0.3.0 リリース後、配布物のドキュメント・設定ファイルに開発プロジェクト内のみで通用する内部 ID (障害 ID / 要件 ID / タスク ID) が残っていることに気付きました。配布物受領者にとっては参照先がなく意味不明なため、削除して具体的な記述に書き換えました。

- v0.3.1: Markdown ファイル群から内部 ID を全件削除
- v0.3.2: v0.3.1 で取りこぼしていた `config/.env.example` (配布物に同梱されていない旧スクリプトの設定例セクション) と `harness/` 配下の docstring・コメントを修正

**コード・要件・設計・テスト成果物には一切変更ありません。**


## マイグレーション (v0.2.1 → v0.3.x)

1. v0.3.2 を取得します。
2. `docs/HARNESS_GUIDE.md` Step 1 を再実行 (hook 再コピー + 新 skill ディレクトリ `harness/skills/cognee-queue-flush` を `~/.claude/skills/` へコピー)。
3. 不要になった `~/.claude/hooks/cognee_remember_flusher.py` を削除し、`crontab -e` から `*/5 * * * * .../cognee_remember_flusher.py` 行を消去。
4. Claude Code を再起動 (新 skill を認識させるため)。
5. 新セッション内で schedule を 1 回だけ登録: `/loop 5m cognee-queue-flush` を入力するか、再起動後も維持したい場合は AI に `CronCreate(cron="*/5 * * * *", prompt="cognee-queue-flush", recurring=true, durable=true)` を実行してもらう。


## まとめ

v0.3.x は「読者から共有いただいた再現条件を前にして、確率対処ではなく **そもそも発火条件が成立しない設計** に作り変えた版」です。

- v0.2.x: 別プロセス flusher が `cognee-mcp` を spawn → 競合する余地あり
- v0.3.0: Claude Code 内 skill が既存 `cognee-mcp` を共有 → 競合する余地が構造上ない

OSS の改善という意味では、報告いただいた事象に対して「自分のコードベースの設計が適切ではなかった」と素直に認めて作り直す機会をいただいた形になりました。

### 謝辞

Zenn 記事 v0.2.1 のコメントで `Could not set lock on file` の事象と再現条件を詳細に共有してくださった **uzuchi 様** に深く感謝いたします。ご報告がなければ、ロック競合発生の余地を残したまま、より長期間にわたって配布を続けていたはずです。本当にありがとうございました。

### リソース

- 英語版 v0.3.2: https://github.com/JapanNomu/tools/releases/tag/v0.3.2
- 日本語版 v0.3.2: https://github.com/JapanNomu/tools-ja/releases/tag/v0.3.2
- 関連 Zenn 記事:
  - [Cognee グラフメモリ × Claude Code — 実用ツールキット公開 (v0.1.10)](https://zenn.dev/japannomu/articles/20260502_cognee-graph-memory-toolkit)
  - [Cognee グラフ記憶ツールキットを Cognee 1.0.5 / Ladybug DB に対応した（v0.2.0）](https://zenn.dev/japannomu/articles/20260504_20260504_cognee-1-0-5-ladybug-v0-2-0)
  - [自作ツールキットが cognee 共同創業者から招待を受け、公式リポへ PR を出すまでの全記録（v0.2.1）](https://zenn.dev/japannomu/articles/20260504_20260504_pr-to-cognee-integrations-v0-2-1)
