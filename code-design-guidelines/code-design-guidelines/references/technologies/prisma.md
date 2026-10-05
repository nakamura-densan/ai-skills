# Prisma Review Guide

PrismaをDBアクセスの基本境界として扱う。DB設計・query・transaction・migrationのレビューも、まずPrisma schemaとPrisma Clientの使い方から確認する。

DB provider固有の知識を先に当てはめない。Prismaの抽象化を越える実装がある場合だけ、`datasource`、migration SQL、raw SQL、DB固有機能を確認する。

## Query design

### N+1

loop内で1件ずつqueryする構造を問題候補として扱う。

確認する。

- relationをまとめて取得できないか
- ID群をbatch取得できないか
- 必要以上のrelationを一括取得して巨大payloadにしていないか
- query回数、DB負荷、payload、cardinalityのどこが実際のボトルネックか

query回数が少ないほど常に良いとは判断しない。

## Transaction / failure safety

nested write、複数query transaction、interactive transaction等から、use caseに合う最小のtransaction boundaryを選ぶ。

transaction内で外部API、メール送信、長時間CPU処理、user input待ち等を行い、DB transactionを長時間保持していないか確認する。

取り消せない外部副作用はDB rollbackでは戻せない。transactionで囲めば処理全体がatomicになるとは判断しない。

## Idempotency / concurrency

read-modify-writeやbatch再実行では、Idempotency（冪等性）、競合検出、retry policyを確認する。

Prismaの通常queryだけで不変条件を安全に保てない場合は、transaction、unique constraint、version field、atomic update、必要に応じたDB固有の仕組みを検討する。

競合対策は「仕組みを入れた」で終わらせず、競合検出後にretryするのか、利用側へ競合として返すのかまでuse caseで決める。

## Schema / constraint / ownership

Prisma schemaを業務上の唯一のSource of Truthと決めつけない。次の責務を区別する。

- DBで常に守るべき不変条件
- Prisma modelとして必要な構造
- API contract
- domain type

`@unique`、relation、required field等、Prisma schemaからDB制約へ反映できる保証は、application codeだけのvalidationへ寄せない。

Prisma schemaだけで表現できない制約やDB機能を使う場合は、migration SQLやDB側定義も確認する。Prisma Clientから見えない保証を、存在しないものとして扱わない。

## Migration / compatibility

schema変更では、最終形だけでなくmigration手順を確認する。

- 既存dataが新しい制約を満たすか
- column / relationのrename・split・mergeで段階移行が必要か
- app旧版と新版が同時稼働しても成立するか
- deploy途中でread / write contractが壊れないか
- rollback可能性をどう扱うか

migration SQLがPrismaの自動生成結果でも、運用上の影響が大きい変更は内容を確認する。

## Index / query performance

indexを追加する場合は、Prisma schema上の定義だけで判断せず、実際のquery pattern、filter、sort、relation、cardinality、write overheadを確認する。

performance上の問題が疑われる場合は、Prismaが生成するqueryや実行計画まで確認する。推測だけでindexやraw SQLを追加しない。

## Prismaの抽象化を越える場合

次が対象に含まれる場合だけ、実際のDB providerとその仕様を追加確認する。

- `$queryRaw` / `$executeRaw`
- migration SQLの手編集
- Prisma schemaで表現できないconstraint / index
- DB固有型・extension・function・trigger
- isolation / locking等のDB固有挙動
- provider固有のperformance問題

この場合も、Skill内に特定DBの時点依存知識を固定しない。対象projectのproviderとversionを確認し、必要な事実だけ公式情報で検証する。

## Version Awareness

Preview / GA、generator、client API、query strategy、migration behavior、provider固有機能等の状態が判断に影響する場合だけ、対象projectの実versionと公式情報を確認する。

詳細は [Version Awareness](../core/version-awareness.md) に従う。
