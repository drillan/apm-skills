# apm-update

`apm.yml` で宣言した APM 依存を最新版に追従するスキルです。`apm.lock.yaml` の `resolved_commit` を更新し、deployed files を最新コンテンツで再配置します。

## 発火条件

- ユーザーが「依存を更新したい」「最新化」「outdated 確認」「`apm update`」と発話
- 既存 PR で `apm.yml` が更新されたとき (CI フック想定)

## 主要コマンド

```bash
# 更新可能性の確認
apm outdated

# 依存の更新 (lockfile 再生成)
apm update

# 更新後の検証 (apm-audit に委譲)
apm audit --ci
```

## 更新で変わるもの・変わらないもの

| 対象 | 挙動 |
| --- | --- |
| `apm.lock.yaml` の `resolved_commit` (40 桁 SHA) | 更新される |
| `apm.lock.yaml` の `content_hash` | 更新される |
| `deployed_files` の中身 (`.claude/skills/` 等) | 更新される |
| `apm.yml` の依存記述 | 変更されない |
| `#v1.0.0` / `#<sha>` で pin した依存 | スキップされる (drift 防止) |

## pin の意味

`#v1.0.0` で pin した依存は `apm update` で**更新されない**仕様です。これは production / CI 用途で意図しない drift を防ぐためです。意図的にバージョン変更したい場合は `apm.yml` の pin 表記を書き換えてから `apm install` します。

## 注意点

- transitive 依存は親パッケージが pin されていても更新される可能性があります。`apm.lock.yaml` の差分で確認します。
- ローカルでスキル md を手動編集していた場合、`apm update` が上書きする可能性があります。事前に git diff で確認します。
- 更新後は `apm.lock.yaml` を必ず commit します (npm 同様、reproducibility の鍵となります)。

## 完全な仕様

[`skills/apm-update/SKILL.md`](https://github.com/drillan/apm-skills/blob/main/skills/apm-update/SKILL.md)
