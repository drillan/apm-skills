# apm-skills

APM (Agent Package Manager) の運用支援スキル集のドキュメントへようこそ。

本パッケージは Claude Code 等のエージェント向けに、APM の install / init / update / uninstall / audit を支援するスキル群を提供します。複数プロジェクトで APM を継続運用する際の知見 (target 切替、`-g` ユーザースコープ、`gh CLI` との混在、policy 違反検知など) を体系化しています。

## 提供スキル

| スキル | 用途 |
| --- | --- |
| `apm-install` | パッケージ追加 (target 選択、`-g` 判断、version pin、collision 解消) |
| `apm-init` | 新規 APM プロジェクトの初期化 (apm.yml 生成、`runtime setup`) |
| `apm-update` | 依存の最新化 (`apm outdated` / `apm update`、pin の挙動) |
| `apm-uninstall` | パッケージ削除 (`--dry-run` 必須、`apm prune`) |
| `apm-audit` | security 監査 (`apm-policy.yml`、SARIF 出力、CI 統合) |

## クイックスタート

### 推奨: APM (Agent Package Manager) で一括インストール

[Microsoft APM](https://github.com/microsoft/apm) を導入してから実行します。

```bash
cd your-project
apm install drillan/apm-skills --target claude
```

これで 5 スキルすべてが `.claude/skills/` 配下に配置されます。`--target` の値で他エージェントに切り替えできます (`codex` / `copilot` / `cursor` / `gemini` 等、複数指定は `claude,copilot` のように comma 区切り)。複数プロジェクトで使い回す場合は `-g` を追加してユーザースコープにインストールします。

### 代替: GitHub CLI で個別インストール

[GitHub CLI (`gh`)](https://github.com/cli/cli) を導入してから実行します。

```bash
cd your-project
gh skill install drillan/apm-skills apm-install --agent claude-code --scope project
```

5 スキルを一括導入する手順、`apm.yml` の例、`gh CLI` 経由のスキルが既に存在する場合の対処など、詳細は [リポジトリ README のインストール節](https://github.com/drillan/apm-skills#インストール) を参照してください。

## 目次

```{toctree}
:maxdepth: 2
:caption: Contents

skills/index
```

## 索引

- {ref}`genindex`
- {ref}`modindex`
- {ref}`search`
