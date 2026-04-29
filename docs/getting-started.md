# APM (Agent Package Manager) を始める

APM は Microsoft が開発している、AI エージェント設定 (skills / agents / instructions / hooks など) のパッケージマネージャです。npm の `package.json` や Cargo の `Cargo.toml` の AI エージェント版と捉えると、機能や狙いがイメージしやすくなります。

このページでは概要から最初のインストールまでを案内します。用語の正確な定義は [概念リファレンス](concepts) を、各操作の詳細は各 [スキルページ](skills/index) を参照してください。

## APM とは何か

### 解決する問題

AI コーディングエージェント (Claude Code、GitHub Copilot、Cursor、Codex、Gemini CLI、OpenCode など) を業務で使うとき、以下のような課題があります。

- 配置場所がエージェント別 (`.claude/skills/`、`.github/skills/`、`.cursor/`、`.codex/` など)
- 依存の調達が手作業 (skill ごとに `gh skill install` を 1 行ずつ、Wiki に手順を書いて新規メンバーに渡す運用)
- 再現性が無い (あるエージェントで動いた構成を、他のマシンで再現する手段が標準化されていない)
- バージョン追跡なし (ある時点の skill バンドルを固定する仕組みが無い)
- セキュリティ検証なし (不可視 Unicode を使った Glassworm 攻撃やポリシー違反の検出機構が無い)

### 提供するもの

これらを `apm.yml` という 1 ファイルに集約し、`apm install` の 1 コマンドで再現します。

- マニフェスト宣言 — 依存パッケージを `apm.yml` に列挙
- ロックファイル — `apm.lock.yaml` が解決済の commit SHA を 40 桁で固定
- マルチエージェント — `--target` で配置先を切替 (claude / copilot / codex / cursor / gemini / opencode)
- セキュリティ — install 時に hidden Unicode をスキャン、`apm-policy.yml` で組織ポリシーを強制
- ガバナンス — Enterprise → Org → Repo の 3 階層継承、CI 統合 (SARIF / Code Scanning)

## CLI 導入

### Linux / macOS

```bash
curl -sSL https://aka.ms/apm-unix | sh
```

### Windows (PowerShell)

```powershell
irm https://aka.ms/apm-windows | iex
```

### 動作確認

```bash
apm --version
```

その他の導入方法 (Homebrew / pip / Scoop) は [APM 公式 README](https://github.com/microsoft/apm) を参照してください。

## 最初の install (1 コマンド)

例として、本プロジェクト `drillan/apm-skills` を Claude Code 用に導入します。

```bash
cd your-project
apm install drillan/apm-skills --target claude
```

これで以下が自動的に行われます。

- `apm.yml` の生成 (依存に `drillan/apm-skills` が記録される)
- `apm.lock.yaml` の生成 (40 桁 SHA + content_hash で pin)
- `.gitignore` への `apm_modules/` 追記
- 5 つのスキルが `.claude/skills/` 配下に配置

`--target` の選択肢は次のとおりです。

| `--target` | 配置先 | 備考 |
| --- | --- | --- |
| `claude` | `.claude/skills/` | Claude Code |
| `copilot` | `.github/skills/` | GitHub Copilot (`--target` 省略時のデフォルト) |
| `codex` | `.codex/` | Codex |
| `cursor` | `.cursor/` | Cursor |
| `gemini` | `.gemini/` | Gemini CLI |
| `opencode` | `.opencode/` | OpenCode |

複数同時に展開するなら `--target claude,copilot` のように comma 区切り、すべて対応するなら `--target all` を使います。

複数プロジェクトで使い回す場合は `-g` を追加し、ユーザースコープにインストールします。

```bash
apm install drillan/apm-skills --target claude -g
```

## apm.yml の最小例

```yaml
name: your-project
version: 1.0.0
dependencies:
  apm:
    - drillan/apm-skills
    - drillan/apm-skills#v0.1.0      # version pin (production / CI で推奨)
  mcp:
    - io.github.github/github-mcp-server
includes: auto
```

`name` / `version` / `description` などのメタデータは `apm init --yes` で自動生成されますが、実プロジェクトに合わせて手で書き換えるのが普通です。

`dependencies.apm` の各要素には次の形式が選べます。

- `owner/repo` — デフォルトブランチ HEAD を取得 (drift する可能性あり)
- `owner/repo#v0.1.0` — git tag に固定
- `owner/repo#abc1234` — 短縮 SHA に固定
- `owner/repo#abcdef0123456789...` — 40 桁 SHA に固定 (mutable tag への耐性が最も強い)

version pin の挙動の詳細は [概念リファレンスの version pin](concepts.md#version-pin) を参照してください。

## 基本ワークフロー

APM パッケージのライフサイクルは 5 段階で構成されます。

| ステップ | 用途 | 主要コマンド | 担当スキル |
| --- | --- | --- | --- |
| 初期化 | 新規 APM プロジェクトの開始 | `apm init --yes`、`apm runtime setup <target>` | [`apm-init`](skills/apm-init) |
| 追加 | 依存パッケージのインストール | `apm install <owner/repo> --target <name>` | [`apm-install`](skills/apm-install) |
| 更新 | 依存の最新化 | `apm outdated`、`apm update` | [`apm-update`](skills/apm-update) |
| 監査 | セキュリティ検証 | `apm audit --ci` | [`apm-audit`](skills/apm-audit) |
| 削除 | 依存の除去 | `apm uninstall <owner/repo>` | [`apm-uninstall`](skills/apm-uninstall) |

各段階で何が変わるかを以下にまとめます。

| ファイル | init | install | update | uninstall |
| --- | --- | --- | --- | --- |
| `apm.yml` | 生成 | 依存追加 | 変更なし | 依存削除 |
| `apm.lock.yaml` | 空で生成 | エントリ追加 + commit SHA 解決 | resolved_commit 更新 (pin 無し時) | エントリ削除 |
| `.<agent>/skills/<name>/` | 変更なし | 配置 | 内容更新 | 削除 |
| `.gitignore` | 変更なし | `apm_modules/` 自動追記 | 変更なし | 変更なし |

## 次に読むもの

- 用語の正確な定義 (target / runtime / scope / pin / policy など): [APM の概念リファレンス](concepts)
- 共通の落とし穴 (本ドキュメント執筆時に遭遇した実例): {ref}`概念リファレンス内の落とし穴セクション <pitfalls>`
- 各操作の詳細手順: [スキル一覧](skills/index)
- 公式ドキュメント (英語): <https://microsoft.github.io/apm/>
