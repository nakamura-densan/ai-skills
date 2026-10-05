---
name: code-design-guidelines
description: コードの保守性を高く保つための設計基準を提供する。新規実装、機能追加、修正、リファクタリングの際に、生成AIが人間とAIの双方にとって理解しやすく、変更・拡張しやすく、検証しやすく、必要以上に複雑でない構造を判断するために使用する。API、画面、バッチ、モジュール、ファイル、Git差分の設計・実装レビューにも同じ基準を適用する。責務、依存、状態、不変条件、副作用、失敗時安全性、診断容易性、契約、技術固有の設計リスクを評価し、品質を実装者の注意や記憶ではなく型・API設計・アーキテクチャ・テスト・静的解析・CIなどの仕組みで維持できる構造を優先する。
---

# Code Design Guidelines

## 目的

コードの保守性を高く保つために、実装・設計・修正・レビューで共通して使用する設計基準を提供する。人間と生成AIの双方が理解しやすく、将来安全に変更・拡張・検証できるコードを、必要以上に複雑にせず維持する。

このSkillでは保守性を、主にUnderstandability（理解容易性）、Changeability（変更・拡張容易性）、Verifiability（検証容易性）、Simplicity（単純性）の4軸へ分解して判断する。動作の正しさは前提条件であり、完成条件ではない。設計パターンや高度なアーキテクチャの採用自体ではなく、将来の理解・変更・検証・運用に必要な負担を減らせているかで判断する。

## Core

保守性を次の4軸で判断する。

1. **Understandability（理解容易性）**: 少ない探索と認知負荷で、意図・状態・処理の流れを理解できるか。
2. **Changeability（変更・拡張容易性）**: 変更箇所を特定でき、局所的かつ漏れなく安全に変更・拡張できるか。
3. **Verifiability（検証容易性）**: 変更対象を小さな範囲でテスト・診断し、正しさを確認できるか。
4. **Simplicity（単純性）**: 上記を満たすために必要な複雑性だけを持ち、不要な状態・依存・抽象化・レイヤーを増やしていないか。

詳細は [Core Principles](references/core/principles.md) を読む。広く知られた設計用語も名称だけから判断せず、同ReferenceのOperational DefinitionをこのSkillでの意味として使う。プロジェクトが用語を明示定義している場合は、プロジェクト固有ルールの優先順位に従う。

## 横断原則

- **Mechanism over Discipline（規律より仕組み）**: 品質を実装者の知識・記憶・注意力だけに依存させない。型、API、責務境界、DB制約、テスト、静的解析、CIなどで正しい実装へ導く。
- **Information Placement（情報配置）**: CodeはHow（どう実現するか）、TestはWhat（何を保証するか）、CommitはWhy（なぜ変更したか）、CommentはWhy / Why not（なぜその方法か／なぜ別案ではないか）を主に担う。
- **必要十分な説明を残す**: 短さを目的にしない。意図を説明する変数・型・処理段階・明示的な分岐は残し、意味のないラッパー、重複、不要な状態・抽象化を削る。

## 利用フロー

[Execution Workflow](references/process/workflow.md) に従う。

1. 新規実装・機能追加・修正・リファクタリングでは設計基準を実装方針へ適用する。設計相談やレビュー依頼では、その用途に応じたモードへ切り替える。
2. `.code-design/context.md`、明示的なDocs・ADR・規約、対象周辺の実装を必要な範囲だけ確認する。
3. 対象とDirect Scopeを定め、必要な場合だけBehavioral Scope、Impact Scopeへ広げる。
4. 判断に関係する使用技術・実バージョン・設定・既存Toolingを確認し、該当するReferenceだけ読む。
5. Coreとプロジェクト固有ルールに沿って設計・実装方針を判断する。
6. レビューではEvidence（根拠）を確認してFindingを作る。実装・設計判断では、Finding形式を強制せず、判断結果を実装方針へ反映する。
7. コードを変更した場合は、対象に応じたtest、type check、lint等で変更を検証する。
8. 長期利用できるプロジェクト事実だけをcontextへ更新する。reportはレビュー時だけ作成する。

事前確認は毎回すべて行わない。判断結果を変え得る情報だけを確認し、既知のcontextで十分なら再探索しない。

## プロジェクト固有ルール

次の優先順位で判断する。

1. 明示された要件・仕様
2. `AGENTS.md`、ADR、設計書、README、コーディング規約などの明示的なプロジェクト判断
3. 言語・フレームワーク・ORM / DBの制約
4. このSkillの一般原則
5. 一般的な好み・スタイル

既存実装パターンは参考情報であり、正式なルールとは限らない。明示ルール自体に将来リスクがある場合は、ルールに従っている事実とルール自体のリスクを分けて扱う。ADR等で既知リスクとして受容済みなら繰り返し指摘しない。

詳細は [Project Context](references/core/project-context.md) を読む。

## Finding

レビュー時は [Finding and Report Format](references/process/findings.md) に従う。

- **MUST FIX**: 正しさ、整合性、安全な変更・再実行、重大な将来障害に直結する。
- **SHOULD FIX**: 現状動作しても、理解・変更・検証・運用のコストを継続的に増やす。
- **CONSIDER**: 文脈依存だが、改善余地として価値がある。

Confidence（確信度）は **High / Medium** を基本とし、Lowの推測的指摘は原則出さない。

「注意する」「コメントを書く」「テストを追加する」だけで終わらず、可能なら型・API・責務境界・制約・静的解析など、誤用を防ぐ仕組みへ落とす。

## コード変更

- 「レビューして」: 調査、指摘、具体的な修正案まで。コードは変更しない。
- 「レビューして修正して」「修正してください」: 必要な事前確認と設計判断を行ったうえで修正する。
- 新規実装・機能追加・リファクタリング: 実装前に必要な文脈と制約を確認し、このSkillの設計基準を実装方針へ反映する。
- 設計相談: 必要な文脈を確認し、複数案がある場合はCoreとプロジェクト制約から判断条件を示す。Findingやreportは作らない。

## Reference Navigation

必要なものだけ読む。SKILL.mdから各Referenceへ直接辿り、Reference同士の多段参照を前提にしない。

### Core

- [Core Principles](references/core/principles.md)
- [Project Context](references/core/project-context.md)
- [Tooling Audit](references/core/tooling.md)
- [Version Awareness](references/core/version-awareness.md)

### Process

- [Execution Workflow](references/process/workflow.md)
- [Finding and Report Format](references/process/findings.md)

### Targets

- [Target Scopes](references/targets/scopes.md)

### Technologies

- [TypeScript](references/technologies/typescript.md)
- [React](references/technologies/react.md)
- [Next.js](references/technologies/nextjs.md)
- [Prisma](references/technologies/prisma.md)
- [Styling](references/technologies/styling.md)
