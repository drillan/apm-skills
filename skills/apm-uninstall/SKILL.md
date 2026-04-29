---
name: apm-uninstall
description: APM パッケージを削除するスキル。`apm uninstall <pkg>` で apm.yml/apm.lock.yaml の依存記述と integrated files (`.claude/skills/` や `.github/skills/` 等) を同期削除する。実行前に必ず `--dry-run` で削除対象を確認する。「APM パッケージを削除」「apm uninstall」「skill を消したい」といった依頼に発火する。`apm prune` で apm.yml に無いパッケージを一括削除する用途にも対応する。
license: MIT
allowed-tools: Bash, Read
---

## 概要

APM 依存を整合的に削除するスキル。`apm.yml` の依存記述、`apm.lock.yaml` のエントリ、deployed files の 3 者を同期して除去する。

## 前提検出

1. **apm.yml 存在**: 無い場合は削除対象が無いので発火しない。
2. **APM CLI 有無**: `apm --version` で確認。
3. **削除対象の特定**: ユーザー指示が曖昧な場合は `apm.yml` の `dependencies.apm` 一覧を読み上げて確認する。

## 手順

### 1. Dry-run で削除対象を確認 (必須)

```bash
cd <project-root>
apm uninstall <owner/repo> --dry-run
```

削除されるファイルとマニフェストエントリが事前表示される。意図しないものが含まれていないかを確認する。

### 2. 実削除

```bash
apm uninstall <owner/repo>
```

実行すると以下が同期更新される:

- `apm.yml` の `dependencies.apm` から該当エントリ削除
- `apm.lock.yaml` の対応エントリ削除
- `deployed_files` で記録されたディレクトリ削除 (`.claude/skills/<name>/` 等)

### 3. ユーザースコープからの削除

```bash
apm uninstall <owner/repo> -g
```

`~/.apm/apm.yml` と `~/.claude/skills/<name>/` 等を同期削除する。

### 4. 一括クリーンアップ (任意)

`apm.yml` に列挙されていないパッケージを `apm_modules/` や deployed files から一掃する。

```bash
apm prune --dry-run    # 削除対象を事前確認
apm prune              # 実行
```

これは「他のブランチで `apm install` した残骸を整理する」「lockfile から外れたパッケージを掃除する」用途。

### 5. git commit

```bash
git add apm.yml apm.lock.yaml .claude/skills/   # 削除分も stage
git commit -m "chore: remove APM dependency <owner/repo>"
```

## 注意点

- **`gh skill install` 由来のスキル**: APM が管理していない (frontmatter に `metadata:` ブロックを持つ) スキルは `apm uninstall` で削除されない。手動で `rm -rf <skill_dir>` する必要がある。
- **`--dry-run` を省略しない**: 削除は破壊的操作なので、CI 等の自動化フロー以外では `--dry-run` で確認してから実行する。
- **transitive 依存の扱い**: パッケージ A が依存していた transitive package が他で使われていない場合、`apm prune` で同時削除候補になる。意図せず必要な依存が消えないよう、`prune` の前は `--dry-run` を必ず通す。
- **`apm_modules/` の残留**: deployed files は削除されるが、`apm_modules/` 配下のキャッシュは残る場合がある。気になる場合は `rm -rf apm_modules` で再ダウンロードを強制できる (次回 `apm install` で復元される)。

## 関連スキル

- **再追加**: `apm-install` (削除後の再導入)
- **検証**: `apm-audit` (削除後の整合性確認)
- **更新**: `apm-update` (削除後に残った依存の最新化)
