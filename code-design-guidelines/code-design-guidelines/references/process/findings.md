# Finding and Report Format

## 目次

- [Severity（重要度）](#severity重要度)
- [Confidence（確信度）](#confidence確信度)
- [原則名だけでFindingを作らない](#原則名だけでfindingを作らない)
- [ソースコード参照リンク](#ソースコード参照リンク)
- [Findingテンプレート](#findingテンプレート)
- [修正案の確認](#修正案の確認)
- [Reportテンプレート](#reportテンプレート)
- [Findingがない場合](#findingがない場合)

## Severity（重要度）

### MUST FIX

現在または将来の正しさ・整合性・安全な変更・再実行・データ保護・重大障害へ直結する問題。

例:

- transaction途中の外部副作用によって不整合が生じる
- status追加時に必須処理が漏れる構造
- 不正stateを通常操作で作れる
- 契約変更で重大なconsumer破壊を検出できない

### SHOULD FIX

現状動作しても、理解・変更・テスト・診断・運用のコストを継続的に増やす構造。

### CONSIDER

文脈依存で、現時点では必須でないが改善価値がある事項。

## Confidence（確信度）

- **High**: コード・設定・契約・公式仕様から直接確認できる。
- **Medium**: 根拠はあるが、未確認のproject decisionやruntime条件で変わり得る。
- **Low**: 原則出さない。追加調査してHigh / Mediumへ上げられなければFindingから外す。

## 原則名だけでFindingを作らない

`DRY違反`、`責務分離が不十分`、`可読性が低い`、`疎結合ではない`のように、設計用語だけを問題名・理由として使わない。

Findingでは、[Core Principles](../core/principles.md) のOperational Definitionに基づき、少なくとも次を具体化する。

- 対象コードで観測できる状態
- その状態が理解・変更・検証・単純性のどれを損なうか
- 問題が起きる仕組み
- 現在または将来の具体的な影響

設計用語は、具体的な問題を短く分類するラベルとしてのみ使う。

## ソースコード参照リンク

Report内でリポジトリ内のコード・設定・schema等を示す場合は、可能な限りMarkdownリンクにして、表示されたReportから対象ファイルへ移動できるようにする。

### 基本形式

リンクの表示名には、リポジトリルートからの相対pathと行番号を含める。

```md
[`src/orders/order.service.ts:42`](../../../src/orders/order.service.ts#L42)
[`src/orders/order.service.ts:42-58`](../../../src/orders/order.service.ts#L42-L58)
```

`.code-design-guidelines/reports/<review>/report.md` からリポジトリルートまでは通常 `../../..` だが、Reportの保存場所を変更した場合は、Reportから対象ファイルまでの相対pathを実際の配置から計算する。絶対ローカルpathはReportへ記録しない。

### 行番号リンク

- Markdownの表示環境が `#L42` / `#L42-L58` のような行アンカーを解釈できる場合は、行番号までリンクする。
- 行アンカーの仕様を確認できない環境では、ファイルへの相対リンクを優先し、表示名に `:42` や `:42-58` を残す。
- Git hosting platform、remote URL、review対象commitを確実に判定でき、platform固有のpermalink形式を正確に生成できる場合は、対象commitへ固定したpermalinkを使用してよい。長期保存するReportでは、後から行位置が変わらないpermalinkを優先する。
- platformやURL形式を推測して壊れたリンクを生成しない。

### Evidenceでの記載

Evidence（根拠）は、リンクだけで終わらせず、その箇所が何を示すかを短く書く。

```md
#### Evidence（根拠）

- [`src/orders/order.service.ts:42-58`](../../../src/orders/order.service.ts#L42-L58) — DB更新後、transaction確定前に外部通知を実行している。
- [`prisma/schema.prisma:120-128`](../../../prisma/schema.prisma#L120-L128) — 再実行を識別する一意制約がない。
```

複数箇所を根拠にする場合は、それぞれを個別リンクにする。

## Findingテンプレート

1つのFindingは `###` で区切り、その構成要素は `####` の小見出しにする。`Target（対象）:` のように項目名と本文を同じ段落構造へ置かない。

```md
### MUST FIX: <短い問題名>

#### Target（対象）

- [`path/to/file.ts:42-58`](../../../path/to/file.ts#L42-L58)

#### Problem（問題）

<何が問題か>

#### Why（理由）

<どのCoreへ影響し、どの仕組みで問題が起きるか>

#### Future Impact（将来影響）

<具体的な仕様変更、状態追加、再実行、障害シナリオ等>

#### Remediation（修正案）

<最小十分な具体策。必要なら代替案と判断条件>

#### Evidence（根拠）

- <コード / 設定 / 公式仕様。リポジトリ内の対象は可能な限りMarkdownリンクにする>

#### Review Metadata（レビュー情報）

- Detection（検出手段）: AI | Compiler | Linter | Test | DB | CI
- Tooling Status（ツール設定状況）: configured | missing | disabled | not-applicable
- Confidence（確信度）: High | Medium
```

Review Metadata（レビュー情報）では、Confidence（確信度）は必ず記載する。Detection（検出手段） / Tooling Status（ツール設定状況）は有益な場合だけ付ける。

## 修正案の確認

修正案自体が新しい負債を作らないか確認する。

- ファイル数・interface数を不必要に増やしていないか
- 単純な条件をStrategy等へ過剰分割していないか
- 一つの問題を直すために複数の新概念を導入していないか
- 仕組みで再発防止できるか
- Project Ruleと矛盾していないか

## Reportテンプレート

```md
# Review Report（レビューレポート）

## Target（対象）
<対象。リポジトリ内のファイルは可能な限りMarkdownリンクにする>

## Summary（概要）
- MUST FIX: n
- SHOULD FIX: n
- CONSIDER: n

## Findings（指摘事項）
...

## Confirmed（確認済み事項）
- <重要観点を確認し、問題がなかったもの>
```

Confirmed（確認済み事項）は、重要項目を実際に確認したことを伝える場合だけ使う。全チェック項目を機械的に並べない。

## Findingがない場合

無理に指摘を作らない。Target（対象）、確認した重要観点、Findingなしという結論、必要なら未確認範囲を明記する。
