# NestJS設計ガイド

## 目次

1. Module（モジュール）の`imports` / `exports`を公開境界として扱う
2. DIトークンはTypeScriptの消去後も実行時に残る形にする
3. Provider Scope（プロバイダーのスコープ）と共有可変状態を一致させる
4. リクエスト処理の役割をNestの実行位置へ合わせる
5. HTTP入力は`ValidationPipe`等で実行時契約へ変換する
6. `ExecutionContext`とメタデータをGuard等の判断材料に使う
7. Express / Fastify固有APIへの依存をアダプター境界へ閉じ込める
8. 設定は`ConfigModule`の検証済み契約として注入する
9. 非同期処理を未待機のPromiseとして逃がさない
10. 機械的な検証・設定をNestの配線へ合わせる

NestJSでは、Coreの原則を「Moduleによる公開境界、DIコンテナの実行時トークン、Providerの寿命、リクエスト処理パイプライン、プラットフォームアダプター、設定・検証API」へ具体化する。

最初にNestJSとTypeScriptの実バージョン、利用プラットフォーム（Express / Fastify等）、有効なグローバルPipe / Guard / Interceptor / Filter、`@nestjs/config`等の主要パッケージを確認する。版依存のAPIや既定動作は [Version Awareness（バージョン依存事項）](../core/version-awareness.md) に従う。

## 1. Module（モジュール）の`imports` / `exports`を公開境界として扱う

NestのModuleはProvider（プロバイダー）を既定で内部へ閉じ、`exports`したProvider（プロバイダー）だけをimport先へ公開する。この仕組みを機能境界として使う。

確認する。

- Feature Module（機能モジュール）が、その機能のControllerとProvider（プロバイダー）をまとめ、外部へ必要なProvider（プロバイダー）だけを`exports`しているか
- 他Module（モジュール）の内部Provider（プロバイダー）を使うために同じProvider（プロバイダー）を各Module（モジュール）へ再登録し、別インスタンスを作っていないか
- `@Global()`でimport関係を見えなくし、依存元を追えない構造にしていないか
- `SharedModule`や`CommonModule`へ無関係なProviderを集積し、公開APIが肥大化していないか
- Dynamic Module（動的モジュール）の`forRoot()` / `forRootAsync()`等を複数箇所で呼び、設定やProvider（プロバイダー）インスタンスを意図せず複製していないか

Module（モジュール）分割はファイル整理のために増やさない。NestのDIグラフ上で公開範囲・初期化・設定所有を分ける意味がある場合に使う。

## 2. DIトークンはTypeScriptの消去後も実行時に残る形にする

NestのDIは実行時メタデータとトークンで依存を解決する。TypeScriptの`interface`や型エイリアスは実行時に消えるため、そのまま注入トークンにはできない。

- classをトークンとして使う場合、`import type`で消去していないか確認する
- interfaceに対する実装差し替えが必要なら、`Symbol`や明示的な定数等のカスタムトークンを定義し、`@Inject()`とProvider定義を対応させる
- 文字列トークンを各所へ直書きして、タイプミスや衝突を実行時まで検出できない構造を増やさない
- `useClass`、`useValue`、`useFactory`、`useExisting`を、生成責任や別名参照の意味に合わせて選ぶ
- `ModuleRef.get()`等による動的取得を通常の依存解決へ広げ、コンストラクタから依存関係を読めなくしていないか確認する

DIエラーを型だけで防げるとは考えず、アプリケーション起動テストでProvider（プロバイダー）グラフの解決も確認する。

## 3. Provider Scope（プロバイダーのスコープ）と共有可変状態を一致させる

Provider（プロバイダー）は既定でsingleton（シングルトン）としてアプリケーション全体へ共有される。singleton（シングルトン）のServiceやRepositoryのフィールドへ、ユーザー、tenant（テナント）、request ID（リクエストID）等のリクエスト固有状態を保存しない。

`Scope.REQUEST` / `Scope.TRANSIENT`を利用する場合は次を確認する。

- request scope（リクエストスコープ）が本当にインスタンス分離を必要とする情報か
- request-scoped Provider（リクエストスコープのプロバイダー）への依存によって、Controller（コントローラー）等の依存チェーンまでrequest scope（リクエストスコープ）へ波及していないか
- 共有DB接続、HTTP Client（HTTPクライアント）、キャッシュ等をrequest scope（リクエストスコープ）へ置き、リクエストごとに不要な生成をしていないか
- WebSocket Gateway（WebSocketゲートウェイ）、Passport Strategy（Passportストラテジー）、Cron等、singleton（シングルトン）前提のコンポーネントへrequest-scoped依存を持ち込んでいないか

リクエスト固有値を下位層へ伝えるためだけに大きなDIサブツリーをrequest scope（リクエストスコープ）へ変える場合は、プロジェクトで採用済みなら`AsyncLocalStorage`等の文脈伝播方式と比較する。

## 4. リクエスト処理の役割をNestの実行位置へ合わせる

NestではMiddleware（ミドルウェア）、Guard（ガード）、Interceptor（インターセプター）、Pipe（パイプ）、Exception Filter（例外フィルター）が異なる実行位置と情報を持つ。横断処理を置く場所は、そのAPIが得られる文脈と実行順序から決める。

- Middleware（ミドルウェア）: ルート固有メタデータを必要としない前処理、低レベルのRequest操作
- Guard（ガード）: `ExecutionContext`やHandler（ハンドラー）のメタデータを使う認証・認可判定
- Pipe（パイプ）: Controller（コントローラー）の引数へ渡る値の検証・変換
- Interceptor（インターセプター）: Handler（ハンドラー）実行前後の横断処理、レスポンス変換、計測、キャッシュ等
- Exception Filter（例外フィルター）: 例外をtransport固有のエラー応答へ変換する境界

認可ルールをMiddlewareへ置いて対象Handler（ハンドラー）のメタデータを再実装したり、入力検証をInterceptor（インターセプター）へ置いたりせず、Nestが提供する実行位置を使う。

グローバル、Controller（コントローラー）、Handler（ハンドラー）単位の登録が混在する場合は、実際のリクエストライフサイクルと適用順を確認する。

## 5. HTTP入力は`ValidationPipe`等で実行時契約へ変換する

DTOのTypeScript型だけではHTTP入力は検証されない。プロジェクトがclass-validator / class-transformerを採用している場合は、`ValidationPipe`の実設定を確認する。

- `whitelist`で未定義プロパティを除去するのか、`forbidNonWhitelisted`で拒否するのかをAPI契約として決める
- `transform`によるDTO / primitive変換を使う場合、暗黙変換で意味が変わらないか確認する
- Controller（コントローラー）が受け取るDTO classと、型だけの`interface`を同一視しない。デコレーターや実行時メタデータが必要ならclassを使う
- Query / Path / Bodyそれぞれの文字列表現から内部型へ変換する場所を一つにする
- Zod等の別Validatorを採用している場合は、Nest標準DTO方式と二重管理しない

検証済みDTOを受け取った後のService（サービス）で、同じHTTP形式検証を繰り返さない。

## 6. `ExecutionContext`とメタデータをGuard等の判断材料に使う

ルート単位の権限、公開設定、tenant条件等をDecoratorで宣言する場合、Guard（ガード）側で`ExecutionContext`と`Reflector`等を使って解釈し、Decoratorと実装を対応させる。

- Handler（ハンドラー） / Controller（コントローラー）のメタデータとGuardの判定規則が別々の文字列・定数へ分裂していないか
- 認証済みユーザー情報を各Guard（ガード）やController（コントローラー）が独自形式でRequestへ書き込んでいないか
- HTTP、GraphQL、Microservice等で`ExecutionContext`から取得すべき引数が異なることを無視していないか
- Custom Decorator（カスタムデコレーター）が内部の多数のGuard（ガード） / Interceptor（インターセプター） / Pipe（パイプ）を隠し、適用効果をコードから推測できなくなっていないか

Decoratorは宣言を短くする手段として使い、重要な認可条件の正本を複数箇所へ分散させない。

## 7. Express / Fastify固有APIへの依存をアダプター境界へ閉じ込める

NestのHTTP層はプラットフォームアダプターを介してExpressまたはFastify等で動く。実際に利用しているアダプターを先に確認する。

- `@Req()` / `@Res()`やExpress / Fastify固有型をService（サービス）層まで伝播させない
- Adapter（アダプター）固有Middleware（ミドルウェア）やPlugin（プラグイン）を使う場合、その依存をbootstrap（起動処理）またはHTTP境界へ局所化する
- Express用パッケージをFastify構成へそのまま適用できると仮定しない
- Nest標準のレスポンス処理で表現できる箇所へ、理由なく低レベルResponse APIを持ち込まない
- アダプター変更を想定していないプロジェクトでも、業務ロジックがHTTP実装詳細へ依存する必要があるかは分けて判断する

## 8. 設定は`ConfigModule`の検証済み契約として注入する

`process.env`を各Provider（プロバイダー）から直接読むのではなく、プロジェクトが`@nestjs/config`を採用している場合は設定読取を集約する。

- 必須環境変数は起動時のSchema / `validate()`等で検証する
- 関連設定は`registerAs()`等の名前空間へまとめ、`ConfigType<typeof ...>`等で型付き注入できる場合は活用する
- `ConfigService.get<T>()`の`T`だけで実行時型が保証されると考えない
- Feature Module（機能モジュール）ごとの`forFeature()`を使う場合、初期化順序に依存したConstructor読取がないか確認する
- 秘密情報を設定オブジェクト全体のログへ出さない

具体的なValidator APIや設定登録方式は対象NestJS / `@nestjs/config`バージョンで確認する。

## 9. 非同期処理を未待機のPromiseとして逃がさない

Controller（コントローラー） / Provider（プロバイダー）から開始する非同期処理では、リクエスト完了後も継続すべき処理か、呼び出し元が結果を待つべき処理かを区別する。

- `void someAsync()`や未`await` Promiseで失敗を観測不能にしていないか
- Queue、Scheduler、Event等のNest連携機構を使う場合、各実行単位をHTTPリクエストとは別の失敗・再実行境界として扱う
- Queue consumerやCron処理が多重実行されても安全か、利用する基盤の再試行仕様と合わせて確認する
- リクエスト内DI文脈がバックグラウンドジョブ（background job）へ自動で引き継がれると仮定しない

Queue / Scheduler等の具体APIは、対象パッケージを実際に使用している場合だけ追加Referenceとして公式仕様を確認する。

## 10. 機械的な検証・設定をNestの配線へ合わせる

- TypeScriptのコンパイル / 型検査と採用済みのlintをCIで実行し、DecoratorやDI以前に検出できる問題は機械検証へ任せる
- Provider解決、Moduleの`exports`、Guard / Pipe / Interceptor等の配線は、`TestingModule`や起動テストで実際のDIグラフを確認する
- グローバル（global）Pipe / Guard / Interceptor / Filter、`@nestjs/config`、HTTPアダプター（adapter）の差が振る舞いへ影響する場合は、E2Eで本番の起動処理（bootstrap）に近い構成を通す
- 既にコンパイル、TestingModule、E2Eで確実に検出できる問題を、AIレビューの主Findingとして重複させない

純粋なServiceロジックはNestを起動せず小さく検証し、フレームワーク配線の検証と分ける。
