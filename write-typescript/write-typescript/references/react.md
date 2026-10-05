# React / Frontend Rules

React / TSX を変更・レビューする場合に適用する。

## UIコンポーネント

- 新しいUI部品を作る前に、プロジェクトで採用している共通UIライブラリや既存componentを確認し、要件を満たすものがあれば優先して再利用する。

## コンポーネント

- 関数コンポーネントを使用し、class componentは使用しない。
- componentの実装を定義するファイルでは、exportする主要componentを1つにすることを基本とする。barrel fileや再export用fileには適用しない。
- 小さく単純なJSXを、コード量を減らす目的だけでcomponentへ切り出さない。
- 複数箇所で再利用する、独立したUI上の責務がある、または親componentの主要なJSX・処理フローを読みやすくできる場合にcomponentへの抽出を検討する。
- 親component専用の補助componentは同じファイルに置いてよいが、独立して再利用・testする単位になったら別ファイルへ分離する。
- 独立したUI上の責務を持つJSX / ReactNode生成処理は、utility functionではなくcomponentとして表現する。ライブラリが要求するrender callbackや、単純なlocal render helperまで機械的にcomponent化しない。
- 同じcomponentを複数の状態で使うためだけに内部stateや条件分岐を複雑化せず、呼び出し側で明示的に使い分けることを検討する。

## Hooks / State

- `useEffect`はReact管理外との同期に限定し、derived state、propsからstateへの単純copy、イベント処理、通常のデータ変換には使用しない。
- `useMemo`と`useCallback`は、実際の性能問題や明確な参照安定性要件がある場合だけ使用する。
- stateは必要最小限にし、propsや既存stateから計算できる値は原則として保持しない。
- custom hookは、Hooksを使うロジックを複数箇所で再利用する、独立したユースケースとして名前を付ける価値がある、または外部システムとの同期を隠蔽する場合に抽出を検討する。
- stateを持つことやcomponentを短くすることだけを理由に、ロジックをcustom hookへ分離しない。
- routingやglobal state管理は、プロジェクトですでに採用しているライブラリと設計に従い、同じ責務を持つ別ライブラリや独自方式を不用意に追加しない。

## Storybook / UI Test

- Storyはcomponentの代表的な使用例とtestに利用できる状態を表現する。
- `any`、compilerを黙らせる目的の型アサーション、非nullアサーションを使用しない。
- user interactionがある場合は`play`関数等、プロジェクトで採用しているinteraction testの仕組みを使用することを検討する。
- `testId`への依存を避け、roleやlabel等のaccessible queryを優先する。
- testを通すためだけにcomponentのstyleや構造を変更せず、accessibilityも確認する。
