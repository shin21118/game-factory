# エージェント運用ルール

## 最初に読むもの

1. このファイルを読む。
2. `../AGENTS.md`、`../STRUCTURE.md`、`../APPLICATION_INDEX.md` を読む。
3. `README.md` を読む。
4. `goals/LOCAL_AGENT_CONTRACT.md` と `operations/SESSION_HANDOFF.md` を読む。
5. 中央ロードマップを確認する。
   - `C:\Users\shin0\aiagent\02_management\goal_roadmap\README.md`
   - `C:\Users\shin0\aiagent\02_management\goal_roadmap\PROGRAM.md`
   - `C:\Users\shin0\aiagent\02_management\goal_roadmap\logs\decisions.md`

## セッション運用

- 開始時は `$repo-session-start` を使う。
- 終了時は `$repo-session-close` を使う。
- プロジェクトの現在地や重要判断が変わったら `$goal-roadmap-sync` を使う。

## Superpowers と OpenSpec

- 新しい生成フローや役割分担は、Superpowersの `brainstorming` で選択肢と失敗条件を整理する。
- 合意した振る舞い変更は `openspec-propose` または `/opsx:propose` でOpenSpec changeへ固定する。
- OpenSpecの `proposal.md`、差分仕様、`design.md`、`tasks.md` が実装可能になるまで実行コードを書かない。
- Superpowersの計画は実行粒度の補助資料として扱い、振る舞い仕様の正本を重複させない。
- 実装では `test-driven-development`、デバッグでは `systematic-debugging`、完了前は `verification-before-completion` を使う。
- 完了前に `openspec validate <change-name>` を実行し、完了後はOpenSpec changeをarchiveする。

## ローカルルール

- 既存 `agentic-game-factory` のコードや履歴を、人間判断なしで移動・削除・置換しない。
- 新旧repoの正本が決まるまで二重実装を始めない。
- 生成速度だけでなく、遊べる品質、評価可能性、失敗からの復旧を重視する。
- 技術スタックと最初のend-to-endフローが合意されるまで実行用スキャフォールドを追加しない。
- 資料、提案、計画、レビュー、引き継ぎは日本語を正本にする。
- 未関係なユーザー変更を上書きしない。
