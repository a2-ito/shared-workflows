# shared-workflows

a2-ito / a2ito の全リポジトリで共有する GitHub Actions の再利用ワークフロー。

| ワークフロー | 調べるもの | 対象 |
| --- | --- | --- |
| [trivy.yml](.github/workflows/trivy.yml) | 依存の脆弱性（lockfile）、IaC の設定ミス、秘密情報のコミット | 全リポジトリ |
| [codeql.yml](.github/workflows/codeql.yml) | 自分たちが書いたコードの脆弱性（SSRF、XSS、インジェクション、Actions の危ない書き方など） | public リポジトリだけ |

- **Trivy** は HIGH 以上が見つかったら失敗する。public リポジトリでは、全重大度の結果を Security タブ（code scanning）にも載せる。private リポジトリの code scanning は有料（GitHub Code Security）なので、失敗させるだけにしている
- **CodeQL** は private リポジトリでは動かない（同じく有料）。Dart（Flutter）など CodeQL が対応していない言語は調べられない
- 見送ると決めた Trivy の検出は、呼び出し側のリポジトリの `.trivyignore.yaml` に理由と見直す期限（`expired_at`）を書く。CodeQL の誤検知などは、Security タブでアラートに理由を付けて閉じる

## 呼び出し方

各リポジトリに `.github/workflows/security.yml` を置く。

```yaml
name: Security

on:
  pull_request:
  push:
    branches: [main]
  schedule:
    - cron: "0 21 * * *" # 毎日 6:00 (JST)
  workflow_dispatch:

permissions:
  contents: read

jobs:
  trivy:
    uses: a2-ito/shared-workflows/.github/workflows/trivy.yml@main
    permissions:
      contents: read
      security-events: write

  # public リポジトリだけ
  codeql:
    uses: a2-ito/shared-workflows/.github/workflows/codeql.yml@main
    with:
      languages: '["actions","javascript-typescript"]'
    permissions:
      contents: read
      security-events: write
```

- `runs-on` でランナーを変えられる（既定 `ubuntu-latest`）。a2ito の private リポジトリは `${{ vars.RUNNER || 'ubuntu-latest' }}` を渡し、org の変数 `RUNNER` で self-hosted に回す
- `languages` は CodeQL の言語名の JSON 配列。`security.yml` を置くと `.github/workflows` ができるので、`actions` はいつも入れる
- Go などのコンパイル言語は autobuild、それ以外はビルドせずに解析する
- CodeQL はリポジトリ設定の default setup と同時に使えない。default setup は無効（not-configured）にしておく
- `@main` で呼ぶので、このリポジトリの main を変えると全リポジトリに効く。main には ruleset で直 push できず、PR を通す

## Trivy の版を上げる

`trivy.yml` の `TRIVY_VERSION` を変える。リリースの tar をチェックサムで確かめてから使うので、
`aquasecurity/trivy-action` のようなタグを書き換えられうる Action には頼っていない。
