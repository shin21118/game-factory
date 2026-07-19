# ローカルエージェント契約

このリポジトリは中央のAgent駆動開発ルールに従います。

- `C:\Users\shin0\aiagent\01_application\AGENTS.md`
- `C:\Users\shin0\aiagent\02_management\goal_roadmap\PROGRAM.md`

## ローカル目的

ゲーム生成を一度きりのデモではなく、企画、生成、評価、修復、公開まで観測できるAgent開発パイプラインとして成立させる。

## 現在のマイルストーン

正本は確定済み（2026-07-19: この repo が正本）。Stage 1 の OpenSpec change `add-stage1-agent-core` の承認を得て、Stage 1（単一 Agent＋複数 Tool）を実装する。

## ローカルコマンド

- OpenSpec検証: `npx --yes @fission-ai/openspec@latest validate --all --strict --no-interactive`
- install / dev / test / build: Stage 1 scaffold で確定（Next.js + TypeScript + SQLite）。

## 編集可能範囲

- `README.md`
- `AGENTS.md`
- `docs/`
- `goals/`
- `operations/`
- `openspec/`
- スタック承認後の実行ファイル

## 承認なしで変更しないもの

- 既存 `agentic-game-factory` のコード、履歴、正本。
- このrepoを後継・再設計・公開用のどれにするかという判断。
- Agent構成、評価ゲート、外部モデル、予算上限、公開範囲。

## 完了条件

- Superpowersで検討した構成判断がOpenSpecのproposal/design/specへ追跡できる。
- 生成結果だけでなく、入力、評価、失敗分類、修復、再検証が観測できる。
- 実装作業では、対象changeの検証と選定スタックのテストを通す。
- 変更、検証、残課題、次の一手をセッション引き継ぎへ残す。
