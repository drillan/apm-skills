# apm-init

新規プロジェクトに APM を導入するスキルです。`apm.yml` のひな形作成と、対象エージェントの runtime setup までを 1 ストロークで完了させます。

## 発火条件

- ユーザーが「APM プロジェクトを始めたい」「`apm.yml` を作成」「APM をセットアップ」と発話
- プロジェクトに `apm.yml` が存在しない状態で APM 関連の操作を求められた場合

## 主要コマンド

```bash
# apm.yml の生成
apm init --yes

# 対象エージェントの永続有効化
apm runtime setup claude
apm runtime setup copilot   # 複数併用も可

# 設定状態の確認
apm runtime status
```

## 生成される apm.yml

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

`name` / `description` は実態に合わせて手動で書き換えることを案内します (例: `pyproject.toml` の name と揃える)。

## runtime setup と --target の関係

`apm runtime setup claude` 済みの環境では `apm install <pkg>` だけで Claude 用配置になります。明示的に他 target に投げたい場合のみ `--target` を使います。`--target` は **その場限りの上書き**、`runtime setup` は **設定の永続化** という関係です。

## 注意点

- 既存 `apm.yml` がある状態で `apm init --yes` を実行すると上書きされる可能性があるため、事前に存在確認を必ず行います。
- 同一プロジェクトで Claude と Copilot 両方を有効化すると `.claude/skills/` と `.github/skills/` の両方に配置されます。git 管理の観点から、本当に必要な target だけ有効化することを推奨します。

## 完全な仕様

[`skills/apm-init/SKILL.md`](https://github.com/drillan/apm-skills/blob/main/skills/apm-init/SKILL.md)
