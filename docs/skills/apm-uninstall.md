# apm-uninstall

APM 依存を整合的に削除するスキルです。`apm.yml` の依存記述、`apm.lock.yaml` のエントリ、deployed files の 3 者を同期して除去します。

## 発火条件

- ユーザーが「APM パッケージを削除」「`apm uninstall`」「skill を消したい」と発話
- 不要になった依存を整理する文脈

## 主要コマンド

```bash
# 必須: 削除対象の事前確認
apm uninstall <owner/repo> --dry-run

# 実削除
apm uninstall <owner/repo>

# user scope からの削除
apm uninstall <owner/repo> -g

# apm.yml に無いパッケージを一括削除
apm prune --dry-run
apm prune
```

## 削除されるもの

`apm uninstall <owner/repo>` で同期更新されるもの:

- `apm.yml` の `dependencies.apm` から該当エントリ
- `apm.lock.yaml` の対応エントリ
- `deployed_files` で記録されたディレクトリ (`.claude/skills/<name>/` 等)

## 注意点

- `gh skill install` で配置済 (frontmatter に `metadata:` を持つ) スキルは APM が管理していないため `apm uninstall` で削除されません。手動で `rm -rf <skill_dir>` する必要があります。
- 削除は破壊的操作です。CI 等の自動化フロー以外では `--dry-run` で確認してから実行します。
- transitive 依存が他パッケージで使われていない場合、`apm prune` で同時削除候補になります。意図せず必要な依存が消えないよう、必ず `--dry-run` を通します。

## 完全な仕様

[`skills/apm-uninstall/SKILL.md`](https://github.com/drillan/apm-skills/blob/main/skills/apm-uninstall/SKILL.md)
