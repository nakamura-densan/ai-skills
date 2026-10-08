# Spring Boot設計ガイド

## 目次

- [1. Beanの依存と寿命をコンテナ定義へ合わせる](#1-beanの依存と寿命をコンテナ定義へ合わせる)
- [2. 関連設定はConfigurationPropertiesへ集約し起動時に検証する](#2-関連設定はconfigurationpropertiesへ集約し起動時に検証する)
- [3. Web境界でHTTP入力と内部モデルを分ける](#3-web境界でhttp入力と内部モデルを分ける)
- [4. TransactionalをAOPプロキシの実行境界として扱う](#4-transactionalをaopプロキシの実行境界として扱う)
- [5. AOPベースのアノテーションを通常のメソッド呼び出しと同一視しない](#5-aopベースのアノテーションを通常のメソッド呼び出しと同一視しない)
- [6. Asyncとタスク実行器を処理容量まで含めて設計する](#6-asyncとタスク実行器を処理容量まで含めて設計する)
- [7. 自動設定を上書きする場合は適用条件を確認する](#7-自動設定を上書きする場合は適用条件を確認する)
- [8. 機械的な検証・設定をApplicationContextへ合わせる](#8-機械的な検証設定をapplicationcontextへ合わせる)
- [9. Spring MVCとWebFluxの実行モデルを混在させない](#9-spring-mvcとwebfluxの実行モデルを混在させない)

Spring Bootでは、Coreの原則を「DIコンテナ、Beanスコープ、AOPプロキシ、設定バインド、HTTP境界、トランザクション、タスク実行、テスト用ApplicationContext」へ具体化する。

最初にSpring BootとSpring Frameworkの実バージョン、Spring MVC / WebFlux、利用中のデータアクセス技術、`@EnableAsync`等の有効化設定、Javaバージョンを確認する。版依存の自動設定やAPIは [Version Awareness（バージョン依存事項）](../core/version-awareness.md) に従う。

## 1. Beanの依存と寿命をコンテナ定義へ合わせる

Spring Bootではコンストラクタ注入（constructor injection）を基本とし、必須依存を生成時に確定できる構造を優先する。

- 必須依存はコンストラクタ引数として受け、可能なら`final`で保持する
- フィールド注入（field injection）で、生成後まで必須依存が確定しない構造を増やさない
- 実装が複数ある場合、`@Qualifier`等を付ける前に「利用側がどの実装を選ぶ責務を持つべきか」を確認する
- コンポーネント走査（Component Scan）と`@Bean`定義で同じ責務のBeanが重複登録されていないか確認する

既定のシングルトンスコープ（singleton scope）はApplicationContext内で共有インスタンスになる。ControllerやService等のsingleton Beanへ、リクエストごとに変わる可変状態をフィールドとして保持しない。

`prototype` / `request` / `session`等の別スコープを使う場合は、その寿命と注入タイミングを確認する。特に`prototype` Beanを`singleton`へ通常注入すると、そのsingleton生成時に解決された同一インスタンスを持ち続ける点を見落とさない。

## 2. 関連設定は`@ConfigurationProperties`へ集約し起動時に検証する

同じ機能の設定値が複数ある場合、`@Value`を各クラスへ散在させるより、意味のまとまりごとに`@ConfigurationProperties`へ集約する。

- 接頭辞（prefix）と型で設定契約を明示する
- 必須値、範囲、形式は`@Validated`とJakarta Bean Validation等で起動時に検出する
- デフォルト値が業務上の意味を変える場合、欠損を無言で補完しない
- `System.getenv()`や`System.getProperty()`を業務ロジックから直接読む箇所を増やさない
- 秘密情報を設定オブジェクトの`toString()`やログへ出さない

`record`によるコンストラクタバインド（constructor binding）等、具体的な記述方法は対象Spring Bootバージョンで確認する。

## 3. Web境界でHTTP入力と内部モデルを分ける

Spring MVCとWebFluxでは実行モデルが異なるため、最初にどちらを使っているか確認する。

HTTP Controllerでは次を確認する。

- `@RequestBody`、`@ModelAttribute`、パス / クエリパラメーター等の外部入力を境界で検証しているか
- `@Valid` / `@Validated`や対象バージョンのメソッド検証（method validation）の挙動を理解しているか
- 永続化Entityや内部ドメインオブジェクトを、そのまま公開HTTP契約として入出力していないか
- HTTPステータス、エラーコード、レスポンス形式を各Controllerで個別に組み立てず、`@ControllerAdvice` / `@ExceptionHandler`等で契約を集約できないか
- 内部例外のクラス名やスタックトレースを外部レスポンスへ漏らしていないか

バリデーション例外の型やmethod validationの起動条件はSpring Frameworkバージョンで変わり得るため、具体的な捕捉対象を固定知識だけで決めない。

## 4. `@Transactional`をAOPプロキシの実行境界として扱う

`@Transactional`は、注釈が書かれているだけで常にトランザクション（transaction）が成立するわけではない。既定のプロキシ方式（proxy mode）では、Springが作るプロキシを経由する呼び出しに対して適用される。

確認する。

- トランザクション境界が、DB更新を一つの業務操作として確定すべきサービス / ユースケース層へ置かれているか
- 同一Bean内の自己呼び出し（self-invocation）によって、`@Transactional`付きメソッドを直接呼び、注釈が効いたつもりになっていないか
- `@PostConstruct`等、プロキシが期待通り働かないライフサイクル中にトランザクションへ依存していないか
- `REQUIRES_NEW`等の伝播属性（Propagation）を、部分コミットを本当に必要とする理由なしに使っていないか
- 長い外部HTTP呼び出しやメール送信等をDBトランザクション内に含め、ロック保持時間や部分失敗を増やしていないか

Spring Frameworkの宣言的トランザクションは、既定では`RuntimeException`と`Error`をロールバック対象とし、検査例外（checked exception）は対象外とする。バージョンや設定で既定値を変更できるため、対象プロジェクトのロールバック設定を確認し、例外型の変更だけでコミット / ロールバック結果が意図せず変わらないよう契約を明示する。

## 5. AOPベースのアノテーションを通常のメソッド呼び出しと同一視しない

`@Transactional`、`@Async`等、プロキシ経由で振る舞いが付加される機能では「メソッドにアノテーションがある」ことと「実行時にインターセプター（interceptor）が通る」ことを分けて考える。

- 自己呼び出し（self-invocation）で付加機能が失われないか
- 対象BeanがSpring管理下か
- メソッド可視性（method visibility）やプロキシ方式が対象バージョンの要件に合うか
- アノテーション（annotation）をprivateな補助メソッドへ移動した結果、期待した機能が働かなくなっていないか

AOP機能ごとに適用条件は異なるため、`@Transactional`の制約をすべての注釈へ機械的に一般化せず、利用機能の公式仕様を確認する。

## 6. `@Async`とタスク実行器を処理容量まで含めて設計する

`@Async`を付けるだけで安全な非同期処理になるとは考えない。

- 既定のプロキシ方式（proxy mode）では、自己呼び出し（self-invocation）で`@Async`が効かないことを考慮する
- どの実行器（`Executor` / `TaskExecutor`）で実行されるかを確認する
- スレッドプール（thread pool）の最大数、キュー（queue）容量、拒否時挙動が負荷特性と合っているか確認する
- `void`の非同期処理で例外を呼び出し側へ返せない場合、失敗を観測・再実行できる設計にする
- トランザクション、`SecurityContext`、MDC等のスレッド依存文脈が非同期境界を自動で越えると仮定しない

仮想スレッド（virtual thread）をSpring Boot設定で利用する場合も、DBコネクションプールや外部サービスの同時実行上限を別に設計する。`spring.threads.virtual.enabled`等の具体設定と自動設定される実行器（executor）は対象Spring Boot / Javaバージョンで確認する。

## 7. 自動設定を上書きする場合は適用条件を確認する

Spring Bootの自動設定（Auto-configuration）はクラスパス、Bean有無、設定値等の条件で有効・無効になる。

- 「動かないから」と無関係な自動設定を広く除外しない
- 独自Beanを追加した結果、Boot側の自動設定が適用されなくなっていないか確認する
- 複数Bean候補がある場合、名前や`@Primary`だけで偶然解決せず、どれが既定かを設計として決める
- 条件付きBeanが環境ごとに変わる場合、起動時の構成をテストや診断情報で確認できるようにする

依存追加だけで自動設定が変化することがあるため、スターター（starter）やライブラリ追加時はBean構成・設定値・起動ログへの影響を確認する。

## 8. 機械的な検証・設定をApplicationContextへ合わせる

- `@ConfigurationProperties`、Bean定義、自動設定等の不整合は、設定検証や必要範囲のApplicationContext起動で早期に検出する
- Controller、JSONバインド、バリデーション、Repository等のフレームワーク境界は、対象バージョンのテストスライスや統合テストを使い分ける
- `@SpringBootTest`は、自動設定や複数層の統合を実際に確認する場合に使い、純粋な業務ロジックまでSpring起動へ依存させない
- 既に起動テスト、ビルド、静的解析で確実に検出できる問題を、AIレビューの主Findingとして重複させない

具体的なテストスライス、AOT、build plugin等の利用可否はSpring Bootバージョンで変わり得るため、対象版の公式Referenceを確認する。

## 9. Spring MVCとWebFluxの実行モデルを混在させない

Spring MVCはServletベース、WebFluxはReactive Streamsベースの実行モデルを持つ。依存が入っているという理由だけで、同じ設計判断を両方へ適用しない。

WebFluxを使う場合は追加で確認する。

- Reactorチェーン（Reactor chain）内へ長時間のブロッキングI/O（blocking I/O）を直接持ち込んでいないか
- ThreadLocal前提の文脈伝播をそのまま利用していないか
- リアクティブトランザクション（reactive transaction）の文脈と命令型トランザクション（imperative transaction）を混同していないか

Spring MVCで通常のブロッキング構成（blocking stack）を採用している場合、リアクティブ型へ置き換えること自体を改善と判断しない。
