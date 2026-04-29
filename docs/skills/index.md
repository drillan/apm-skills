# スキル一覧

`apm-skills` が提供する 5 つのスキルの個別ドキュメントです。各ページは元の `SKILL.md` の主要部分を抜粋しています。完全な仕様は GitHub リポジトリの `skills/<name>/SKILL.md` を参照してください。

```{toctree}
:maxdepth: 1

apm-install
apm-init
apm-update
apm-uninstall
apm-audit
```

## ワークフロー全体像

スキルは互いに委譲しあう設計です。代表的な呼び出し関係は以下の通りです。

| 起点 | 委譲先 |
| --- | --- |
| `apm-install` | `apm-init` (apm.yml 不在時)、`apm-audit` (install 後の検証) |
| `apm-init` | `apm-install` (依存追加)、`apm-audit` (CI セットアップ) |
| `apm-update` | `apm-audit` (更新後の security 確認) |
| `apm-uninstall` | `apm-audit` (削除後の整合性確認) |

`apm-audit` は他のスキルの後段として呼ばれることが多く、CI に組み込んでおくと PR 単位で自動的に security 検証が走ります。
