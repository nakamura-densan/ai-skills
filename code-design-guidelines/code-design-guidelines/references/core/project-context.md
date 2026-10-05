# Project Context Cache

## 目的

`.code-design/context.md` を、プロジェクトを毎回ゼロから探索しないための索引兼キャッシュとして使う。

正式なArchitecture DocumentationやADRの代わりにはしない。削除して再生成できる非権威的キャッシュとして扱う。

## 推奨形式

```md
# Code Design Context

Last analyzed commit: <hash>
Updated at: <ISO datetime>

## Explicit sources
- docs/architecture.md — overall architecture
- docs/adr/001-tenant-rls.md — tenant isolation decision
- eslint.config.mjs — lint policy

## Technology
- TypeScript: x.y
- React: x.y
- Next.js: x.y
- Prisma: x.y
- Prisma datasource provider: <provider>

## Observed architecture
- [High] API application layer is under src/application/... Evidence（根拠）: ...
- [Medium] Shared API types appear to be generated from ... Evidence（根拠）: ...

## Authoritative owner / source of truth
- Order status: ...
- API schema: ...
```

## 情報分類

### Explicit（明示）

ADR、設計書、規約等で明示された事実。

### Observed（観測）

代表実装や設定から推測した構造。High / Medium / LowのConfidence（確信度）を付ける。

Observedを正式ルールとして扱わない。

## 鮮度管理

保存済みcommitと現在commitを比較する。

次が変更されていれば該当項目だけ再確認する。

- architecture docs / ADR
- package / lockfile
- compiler / lint config
- schema / migration
- core module / shared contract

毎回全件再分析しない。

## 複数AIでの利用

同一worktreeで複数AIが動く場合:

1. 開始時にcontextを読む
2. 必要な追加調査を行う
3. 最後にdurable factだけmergeする
4. 作業中に細かく書き続けない

原則としてgitignore対象にし、merge conflictを避ける。

## 保存しない情報

- Finding
- Severity
- 修正案
- 一時的な仮説
- その実行だけの探索メモ
- 既存Docsの長い要約
