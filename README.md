# cw-workflows

COUNTERWORKS 各リポジトリから呼び出して使う **Reusable GitHub Actions Workflow** の集約リポジトリ。

PLATFORM-1281 / PLATFORM-1283 のシークレット漏洩防止施策の一環として作成 (gitleaks workflow が最初の workflow)。

## 提供 Workflow

### `.github/workflows/gitleaks.yml`

[gitleaks](https://github.com/gitleaks/gitleaks) (OSS, MIT) でリポジトリ内のシークレット混入を検出する Reusable Workflow。

- 公式 README で推奨されている **Docker image (`ghcr.io/gitleaks/gitleaks`)** を `docker run` で呼び出す方式
- 1 vCPU の **`ubuntu-slim`** ランナーで実行 (15 分上限の軽量ランナー)
- 公式 `gitleaks/gitleaks-action` は Organization 利用に商用ライセンスが必要なため不採用
- v8.19.0 以降のサブコマンド名変更に対応済み (`detect` → `git`)

#### Inputs

| name | type | default | 説明 |
| --- | --- | --- | --- |
| `gitleaks-version` | string | `latest` | `ghcr.io/gitleaks/gitleaks` のタグ。新ルールを取り込むため `latest` を既定。挙動を固定したい場合は `v8.30.1` のように pin |
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
    branches: [main, master, develop]
  schedule:
    - cron: "0 0 * * 1"
  workflow_dispatch:

jobs:
  scan:
    uses: COUNTERWORKS/cw-workflows/.github/workflows/gitleaks.yml@main
```

リポ固有の allowlist が必要なら、リポ直下に `.gitleaks.toml` (例: [`config/gitleaks-baseline.toml`](./config/gitleaks-baseline.toml) を起点に拡張) を配置し、caller workflow で `config-path: .gitleaks.toml` を渡す。

## バージョニング方針

- **Reusable Workflow 側**: 現状は各リポから `@main` で参照 (initial rollout)。安定後は `v1.x.x` のタグを切り、各リポは pin に変更する想定 (Renovate / Dependabot で更新管理)
- **gitleaks image 側**: デフォルト `latest` で常に最新ルール (新サービスのトークンパターン等) を取り込む。再現性が必要な検証用途では caller workflow 側で `gitleaks-version: v8.x.x` のように pin できる

## 関連

- 親チケット: [PLATFORM-1281](https://counter-works.atlassian.net/browse/PLATFORM-1281)
- 設計チケット: [PLATFORM-1283](https://counter-works.atlassian.net/browse/PLATFORM-1283)
- gitleaks 本体: <https://github.com/gitleaks/gitleaks>
