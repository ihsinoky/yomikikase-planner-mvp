# yomikikase-planner-mvp

[![CI](https://github.com/<OWNER>/yomikikase-planner-mvp/actions/workflows/ci.yml/badge.svg)](https://github.com/<OWNER>/yomikikase-planner-mvp/actions/workflows/ci.yml)

## Continuous Integration

GitHub Actions で push と pull_request をトリガーに CI を実行します。Node.js 20 環境で npm のキャッシュを `~/.npm` に保持しつつ、依存関係をインストールしてから `npm run lint`、`npm run test`、`npm run build` を順に実行します（スクリプトが存在しない場合はスキップされます）。

### 使い方
1. リポジトリを GitHub に公開したら、上記バッジの `<OWNER>` を自分の GitHub ユーザー名または組織名に置き換えてください。
2. `package.json` / `package-lock.json` を追加し、必要な npm スクリプト（`lint`、`test`、`build` など）を定義してください。ローカルでは同じコマンドで確認できます。
3. Wrangler を用いた Cloudflare Workers デプロイ前検証を行う場合は `wrangler.toml` を追加し、`wrangler deploy --dry-run` が通るように API トークン等を GitHub Secrets に設定してください（`deploy-dry-run` ジョブは `wrangler.toml` が存在する場合のみ動作します）。

このワークフローによって、PR/ブランチのビルドとテストが自動化され、デプロイ検証も追加しやすくなります。
