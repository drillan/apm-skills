# apm-install

APM パッケージを既存プロジェクトに追加するスキルです。target (配置先エージェント)、scope (project / user)、version pin の 3 軸の判断を支援します。

## 発火条件

- ユーザーが「APM パッケージを入れたい」「skill を追加して」「`apm install`」と発話
- `apm.yml` が存在するプロジェクトで依存追加を求められた場合

## 主要コマンド

```bash
# 基本
apm install <owner/repo> --target claude

# ユーザースコープ
apm install <owner/repo> --target claude -g

# Version pin (production 推奨)
apm install <owner/repo>#v1.0.0 --target claude

# 副作用ゼロのプレビュー
apm install <owner/repo> --target claude --dry-run -v
```

## target の選択肢

| target | 配置先 | 備考 |
| --- | --- | --- |
| `claude` | `.claude/skills/` | Claude Code |
| `copilot` | `.github/skills/` | GitHub Copilot (デフォルト) |
| `codex` | `.codex/` | Codex (`-g` 非対応) |
| `cursor` | `.cursor/` | Cursor (一部プリミティブ非対応 in `-g`) |
| `gemini` | `.gemini/` | Gemini CLI |
| `opencode` | `.opencode/` | OpenCode |

複数指定は `claude,copilot` のように comma 区切り、全部対応なら `all`。

## 注意点

- `gh skill install` で配置済のスキルは frontmatter の `metadata:` ブロックで識別され、APM は上書きを拒否します (`1 file skipped` 警告)。`--force` で上書きするか、`rm -rf` で削除してから再 install します。
- Codex は user scope (`-g`) 非対応 — project scope のみ利用可。
- `--target` 省略時は `.github/skills/` (Copilot) に配置されます。Claude Code で使うなら明示が必須です。

## 完全な仕様

[`skills/apm-install/SKILL.md`](https://github.com/drillan/apm-skills/blob/main/skills/apm-install/SKILL.md)
