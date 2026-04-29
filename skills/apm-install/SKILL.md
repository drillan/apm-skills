---
name: apm-install
description: APM パッケージを Claude Code 等のエージェント設定に追加するスキル。`apm install <owner/repo>` の実行、`--target` (claude/copilot/codex/cursor/gemini 等) の選択、ユーザースコープ (`-g`) の判断、`#v1.0.0` 等での version pin 提案、collision 検知時の `--force` 判断を行う。「APM パッケージを入れたい」「skill を追加して」「apm install」といった依頼や、apm.yml が存在するプロジェクトで依存追加を求められたときに発火する。
license: MIT
allowed-tools: Bash, Read, Write, Edit
---

## 概要

APM (Agent Package Manager) パッケージを既存プロジェクトに追加するスキル。target (配置先エージェント)、scope (project / user)、version pin の 3 軸の判断を支援する。

## 前提検出

実行前に以下を順に確認する。

1. **APM CLI 有無**: `apm --version` が成功するか。失敗した場合は導入手順 (`curl -sSL https://aka.ms/apm-unix | sh`) を案内し処理停止。
2. **apm.yml 有無**: 無い場合は `apm-init` への委譲を提案 (新規プロジェクトの初期化が先)。
3. **対象 target**: ユーザー指示 → `apm runtime status` → プロジェクト内ディレクトリ (`.claude/` / `.github/` 等) の順に推論。

## 手順

### 1. 基本インストール

```bash
cd <project-root>
apm install <owner/repo> --target <name>
```

`<name>` の候補:

- `claude` (Claude Code, `.claude/skills/`)
- `copilot` (GitHub Copilot, `.github/skills/`)
- `codex` (Codex, `.codex/`)
- `cursor` (Cursor, `.cursor/`)
- `gemini` (Gemini CLI, `.gemini/`)
- `opencode` (OpenCode, `.opencode/`)

複数指定は `claude,copilot` のように comma 区切り、全部対応なら `all`。

`--target` を省略すると `.github/skills/` (Copilot 用) に配置される (auto-detection の規定値)。

### 2. ユーザースコープ (任意)

複数プロジェクトで使い回すなら `-g` (`--global`) を追加。manifest が `~/.apm/`、スキルが `~/.claude/skills/` 等に配置される。

```bash
apm install <owner/repo> --target claude -g
```

user scope の対応状況は target ごとに異なる:

- 完全対応: `claude` / `gemini` / `copilot-cowork`
- 部分対応: `copilot` / `cursor` / `opencode` (一部プリミティブ非対応)
- 非対応: `codex` (project scope のみ利用可)

### 3. Version pin (production / CI で推奨)

drift 防止のため `#v1.0.0` (tag) または `#<full-sha>` で pin する。

```bash
apm install owner/repo#v1.2.0 --target claude
# または full SHA
apm install owner/repo#abc1234567890abcdef1234567890abcdef1234567 --target claude
```

CI で固定 SHA を取得する例:

```bash
SHA=$(gh api repos/owner/repo/commits/main --jq '.sha')
apm install "owner/repo#$SHA" --target claude
```

### 4. Dry-run で副作用ゼロのプレビュー

```bash
apm install owner/repo --target claude --dry-run -v
```

何が `.claude/skills/` 等に配置されるかを事前確認できる (ただし `apm.yml` 自体は dry-run でも作成される)。

## 注意点

- **`gh skill install` で配置済のスキルがある場合**: APM は frontmatter の `metadata:` ブロックでそれを検知し、`1 file skipped -- local files exist, not managed by APM` の警告を出して上書きを拒否する。APM 管理に統一したい場合は `--force` で上書きするか、該当スキルを `rm -rf <skill_dir>` で削除してから再 install する。
- **Codex の user-scope**: 非対応。`-g` を付けても展開されないので project scope を使う。
- **配置先の確認**: `.gitignore` に `apm_modules/` が自動追記されるが、`.claude/skills/` 等の deployed files は git 管理対象として commit する設計。
- **transitive 依存**: パッケージ自体が他 APM パッケージや MCP を depend する場合、信頼性確認のため `--trust-transitive-mcp` の利用は慎重に判断する。

## 関連スキル

- **前提**: `apm-init` (apm.yml が無いプロジェクトの初期化)
- **検証**: `apm-audit` (install 後の security 確認、policy 違反検出)
- **削除**: `apm-uninstall` (依存からの除去 + integrated files 削除)
- **更新**: `apm-update` (lockfile を最新に再生成)
