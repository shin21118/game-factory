# game-factory 段階構成設計（全体設計）

Status: レビュー済み設計（2026-07-19）
前提: [DEVELOPMENT_BRIEF.md](DEVELOPMENT_BRIEF.md) の確定済み前提に従う。

> この文書は 5 Stage 全体の設計方針を示す。**振る舞いの正本は openspec/ に固定する**。各 Stage の着手時に、この設計から該当部分を OpenSpec change として propose し、承認後に実装する。この文書自体は実装の根拠にしない。

## 0. 設計の骨格: 固定 Workflow の中に Agent を置く

原則「Agent より先に固定 Workflow を検討」を構造で表現する。

```
[固定 Workflow（決定論的な状態機械。コードで制御）]
 企画入力 → 計画生成 → 【人間承認】→ 生成実行 → 検証 → 【差分承認】→ 確定
                │                        │
                └── ここだけ LLM ────────┘
                    「計画生成」= 単発 LLM 呼び出し（Agent loop 不要）
                    「生成実行」= 単一 Agent loop（Tool を選んで実行を繰り返す）
```

- run 全体の遷移（どの状態から何へ進めるか、どこで人間を待つか）はコードが決める。LLM は決めない。
- LLM の自由裁量は「生成実行」ステップの内側の Tool 選択に限定する。これが旧repoの「16 phase を LLM 呼び出し境界にして coherence を失った」失敗の反転。
- Multi-Agent 化は Stage 5 の評価まで行わない。

## 1. Stage 構成（PROGRAM.md Block 20–24 と 1:1）

### Stage 1（Block 20）単一 Agent＋複数 Tool

完成物: 企画入力 → 計画 → 承認 → 許可された Tool でファイル生成 → 結果プレビュー。

- **Agent loop（自作）**: Anthropic SDK の messages API を while loop で回す。tool_use を受けたら Tool Registry 経由で実行し、tool_result を返す。1 step ごとに State（messages、生成ファイル一覧、step 数）を DB に永続化。
- **Tool Registry**: Tool を宣言的に登録（name、説明、引数 schema、権限宣言）。Agent には Registry 経由でのみ実行させる。
- **State**: `Run`（id、企画入力、状態、計画、費用、step 数）と `RunEvent`（append-only のイベントログ）。
- **停止条件**: max 30 step、同一 Tool エラー連続 3 回、ユーザーキャンセル、（Stage 2 から）費用上限。停止時も State は保存され原因が記録される。
- **人間承認**: 計画承認（生成開始前）。生成物はプレビューのみで、確定は Stage 4 の差分承認まで手動 git 操作。
- **UI（最小）**: 企画入力フォーム／計画の表示と承認・却下／生成イベントの一覧／Canvas ゲームの iframe プレビュー。

Stage 1 の Tool セット:

| Tool | 説明 | ファイル権限 | Shell | Network |
|---|---|---|---|---|
| `list_files` | run workspace の一覧 | `workspace/runs/<runId>/` 読み | なし | なし |
| `read_file` | ファイル読み取り | 同上 読み | なし | なし |
| `write_file` | ゲームファイル生成・上書き | `workspace/runs/<runId>/` 書き | なし | なし |
| `check_syntax` | JS/HTML の構文チェック | 同上 読み | 固定コマンドのみ（例: `node --check`） | なし |

上記以外（任意 shell、外部 network、workspace 外のファイル）は Tool として存在させない。「権限で禁止」より先に「Tool を与えない」。

### Stage 2（Block 21）Harness・Sandbox・Budget

完成物: 権限の宣言と強制、Sandbox 実行、上限と強制停止、監査ログ。

- **権限の強制**: Tool Registry の権限宣言を実行層で検証（path の正規化と prefix 検査、コマンド allowlist）。Agent のプロンプト指示に依存しない。
- **Sandbox**: 生成ゲームの実行は sandbox 属性付き iframe（`allow-scripts` のみ、same-origin 禁止）を最小実装とする。プロセス分離・コンテナへの拡張は調査タスクの結果で判断。
- **Budget**: run 単位 $1・30 step、月間 $30。API レスポンスの usage から費用を積算し `CostLedger` に記録。超過時は loop を強制停止し、状態 `HALTED_BUDGET` で保存。上限値の変更は人間承認。
- **監査ログ**: すべての Tool 呼び出し（引数、結果、拒否理由）と権限拒否を `RunEvent` に記録。

### Stage 3（Block 22）非同期 Job・Checkpoint

完成物: 長時間処理の Job 化、進捗、キャンセル、再開、Retry、二重実行防止。

- **Queue/Worker**: Agent 実行を HTTP リクエストから切り離し、Node worker が Job を取得して実行。具体実装（DB ベース自作か軽量ライブラリ）は調査後に確定。
- **Job 状態機械**: `PENDING → RUNNING → WAITING_APPROVAL → RUNNING → DONE / FAILED / CANCELLED / HALTED_*`。遷移はコードで制御（旧repo run-store の状態機械パターンを参照。コードは移植しない）。
- **Checkpoint**: Stage 1 で導入済みの step 単位永続化を再開に接続。失敗・停止した run を最後の checkpoint から resume できる。
- **冪等性**: Job 投入に idempotency key。同一 run の二重実行を DB 制約で防止。Retry は回数上限付き。

### Stage 4（Block 23）Agent 操作 UI・差分承認

完成物: 進捗タイムライン、生成物プレビュー、Git 差分、承認・却下・再実行。

- **進捗タイムライン**: `RunEvent` を時系列表示（計画、Tool 呼び出し、承認、停止、費用）。
- **差分承認**: run workspace を git 管理し、生成・修正を diff として提示。承認で commit（確定）、却下で破棄、コメント付き再実行で Agent へ差し戻し。**承認前に成果物を確定しない**。
- **監査と rollback**: 確定は常に git 履歴に残り、巻き戻せる。

### Stage 5（Block 24）分業判断・総合リリース

完成物: 単一 Agent の評価と、Multi-Agent 分離の要否判断（文書）。

- 複数 run の費用・step 数・承認却下率・失敗分類を集計し、「どこがボトルネックか」を根拠に Planner／Executor／Reviewer 分離の費用対効果を評価。
- 分離する場合も、しない場合も、理由を説明できる形で記録（decisions.md へ同期）。
- 全 Stage の設計判断のまとめ（P4 の学習成果物）。

## 2. openspec への固定の提案

capability（`openspec/specs/` の単位）を次の 6 つとする。Stage と 1:1 ではなく、振る舞いの責務単位で切る:

| capability | 内容 | 主に固定される Stage |
|---|---|---|
| `agent-run` | run のライフサイクル、Agent loop、State、停止条件 | 1, 3 |
| `tool-registry` | Tool の宣言、引数 schema、権限宣言と強制 | 1, 2 |
| `sandbox-execution` | 生成物の隔離実行 | 2 |
| `budget-control` | 費用・step 上限、積算、強制停止 | 2 |
| `job-orchestration` | Queue、Job 状態機械、Checkpoint、冪等性 | 3 |
| `approval-flow` | 人間承認ポイント、差分承認、監査、rollback | 1, 4 |

運用: 各 Stage 着手時に change を 1 つ propose（`add-stage1-agent-core` から開始）→ 承認 → 実装 → `openspec validate` → archive で specs へ反映。複数 Stage を 1 change に詰めない。

## 3. データモデル（初期案。詳細は OpenSpec change で確定）

- `Run`: id、企画入力、計画、状態、費用累計、step 数、作成・更新時刻。
- `RunEvent`: runId、種別（plan / tool_call / tool_denied / approval / halt / error / cost）、payload、時刻。append-only。
- `CostLedger`: runId、API 呼び出し単位の usage・cost、月次集計ビュー。
- `Job`（Stage 3）: runId、idempotency key、状態、retry 回数、checkpoint 参照。

## 4. 旧repoから参照する設計パターン（コード移植はしない）

| 参照元（agentic-game-factory） | 使い方 |
|---|---|
| `docs/REBUILD_KNOWLEDGE_TRANSFER.md` | 設計全体の一次参照（失敗 root cause と Don'ts） |
| `control-plane/src/run-store.js` | 状態機械・resume・events.jsonl の設計パターン |
| `control-plane/src/observability/cost-ledger.js` | 費用 schema の参考 |
| `control-plane/src/real-executors/validators/runtime-sandbox-evaluator.js` | Playwright による「実際に遊べるか」検証（Stage 2 以降の検証 Tool の参考） |
| `docs/operations/sprints/sprint5_5/QUALITY_CHARTER/` | 品質観点の語彙（ただし多軸 gate の再現はしない） |
