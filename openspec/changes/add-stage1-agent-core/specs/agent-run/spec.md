# agent-run（delta）

## ADDED Requirements

### Requirement: run は固定 Workflow の状態機械に従う

システムは run のライフサイクルを `DRAFT → PLANNING → WAITING_PLAN_APPROVAL → GENERATING → PREVIEW_READY` の固定 Workflow として管理し、状態遷移をサーバ側コードのみが行わなければならない（SHALL）。LLM の出力が状態を直接変更してはならない。

#### Scenario: 承認前に生成が始まらない

- **WHEN** run が `WAITING_PLAN_APPROVAL` 状態で、計画承認がまだ行われていない
- **THEN** 生成（GENERATING）を開始する API はエラーを返し、状態は変化しない

#### Scenario: 却下された計画は生成に進まない

- **WHEN** ユーザーが計画を却下する
- **THEN** run は `REJECTED` 状態になり、Tool 実行は一切行われない

### Requirement: Agent loop は単一 Agent で step 単位に永続化する

システムは生成実行を単一の Agent loop（Anthropic SDK の自作 loop）で行い、1 step（1 回の assistant 応答）ごとに run の State（messages、step 数、生成ファイル一覧）を DB に永続化しなければならない（SHALL）。複数 Agent の並行実行や Agent 間委譲を行ってはならない。

#### Scenario: step ごとの State 保存

- **WHEN** Agent loop が 1 step を完了する
- **THEN** その時点の messages スナップショットと step 数が DB に保存され、プロセスが落ちても直近 step までの State が残る

### Requirement: 停止条件で必ず停止し原因を記録する

システムは次の停止条件のいずれかを満たしたとき Agent loop を停止し、原因を示す状態（`HALTED_MAX_STEPS` / `HALTED_TOOL_ERRORS` / `CANCELLED`）と停止イベントを記録しなければならない（SHALL）: 30 step 到達、同一 Tool のエラー連続 3 回、ユーザーキャンセル。

#### Scenario: step 上限で停止

- **WHEN** Agent loop が 30 step に到達する
- **THEN** loop は停止し、run は `HALTED_MAX_STEPS` になり、停止イベントが RunEvent に記録される

#### Scenario: Tool エラー連続で停止

- **WHEN** 同一 Tool の実行エラーが 3 回連続する
- **THEN** loop は停止し、run は `HALTED_TOOL_ERRORS` になり、直近のエラー内容がイベントに残る

### Requirement: run の行動と費用を追跡できる

システムはすべての計画生成、Tool 呼び出し、承認、停止、エラーを append-only の RunEvent として記録し、LLM API の usage を CostLedger に記録しなければならない（SHALL）。

#### Scenario: イベントの追跡

- **WHEN** run が完了または停止した後にユーザーがイベント一覧を開く
- **THEN** 計画・各 Tool 呼び出し（引数と結果の要約）・承認・停止が時系列で確認できる
