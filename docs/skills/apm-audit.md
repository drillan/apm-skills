# apm-audit

APM パッケージのセキュリティ検証を行うスキルです。**File presence IS execution** (配置されたファイルが LLM に即読み込まれる) という APM の前提のもと、不可視 Unicode 攻撃検出と組織ポリシー違反検知の 2 系統をカバーします。

## 発火条件

- ユーザーが「security チェック」「policy 違反確認」「`apm audit`」「CI に統合」と発話
- 新規 PR で APM 依存追加・更新があった場面

## 主要コマンド

```bash
# ローカル audit (依存追加直後)
apm audit --ci

# Policy ベースの監査 (ポリシー含む)
GITHUB_TOKEN=$(gh auth token) apm audit --ci --policy org --no-cache

# SARIF 出力 (Code Scanning 連携)
apm audit --ci --policy org --no-cache -f sarif -o policy-report.sarif
```

## 検査内容

| 検査 | 説明 |
| --- | --- |
| hidden Unicode | ゼロ幅文字 / RTL 文字 / Glassworm 系の不可視シーケンス検出 |
| lockfile 整合性 | `apm.lock.yaml` の `content_hash` と実体の SHA-256 突合 |
| 構造検証 | フォーマット異常やマニフェスト不整合 |
| Policy 違反 | `apm-policy.yml` の `denylist` / `allowlist` ルール違反 |

## apm-policy.yml の最小例

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

`fetch_failure_default` のデフォルトは `pass` (fail-open) ですが、安全運用なら `block` (fail-closed) を明示します。継承順は **Enterprise → Org → Repo** で、子側は厳しくしかできません (緩めることは不可)。

## GitHub Actions 統合

```yaml
- name: Baseline checks
  run: apm audit --ci

- name: Policy checks
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: apm audit --ci --policy org --no-cache -f sarif -o policy-report.sarif

- uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: policy-report.sarif
```

リポジトリ設定 → Rules → Rulesets で `APM Audit / audit` を **Require status checks to pass** に追加すると、policy 違反時の merge を物理的にブロックできます。

## 注意点

- `--no-policy` は `apm install` 時のエスケープハッチですが、`apm audit --ci` は迂回できません。CI で確実に止まる設計です。
- `--no-cache` はポリシー検査でキャッシュを使わず常に最新を取得する設定です。CI では必ず付けます。
- ローカル install 時にも基本的なチェック (Unicode + 構造) は走るので、CI と二重化されます。

## 完全な仕様

[`skills/apm-audit/SKILL.md`](https://github.com/drillan/apm-skills/blob/main/skills/apm-audit/SKILL.md)
