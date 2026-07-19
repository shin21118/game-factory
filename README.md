# game-factory

## 目的

UI から AI Agent を操作し、ゲームの企画・コード・アセット・テストを段階的に生成するツール（v4 Project 4 の正本 repo）。目的の正本は [docs/DEVELOPMENT_BRIEF.md](docs/DEVELOPMENT_BRIEF.md)。

## 現在の状態

- **正本確定（2026-07-19）**: この repo が P4 の正本。既存 `agentic-game-factory`（GitHub `shin21118/agentic-game-factory`）は読み取り専用の参照アーカイブ。コード移植はしない。
- 開発brief 確定、段階設計（[docs/STAGE_DESIGN.md](docs/STAGE_DESIGN.md)）、Codex 依頼書（[docs/CODEX_REQUEST.md](docs/CODEX_REQUEST.md)）作成済み。
- Stage 1 の OpenSpec change `add-stage1-agent-core` を提案済み（人間承認待ち）。
- 実行用コードは未作成。実装開始は change 承認後（着手予定 11/23）。

## 確定済みスタック

TypeScript 一体型: Next.js（UI）＋ Node worker（非同期Job、Stage 3〜）＋ SQLite（Stage 3 で PostgreSQL 再判断）。Agent loop は Anthropic SDK による自作。

## 検証方法

```powershell
npx --yes @fission-ai/openspec@latest validate --all --strict --no-interactive
```

実装開始後は、生成物の再現性、評価ゲート、失敗分類を自動検証へ追加します。

## 正本

- 振る舞い仕様: `openspec/specs/`
- 進行中の変更: `openspec/changes/`
- ローカル制約: `goals/LOCAL_AGENT_CONTRACT.md`
- セッション状態: `operations/SESSION_HANDOFF.md`

## 中央ロードマップ

- `C:\Users\shin0\aiagent\02_management\goal_roadmap\PROGRAM.md`
- `C:\Users\shin0\aiagent\02_management\goal_roadmap\logs\decisions.md`
