# game-factory 開発brief（確定版）

Status: 確定（2026-07-19 レビュー完了）
Last updated: 2026-07-19

> この文書は Project 4（v4、11/23–12/27）の開発目的の正本です。振る舞いの詳細仕様は `openspec/` を正本とし、この文書はスコープと判断の背景を保持します。段階設計は [STAGE_DESIGN.md](STAGE_DESIGN.md)、Codex への依頼は [CODEX_REQUEST.md](CODEX_REQUEST.md) を参照。

## 確定済みの前提（2026-07-19 判断）

- **正本**: このrepo `game-factory` が P4 の正本。既存 `agentic-game-factory` は読み取り専用の参照アーカイブ（`goal_roadmap/logs/decisions.md` 2026-07-19 記録）。
- **旧repoからの持ち込み**: wisdom と設計参照のみ。一次参照は旧repo `docs/REBUILD_KNOWLEDGE_TRANSFER.md`。コードの直接移植はしない。
- **対象ゲーム**: HTML5 Canvas ミニゲーム（単一ジャンル・少数ファイル・依存最小・ブラウザで即プレイ）。
- **技術スタック**: TypeScript 一体型。Next.js（UI）＋ Node worker（非同期Job）＋ SQLite または PostgreSQL。Agent loop は Anthropic SDK による自作（Claude Agent SDK は参照資料に留める）。
- **予算デフォルト**: $1/run・30 step/run・$30/月。超過時は必ず停止（変更時は人間承認）。
- **モデル既定値（2026-07-20 仮決め。着手時に単価・世代を再確認）**: 計画・生成とも `claude-sonnet-4-6`。vision 判定（スクリーンショット確認）は `claude-haiku-4-5`。品質不足を実測で確認したときのみ `claude-opus-4-8` へ昇格。根拠: Sonnet 4.6 なら 1 run 概算 $0.7–0.9 で $1/run 上限と整合（prompt caching 有効時）。

---

# 第1層: P4 スコープ（12/27 までの正本）

## 目的

UI から Agent を操作し、ゲームの企画・コード・アセット・テストを段階的に生成するツールを、5つの Stage（PROGRAM.md Block 20–24 と 1:1 対応）で完成させる。

学習の核は、Agent loop、Tool と権限、Harness、Sandbox、Budget と停止条件、非同期 Job と Checkpoint、Agent UX（進捗・差分承認・監査ログ）、Multi-Agent 化の判断を、実装から理解し Codex に指示・レビューできるようになること。

## 対象ユーザー

開発者本人。生成されるミニゲームのプレイヤーは「ブラウザで開いて数秒で遊び方が分かる人」を想定するが、P4 期間中の公開・集客は対象外。

## 作るもの（5 Stage）

1. **Stage 1（Block 20）単一 Agent＋複数 Tool**: 企画入力 → Agent が計画 → 人間承認 → 許可された Tool でファイル生成 → 結果プレビュー。
2. **Stage 2（Block 21）Harness・Sandbox・Budget**: Tool 権限（ファイル・Shell・ネットワーク）の宣言と強制、生成物の Sandbox 実行、Token・Cost・Step 上限と強制停止、監査ログ。
3. **Stage 3（Block 22）非同期 Job・Checkpoint**: 長時間処理の Job 化、進捗、キャンセル、再開、Retry、二重実行防止。
4. **Stage 4（Block 23）Agent 操作 UI・差分承認**: 進捗タイムライン、生成物プレビュー、Git 差分、承認・却下・再実行。
5. **Stage 5（Block 24）分業判断・総合リリース**: 単一 Agent を評価し、Multi-Agent 分離の要否を根拠付きで判断。

## 設計原則（旧repoの教訓）

- 最初から Multi-Agent にしない。分離は Stage 5 で費用対効果を評価してから。
- Agent より先に固定 Workflow を検討する。「企画 → 計画 → 承認 → 生成 → 検証 → 承認」の骨格は決定論的な状態機械で持ち、LLM の自由裁量は各ステップの内側に限定する。
- phase の過分割をしない（旧repoは 16 phase で design coherence を失った）。
- prompt にルールを累積追加しない（旧repoは halt のたびに ABSOLUTE RULE を継ぎ足して 60KB まで肥大し、矛盾した指示の束になった）。system prompt はゴール・制約・Tool 説明のみの安定した内容に保つ。サイズ自体より「累積で操縦しないこと」と「安定 prefix を保って prompt caching を効かせること」が本質（現行モデルは過度に prescriptive な指示でむしろ品質が下がる）。
- semantic 品質を regex で検出しようとしない。検証は「実行できるか」（runtime evidence）＋ vision 判定（スクリーンショットの LLM 確認）＋人間承認の 3 層にする。旧repoの 28 軸 gate は再現しない。
- 形式的な gate の PASS を成功扱いしない。stub を実装済みとみなさない。

## 非目標（P4 期間中）

- 流行調査・ゲーム分析・pattern synthesis などの企画上流 pipeline。
- 複数ジャンル・複数 engine・実ストア公開・収益化。
- 自動 QA・自動修復ループの高度化（人間承認で代替する）。
- 見た目だけのデモ、遊べない偽の export、gate の実効性のない多軸評価 framework。

## 成功条件（判定可能な形）

- 各 Stage が PROGRAM.md Block 20–24 の完了条件を満たす:
  - 最初から Multi-Agent にしていない。Tool 権限と停止条件を明文化した。
  - Agent が許可範囲外を操作できない。上限超過時に必ず停止する。
  - ページを閉じても処理状態が失われない。同じ Job を重複実行しない。
  - Agent が何をしたか追跡できる。ユーザー承認前に成果物を確定しない。
  - Agent 追加（Multi-Agent 化）の理由を説明できる。
- 企画入力から遊べる HTML5 Canvas ミニゲーム 1 本の生成・プレビュー・承認までを、1 つの run として追跡・再開できる。
- 各 run の入力、出力、モデル、費用、step 数、承認履歴を UI で確認できる。

---

# 第2層: 将来構想（North Star、P4 スコープ外）

P4 完了後に再評価する構想。P4 の設計判断を縛らないが、方向性の文脈として残す。

- 公開可能なゲームを継続的に作り、売上・広告・利用反応と、制作システム自体の価値化を狙う。
- 企画上流（調査・分析・synthesis）や自動修復ループの拡張。ただし旧repoの教訓により、phase 追加は「固定 Workflow で足りないことが実証されてから」。
- 複数 run の結果から当たりやすい企画・表現を学習し、品質・速度・コストを改善する。
- 小規模ゲーム開発者・企画者への制作工程の提供。

---

# 残論点の仕分け

## 今決める → 決定済み（冒頭「確定済みの前提」参照）

正本関係、対象ゲーム種別、技術スタック、Agent loop 実装方針、予算デフォルト。

## 調査後に決める（Block 20 着手前の調査タスク）

- Sandbox の実現方式: iframe sandbox を最小とし、プロセス分離・コンテナの要否（OWASP LLM／コンテナ資料の調査後）。
- Model 選定の最終確認（既定値は Sonnet 4.6 で仮決め済み。着手時 11/23 の単価・モデル世代で再確認するだけ）。
- Queue/Worker の具体実装（DB ベース自作か軽量ライブラリか。DDIA 該当章の学習後）。
- 生成ゲームの置き場所と共有方法（itch.io 等は P4 後の判断でよい）。

## 実装時に決める（Codex と対話しながら承認）

- Tool の細目仕様（引数 schema、エラー分類）。
- 検証チェックの細目（構文チェック、起動確認、操作可能性）。
- 人間 playtest の挿入位置の細部。
- 旧repoパターン（run-store 状態機械、cost-ledger schema、Playwright runtime 検証）の個別採用。
- Multi-Agent 分離の要否（Stage 5 のテーマそのもの）。
