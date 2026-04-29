---
name: apm-init
description: APM プロジェクトを新規セットアップするスキル。`apm init --yes` で apm.yml ひな形を生成し、`apm runtime setup <target>` で対象エージェント (claude/copilot/codex/cursor/gemini 等) を有効化する。「APM プロジェクトを始める」「apm.yml を作成」「APM をセットアップ」といった依頼、または apm.yml が無いプロジェクトで APM 関連操作を求められたときに発火する。
license: MIT
allowed-tools: Bash, Read, Write
---

## 概要

新規プロジェクトに APM (Agent Package Manager) を導入するスキル。`apm.yml` のひな形作成と、対象エージェントの runtime setup までを 1 ストロークで完了させる。

## 前提検出

1. **APM CLI 有無**: `apm --version` で導入確認。未導入なら案内して処理停止。
2. **既存 apm.yml**: 既に存在する場合は本スキルを発火させず、`apm-install` 等の別スキルへ案内する。
3. **対象 target の確認**: ユーザーに採用するエージェント (Claude Code / Copilot / Codex / Cursor / Gemini / OpenCode) を確認する。

## 手順

### 1. apm.yml の生成

非対話的にひな形を作成する。

```bash
cd <project-root>
apm init --yes
```

生成される apm.yml の典型例:

```yaml
name: <project-name>
version: 1.0.0
description: APM project for <project-name>
author: <user>
dependencies:
  apm: []
  mcp: []
includes: auto
scripts: {}
```

`name` / `description` はリポジトリの実態に合わせて手動で書き換えることを案内する (例: 既存 `pyproject.toml` の name と揃えるなど)。

### 2. 対象 target の有効化

`--target` を毎回付けるより、`apm runtime setup` で永続有効化する方が idiomatic。

```bash
apm runtime setup claude
```

複数 target を併用する場合は順次実行する。

```bash
apm runtime setup claude
apm runtime setup copilot
apm runtime setup codex
```

設定状態は `apm runtime status` で確認できる。

### 3. .gitignore の整備

APM が `apm_modules/` を自動追加するが、念のため確認する。

```bash
grep -q '^apm_modules/' .gitignore || echo 'apm_modules/' >> .gitignore
```

### 4. 初回 install (任意の依存追加)

依存パッケージを最初から決めている場合はそのまま `apm-install` に委譲する。

## 注意点

- **`apm init --yes` の上書き挙動**: 既存 apm.yml がある状態で実行すると上書きされる可能性があるため、事前に存在確認を必ず行う。
- **runtime setup と `--target` の関係**: `apm runtime setup claude` 済みの環境では `apm install <pkg>` だけで Claude 用配置になる。明示的に他 target に投げたい場合のみ `--target` を使う。
- **`includes: auto` の意味**: プロジェクト内の primitive (`.apm/` 配下の skills/agents/etc.) を自動取り込みする設定。明示的に絞りたい場合は `includes` を配列で書き直す。
- **複数 target の落とし穴**: 同一プロジェクトで Claude と Copilot 両方を有効化すると `.claude/skills/` と `.github/skills/` の両方に配置される。CI / git 管理の観点から、本当に必要な target だけ有効化することを推奨する。

## 関連スキル

- **続き**: `apm-install` (依存パッケージ追加)
- **削除**: `apm-uninstall` (誤って入れた依存の除去)
- **検証**: `apm-audit` (security チェックの初期セットアップ)
