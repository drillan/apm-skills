# apm-skills

APM (Agent Package Manager) の運用支援スキル集のドキュメントへようこそ。

本パッケージはエージェント非依存で APM の install / init / update / uninstall / audit を支援するスキル群を提供します (Claude Code / Copilot / Codex / Cursor / Gemini / OpenCode 等で利用可)。複数プロジェクトで APM を継続運用する際の知見 (target 切替、`-g` ユーザースコープ、`gh CLI` との混在、policy 違反検知など) を体系化しています。

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

[Microsoft APM](https://github.com/microsoft/apm) を導入してから、対象エージェントを `--target` で指定して実行します。

```bash
cd your-project
apm install drillan/apm-skills --target <agent>
```

`<agent>` の選択肢: `claude` / `copilot` / `codex` / `cursor` / `gemini` / `opencode` 等。複数指定は `claude,copilot` のように comma 区切り、全対応なら `all`。配置先はエージェント別 (Claude Code なら `.claude/skills/`、GitHub Copilot なら `.github/skills/` 等)。複数プロジェクトで使い回す場合は `-g` を追加してユーザースコープにインストールします。

### 代替: GitHub CLI で個別インストール

[GitHub CLI (`gh`)](https://github.com/cli/cli) を導入してから、`--agent` で対象エージェントを指定して実行します。

```bash
cd your-project
gh skill install drillan/apm-skills apm-install --agent <agent> --scope project
```

`<agent>` の値は `claude-code` / `github-copilot` / `cursor` / `codex` / `gemini-cli` / `antigravity` 等から選択します。5 スキルを一括導入する手順、`apm.yml` の例、`gh CLI` 経由のスキルが既に存在する場合の対処など、詳細は [リポジトリ README のインストール節](https://github.com/drillan/apm-skills#インストール) を参照してください。

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
