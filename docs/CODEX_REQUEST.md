# Codex 開発依頼書（P4 game-factory）

Status: 確定（2026-07-19）。PROGRAM.md §7「Codexへ開発させる標準手順」に準拠。
前提文書: [DEVELOPMENT_BRIEF.md](DEVELOPMENT_BRIEF.md)（目的の正本）／[STAGE_DESIGN.md](STAGE_DESIGN.md)（全体設計）／[UX_DESIGN.md](UX_DESIGN.md)（UI/UX の正本）／`openspec/changes/add-stage1-agent-core/`（Stage 1 の振る舞い正本）

## 0. 実装前の定義（§7.1 の 9 項目）

| 項目 | 内容 |
|---|---|
| 誰が使うか | 開発者本人（Agent 操作 UI の利用者） |
| 何ができれば完成か | 各 Stage の完了条件（下表）＋ PROGRAM.md Block 20–24 の完了条件 |
| 今回作らないもの | Multi-Agent、企画上流 pipeline、自動修復ループ、複数ジャンル・実ストア公開・収益化 |
| 入力と出力 | 入力: ゲーム企画テキスト。出力: HTML5 Canvas ミニゲーム（少数ファイル）＋ run の追跡記録 |
| 保存するデータ | Run / RunEvent（append-only）/ CostLedger / Job（Stage 3〜）。SQLite から開始 |
| AIに任せる処理 | 計画生成（単発呼び出し）と、生成実行ステップ内の Tool 選択・ファイル生成のみ |
| 使用モデル | 既定: `claude-sonnet-4-6`（計画・生成）、`claude-haiku-4-5`（vision self-check）。Opus 昇格は品質不足の実測後・人間承認（STAGE_DESIGN.md §4） |
| コードで処理する部分 | run の状態遷移、Tool 権限の強制、停止条件、費用積算、Queue/Job、承認フロー、UI |
| セキュリティ・倫理上の制約 | 下記「Tool 権限表」「費用上限」「人間承認ポイント」。API キーは `.env`（コミット禁止） |
| テスト項目 | 各 OpenSpec change の Scenario を必ずテスト化（特に: 承認前に生成しない、workspace 外へ書けない、上限で必ず停止、二重実行しない） |

## 1. 運用ルール（毎回の依頼に適用）

- **OpenSpec ゲート**: 実装は承認済みの OpenSpec change がある範囲のみ。Stage 着手時にまず `openspec-propose` で change を作り、人間承認を得てから実装する。Stage 1 は `add-stage1-agent-core` が承認対象として作成済み。
- 一度に 1 機能だけ依頼する。既存テストを先に確認させる。
- 実装後に lint・test・build を実行し、変更ファイル一覧と理由を説明する。
- 完了前に `openspec validate <change-name> --strict` を通す（CLI: `npx --yes @fission-ai/openspec@latest`）。
- stub・placeholder を「実装済み」と報告しない。gate を無効化して完了扱いにしない（旧repoの anti-dwarf 禁則）。
- 旧 `agentic-game-factory` のコードを移植しない。参照は設計パターンのみ（STAGE_DESIGN.md §5）。
- モデル・API の使い方は STAGE_DESIGN.md §4 の前提（adaptive thinking、structured outputs、streaming、prompt caching 前提の安定 prefix）に従う。旧世代のパラメータ（`budget_tokens`、`temperature`）を書かない。

## 2. Codex への最初の依頼文（§7.2 準拠・コピーして使う）

```text
docs/DEVELOPMENT_BRIEF.md、docs/STAGE_DESIGN.md、docs/UX_DESIGN.md、openspec/changes/add-stage1-agent-core/ を読んでください。

まだコードを書かず、次を提示してください。

1. 要件の理解（Stage 1 の範囲と非目標）
2. 最小の縦切り（tasks.md の順序への意見を含む）
3. 画面・API・データ構造
4. 各層の責務（固定 Workflow／Agent loop／Tool Registry／UI）
5. 技術選定の理由と代替案（Next.js 構成内での選択。スタック自体は確定済み）
6. セキュリティ・プライバシー上のリスク（特に生成コードの実行と path traversal）
7. テスト計画（OpenSpec の各 Scenario との対応表）
8. 今回作らないもの
```

## 3. Stage 別タスク分解・完了条件・レビュー観点

### Stage 1（Block 20、11/23–11/29）単一 Agent＋複数 Tool

- タスク分解: `openspec/changes/add-stage1-agent-core/tasks.md` を正本とする（scaffold → データ層 → 固定 Workflow → Tool Registry / Agent loop → UI → 検証）。
- 完了条件: 同 change の全 Scenario がテストで通る。実 run 1 本でブラウザで遊べるゲームが生成される。最初から Multi-Agent にしていない。Tool 権限と停止条件が明文化されている。
- レビュー観点:
  - 状態遷移がサーバ側コードのみで行われているか（LLM 出力が状態を直接変えていないか）。
  - Agent loop が step ごとに永続化しているか。system prompt が安定した内容（ゴール・制約・Tool 説明のみ）で、ルールの累積追加になっていないか。
  - 現行 API 前提が守られているか: adaptive thinking（`budget_tokens`・`temperature` なし）、計画は structured outputs、長い生成は streaming、`stop_reason: "refusal"` を含む分岐処理、usage の cache_read / cache_creation 記録。
  - prompt caching が実際に効いているか（cache_read_input_tokens がゼロなら prefix を汚す実装を疑う）。
  - Tool が Registry 経由でのみ実行され、4 Tool 以外が LLM に公開されていないか。
  - `write_file` の path 正規化と workspace 外拒否のテストがあるか。
  - 承認 API を経ずに GENERATING へ遷移できないことのテストがあるか。
  - UI が UX_DESIGN.md §7 の Stage 1 線引きに収まり、§3 の表示規則（承認はインラインカード＋上部バナー、タイムラインは RunEvent を単一情報源として描画）に従っているか。

### Stage 2（Block 21、11/30–12/6）Harness・Sandbox・Budget

- 着手時に change `add-stage2-harness-sandbox-budget` を propose（capability: `tool-registry` 拡張、`sandbox-execution` 新規、`budget-control` 新規）。
- タスク分解の目安: 権限強制の実行層検証（path 正規化・コマンド allowlist）→ iframe sandbox の属性固定 → 費用積算と強制停止 → 監査ログの網羅 → 権限逸脱テスト → vision self-check Tool（`run_visual_check`）の追加検討。
- 完了条件: Agent が許可範囲外を操作できない（テストで証明）。上限超過時に必ず停止する（$1/run・30 step・$30/月）。すべての Tool 呼び出しと拒否が監査ログに残る。
- レビュー観点:
  - 権限強制がプロンプト指示ではなく実行層の検証で行われているか。
  - 上限値がコード埋め込みでなく設定として持たれ、変更に人間承認が要るか。
  - 超過停止後も State が保存され原因（HALTED_BUDGET）が追跡できるか。
  - sandbox iframe が `allow-scripts` のみで same-origin を許していないか。

### Stage 3（Block 22、12/7–12/13）非同期 Job・Checkpoint

- 着手時に change `add-stage3-async-jobs`（capability: `job-orchestration` 新規、`agent-run` 拡張）。Queue/Worker の具体実装は propose 前の調査結果（BRIEF「調査後に決める」）を反映。
- タスク分解の目安: Job テーブルと状態機械 → worker プロセス → 進捗のポーリング/ストリーム → キャンセル → checkpoint からの resume → idempotency key と二重実行防止 → retry 上限。
- 完了条件: ページを閉じても処理状態が失われない。同じ Job を重複実行しない。失敗 run を checkpoint から再開できる。
- レビュー観点:
  - 二重実行防止が DB 制約（unique）で保証されているか（アプリ層チェックのみは不可）。
  - resume が「最後に保存された step」から正しく再構築されるか。
  - retry が上限付きで、retry 対象と非対象のエラーが分類されているか。

### Stage 4（Block 23、12/14–12/20）Agent 操作 UI・差分承認

- 着手時に change `add-stage4-approval-ui`（capability: `approval-flow` 拡張）。
- タスク分解の目安: run workspace の git 化 → 差分表示 → 承認 = commit・却下 = 破棄 → コメント付き再実行（Agent への差し戻し）→ 進捗タイムライン UI → 監査表示（費用・step・承認履歴）。
- 完了条件: Agent が何をしたか UI で追跡できる。ユーザー承認前に成果物を確定しない。承認・却下・再実行が動く。
- レビュー観点:
  - 承認操作なしに commit される経路が存在しないか。
  - 却下時に workspace が安全に巻き戻るか。
  - タイムラインが RunEvent の正本から描画されているか（別集計を持って乖離していないか）。

### Stage 5（Block 24、12/21–12/27）分業判断・総合リリース

- 実装より評価が主。複数 run の費用・step・却下率・失敗分類の集計と、Multi-Agent 分離の要否判断文書を作る。
- 完了条件: Agent 追加（または追加しない）の理由を説明できる。全 Stage の設計判断がまとまり、goal_roadmap の decisions.md に同期されている。
- レビュー観点: 判断がデータ（集計）に基づいているか。「分離しない」も正当な結論として扱われているか。分離する場合、最初の候補が独立 Reviewer（fresh context の検証 Agent）になっているか — Planner/Executor の生成側分割は旧repoの phase 分割の失敗と同型であり、根拠がない限り選ばない。

## 4. Tool 権限表（正本。変更は人間承認＋OpenSpec change）

| Tool | 導入 Stage | ファイル | Shell | Network | 備考 |
|---|---|---|---|---|---|
| `list_files` | 1 | `workspace/runs/<runId>/` 読み | なし | なし | |
| `read_file` | 1 | 同上 読み | なし | なし | |
| `write_file` | 1 | 同上 書き | なし | なし | path 正規化・範囲外拒否必須 |
| `check_syntax` | 1 | 同上 読み | 固定コマンドのみ | なし | 例: `node --check` |
| `run_playtest`（候補） | 2–3 | 同上 読み | Playwright 起動（固定） | localhost のみ | 旧repoの runtime 検証パターン参照 |
| `run_visual_check`（候補） | 2–3 | 同上 読み | なし | Anthropic API のみ | スクリーンショットを Haiku 4.5 で判定（タイトル画面・アセット描画の確認）。判定コストも CostLedger に記録 |
| 任意 shell / 外部 network / workspace 外 | — | 禁止 | 禁止 | 禁止 | Tool として存在させない |

## 5. 費用上限（正本）

- 1 run: **$1**、30 step。月間: **$30**。（根拠: 既定モデル Sonnet 4.6・prompt caching 有効で 1 run 概算 $0.7–0.9。2026-07 時点の単価。）
- 超過時は強制停止（`HALTED_BUDGET`）し、State を保存して UI に通知する。自動でのリトライ・上限引き上げはしない。
- 上限値の変更は人間承認を経て設定を更新する（コード変更で回避しない）。
- 開発中の Codex 自身の動作確認呼び出しも同じ CostLedger に記録する。

## 6. 人間承認ポイント（正本）

| # | 地点 | 内容 |
|---|---|---|
| 1 | Stage 着手前 | OpenSpec change の承認（proposal / design / tasks / spec delta） |
| 2 | 生成開始前 | 計画承認（生成予定ファイル・仕様要約・想定 step 数） |
| 3 | 成果物確定前 | 差分承認（Stage 4 から。承認 = commit、却下 = 破棄） |
| 4 | 上限・権限の変更時 | 費用上限、Tool 権限表、停止条件の変更 |
| 5 | 公開・エクスポート | P4 期間中はすべて手動（自動公開なし） |
