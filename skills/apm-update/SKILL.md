---
name: apm-update
description: APM 依存を最新版に更新するスキル。`apm outdated` で更新可能なパッケージを確認し、`apm update` で apm.lock.yaml を再生成する。pin 済 (`@v1.0.0` 等) のものはスキップされる仕様。「依存を更新したい」「outdated 確認」「apm update」「最新化」といった依頼に発火する。更新後は `apm-audit` での security 検証を案内する。
license: MIT
allowed-tools: Bash, Read
---

## 概要

`apm.yml` で宣言した APM 依存を最新版に追従するスキル。`apm.lock.yaml` の `resolved_commit` を更新し、deployed files (`.claude/skills/` 等) を最新コンテンツで再配置する。

## 前提検出

1. **apm.yml 存在**: 無い場合は更新対象が無いので発火しない。
2. **APM CLI 有無**: `apm --version` で確認。
3. **clean working tree**: 更新は git 履歴に乗るため、コミット前の変更は事前に整理することを案内する (必須ではない)。

## 手順

### 1. 更新可能性の確認

```bash
cd <project-root>
apm outdated
```

各依存について「現在の resolved_commit」と「remote 最新」が比較表示される。pin (`@v1.0.0`、`@<sha>`) されたものはスキップされる旨が明示される。

### 2. 依存の更新

```bash
apm update
```

実行すると以下が更新される:

- `apm.lock.yaml` の `resolved_commit` (40 桁 SHA)
- `apm.lock.yaml` の `content_hash`
- `deployed_files` で示されるディレクトリ内のスキル本体

`apm.yml` は変更されない (pin 値はそのまま、unpin の依存だけ最新追従)。

### 3. 個別 / 全体の指定

特定パッケージのみ更新する場合は `apm update --help` で対応オプションを確認する (バージョンによっては個別指定がサポートされる)。サポートが無い場合は該当パッケージを `apm uninstall` → `apm install` の手順で更新する代替案を提示する。

### 4. 更新後の検証

更新内容に security 問題が無いか確認する。

```bash
apm audit --ci
```

検出された場合は `apm-audit` スキルに委譲する。

### 5. git commit

```bash
git add apm.lock.yaml .claude/skills/   # target に応じて配置先を調整
git commit -m "chore: update APM dependencies"
```

## 注意点

- **pin の効果**: `@v1.0.0` や `@<sha>` で pin した依存は `apm update` で**更新されない**。これは仕様で、production / CI 用途の drift 防止が意図。意図的にバージョン変更したい場合は `apm.yml` の pin 表記を書き換えてから `apm install` する。
- **transitive 依存**: 親パッケージが pin されていても、その親が依存する transitive な package は更新される可能性がある。`apm.lock.yaml` の差分で確認する。
- **deployed files の競合**: ローカルでスキル md を手動編集していた場合、`apm update` が上書きする可能性がある。事前に `--dry-run` (もし対応していれば) や git diff で確認する。
- **lockfile を commit する設計**: APM は npm 同様、`apm.lock.yaml` を commit 対象とする。これにより別の開発者が `apm install` (引数なし) で同一バージョンを再現できる。

## 関連スキル

- **検証**: `apm-audit` (更新後の security チェック)
- **削除**: `apm-uninstall` (不要になった依存の除去)
- **追加**: `apm-install` (新規依存の追加 / pin の書き換え)
