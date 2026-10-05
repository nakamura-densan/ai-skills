# Styling Review Guide

## 目次

- [1. スタイリング方式を先に特定する](#1-スタイリング方式を先に特定する)
- [2. 共通の設計基準](#2-共通の設計基準)
- [3. Runtime CSS-in-JS](#3-runtime-css-in-js)
- [4. Component libraryのstyling API](#4-component-libraryのstyling-api)
- [5. Zero-runtime styling](#5-zero-runtime-styling)
- [6. CSS / CSS Modules等](#6-css--css-modules等)
- [7. 複数方式の併存](#7-複数方式の併存)
- [8. Version Awareness](#8-version-awareness)

## 1. スタイリング方式を先に特定する

画面・Componentの設計判断やレビューでは、CSSファイルの存在を前提にしない。対象プロジェクトが実際に採用しているスタイリング方式を確認してから、その方式に合う基準を適用する。

確認対象:

- `package.json` / lockfileの依存関係
- framework / component libraryの設定
- build plugin / bundler設定
- 対象周辺のimport
- theme / token定義
- 代表的なComponentの実装

依存パッケージが存在するだけで採用方式を決めない。直接importされているか、Component libraryの内部依存か、実際にどのAPIでstyleを記述しているかを確認する。

代表的な方式として次を識別する。これは固定一覧ではない。

- CSS / SCSS / CSS Modules
- Runtime CSS-in-JS（例: Emotion）
- Component libraryが提供するstyling API（例: Material UIの`sx`、`styled`、theme customization）
- Zero-runtime styling（例: vanilla-extract）
- その他、プロジェクトで採用されているutility / styling library

複数方式が併存する場合は、どの方式が主で、どの境界で使い分けているかを確認する。単一方式への統一を機械的に要求しない。

## 2. 共通の設計基準

スタイリング方式に関係なく、次を確認する。

### Styling responsibility（スタイル責務）

Component自身、親layout、theme、共通tokenのどこが責務を持つかを明確にする。

- 大きなlayoutは原則として親が制御する
- 子Componentがmarginやpositionのmagic numberで親layoutを打ち消し続ける構造を避ける
- Component内部の実装詳細へ外部styleが過度に依存しない
- global styleから個別Componentの内部構造へ広く侵入しない

### Theme / Token ownership（Theme・Tokenの所有責任）

色、spacing、typography、radius等で同じ意味を持つ値は、プロジェクトのthemeやdesign tokenをSource of Truthとして扱う。

同じ数値や色という理由だけで共通化しない。同じ意味と変更理由を持つ値かで判断する。

既存のtheme / token機構がある場合、独自の定数体系を並行して増やさない。

### Variant / State（Variant・状態）

Componentの状態やvariantとstyleの対応を追跡しやすくする。

- 同じvariant条件を複数箇所へ重複させない
- 不可能なstyle stateを作りやすいbooleanの組み合わせを増やさない
- Component APIとstyle variantの対応を予測可能にする

### Override（上書き）

利用ライブラリが提供する正式な拡張・override方法を優先する。

生成class名、非公開DOM構造、過度に深いselector、`!important`の連鎖など、内部実装へ強く依存する上書きを問題候補とする。

### 最小十分な方式を使う

既存のスタイリング方式で表現できる変更のために、新しいstyling libraryや別方式を追加しない。新方式の導入は、既存方式では解決しにくい具体的な問題と移行・併存コストを比較して判断する。

## 3. Runtime CSS-in-JS

Emotion等のRuntime CSS-in-JSが実際に使われている場合、そのライブラリの責務とproject conventionに沿って確認する。

重点:

- theme / tokenの利用方法が一貫しているか
- `css` / `styled`等の使い分けがプロジェクト内で予測可能か
- 動的styleが必要な理由を追跡できるか
- styleのためだけに不要なComponent stateや計算を増やしていないか
- SSR、style挿入順、hydration等が設計判断に影響する場合、現在の構成と公式情報を確認しているか

Runtime costや再生成を性能問題として指摘する場合は、ライブラリ名だけから一般化せず、実際の実装・計測・公式仕様に根拠を持つ。

## 4. Component libraryのstyling API

Material UI等のComponent libraryを使う場合、内部のstyle engineだけでなく、アプリケーションが実際に利用している公開styling APIを特定する。

Material UIであれば、対象projectで`sx`、`styled`、theme customization、component variants / overrides等のどれを採用しているかを確認する。具体的な推奨APIやversion依存挙動はSkillへ固定せず、必要な場合だけ現在の公式情報を確認する。

責務の目安:

- application全体のsemanticな共通ルール → theme / token
- Component固有の見た目・variant → Component側のstyling
- 親子配置 → layoutを所有する側
- 一度だけ必要な局所調整 → project conventionに沿ったlocal styling

同じsemantic ruleが繰り返されている場合はthemeや共通Componentへの集約を検討する。一度しか現れないstyleまで抽象化しない。

## 5. Zero-runtime styling

vanilla-extract等のZero-runtime方式が実際に使われている場合、build時にstyleを生成する設計意図を保つ。

重点:

- theme contract / token等の正本が明確か
- staticに表現できるstyleを、不要なruntime style処理へ戻していないか
- runtime値が必要な場合、その方式がプロジェクトとライブラリの正式な仕組みに沿っているか
- recipe / utility abstractionを、実際に繰り返すvariantやsemantic patternに対して使っているか
- style定義の分割が細かすぎて、Componentの見た目を理解するための探索を増やしていないか

特定APIの利用可否や推奨状態はversion依存になり得るため、必要な場合は [Version Awareness](../core/version-awareness.md) に従う。

## 6. CSS / CSS Modules等

CSSまたはCSS Modules等を直接使う場合は、cascade、inheritance、specificity、selector scopeを確認する。

問題候補:

- specificity競争を`!important`や深いselectorで解決している
- selectorの適用範囲が広く、別Componentの変更で影響する
- 親から自然にinheritするpropertyを子で大量に再指定している
- Componentの責務外の子孫構造へ強く依存している
- size、position、margin等が相殺し、layout responsibilityが分からない

CSS Custom Propertiesを使う場合も、単なる値の共通化ではなくsemantic tokenとしての所有責任を見る。

## 7. 複数方式の併存

複数のstyling systemが存在すること自体をFindingにしない。次を確認する。

- 方式の使い分けに明確な境界があるか
- 同じComponent内で理由なく複数方式を混在させていないか
- theme / tokenのSource of Truthが分裂していないか
- overrideや優先順位がライブラリ間の偶然の挙動へ依存していないか
- 移行途中であれば、新旧方式の境界と移行方針を追跡できるか

既存方式を統一・移行する提案は、理解・変更・検証コストが実際に下がる場合に限る。

## 8. Version Awareness

styling libraryやcomponent libraryのDeprecated / Legacy /推奨API、SSR統合方法、build設定等は時間とともに変わり得る。

Skill内の固定知識だけで断定せず、対象projectの実バージョンを確認し、判断に必要な場合だけ現在の公式情報を確認する。
