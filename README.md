# game-factory

## 目的

AI Agentによるゲームの企画、生成、評価、修復、公開を再現可能な開発パイプラインとして扱うための新規プロジェクト候補です。

既存の `C:\Users\shin0\aiagent\99_others\agentic-game-factory` と責務が重なるため、このリポジトリを後継、再設計、公開用のどれとして使うかを決めるまでは既存コードを移植しません。

## 現在の状態

- GitHub remote 接続済み。
- Agent制御スケルトンとOpenSpecを導入済み。
- 実行用コードは未作成。
- 既存 `agentic-game-factory` との統合方針は要確認。

## 最初のマイルストーン

- 新旧リポジトリの役割と正本を決定する。
- 最小の生成対象、評価ゲート、失敗時の修復フローを定義する。
- 最初のend-to-endフローをOpenSpec changeとして合意する。

## 実行方法

未決定です。既存リポジトリのスタックを自動的にコピーしません。

## 検証方法

```powershell
openspec validate --all --strict --no-interactive
```

実装開始後は、生成物の再現性、評価ゲート、失敗分類を自動検証へ追加します。

## 正本

- 振る舞い仕様: `openspec/specs/`
- 進行中の変更: `openspec/changes/`
- ローカル制約: `goals/LOCAL_AGENT_CONTRACT.md`
- セッション状態: `operations/SESSION_HANDOFF.md`

## 中央ロードマップ

- `C:\Users\shin0\aiagent\02_management\goal_roadmap\operations\projects\project-registry.md`
- `C:\Users\shin0\aiagent\02_management\goal_roadmap\operations\agent\agent-driven-development.md`
