# React Review Guide

## State設計

State & Invariant Integrity（状態と不変条件の整合性）を優先して確認する。

- 同一概念を複数stateで重複保持していないか
- propsや既存stateから計算できる値をstateへ保存していないか
- 不可能な状態組み合わせを表現できないか
- selection等で、object全体とIDを二重管理していないか
- 状態遷移が複数component / hookへ散っていないか

関連stateをまとめる場合も、単一概念として一緒に変化するかで判断する。近くにあるという理由だけで一つのobjectへまとめない。

## Effect

Effectは外部システムとのSynchronization（同期）が必要な処理に使う。

次の構造は問題候補になる。

- props / stateからderived stateを作るためのEffect
- event handlerで実行できる処理をEffectへ移している
- state A変更 → Effect → state B変更という同期鎖
- dependency抑制を前提に成立しているEffect
- setupとcleanupが非対称なsubscription / listener

Effectの有無ではなく、外部システムとの同期が本当に必要かで判断する。

## Component / Hook分割

statefulならcustom hook、statelessならutilityという機械的分離は行わない。

分割によって次のどれかが得られる場合に価値がある。

- 独立した責務・変更理由が明確になる
- 複雑なstate transitionを隠せる
- 同じbehaviorを複数箇所で再利用できる
- component本体のmain flowが読みやすくなる

数行の単純処理を追うためだけに別fileを開く必要が生じるなら、Local Reasoning（局所的理解可能性）が悪化していないか確認する。

## Memoization

memoizationは正しさではなくperformanceのための仕組みとして扱う。

- measurementや明確な再計算コストなしに大量導入していないか
- dependency安定化のためのmemoizationが、Effect設計の複雑さを隠していないか
- projectが自動最適化の仕組みを採用している場合、その設定を確認せず手動memoizationを増減していないか

仕組みの提供状況や推奨方法がversionに依存する場合は、[Version Awareness](../core/version-awareness.md) に従う。

## Testability

複雑なUI判定、validation、business calculationがcomponent lifecycleと強結合している場合、入力と出力が明確なlogicとして分離できないか確認する。

一方で、DOM interactionやReact固有behaviorまで無理にpure function化しない。ユーザーから見たbehaviorはcomponent testで保証する。
