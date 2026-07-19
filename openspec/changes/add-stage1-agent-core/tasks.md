# Tasks: Stage 1 単一 Agent＋複数 Tool

## 1. Scaffold（承認後に着手）

- [ ] 1.1 Next.js + TypeScript + SQLite の最小 scaffold（lint / test / build が通る空実装）
- [ ] 1.2 `.env.example` に `ANTHROPIC_API_KEY` を追加（実キーはコミットしない）
- [ ] 1.3 `workspace/runs/` を作成し `.gitignore` に追加

## 2. データ層

- [ ] 2.1 `Run` / `RunEvent` / `CostLedger` の schema とマイグレーション
- [ ] 2.2 RunEvent の append-only 書き込みと読み出し
- [ ] 2.3 単体テスト（状態遷移が許可された遷移のみか）

## 3. 固定 Workflow

- [ ] 3.1 状態機械（DRAFT → PLANNING → WAITING_PLAN_APPROVAL → GENERATING → PREVIEW_READY / REJECTED / HALTED_* / CANCELLED）
- [ ] 3.2 PLANNING: 単発 LLM 呼び出しで構造化計画を生成し保存
- [ ] 3.3 計画の承認・却下 API（承認前に GENERATING へ進めないことをテスト）

## 4. Tool Registry と Agent loop

- [ ] 4.1 Tool Registry（宣言的登録、引数 schema 検証、permissions 宣言）
- [ ] 4.2 Stage 1 の 4 Tool 実装（list_files / read_file / write_file / check_syntax）
- [ ] 4.3 `write_file` の workspace 外書き込み拒否（テスト必須）
- [ ] 4.4 Agent loop（step 永続化、tool_use 実行、tool_result 返却）
- [ ] 4.5 停止条件（30 step / エラー連続 3 回 / キャンセル）のテスト
- [ ] 4.6 usage の CostLedger 記録

## 5. UI（最小）

- [ ] 5.1 企画入力フォームと run 作成
- [ ] 5.2 計画表示と承認・却下
- [ ] 5.3 RunEvent の一覧表示
- [ ] 5.4 Canvas ゲームの sandbox 属性付き iframe プレビュー

## 6. 検証

- [ ] 6.1 `openspec validate add-stage1-agent-core --strict` 通過
- [ ] 6.2 lint / test / build 通過
- [ ] 6.3 実 run 1 本: 企画入力 → 計画承認 → 生成 → ブラウザで遊べることを人間が確認
- [ ] 6.4 停止条件の実動確認（step 上限で HALTED_MAX_STEPS になる）
