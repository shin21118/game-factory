# Change: Stage 1 単一 Agent＋複数 Tool のコア導入

## Why

P4（v4 Block 20、2026/11/23–11/29）の最初の縦切りとして、「企画入力 → 計画 → 人間承認 → 許可された Tool でのファイル生成 → 結果プレビュー」を成立させる。正本判断（2026-07-19: `game-factory` を正本、旧repoは参照アーカイブ）と `docs/DEVELOPMENT_BRIEF.md`（確定版）、`docs/STAGE_DESIGN.md` の Stage 1 設計に基づく。

旧 `agentic-game-factory` の教訓により、run 全体の遷移は固定 Workflow（決定論的な状態機械）が持ち、LLM の自由裁量は生成実行ステップ内の Tool 選択に限定する。最初から Multi-Agent にしない。

## What Changes

- 新規 capability `agent-run`: run のライフサイクル（固定 Workflow）、自作 Agent loop、State 永続化、停止条件。
- 新規 capability `tool-registry`: Tool の宣言的登録と、Stage 1 の 4 Tool（list_files / read_file / write_file / check_syntax）。権限は run workspace 配下に限定。
- 新規 capability `approval-flow`: 計画承認（生成開始前）の人間承認ポイント。
- 最小 UI: 企画入力、計画の承認・却下、イベント一覧、Canvas ゲームの iframe プレビュー。
- 技術スタック確定の反映: Next.js + TypeScript + Anthropic SDK + SQLite（Stage 3 で PostgreSQL 移行を再判断）。

## Non-Goals（この change ではやらない）

- 権限強制の実行層検証・Sandbox・Budget（Stage 2 / `add-stage2-harness-sandbox-budget`）。
- 非同期 Job・Queue・resume（Stage 3）。ただし State の step 単位永続化は Stage 1 で行う。
- Git 差分承認 UI（Stage 4）。Stage 1 では生成物はプレビューのみで確定しない。
- Multi-Agent、企画上流 pipeline、自動修復ループ。

## Impact

- Affected specs: `agent-run`（新規）、`tool-registry`（新規）、`approval-flow`（新規）
- Affected code: 新規 scaffold（`openspec validate` 通過と本 change の承認まで実行コードを書かない）
- 予算: 開発時の動作確認も $1/run・30 step・$30/月 のデフォルトに従う（強制停止の実装は Stage 2。Stage 1 では loop 内の step 上限として実装）
