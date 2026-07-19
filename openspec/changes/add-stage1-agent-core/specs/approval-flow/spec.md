# approval-flow（delta）

## ADDED Requirements

### Requirement: 計画は人間承認を経なければ実行されない

システムは Agent の生成実行前に、生成予定ファイル一覧とゲーム仕様要約を含む計画をユーザーに提示し、明示的な承認操作を得なければならない（SHALL）。承認・却下の操作と操作時刻は RunEvent に記録する。

#### Scenario: 計画承認で生成が始まる

- **WHEN** ユーザーが提示された計画を承認する
- **THEN** run は `GENERATING` に遷移し、承認イベントが記録される

#### Scenario: 承認操作なしのタイムアウト

- **WHEN** 計画提示後にユーザーが操作しない
- **THEN** run は `WAITING_PLAN_APPROVAL` に留まり、自動承認は行われない

### Requirement: 生成物は承認前に確定されない

システムは Stage 1 において生成物をプレビュー（sandbox 属性付き iframe）としてのみ提示し、自動で公開・配布・リポジトリ確定を行ってはならない（SHALL）。Git 差分承認による確定は Stage 4 で導入する。

#### Scenario: プレビュー止まり

- **WHEN** 生成が `PREVIEW_READY` に到達する
- **THEN** 生成物は run workspace 内とプレビューでのみ参照可能で、git commit・外部公開は行われない
