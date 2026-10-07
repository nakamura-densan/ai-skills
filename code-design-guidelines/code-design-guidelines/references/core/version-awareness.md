# Version Awareness

## 目的

時間とともに変わる技術情報をSkillへ固定せず、設計判断・実装・レビューを行う時点のプロジェクトと公式情報に基づいて判断する。

Skill内には、長期間有効な設計原則と確認手順だけを保持する。Deprecated（公式非推奨）、Removed（削除済み）、Legacy、Preview / GA、既定動作、機能提供状況、ブラウザ対応状況など、変化し得る事実は固定知識として保持しない。


## ステータス語の意味

- **Deprecated（公式非推奨）**: 対象versionへ適用できる公式情報が、当該API・機能をdeprecatedまたは同等の公式非推奨状態として明示している場合だけ使う。
- **Removed（削除済み）**: 対象versionでは当該API・機能を利用できないことを公式情報で確認した場合に使う。
- **Legacy**: 公式情報またはプロジェクト文書がその語を明示している場合に限って使う。意味が曖昧な場合は`Legacy`と要約せず、保守のみ・旧方式・新規利用非推奨など確認できた具体的な状態を書く。
- **Discouraged（設計上の非推奨）**: 公式statusではなく、このSkillまたはプロジェクトの設計判断として避ける方がよい状態を指す。

これらのラベルを印象で付けない。ラベルより、対象versionで何が利用でき、何が推奨され、どの互換性制約があるかという具体的事実を優先する。

## 確認手順

1. `package.json`、lockfile、runtime設定、DB情報等から、対象プロジェクトの実バージョンと有効な設定を確認する。
2. 判断がバージョン依存の事実に依存する場合だけ、その時点の公式ドキュメント、Migration Guide、Release Note、仕様書等で確認する。
3. 公式の状態と、このSkillの設計上の評価を区別する。Skill上のDiscouraged（設計上の非推奨）を、公式Deprecatedと表現しない。
4. 対象バージョンへ適用できる根拠が確認できなければ、バージョン依存の断定を設計判断・実装方針・Findingの根拠にしない。
5. 確認した版依存情報は必要な出力へ根拠とともに残し、Skill本体へ追記しない。Reviewではreportへ記録する。

## 情報源

版依存の判断では、可能な限り次の順で確認する。

1. 対象技術の公式ドキュメント
2. 公式Migration Guide / Upgrade Guide
3. 公式Release Note / Changelog
4. 言語仕様・標準仕様・公式互換性データ

ブログや既存コードの印象だけで、Deprecated、Removed、非互換などを断定しない。

## Project Contextへ保存する情報

`.code-design-guidelines/cache/context.md` には、プロジェクト自身のバージョンや有効設定は保存してよい。

保存しないもの:

- 「version Xではfeature YがDeprecated」といった外部技術の時点依存知識
- 将来の削除予定
- 最新版だけに基づく推奨状態
- 一時的なPreview / Experimental status
