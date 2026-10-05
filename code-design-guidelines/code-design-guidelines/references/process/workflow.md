# Execution Workflow

## 目次

- [1. 利用モードを決める](#1-利用モードを決める)
- [2. 共通の事前確認](#2-共通の事前確認)
- [3. Scopeを決める](#3-scopeを決める)
- [4. 技術・バージョン・Toolingを確認する](#4-技術バージョンtoolingを確認する)
- [5. Coreを適用する](#5-coreを適用する)
- [6. 利用モードごとに処理する](#6-利用モードごとに処理する)
- [7. 変更を検証する](#7-変更を検証する)
- [8. contextとreportを更新する](#8-contextとreportを更新する)

## 1. 利用モードを決める

このSkillは、コードを書く・変更する場面で設計基準を適用することを既定用途とする。ユーザーの依頼に応じて次のモードを使い分ける。

- **Implementation（実装）**: 新規実装、機能追加、修正、リファクタリングで設計基準を実装へ反映する。
- **Design Guidance（設計判断）**: 実装前または実装中に、構造、責務、状態、契約、依存関係等を判断する。
- **Review（レビュー）**: 既存コードや変更内容を同じ設計基準で評価し、Findingを報告する。
- **Review & Fix（レビュー後の修正）**: Findingを確認したうえで必要な修正まで行う。

「変更した内容をレビュー」は原則Git差分を起点にする。「注文APIを実装」「この画面をリファクタリング」のような依頼では、対象機能そのものを起点にする。

## 2. 共通の事前確認

設計判断・実装・レビューのいずれでも、判断結果を変え得るプロジェクト文脈を先に確認する。

`.code-design/context.md` があれば最初に読む。その後、必要な範囲で次を確認する。

- `AGENTS.md`
- README
- ADR
- architecture / design docs
- coding conventions
- package manifest / lockfile
- compiler / linter / formatter config
- representative implementation
- OpenAPI / schema / migration

毎回すべて読む必要はない。対象の責務、制約、既存の設計判断を理解するために必要な情報だけ確認する。contextが新しく、今回の判断に必要な事実を十分含む場合は同じ調査を繰り返さない。

既存コードは観測材料であり、正式なルールとは限らない。

## 3. Scopeを決める

### Direct Scope（直接範囲）

対象そのものと、処理成立または設計判断に直接必要な依存を確認する。

### Behavioral Scope（振る舞い範囲）

意味、副作用、transaction、state transition、不変条件を理解するために必要な範囲。必要な場合だけ広げる。

### Impact Scope（影響範囲）

契約、共有schema、公開API、共通component等の変更で利用側への影響が疑われる場合に確認する。

「念のため」でrepository全体へ広げず、判断に必要な仮説を持って探索する。対象別のScopeは [Target Scopes](../targets/scopes.md) を参照する。

## 4. 技術・バージョン・Toolingを確認する

判断に影響する場合だけ、`package.json`、lockfile、framework config、runtime config、Prisma schema等から実際のバージョンと有効な設定を把握し、該当する技術Referenceを読む。

版依存の事実が判断に影響する場合は、[Version Awareness](../core/version-awareness.md) に従い、その時点の公式情報で確認する。時点依存の知識をSkill内の記述だけから断定しない。

続けて、必要な範囲でTooling Auditを行う。型、linter、test、Prisma schema / DB制約、CI等ですでに機械的に担保されている事項を、人間や生成AIの注意力へ戻さない。

画面・Componentのstylingが判断対象に含まれる場合は、CSSを前提にせず、依存関係・import・設定・代表実装から実際のstyling library / styling APIを特定し、[Styling](../technologies/styling.md) を適用する。packageに存在するだけの内部依存を、アプリケーションの採用方式と誤認しない。

## 5. Coreを適用する

設計用語は [Core Principles](../core/principles.md) のOperational Definitionに従う。`DRY`、`責務分離`、`Boundary`、`Simplicity`等の名称だけを根拠に、共通化・分割・抽象化を決めない。

チェックリストを機械的に消化せず、対象と依頼内容からリスクが高い観点を重点的に確認する。

最低限、次の4問を判断基準にする。

1. 理解するための探索・認知負荷は必要以上に大きくないか。
2. 変更・追加時の影響範囲と変更箇所を追跡できるか。
3. 対象ロジックを小さな範囲で検証・診断できるか。
4. 責務分離、抽象化、state、dependencyが必要以上に複雑ではないか。

典型的な兆候と確認観点:

- 多数のboolean / state同期 → State & Invariant Integrity
- 呼び出し順序への暗黙依存 → Side Effect & Temporal Coupling
- DB + external I/O → Failure Safety
- API / schema共有 → Information Ownership / Contract & Compatibility
- 深い呼び出し階層 → Local Reasoning
- wrapper / service / repositoryの多段委譲 → Minimal Sufficient Design
- エラー分類・ログ → Diagnosability
- 追加区分・状態 → Evolution Safety

## 6. 利用モードごとに処理する

### Design Guidance（設計判断）

Core、プロジェクト固有ルール、既存の責務境界から判断する。選択肢が複数ある場合は、採用条件とtrade-offを示す。

Finding形式やSeverityは使わない。設計を高度にすることを目的にせず、必要十分な構造を選ぶ。

### Implementation（実装）

実装前に、今回守るべき責務、契約、不変条件、失敗時の扱い、検証方法を把握する。そのうえでCoreに沿って実装する。

既存実装へ合わせる場合も、既存パターンが明示ルールなのか、単なる慣習なのかを区別する。問題のある既存構造を無条件に複製しない。

### Review（レビュー）

Findingは具体的なコード、設定、契約、実行経路へ結び付ける。「一般的に良くない」だけではFindingにしない。

少なくとも、次のいずれかをEvidence（根拠）として示す。

- 現在の誤動作・不整合
- 将来の具体的な変更シナリオ
- 検証・診断の困難さ
- 暗黙の前提
- 実装者の記憶へ依存する手順
- 対象バージョンへ適用できる公式制約

修正案は、問題を解消し、最小十分で、認知負荷や将来の変更箇所を増やさないものにする。詳細は [Finding and Report Format](findings.md) に従う。

### Review & Fix（レビュー後の修正）

Reviewで確認した問題と修正方針を基に変更する。修正案自体が新しい抽象化・依存・状態を不必要に増やしていないか再確認する。

## 7. 変更を検証する

コードを変更した場合は、対象に応じて既存の検証手段を使う。

- type check
- linter / formatter
- unit / integration / e2e test
- migration / schema validation
- build

すべてを機械的に実行するのではなく、変更内容を検証できる最小十分な組み合わせを選ぶ。既存のCIやproject commandを優先し、Skill専用の独自scriptを安易に増やさない。

## 8. contextとreportを更新する

### context

`.code-design/context.md` には、今後も使える事実だけを保存する。

- 重要Docs / ADRのpath
- architecture boundary
- stack / version
- tooling config
- authoritative owner / source of truth
- representative implementation path
- observed architecture fact + Evidence（根拠） + Confidence（確信度）
- 最終確認commit

Finding、Severity、修正案、一時的な仮説、その実行だけの探索メモは保存しない。

### report

ReviewまたはReview & Fixでは、`.code-design/reports/YYYYMMDD-HHMMSS-<target>/report.md` を作る。

Design GuidanceやImplementationではreportを作らない。ユーザーが明示的に求めた場合だけ作成する。

チャットには依頼に必要な結果だけを返し、探索ログを大量に出さない。
