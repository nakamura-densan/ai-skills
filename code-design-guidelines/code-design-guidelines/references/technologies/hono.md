# Hono設計ガイド

## 目次

1. Fetch標準部分とランタイム固有部分を分ける
2. `Context`のBindingsとVariablesを型付きの境界として使う
3. Middlewareの前後実行と登録順を設計に含める
4. 外部入力はValidatorを通した値だけを内部へ渡す
5. RPCを使う場合はRoute定義そのものを契約の正本にする
6. グローバルエラー / Not Found処理とRPC型の範囲を区別する
7. 大規模RPCでは型推論コストも設計対象にする
8. Route構成は`app.route()`を基本にし、deprecated APIを固定知識にしない
9. ランタイムBindingsを設定とインフラ依存の正本にする
10. 機械的な検証・設定をFetch境界とRPC型へ合わせる

Honoでは、Coreの原則を「Web標準のRequest / Response、実行ランタイム、`Context`、Middleware（ミドルウェア）チェーン、Validator（検証器）、ルート型推論、RPC契約」へ具体化する。

最初にHonoとTypeScriptの実バージョン、実行ランタイム（Cloudflare Workers / Node.js / Bun / Deno等）、利用アダプター、Validator、RPC利用有無を確認する。ランタイムや版に依存するAPIは [Version Awareness（バージョン依存事項）](../core/version-awareness.md) に従う。

## 1. Fetch標準部分とランタイム固有部分を分ける

HonoのHandler（ハンドラー）はWeb標準の`Request` / `Response`を中心に複数ランタイムで動く一方、起動方法、Bindings、WebSocket、静的ファイル、background（バックグラウンド）処理等はランタイムごとに異なる。

- 通常のルーティング、入力処理、レスポンス生成は可能な範囲でHono / Web標準APIへ寄せる
- Cloudflare Workers、Node.js、Bun、Deno等の固有APIを業務ロジックへ直接伝播させない
- `@hono/node-server`やCloudflare向けPackage（パッケージ）等、アダプター固有処理をentry point（エントリーポイント）やinfrastructure（インフラストラクチャ）境界へ局所化する
- Node.jsではServer（サーバー）のclose処理、Cloudflare WorkersではBindingsや`ExecutionContext`等、実際のランタイムが持つライフサイクルを確認する
- 「Honoで動くから全ランタイムで同じ」と仮定せず、ファイル、Socket、process API、background API等の差を確認する

## 2. `Context`のBindingsとVariablesを型付きの境界として使う

`Context`はリクエスト単位の情報と実行環境への入口になる。用途の異なる値を同じ場所へ無秩序に置かない。

- ランタイムから供給される環境値・KV・DB binding（DBバインディング）等は`Bindings`として`Hono<{ Bindings: ... }>`へ型付けする
- Middleware（ミドルウェア）が現在のリクエストへ付加する認証ユーザー等は`Variables`として定義し、`c.set()` / `c.get()`のキーと型を固定する
- リクエスト固有情報をmodule global（モジュールグローバル）変数へ保存しない
- `c.env`と`process.env`等を混在させ、ランタイム差を各Handler（ハンドラー）へ漏らさない
- `Variables`へ巨大なService Locator（サービスロケーター）やmutable container（可変コンテナ）を入れて依存関係を隠さない

`c.set()` / `c.get()`の値は現在のリクエスト寿命に属するものとして扱う。

## 3. Middlewareの前後実行と登録順を設計に含める

Hono Middleware（ミドルウェア）はHandlerの前後で実行され、`await next()`を境に処理が入れ子状になる。順序と対象path（パス）が振る舞いへ直接影響する。

- 認証、CORS、計測、response header等がどのpathへどの順序で適用されるか確認する
- 次の処理を実行するMiddleware（ミドルウェア）では`await next()`を正しく呼ぶ
- Middleware（ミドルウェア）自身が`Response`を返して処理を終了する場合と、後続へ委譲する場合を混同しない
- `app.use('*', ...)`等の広い適用範囲に、特定route（ルート）だけの高コスト処理を置かない
- Middleware（ミドルウェア）の登録順へ依存する値を、暗黙の前提として離れたHandler（ハンドラー）から利用しない

例外処理や`next()`の挙動はHonoバージョンで確認し、Express等のMiddleware（ミドルウェア）モデルをそのまま当てはめない。

## 4. 外部入力はValidatorを通した値だけを内部へ渡す

Hono本体のValidator（検証器）は薄い境界機構であり、入力の意味検証は明示的に定義する。

- `validator()`または採用済みのZod / Standard Schema系Validator（検証器）でquery、param、form、json等を検証する
- Handler（ハンドラー）では未検証の生入力を再度読むより、`c.req.valid()`から検証済み値を取得する
- URL path / queryは文字列として到着することを前提に、数値等への変換をSchema（スキーマ）やValidator（検証器）へ集約する
- TypeScript型やHono RPCの推論だけで実行時入力が正しいとみなさない
- 複数のValidator（検証）ライブラリを同じAPI契約へ重ね、型と検証ルールの正本を分裂させない

## 5. RPCを使う場合はRoute（ルート）定義そのものを契約の正本にする

Hono RPCはServerのRoute型からClientの入力・出力型を推論する。RPCを使う場合は、その型推論が維持されるRoute構造にする。

- Client（クライアント）へ公開する`AppType`は、実際にroute（ルート）を登録した変数から`typeof`で導出する
- Route（ルート）を分割する場合は`app.route()`等で型が連鎖する構造を使い、最上位で型情報を失わない
- RPC対象Route（ルート）では、成功・失敗それぞれのHTTP status（ステータス）を`c.json(..., status)`等で明示し、Client（クライアント）がresponse union（レスポンスのユニオン型）を判別できるようにする
- Server（サーバー） / Client（クライアント）を分ける場合は、RPC要件として必要なTypeScript `strict`設定等を実プロジェクトで確認する
- API契約をRPC型と別の手書きinterfaceで二重定義しない

RPCを利用しない公開HTTP APIでは、OpenAPI等の既存契約方式があるならそちらを優先し、Hono RPCを追加すること自体を目的にしない。

## 6. グローバルエラー（Global error） / Not Found処理とRPC型の範囲を区別する

`app.onError()`や`app.notFound()`で共通エラー形式を定義できるが、RPC Clientの型推論へ自動的に全てのGlobal response（グローバルレスポンス）が含まれるとは限らない。

- Handler（ハンドラー）が返す業務上の4xxと、予期しない例外から作る5xxを分ける
- `app.onError()`で内部例外メッセージやstackを外部へそのまま返さない
- RPC利用時、グローバルエラーレスポンス（Global error response）もClient（クライアント）契約へ含める必要がある場合は、対象Honoバージョンの`ApplyGlobalResponse`等の仕組みを確認する
- `c.notFound()`の型と実際のnot-found responseをClient（クライアント）側が必要とする場合、対象版の型拡張方式を確認する

エラー形式をRoute（ルート）ごとに独立実装し、同じエラーコードが異なるJSON形状を返す状態を増やさない。

## 7. 大規模RPCでは型推論コストも設計対象にする

Hono RPCはRoute（ルート）数が増えるとTypeScriptの型インスタンス化コストがIDE性能へ影響することがある。

- 巨大な単一の`Hono`型へ全Routeを集めるより、機能単位のsub app（サブアプリケーション）へ分けて`app.route()`で合成できないか確認する
- Client（クライアント）側も必要に応じて機能単位の型付きClient（クライアント）へ分割する
- 型推論が重い場合、公式が案内する事前コンパイル済みClient型等を対象バージョンで検討する
- 型性能対策のためにHTTP契約を手書き型へ戻し、Server（サーバー）との整合性保証を失わない

型安全性による変更検出と、編集時の応答性の両方を確認する。

## 8. Route構成は`app.route()`を基本にし、deprecated APIを固定知識にしない

HonoでHono sub app（サブアプリケーション）を組み合わせる場合は、現在の公式APIである`app.route()`を基本候補にする。

- routeのbase pathとsub app（サブアプリケーション）内部pathが重複・欠落していないか確認する
- Middleware（ミドルウェア）適用path（パス）とsub app（サブアプリケーション）のmount（マウント）位置を合わせる
- RPC利用時はroute（ルート）登録の戻り値をchainして型情報を保持する
- 他Framework（フレームワーク）のapplication（アプリケーション）を組み込む場合は、対象版で推奨されるMount Middleware（マウント用ミドルウェア）等を確認する

`app.mount()`等のdeprecated / removed状態は時点依存なので、Skill内の記憶だけで断定せず対象Honoバージョンの公式情報を確認する。

## 9. ランタイムBindingsを設定とインフラ依存の正本にする

Cloudflare Workers等ではKV、D1、R2、Secrets等がBindingsとして供給される。実行環境が提供するresource（リソース）を型なし文字列として各Handler（ハンドラー）から参照しない。

- Wrangler等が型生成を提供している場合は、生成型を`Bindings`の正本として利用できないか確認する
- Binding名を複数ファイルへ重複定義しない
- secretと公開設定を同じログやresponseへ展開しない
- Node.js等、Bindings（バインディング）モデルを持たないランタイムでCloudflare固有の`c.env`前提を持ち込まない

ランタイム設定ファイルとTypeScript型の生成方法は対象Platformの公式仕様も確認する。

## 10. 機械的な検証・設定をFetch境界とRPC型へ合わせる

- TypeScriptのコンパイル / 型検査をCIで実行し、RPCのRoute型、Bindings、Variablesの不整合を型検査で表面化させる
- HTTP契約、Middleware、Validator、`onError()`等は`app.request()`や採用済みの型付きClientを使い、実際のapp構成を経由して検証する
- ランタイム固有BindingsやAPIがある場合は、型生成やプラットフォーム公式のテスト環境を利用できるか確認し、Node.jsだけの代替テストで済ませない
- 既に型検査、Fetch境界テスト、プラットフォーム検証で確実に検出できる問題を、AIレビューの主Findingとして重複させない

単純な業務ロジックはHono `Context`から分離し、通常のTypeScript関数として小さく検証できる形を保つ。
