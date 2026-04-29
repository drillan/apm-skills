# apm-skills

APM (Agent Package Manager) の運用支援 Agent Skills パッケージ。GitHub の `gh skill install` または APM 自身で配布可能。

## 提供スキル (5 個)

| スキル | 用途 | 自動発火条件 |
|---|---|---|
| [`apm-install`](skills/apm-install/SKILL.md) | パッケージ追加 (target / `-g` / version pin / collision 解消) | 「APM パッケージを入れたい」「skill を追加」 |
| [`apm-init`](skills/apm-init/SKILL.md) | 新規 APM プロジェクトの初期化 (apm.yml 生成、`runtime setup`) | 「APM プロジェクトを始める」、apm.yml 不在 |
| [`apm-update`](skills/apm-update/SKILL.md) | 依存の最新化 (`apm outdated` / `apm update`) | 「依存を更新」「最新化」 |
| [`apm-uninstall`](skills/apm-uninstall/SKILL.md) | パッケージ削除 (`--dry-run` 必須、`apm prune`) | 「APM パッケージを削除」「skill を消したい」 |
| [`apm-audit`](skills/apm-audit/SKILL.md) | セキュリティ監査 (Glassworm 検出、SARIF、CI 統合) | 「policy チェック」「security 確認」 |

## 対象エージェント

主想定は **Claude Code**。`gh skill install --agent <name>` で claude-code / github-copilot / cursor / codex / gemini-cli / antigravity に対応 (形式互換、発動精度はエージェント依存)。APM 側は `--target claude` / `--target codex` 等で同等の切替が可能。

## インストール

エージェント別に APM の `--target` または gh CLI の `--agent` で配置先を切り替える (Claude Code は `.claude/skills/`、GitHub Copilot は `.github/skills/`、その他もそれぞれの規約)。git で管理することを前提とする。

### 推奨: APM (Agent Package Manager) で一括導入

`--target claude` 指定で 1 コマンド導入。`apm.yml`、`apm.lock.yaml`、`.gitignore` が自動生成され、5 スキルが `.claude/skills/` 配下に配置される。

```bash
cd your-project
apm install drillan/apm-skills --target claude
git add .claude/skills/ apm.yml apm.lock.yaml .gitignore
git commit -m "chore: install apm-skills via APM"
```

`--target` を省略すると `.github/skills/` (Copilot 用レイアウト) に配置される。他エージェントを使う場合は `claude` の部分を `codex` / `copilot` / `cursor` / `gemini` 等に置き換え (複数なら `claude,copilot` のように comma 区切り、全部対応なら `all`)。

別の開発者がリポジトリを clone した後は、引数なしで再現できる。

```bash
apm install   # apm.yml の内容で全依存を復元 (target は既存ディレクトリから auto-detect)
```

CI や production では drift 防止のため tag/SHA で pin することを推奨 (例: `drillan/apm-skills@v0.1.0`。リリース後に利用可)。

APM 本体の導入手順は https://github.com/microsoft/apm を参照 (例: `curl -sSL https://aka.ms/apm-unix | sh`)。

#### ユーザースコープに入れる場合

複数プロジェクトで使い回すなら `-g` (`--global`) を付ける。manifest が `~/.apm/` に、スキルが `~/.claude/skills/` に配置される。

```bash
apm install drillan/apm-skills --target claude -g
```

user scope の対応状況は target ごとに異なる:

- 完全対応: `claude` / `gemini` / `copilot-cowork`
- 部分対応: `copilot` / `cursor` / `opencode` (一部のプリミティブ非対応)
- 非対応: `codex` (project scope のみ利用可)

アンインストールは `apm uninstall drillan/apm-skills -g`。

#### gh CLI 経由のスキルが既に存在する場合

`gh skill install` で配置したスキルは frontmatter の `metadata:` ブロックで識別される。APM はそれを検知して上書きを拒否し、`1 file skipped -- local files exist, not managed by APM` の警告を出す。APM 管理に統一する場合は `--force` で上書きするか、該当スキルを `rm -rf <skill_dir>` で削除してから再 install する。

### 代替: GitHub CLI で一括インストール

APM を導入できない環境では `gh skill install` をシェルループでまとめて実行する。

```bash
cd your-project
for s in apm-install apm-init apm-update apm-uninstall apm-audit; do
  gh skill install drillan/apm-skills "$s" --agent claude-code --scope project
done
git add .claude/skills/ && git commit -m "chore: install apm-skills"
```

ユーザースコープに入れる場合は `--scope user`。他エージェントを使う場合は `--agent` の値を `github-copilot` / `cursor` / `codex` / `gemini-cli` / `antigravity` 等に置き換える。Claude Code 以外の多くは `.agents/skills/` を共有する規約なので、`git add` 先もそれに合わせる。各スキルは description の発火条件で APM 不使用プロジェクトでは発動しないため、user スコープでも誤発火しない。

## 典型シナリオ

### A. 新規プロジェクトに APM を導入

```
User: "このプロジェクトに APM を導入してスキルを追加したい"
```

→ `apm-init` 発火 → `apm init --yes` で apm.yml 生成 → `apm runtime setup claude` で Claude Code 有効化 →
   `apm-install` 委譲で実依存追加 → `.claude/skills/` 配下に配置確認

### B. 既存プロジェクトに新しい APM パッケージを追加

```
User: "drillan/sphinx-skills を入れて"
```

→ `apm-install` 発火 → apm.yml 検出 → `--target claude` 推論 → `apm install drillan/sphinx-skills --target claude` →
   collision 検知時は `--force` 判断 → `apm.lock.yaml` 更新 → commit 案内

### C. 依存を最新化する

```
User: "依存を最新版に更新して"
```

→ `apm-update` 発火 → `apm outdated` で更新可能性確認 → `apm update` 実行 → lockfile 差分 →
   `apm-audit` 委譲で security 検証 → commit 案内

### D. CI に security 監査を組み込む

```
User: "PR で APM パッケージの security チェックを自動化したい"
```

→ `apm-audit` 発火 → `apm-policy.yml` の最小構成を提案 → `.github/workflows/apm-audit.yml` 生成 →
   SARIF アップロード設定 → Code Scanning 連携 → branch protection rule の案内

### E. 不要になった依存を削除

```
User: "もう使わない skill を消したい"
```

→ `apm-uninstall` 発火 → `apm uninstall <pkg> --dry-run` で削除対象確認 → 実行 →
   apm.yml / apm.lock.yaml / deployed files の同期削除 → commit 案内

## 開発・コントリビュート

### 検証

```bash
uv sync --group dev
uv run pytest tests/ -v                          # validator のテスト
uv run python scripts/validate_skills.py skills  # 全 SKILL.md の frontmatter 検証
uv run ruff check . && uv run ruff format .
uv run mypy scripts tests
```

### CI

`.github/workflows/ci.yml` が PR/push 時に上記検証を全実行する。`docs.yml` は `docs/**` の変更時に Sphinx ビルドして GitHub Pages にデプロイする。

### 関連プロジェクト

- [drillan/sphinx-skills](https://github.com/drillan/sphinx-skills) — Sphinx + MyST ドキュメント開発スキル集
- [Microsoft APM](https://github.com/microsoft/apm) — 本パッケージが対象とする APM 本体

## ライセンス

MIT (`LICENSE` 参照)
