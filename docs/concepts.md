# APM の概念リファレンス

APM の運用で頻出する用語と、実機検証で見えた挙動をまとめます。各スキルページとの棲み分けは「ここで概念を定義し、各スキルで操作手順を示す」ものです。

## 用語一覧

| 用語 | 一行説明 | 詳細 |
| --- | --- | --- |
| target | スキルが配置されるエージェント | [target](#target) |
| runtime | LLM 実行環境 (target とは別概念) | [runtime](#runtime) |
| scope | 配置範囲 (project / user) | [scope](#scope) |
| manifest | 依存宣言ファイル `apm.yml` | [manifest](#manifest) |
| lockfile | 解決済バージョンの固定 `apm.lock.yaml` | [lockfile](#lockfile) |
| deployed_files | lockfile に記録された配置パス | [deployed_files](#deployed-files) |
| version pin | 依存バージョンの固定 (`#v1.0.0` など) | [version pin](#version-pin) |
| collision | 既存ファイルとの配置衝突 | [collision](#collision) |
| policy | 組織ポリシー (`apm-policy.yml`) | [policy](#policy) |
| hidden Unicode | 不可視文字攻撃 (Glassworm 系) | [hidden Unicode](#hidden-unicode) |

(target)=
## target (配置先エージェント)

`--target` は、スキルがファイルとして配置されるエージェントを指定するオプションです。target ごとに配置先ディレクトリが決まっています。

| `--target` | 配置先 | エージェント |
| --- | --- | --- |
| `claude` | `.claude/skills/` | Claude Code |
| `copilot` | `.github/skills/` | GitHub Copilot |
| `codex` | `.codex/` | Codex |
| `cursor` | `.cursor/` | Cursor |
| `gemini` | `.gemini/` | Gemini CLI |
| `opencode` | `.opencode/` | OpenCode |

複数指定は `claude,copilot` のように comma 区切り、全対応なら `all` を指定します。

`--target` を省略するとデフォルトで `.github/skills/` (Copilot 用) に配置されます。Claude Code で使うなら明示しないと意図しない場所に行く点に注意してください。

(runtime)=
## runtime (LLM 実行環境)

`apm runtime` は LLM 実行環境を扱うサブコマンドで、target とは別概念です。

| 概念 | 意味 |
| --- | --- |
| target | スキルファイルがどこに配置されるか |
| runtime | どの LLM (Copilot / Codex / Gemini など) で実行するか |

代表的なコマンドは次のとおりです。

```bash
apm runtime status               # 現在の active runtime と preference order を表示
apm runtime setup claude         # Claude Code 用の永続的な配置を有効化
```

`apm runtime setup claude` を一度実行すると、以降の `apm install <pkg>` で `--target` を毎回指定しなくても Claude Code 用の配置先に展開されます。`--target` は「その場限りの上書き」、`apm runtime setup` は「設定の永続化」という関係です。

(scope)=
## scope (project / user)

スキルを置く範囲は project scope と user scope の 2 種類です。

| scope | manifest | 配置先 (claude の例) | 用途 |
| --- | --- | --- | --- |
| project (デフォルト) | `./apm.yml` | `./.claude/skills/` | プロジェクト固有の依存 |
| user (`-g`) | `~/.apm/apm.yml` | `~/.claude/skills/` | 複数プロジェクトで使い回す依存 |

target ごとの user-scope サポートは差があります (`apm install --help` の出力より):

- 完全対応: `claude` / `gemini` / `copilot-cowork`
- 部分対応: `copilot` / `cursor` / `opencode` (一部のプリミティブ非対応)
- 非対応: `codex` (project scope のみ利用可)

(manifest)=
## manifest (apm.yml)

`apm.yml` は依存宣言とパッケージメタデータを兼ねるマニフェストファイルです。

```yaml
name: my-project
version: 1.0.0
description: APM project for my-project
author: driller
dependencies:
  apm:
    - drillan/apm-skills#v0.1.1
  mcp: []
includes: auto
scripts: {}
```

`apm install <pkg>` を引数付きで実行すると、`apm.yml` が無ければ自動生成され、`dependencies.apm` に依存が追記されます。引数なしで `apm install` を打てば既存 `apm.yml` の内容を再現します。

publisher 側 (= スキルを配布するリポジトリ) でも同じ `apm.yml` を置きますが、こちらは「自分自身が何という名前のパッケージか」を宣言するメタデータが主目的です。

(lockfile)=
## lockfile (apm.lock.yaml)

`apm.lock.yaml` は解決済の commit SHA と content hash を記録するロックファイルです。

```yaml
lockfile_version: '1'
generated_at: '2026-04-29T05:54:33.191118+00:00'
apm_version: 0.10.0
dependencies:
  - repo_url: drillan/apm-skills
    host: github.com
    resolved_commit: 27edc8f0e6a182754a2e0f8fcbe5f45e52f9c187
    package_type: skill_bundle
    deployed_files:
      - .claude/skills/myst-authoring
      - .claude/skills/rst-to-myst
      ...
    content_hash: sha256:2a18ba31cbf859c01...
```

役割は npm の `package-lock.json` や Cargo の `Cargo.lock` と同等です。**この lockfile を必ず commit してください**。pin 無しの依存も lockfile が同じ SHA を保持していれば再現可能です。

(deployed-files)=
## deployed_files

lockfile に記録された配置パスです。`apm uninstall` 実行時に同期削除する根拠として使われます。`gh CLI` 経由でファイルを直接置いた場合 (frontmatter に `metadata:` ブロックがある) は APM の管理下に無いため、`deployed_files` には含まれず、APM はそのファイルを上書きしません。

(version-pin)=
## version pin

依存バージョンを固定する構文です。区切り文字は `#` (URL fragment / git ref) を使います。npm 風の `@` は使えません。

| 形式 | 意味 |
| --- | --- |
| `owner/repo` | デフォルトブランチ HEAD を取得 (pin 無し) |
| `owner/repo#v1.0.0` | git tag を指定 |
| `owner/repo#abc1234` | 短縮 SHA を指定 |
| `owner/repo#abcdef01234567...` | 40 桁 SHA を指定 |

### pin の効果

- `apm update` の挙動: pin 済の依存はスキップされる (drift 防止)
- 再現性: lockfile が同じなら pin の有無を問わず再現可能
- セキュリティ: tag pin は mutable tag (同名 tag への commit 付け替え) の影響を受けるが、SHA pin はその影響も排除

production / CI では tag または SHA で pin することを推奨します。個人の開発環境で「自分が publisher を兼ねる」ケースでは pin 無しの方が便利な場合もあります。

(collision)=
## collision (skill 配置の衝突)

APM が把握していないファイルが配置先にある場合、APM は上書きを拒否し警告を出します。

```text
1 file skipped -- local files exist, not managed by APM
```

識別の仕組みは frontmatter の `metadata:` ブロックの有無です。`gh skill install` で配置されたスキルはこのブロックを持つため、APM は「自分のものではない」と判断します。

対処法は 2 つです。

- `--force` で上書き (APM 管理下に統一)
- 該当スキルを `rm -rf <skill_dir>` で削除してから再 install

(policy)=
## policy (apm-policy.yml)

組織レベルの allowlist / denylist を宣言するポリシーファイルです。例:

```yaml
policy_version: '1'
allowlist:
  hosts:
    - github.com
denylist:
  patterns:
    - '*/evil-*/**'
    - '*/test-fake-*/**'
fetch_failure_default: block
```

`fetch_failure_default` のデフォルトは `pass` (fail-open) で、ポリシー取得失敗時は警告のみで通過します。安全運用には `block` (fail-closed) を明示してください。

継承順は **Enterprise → Org → Repo** で、子側は親より厳しくしかできません (緩めることは不可)。

`apm audit --ci --policy org` で CI に組み込め、SARIF 出力 (`-f sarif`) で GitHub Code Scanning にも連携できます。

(hidden-unicode)=
## hidden Unicode (Glassworm 対策)

APM は **File presence IS execution** の哲学を採っており、配置されたファイルは LLM に直接読み込まれます。ゼロ幅文字や RTL 文字を仕込んだ Glassworm 系の攻撃を防ぐため、APM は次の 2 段階で hidden Unicode をスキャンします。

| タイミング | コマンド | 用途 |
| --- | --- | --- |
| install 時 | (自動) | consumer 側の事故を install 時点で検出 |
| publisher 側 | `apm audit --file <SKILL.md>` | 自分が公開するファイルを事前検証 |

publisher が `apm audit --file` を CI に組み込んでおくと、PR 単位で混入を検出して merge を止められます。混入が見つかった場合は `apm audit --strip` で除去できます。

(pitfalls)=
## 共通の落とし穴 (実例)

本ドキュメント執筆時に実際に踏んだ落とし穴を記録しておきます。

### 1. pin 構文は `@` ではなく `#`

npm の習慣で `@v1.0.0` と書くと失敗します。

```text
[x] drillan/apm-skills@v0.1.0 -- Invalid repository path component: apm-skills@v0.1.0
```

正しくは `#v1.0.0` です。`#` は URL fragment / git ref の区切り記号で、microsoft/apm-sample-package の `apm.yml` も同じ構文を採っています。

### 2. gh CLI 経由のスキルとの collision

`gh skill install` で先にスキルを置いていた状態で `apm install` を流すと、collision 警告で skip されます。識別キーは frontmatter の `metadata:` ブロックです。

```yaml
metadata:
    github-path: skills/sphinx-init
    github-ref: refs/heads/main
    github-repo: https://github.com/drillan/sphinx-skills
    github-tree-sha: ...
```

これがあるファイルを APM は触りません。APM 管理に統一する場合は `--force` か `rm -rf` で対処します。

### 3. Codex の user-scope は非対応

`apm install --target codex -g` を実行すると以下の警告が出ます。

```text
Targets without native user-level support: codex
```

Codex は project scope (`-g` 無し) のみ利用可能です。複数プロジェクトで使い回したい場合は、各プロジェクトの `apm.yml` で個別に依存宣言してください。

### 4. dry-run でも `apm.yml` は生成される

`apm install --dry-run` の最後には "no changes made" と表示されますが、`~/.apm/apm.yml` (user scope の場合) や `./apm.yml` (project scope の場合) の作成は走ります。

完全な副作用ゼロを期待するなら、別ディレクトリでテストするか、テスト後に `rm -rf` で消すのが確実です。

### 5. `--target` 省略時のデフォルトは Copilot

`--target` 無しで実行すると `.github/skills/` (Copilot 用) に配置されます。Claude Code 用なら明示が必要です。

事前に `apm runtime setup claude` を 1 度実行しておけば、以降の `apm install` で `--target` 省略時も Claude Code 用に展開されます。

### 6. content hash mismatch 警告

同じ pin (`#v0.1.0` など) で content が変わっていた場合、APM は supply-chain attack を疑って install を拒否します。

```text
Content hash mismatch for drillan/apm-skills: expected sha256:..., got sha256:...
```

意図的な変更 (例: tag を別 commit に付け替えた、v0.1.1 をリリースして pin を切り替えた) なら `apm install --update` で承諾できます。見覚えがない場合は本物の改竄を疑ってください。

## 比較表

### project vs user scope

| 観点 | project (デフォルト) | user (`-g`) |
| --- | --- | --- |
| 配置範囲 | プロジェクト直下 | ユーザーホーム |
| manifest | `./apm.yml` | `~/.apm/apm.yml` |
| 配置先 (claude) | `./.claude/skills/` | `~/.claude/skills/` |
| target 対応 | 全 target | claude / gemini / copilot-cowork は完全、codex は非対応 |
| 用途 | プロジェクト固有の依存 | 複数プロジェクトで使い回す依存 |
| アンインストール | `apm uninstall <pkg>` | `apm uninstall <pkg> -g` |

### pin あり vs pin なし

| 観点 | pin あり (`#v0.1.0`) | pin なし |
| --- | --- | --- |
| 再現性 | 常に同一 commit | lockfile が同じなら再現 |
| `apm update` の挙動 | スキップ (drift 防止) | 最新追従 (drift 発生可能性) |
| 新機能 / bug fix の取得 | 手動でバンプが必要 | 自動 (update 走らせれば) |
| 意図しない breaking change | 起こらない | 起こり得る (lockfile 更新時) |
| mutable tag 攻撃への耐性 | tag pin → 影響あり、SHA pin → なし | HEAD 改竄に脆弱 |
| メンテナンスコスト | 高 (バンプの判断と PR) | 低 (放置でも動く) |
| 推奨用途 | production / CI / 安定運用 | 個人の試行錯誤 / 自分が publisher 兼任 |
