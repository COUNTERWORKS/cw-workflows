# cw-workflows

COUNTERWORKS 各リポジトリから呼び出して使う共有 **Reusable GitHub Actions Workflow** の集約リポジトリ。シークレット漏洩防止 (gitleaks) を皮切りに、Org 共通のセキュリティ・品質チェック系 Workflow を追加していく予定。

## 提供 Workflow

### `.github/workflows/gitleaks.yml`

[gitleaks](https://github.com/gitleaks/gitleaks) (OSS, MIT) でリポジトリ内のシークレット混入を検出する Reusable Workflow。

- gitleaks 公式 release tarball を **バイナリ直接 install** して実行
- 1 vCPU の **`ubuntu-slim`** ランナーで実行 (15 分上限の軽量ランナー)
- 公式 Docker image (`ghcr.io/gitleaks/gitleaks`) を `docker run` する方式も推奨されているが、`ubuntu-slim` には Docker daemon が無いためバイナリ install 方式を採用
- 公式 `gitleaks/gitleaks-action` は Organization 利用に商用ライセンスが必要なため、OSS バイナリを直接呼ぶ自前実装としている
- v8.19.0 以降のサブコマンド名変更に対応済み (`detect` → `git`)

#### Inputs

| name | type | default | 説明 |
| --- | --- | --- | --- |
| `gitleaks-version` | string | `latest` | gitleaks のリリースタグ。`latest` の場合は GitHub Releases API で最新を解決する。挙動を固定したい場合は `v8.30.1` のように pin |
| `config-path` | string | `""` | リポ内の `.gitleaks.toml` パス。空ならデフォルトルール |
| `fail-on-leak` | boolean | `true` | 検出時にジョブを失敗させるか |
| `fetch-depth` | number | `0` | `actions/checkout` の fetch-depth (0 = 全履歴) |
| `scan-mode` | string | `git` | `git` = 履歴スキャン / `dir` = ディレクトリスキャン |
| `additional-args` | string | `""` | `--log-opts="--all main..HEAD"` 等の追加引数 |

SARIF レポートは常に artifact `gitleaks-sarif` として 14 日保持される (検出なしの場合は空)。

## 使い方 (呼び出し側リポ)

各リポの `.github/workflows/gitleaks.yml` に [`examples/caller-workflow.yml`](./examples/caller-workflow.yml) をコピーして配置する。最小構成:

```yaml
name: gitleaks
on:
  pull_request:
  push:
    branches: [main]   # ← 各リポの default branch を 1 本だけ指定 (例: develop)
  schedule:
    - cron: "0 0 * * 1"
  workflow_dispatch:

jobs:
  scan:
    uses: COUNTERWORKS/cw-workflows/.github/workflows/gitleaks.yml@main
```

リポ固有の allowlist が必要なら、リポ直下に `.gitleaks.toml` (例: [`config/gitleaks-baseline.toml`](./config/gitleaks-baseline.toml) を起点に拡張) を配置する。gitleaks はリポルートの `.gitleaks.toml` を自動で読み込むため、caller workflow 側での明示指定は不要。サブディレクトリに置く場合のみ `config-path` を渡す。

## バージョニング方針

- **Reusable Workflow 側**: 現状は各リポから `@main` で参照 (initial rollout)。安定後は `v1.x.x` のタグを切り、各リポは pin に変更する想定 (Renovate / Dependabot で更新管理)
- **gitleaks image 側**: デフォルト `latest` で常に最新ルール (新サービスのトークンパターン等) を取り込む。再現性が必要な検証用途では caller workflow 側で `gitleaks-version: v8.x.x` のように pin できる

## 関連

- gitleaks 本体: <https://github.com/gitleaks/gitleaks>
