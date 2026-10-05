# TypeScript Review Guide

## 設計原則

型を説明のためだけでなく、Changeability（変更・拡張容易性）とMechanism over Discipline（規律より仕組み）のために使う。

型が複雑になるほどよいとは判断しない。実装を理解するために型パズルの解読が必要なら、Cognitive Load（認知負荷）を問題視する。

## 型で変更漏れを検出する

状態・区分の追加時に処理漏れが問題になる場合、literal union / discriminated unionとexhaustive checkingを使い、未対応箇所をcompilerへ表面化させる。

この観点は、全文検索や実装者の記憶だけで変更箇所を探す構造より優先する。

## 型安全性を弱める構文

### `any`

通常はDiscouraged（設計上の非推奨）。型システムを迂回し、変更漏れや誤用をcompilerが検出できなくなる。

外部境界の不明データは`unknown`を使い、validationやtype guardで絞り込めないか確認する。

### `as`

Caution（注意）。型assertionはruntime保証を増やさない。

次を確認する。

- library typing不足など、局所的な理由があるか
- validation / narrowingで安全に表現できないか
- assertionが上流の設計問題を隠していないか

`as const`やimport/export alias等、意味の異なる用途と区別する。

### non-null assertion `!`

nullabilityの成立条件が構造や制御フローから分からないまま`!`で抑制している場合は問題視する。

## 状態表現

- 同一概念を複数booleanやoptional propertyの組み合わせで表し、不可能状態を作っていないか確認する
- behaviorを変えるidentifierは、生のstring / numberより意味のある型で表せないか検討する
- enum / union等の選択は、既存規約、interop、runtime表現、可読性を踏まえて決める。機械的に一方を禁止しない

## 型の再利用とAuthoritative Owner / Source of Truth

同じDomain Concept（ドメイン概念）やContract（契約）を表す型を複数箇所で独立管理しない。utility type、schema、code generation等を使い、どこがAuthoritative Owner（正本となる所有者）か明確にする。

構造が同じという理由だけで型を共通化しない。異なる責務・異なる変更理由を持つDTOやmodelは、現在のproperty構成が同じでも別の型として維持できる。

多段conditional typeや過剰なgenericで理解が難しくなる場合は、明示的な型の方を優先する。

## 関数・変数

- `const`を基本とし、再代入が必要な場合だけ`let`を使う
- 引数の参照型を意図せずmutateしない。変更結果を戻り値で明示できないか確認する
- 引数数が多い場合、単にobjectへ包む前に責務過多や情報のまとまりを確認する
- function名は動作を示し、`check`のような曖昧語より`is` / `exists` / `can` / `needs` / `validate`等で意味を区別する

## 日付

projectで日付ライブラリを採用している場合、timezoneを含む日付計算の抽象化を統一する。境界で別形式へ変換する必要がある場合も、変換責務を局所化する。

## Compiler設定

`tsconfig`の実設定を確認する。型安全性を高める設定を提案するときは、設定を強くすること自体を目的にせず、実際に防げる不具合とmigration costを示す。

設定項目の利用可否や意味が対象versionに依存する場合は、[Version Awareness](../core/version-awareness.md) に従う。
