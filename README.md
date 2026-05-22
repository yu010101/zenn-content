# zenn-content

Zenn記事をGitHub連携で公開するリポ。

## 公開手順
1. zenn.dev → アカウント作成 → GitHubからのデプロイ → 本リポを連携（一度だけ）
2. 各 `articles/*.md` の `published: false` → `true` に変えて push すると公開
3. 以後、push するだけで自動公開・更新

記事3本: aiki-business-ops-intro / aiki-mlb-ops-realdata / ai-agent-safety-gate
