# Design: Stage 1 単一 Agent＋複数 Tool

## Context

`docs/STAGE_DESIGN.md` §0–1 を前提とする。ここでは Stage 1 の実装判断のみ記す。

## 固定 Workflow と Agent の境界

```
DRAFT → PLANNING → WAITING_PLAN_APPROVAL → GENERATING → PREVIEW_READY
                     │ 却下                    │ 停止条件
                     └→ REJECTED               └→ HALTED_*（原因付き）
```

- 状態遷移はサーバ側コードのみが行う。LLM の出力が状態を直接変えない。
- PLANNING は単発 LLM 呼び出し（Agent loop を使わない）。出力は構造化された計画（生成予定ファイル一覧、ゲーム仕様の要約、想定 step 数）。
- GENERATING が唯一の Agent loop。while loop で messages API を呼び、tool_use → Tool Registry 実行 → tool_result を繰り返す。

## Agent loop の実装方針

- Anthropic SDK（TypeScript）を直接使う。Claude Agent SDK は使わない（学習目的のため loop・停止条件・State を自分の設計で持つ）。
- 1 step = 1 回の assistant 応答。step ごとに Run の messages スナップショットと step 数を SQLite に保存。
- system prompt はゴール・制約・Tool 説明のみの安定した内容とし、ルールの累積追加をしない（旧repoの 60KB 肥大の教訓。サイズより「累積で操縦しない」ことが本質）。安定 prefix を保ち prompt caching を効かせる（可変値を先頭側に入れない、Tool 順序固定）。
- 現行 API の前提: モデル既定は `claude-sonnet-4-6`。thinking は adaptive（`budget_tokens`・`temperature` は送らない）、深さは `output_config.effort`。PLANNING の構造化計画は structured outputs（`output_config.format`）で schema 保証。長い生成は streaming で受け、`stop_reason`（`end_turn` / `tool_use` / `max_tokens` / `refusal`）を分岐処理する。usage（cache_read / cache_creation 含む）を CostLedger に記録。

## 停止条件（Stage 1 実装分）

| 条件 | 挙動 |
|---|---|
| 30 step 到達 | `HALTED_MAX_STEPS` で停止・保存 |
| 同一 Tool のエラー連続 3 回 | `HALTED_TOOL_ERRORS` で停止・保存 |
| ユーザーキャンセル | `CANCELLED` で停止・保存 |
| LLM が終了を宣言（end_turn） | `PREVIEW_READY` へ遷移 |

## Tool 権限（Stage 1 の範囲）

- Tool 定義に `permissions` を宣言（read/write の path prefix、許可コマンド）。Stage 1 では宣言と Registry 経由実行までを必須とし、path 正規化などの強制検証の完全化は Stage 2 の受け入れ条件とする。ただし `write_file` の `workspace/runs/<runId>/` 外への書き込み拒否だけは Stage 1 でも実装する（最低限の安全）。
- shell は `check_syntax` の固定コマンドのみ。任意コマンド Tool は存在させない。

## データ

- SQLite（ファイル DB）。`Run` / `RunEvent`（append-only）/ `CostLedger`（usage 記録のみ。強制停止は Stage 2）。
- run workspace は `workspace/runs/<runId>/` に隔離。`.gitignore` 対象（成果物の git 管理・差分承認は Stage 4 で導入）。

## Alternatives considered

- Claude Agent SDK 採用 → 実装は速いが学習核が隠れるため不採用（参照資料には使う）。
- FastAPI backend 分離 → 2 言語構成は 5 週間に過大。不採用。
- 最初から Queue/Worker → Stage 3 の主題。Stage 1 は同期実行＋step 永続化で足りる。
