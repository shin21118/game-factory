# tool-registry（delta）

## ADDED Requirements

### Requirement: Tool は Registry に宣言的に登録され Registry 経由でのみ実行される

システムは Tool を name、説明、引数 schema、permissions（read/write の path prefix、許可コマンド）付きで Tool Registry に登録し、Agent からの Tool 実行を Registry 経由に限定しなければならない（SHALL）。Registry に登録されていない Tool の実行要求は拒否し、拒否イベントを記録する。

#### Scenario: 未登録 Tool の拒否

- **WHEN** Agent が Registry に存在しない Tool の実行を要求する
- **THEN** 実行は拒否され、`tool_denied` イベントが RunEvent に記録され、Agent には拒否理由が tool_result として返る

#### Scenario: 引数 schema 違反の拒否

- **WHEN** Agent が schema に合わない引数で Tool を呼び出す
- **THEN** Tool 本体は実行されず、検証エラーが tool_result として返る

### Requirement: Stage 1 の Tool セットは 4 つに限定される

システムは Stage 1 において `list_files`、`read_file`、`write_file`、`check_syntax` の 4 Tool のみを Agent に公開しなければならない（SHALL）。任意 shell 実行、外部ネットワークアクセス、run workspace 外を対象とする Tool を提供してはならない。

#### Scenario: 公開 Tool の限定

- **WHEN** Agent loop が LLM へ Tool 定義を渡す
- **THEN** 渡される Tool は上記 4 つのみである

### Requirement: write_file は run workspace 外へ書き込めない

システムは `write_file` の書き込み先を `workspace/runs/<runId>/` 配下に限定し、path 正規化のうえで範囲外（`..` や絶対パスを含む）への書き込みを拒否しなければならない（SHALL）。

#### Scenario: パストラバーサルの拒否

- **WHEN** Agent が `../../etc/config` のような workspace 外パスへの書き込みを要求する
- **THEN** 書き込みは行われず、拒否理由が tool_result と RunEvent に記録される
