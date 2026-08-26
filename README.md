# PRドキュメントプレビュー

PR作成時にMarkdownドキュメント（`docs/`）をMkDocs MaterialでビルドしGitHub Pagesの
`gh-pages`ブランチ配下（`pr-preview/pr-<PR番号>/`）にデプロイし、GitHub Deployments API
経由でPRのチェック欄・Environments欄にプレビューリンクを表示する仕組み。

## 構成

- `mkdocs.yml` / `requirements-docs.txt` — MkDocs Materialのビルド設定
- `docs/` — ドキュメント本体（サンプルページのみ）
- `.github/workflows/docs-pr-preview.yml` — PR作成・更新時にビルド→デプロイ→Deployment作成→PRコメント
- `.github/workflows/docs-pr-preview-cleanup.yml` — PRクローズ時にプレビュー削除→Deploymentをinactive化

## リポジトリ作成後、実際の運用前に必ず確認すること

このプロジェクトはリポジトリ未作成の段階で叩き台として実装したもの。GitHub上にリポジトリを
作成したら、以下を必ず確認・調整すること（引き継ぎ資料の「前提として確認が必要な事項」に対応）。

1. **GitHub Pagesのビルド方式**
   `Settings > Pages` で `Build and deployment > Source` が **「Deploy from a branch」→ `gh-pages`**
   になっていることを確認する。「GitHub Actions」ビルドタイプだとこのワークフローの
   `pr-preview/pr-<番号>/`サブディレクトリ運用は成立しない。

2. **ドキュメント格納ディレクトリ名**
   `docs/`で仮置きしている。実際のディレクトリ名が異なる場合は `mkdocs.yml`の`docs_dir`と
   両ワークフローの`paths:`トリガー条件を修正する。

3. **Pythonバージョン方針**
   `3.12`で仮置きしている（ローカル検証はPython 3.13.7で実施し、ビルド自体は問題なく通ることを
   確認済み）。リポジトリの方針に合わせて`docs-pr-preview.yml`の`python-version`を修正する。

4. **`GITHUB_TOKEN`のデフォルト権限**
   ワークフロー側で`permissions: deployments: write`は明示済み。ただしOrganization設定側で
   Actionsのデフォルト権限が制限されている場合（`Settings > Actions > General > Workflow permissions`
   が Organization/Enterprise ポリシーでReadonlyに固定されているなど）、ワークフロー側の指定だけでは
   `deployments: write`が有効にならないことがあるため、Organization管理者に確認する。

## 実装した内容（Deployment Status対応）

### `docs-pr-preview.yml`

- `permissions`に`deployments: write`を追加
- `Deploy preview to gh-pages`の後に`actions/github-script@v7`のステップを追加し、以下を実施:
  1. 同一`environment`（`pr-<PR番号>`）の既存Deploymentをすべて`inactive`にする
     （再pushのたびにDeploymentが際限なく積み上がるのを防止）
  2. `ref: pull_request.head.sha`で新規Deploymentを作成（`auto_merge: false`,
     `required_contexts: []`でステータスチェック待ち・マージ処理によるブロックを回避、
     `transient_environment: true`で一時的な環境であることを明示）
  3. `state: "success"`, `environment_url: <プレビューURL>`のDeployment Statusを作成
- PRコメントのプレビューURLは、Deployment作成ステップの出力（`steps.deployment.outputs.preview_url`）
  を参照する形に変更（URL文字列の二重管理を避けるため）

### `docs-pr-preview-cleanup.yml`

- `permissions`に`deployments: write`を追加
- プレビューディレクトリ削除の後に、同一`environment`のDeploymentを取得し、
  すべて`state: "inactive"`のDeployment Statusを追加するステップを追加

## 未実施（要ユーザー側対応）

以下はGitHub上に実リポジトリが存在しないと検証できないため、このセッションでは未実施。
リポジトリ作成・push・PR作成はいずれも外部への公開を伴う操作のため、実施前に別途確認する。

- [ ] 実リポジトリ作成、Pages設定の確認（上記1参照）
- [ ] テストPRを作成し、以下を確認
  - [ ] PRのチェック欄に`pr-<PR番号>`環境のDeploymentが表示され、「View deployment」から
        正しいプレビューURLに遷移できる
  - [ ] PR右サイドバー「Environments」欄にも表示される
  - [ ] 同一PRへの複数回pushでDeploymentが増殖せず、古いものは`inactive`になる
  - [ ] 2つのPRを同時に開いても互いのプレビューディレクトリ・Deploymentが干渉しない
  - [ ] PRクローズ/マージでプレビューディレクトリが削除され、Deploymentが`inactive`になる
  - [ ] EMU未認証のブラウザセッションからプレビューURLにアクセスした場合、
        組織のアクセス制御でブロックされることを確認（GitHub Pagesの可視性設定に依存するため、
        リポジトリがPrivateかつOrganizationの設定次第。Publicリポジトリの場合はGitHub Pagesも
        誰でも閲覧可能になる点に注意——EMU/EntraID認証だけでは自動的にPages閲覧を制限しない
        ケースがあるため、要実機確認）

## ローカルでのビルド確認方法

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-docs.txt
mkdocs build --strict --site-dir site
```
