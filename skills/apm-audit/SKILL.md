---
name: apm-audit
description: APM パッケージの security 監査を行うスキル。`apm audit --ci` で hidden Unicode (Glassworm 攻撃) 検出と `apm-policy.yml` の denylist 違反を検査する。GitHub Actions に組み込む場合は SARIF 出力で Code Scanning 連携可能。「security チェック」「policy 違反確認」「apm audit」「CI に統合」といった依頼や、新規 PR で APM 依存追加・更新があった場面で発火する。`apm-policy.yml` の作成支援も含む。
license: MIT
allowed-tools: Bash, Read, Write, Edit
---

## 概要

APM パッケージのセキュリティ検証を行うスキル。**File presence IS execution** (配置されたファイルが LLM に即読み込まれる) という APM の前提のもと、不可視 Unicode 攻撃検出と組織ポリシー違反検知の 2 系統をカバーする。

## 前提検出

1. **APM CLI 有無**: `apm --version`。
2. **apm.yml 存在**: 監査対象が無いとエラー伝播。
3. **`apm-policy.yml` の有無**: 既存ならそれに従って検査、無ければ新規作成を提案。

## 手順

### 1. ローカル audit (依存追加直後)

```bash
cd <project-root>
apm audit --ci
```

検査内容:

- **hidden Unicode**: ゼロ幅文字 / RTL 文字 / Glassworm 系の不可視シーケンス検出
- **lockfile 整合性**: `apm.lock.yaml` の `content_hash` と実体の SHA-256 突合
- **基本的な構造検証**: フォーマット異常やマニフェスト不整合

違反検出時は exit 1 で停止。

### 2. Policy ベースの検査

`apm-policy.yml` で許可・禁止のパッケージ規則を宣言する。

例: `apm-policy.yml` の最小構成

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

ポリシー込みで監査:

```bash
GITHUB_TOKEN=$(gh auth token) apm audit --ci --policy org --no-cache
```

`--policy org` は組織レベルのポリシーをロードするオプション。継承順は **Enterprise → Org → Repo** で、子側は厳しくしかできない (緩めることは不可)。

### 3. SARIF 出力で Code Scanning 連携 (GitHub Actions)

`.github/workflows/apm-audit.yml` の例:

```yaml
name: APM Audit
on:
  pull_request:
  push:
    branches: [main]

jobs:
  audit:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@v4

      - name: Install APM
        run: curl -sSL https://aka.ms/apm-unix | sh

      - name: Baseline checks
        run: apm audit --ci

      - name: Policy checks (SARIF)
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: apm audit --ci --policy org --no-cache -f sarif -o policy-report.sarif

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: policy-report.sarif
```

これで PR 時に違反が GitHub の Code Scanning タブで可視化される。

### 4. Branch protection への組み込み

リポジトリ設定 → Rules → Rulesets で `APM Audit / audit` を **Require status checks to pass** に追加することで、policy 違反時の merge を物理的にブロックできる。

## 注意点

- **`fetch_failure_default`**: デフォルトは `pass` (fail-open) で、policy 取得失敗時は警告のみで通過する。安全運用なら `block` (fail-closed) を明示的に設定する。
- **`--no-cache` の意味**: ポリシー検査はキャッシュされた policy を使わず常に最新を取得する設定。CI では必ず付ける。
- **ローカル install 時の自動 audit**: `apm install` 実行時にも基本的なチェック (Unicode + 構造) は走るので、CI と二重化される設計。`apm audit --ci` はそれを CI で再現するためのコマンド。
- **`--no-policy` の用途**: 意図的にポリシーを無視するエスケープハッチ (例: 違反パッケージを CI 検証用に install する) だが、`apm audit --ci` 自体は迂回できない。Policy enforcement の観点では `apm install --no-policy` を使ってもダメで、CI で確実に止まる。

## 関連スキル

- **追加時の自動チェック**: `apm-install` (install 内部で基本 audit が走る)
- **更新後の確認**: `apm-update` (更新後に再 audit 推奨)
- **削除整合性**: `apm-uninstall` (削除後に lockfile 整合性を audit で確認)
