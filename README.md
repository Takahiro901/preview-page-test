# PRドキュメントプレビュー

PR作成時にMarkdownドキュメント（`docs/`）をMkDocs MaterialでビルドしGitHub Pagesの
`gh-pages`ブランチ配下（`pr-preview/pr-<PR番号>/`）にデプロイし、GitHub Commit Status
（Statuses API）経由でPRのチェック一覧にプレビューリンクを表示する仕組み。

## 構成

- `mkdocs.yml` / `requirements-docs.txt` — MkDocs Materialのビルド設定
- `docs/` — ドキュメント本体（サンプルページのみ）
- `.github/workflows/docs-pr-preview.yml` — PR作成・更新時にビルド→gh-pagesへデプロイ→Commit Status作成→PRコメント
- `.github/workflows/docs-pr-preview-cleanup.yml` — PRクローズ時にプレビュー削除→Commit Statusを`failure`に書き換え

## 重要: 当初案（Deployment Status）からCommit Statusへ変更した経緯

引き継ぎ資料では「GitHub Deployments API（Environment）を使えばPRのチェック欄・右サイドバー
Environments欄にリンクが出る」という前提だったが、実リポジトリ（`Takahiro901/preview-page-test`、
一時public化）で実機検証した結果、**Deployments APIで作ったDeployment Statusは、PRのChecksタブにも
右サイドバーのEnvironments欄にも表示されない**ことを確認した（表示されるのは会話タイムライン中の
「deployed to pr-N — View deployment」という1行のみで、PRコメントと同程度の視認性しかない）。

代わりに**Commit Status（Statuses API, `createCommitStatus`）**を使うと、GitHub Actionsのcheck run
（CI結果）と同じ「チェック一覧」（`gh pr checks`で見えるもの＝PR下部のマージボックスが参照するのと
同じデータ）に、クリック可能なリンク付きで確実に表示されることを確認した。そのため実装は
Commit Status方式に置き換えている。Deployments API自体は使用していない
（`permissions`も`deployments: write`ではなく`statuses: write`）。

## リポジトリ作成後、実際の運用前に必ず確認すること

1. **GitHub Pagesのビルド方式**
   `Settings > Pages` で `Build and deployment > Source` が **「Deploy from a branch」→ `gh-pages`**
   になっていることを確認する。「GitHub Actions」ビルドタイプだとこのワークフローの
   `pr-preview/pr-<番号>/`サブディレクトリ運用は成立しない。

2. **GitHub Pagesのプラン制約**
   privateリポジトリでGitHub Pagesを有効化するには、個人アカウントならGitHub Pro、
   OrganizationならTeam/Enterprise Cloud相当のプランが必要（GitHub Freeではprivateリポジトリの
   Pagesは有効化できないことを実機で確認済み）。対象リポジトリ・Organizationのプランを確認すること。

3. **ドキュメント格納ディレクトリ名**
   `docs/`で仮置きしている。実際のディレクトリ名が異なる場合は `mkdocs.yml`の`docs_dir`と
   両ワークフローの`paths:`トリガー条件を修正する。

4. **Pythonバージョン方針**
   `3.12`で仮置きしている（ローカル・CI双方で`mkdocs build --strict`が通ることを確認済み）。
   リポジトリの方針に合わせて`docs-pr-preview.yml`の`python-version`を修正する。

5. **`GITHUB_TOKEN`のデフォルト権限**
   ワークフロー側で`permissions: statuses: write`は明示済み。ただしOrganization設定側で
   Actionsのデフォルト権限が制限されている場合（`Settings > Actions > General > Workflow permissions`
   がOrganization/Enterpriseポリシーで固定されているなど）、ワークフロー側の指定だけでは
   有効にならないことがあるため、Organization管理者に確認する。

6. **EMU認証によるアクセス制御の実効性**
   GitHub Pagesが「社内メンバー限定」になるかどうかは、リポジトリがPrivate/Internalであることと
   Pages自体の可視性設定（Enterprise Cloud機能）に依存する。Publicリポジトリの場合はGitHub Pagesも
   誰でも閲覧可能になるため、本番相当のOrganization設定で必ず実機確認すること。

## 実機検証で確認できたこと

`Takahiro901/preview-page-test`（一時public化）にテストPRを作成し、以下を確認済み。

- [x] `mkdocs build --strict`がローカル・CI双方で成功する
- [x] PRのチェック一覧（`gh pr checks`で確認）に`docs-preview/pr-<番号>`が表示され、
      クリックで正しいプレビューURL（実際にMkDocs Materialでビルドされたページ）に遷移できる
- [x] 同一PRへの複数回pushで、Commit Statusは同一contextを上書きするだけなので増殖しない
      （Deployments API時代に必要だった「旧デプロイをinactiveにする」処理が不要になった）
- [x] 2つのPRを同時に開いても、`pr-preview/pr-1/`と`pr-preview/pr-2/`が独立して存在し、
      互いにも本番の`index.html`にも干渉しない
- [x] PRクローズで`gh-pages`上のプレビューディレクトリが削除され、Commit Statusが
      `failure`＋「プレビューは削除されました」に書き換わる

## 未確認（要ユーザー側での実機確認）

- [ ] PR右サイドバー・マージボックスでの見た目を、実際にログインしたブラウザで目視確認
      （`gh pr checks`ではデータ上の存在は確認済みだが、このセッションからは認証済みブラウザで
      画面を直接見る手段がなかったため）
- [ ] EMU未認証のブラウザセッションからのアクセスブロック（本番相当のOrganization/Private環境が必要）

## ローカルでのビルド確認方法

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-docs.txt
mkdocs build --strict --site-dir site
```
