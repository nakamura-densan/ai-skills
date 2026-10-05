# AI Skills
AIエージェントで利用する、再利用可能なSkillをまとめたリポジトリです。
主に社内での利用を目的として公開しています。

## Skills

| Skill | Description |
| --- | --- |
| [grill-trace](./grill-trace/) | grilling Skill が構成した意思決定の質問を HTML 質問票として提示し、回答過程を証跡として保存するためのSkill |
| [code-design-guidelines](./code-design-guidelines/) | コードの理解容易性・変更容易性・検証容易性などを重視し、実装や設計レビューを支援するためのSkill |
| [write-typescript](./write-typescript/) | TypeScript / TSXの実装・修正・リファクタリング・コードレビュー時に、保守しやすく型安全なコードを書くためのSkill |

各Skillの詳細な用途や利用方法については、それぞれのディレクトリにある `README.md` を参照してください。

## Repository Structure

各Skillは、人向けのドキュメントとAIが利用するSkill本体を分離して管理します。

```text
ai-skills/
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── .github/
│
├── <skill-name>/
│   ├── README.md
│   └── <skill-name>/
│       ├── SKILL.md
│       ├── references/
│       ├── scripts/
│       └── assets/
│
└── ...
```

- `<skill-name>/README.md`
  - Skillの概要、用途、利用方法など、人が参照するためのドキュメントです。
- `<skill-name>/<skill-name>/`
  - AIが利用するSkill本体です。
  - このディレクトリ単体で利用できる構成とします。
- `SKILL.md`
  - Skillのエントリポイントです。
- `references/`、`scripts/`、`assets/`
  - Skillで必要な場合のみ配置します。

## License

ライセンスについては [LICENSE](./LICENSE) を参照してください。