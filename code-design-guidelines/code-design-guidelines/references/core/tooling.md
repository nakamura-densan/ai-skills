# Tooling Audit

## 目的

AIが意味・構造の判断へ集中し、compiler / linter / formatter / test / DB / CIで確実に検出できる問題を人手ルールへ戻さない。

## 確認項目

### TypeScript

- `strict`
- `strictNullChecks`
- `noUncheckedIndexedAccess`
- `exactOptionalPropertyTypes`
- project references / build mode（該当時）

設定を強くすること自体を目的にしない。既存コードへの影響と、実際に防げる不具合を確認する。

### Linter / Formatter

- ESLint / Biome等がCIで実行されるか
- type-aware rulesが必要な箇所で有効か
- ignore / overrideで重要領域が外れていないか
- warningを大量放置してsignalが死んでいないか
- formatter opt-outに明示理由があるか

### Dependency / Dead code

既にprojectで採用されているdependency-cruiser、Knip、SonarQube、Semgrep等があれば活用する。このSkillの利用だけを目的とした新しい独自scriptを安易に追加しない。

### Test / CI

- unit / integration / e2eの責務が分かれているか
- 必須testがCI gateになっているか
- flaky testを恒常的にretryで隠していないか

### DB / Schema

- unique / foreign key / check / not null等で守れる不変条件をapplicationの注意力だけに任せていないか
- migrationでcontract変更を追跡できるか

## 指摘方針

既に機械的に担保されている問題:

- Findingの主題にしない
- 必要なら「Toolingで担保済み」と確認だけ残す

機械化できるが未設定:

- 個別修正の羅列より、再発防止できるtool/configの導入・有効化を優先して提案する
- 新規toolの保守コストが高い場合は導入しない
