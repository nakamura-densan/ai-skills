# Next.js Review Guide

## 実構成を先に確認する

router方式、rendering方式、deployment方式、利用versionを確認し、異なる世代・方式のAPIや設計前提を混同しない。

機能の提供状況、既定動作、推奨APIがversionに依存する場合は、[Version Awareness](../core/version-awareness.md) に従う。

## Server / Client boundary

Server / Clientの境界を、データ所有と実行環境の責務に合わせる。

確認する。

- client側へ不要なmodule graphやdata access責務を広げていないか
- server-onlyな依存をclient側から参照できる構造になっていないか
- serializableでない値を境界越しに渡していないか
- server側で完結できる処理のために不要なHTTP round tripを増やしていないか
- browser API、interaction、local state等、client executionが必要な範囲を必要以上に広げていないか

## Data fetching / BFF

データ取得経路を増やすほどよいとは判断しない。Server側から直接扱えるresourceを、理由なく別のserver endpoint経由にしない。

Clientからserver resourceへアクセスする必要がある場合は、公開契約、authorization、validation、error mappingの責務を明確にする。

mutation系の仕組みをqueryへ流用するなど、APIの用途を本来の実行モデルとずらしていないか確認する。

## Cache / freshness

cacheそのものではなく、Freshness（鮮度）の責任がどこにあるかを確認する。

- invalidationのownerが明確か
- 同じdata sourceをserver / clientで別々に管理していないか
- stale dataを許容できる期間が業務要件と一致するか
- rendering / cache behaviorを暗黙のdefaultへ依存していないか

具体的なcache APIやdefault behaviorはversionで変わり得るため、必要な場合だけ公式情報を確認する。

## Compatibility

公開route、client bundle、server function等の境界では、旧新実装の同時稼働やdeployment切替中の互換性を確認する。

contract変更がある場合は、Consumer（利用側）を追跡できる仕組みと移行手順を優先する。
