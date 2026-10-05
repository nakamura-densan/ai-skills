# Target Scopes

## API

Direct Scope:

- route / controller / handler
- request / response schema
- use case / application service
- domain rule
- repository / DB access
- auth / authorization
- transaction boundary
- test

必要に応じて:

- OpenAPI / generated client
- callers / frontend consumer
- external API
- event / queue

重点:

- contract ownership
- validation responsibility
- transaction + irreversible side effect
- error classification
- contract / deployment compatibility

## Screen

Direct Scope:

- route / page
- component
- local state / hook
- validation
- API client
- styling / theme / token
- test

重点:

- redundant / contradictory state
- derived state
- effect-based synchronization
- state ownership
- component responsibility
- styling方式とtheme / token ownership
- server/client boundary（Next.js）

## Batch

Direct Scope:

- scheduler / trigger
- entrypoint
- processing loop
- use case
- repository
- external I/O
- logging / metrics
- retry / idempotency / recovery

重点:

- resume / rerun
- partial failure
- irreversible side effect
- per-item failure isolation
- transaction size
- operational diagnostics

## Module / File

公開API、責務、依存方向、state ownership、呼び出し階層、testabilityを確認する。

単一fileだけでは判断できない場合、Behavioral Scopeへ必要最小限広げる。

## Git Diff

diffを起点にするが、diffだけで完結させない。

1. 変更意図を推定する
2. 変更されたpublic contract、state、schema、DB、shared typeを特定する
3. direct dependenciesを確認する
4. 将来影響や漏れが疑われる場合だけconsumerを追う
5. 既存のunrelated debtを大量に持ち込まない

指摘は原則として今回の変更で導入・悪化・露呈した問題へ集中する。
