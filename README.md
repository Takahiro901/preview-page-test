# PRドキュメントプレビュー

PR作成時にMarkdownドキュメント（`docs/`）をMkDocs MaterialでビルドしGitHub Pagesの
`gh-pages`ブランチ配下（`pr-preview/pr-<PR番号>/`）にデプロイする。プレビューリンクは
**PRのDescription（本文）自体**に埋め込むのを主な通知手段とし、補助的にPRのチェック一覧
（Commit Status）とPRコメントにも表示する。

## 構成

- `mkdocs.yml` / `requirements-docs.txt` — MkDocs Materialのビルド設定
- `docs/` — ドキュメント本体（サンプルページのみ）
- `.github/workflows/docs-pr-preview.yml` — PR作成・更新時に ビルド → gh-pagesへデプロイ →
  Commit Status作成 → **PR Description更新** → PRコメント
- `.github/workflows/docs-pr-preview-cleanup.yml` — PRクローズ時に プレビュー削除 →
  Commit Statusを`failure`に書き換え → **PR Descriptionからブロック削除** → PRコメント更新

## 重要: 通知方法を2段階で見直した経緯

引き継ぎ資料の当初案は「GitHub Deployments API（Environment）を使えばPRのチェック欄・右サイドバー
Environments欄にリンクが出る」というものだったが、実リポジトリ（`Takahiro901/preview-page-test`、
一時public化）で実機検証した結果、**Deployments APIで作ったDeployment Statusは、PRのChecksタブにも
右サイドバーのEnvironments欄にも表示されない**ことを確認した（表示されるのは会話タイムライン中の
「deployed to pr-N — View deployment」という1行のみで、PRコメントと同程度の視認性しかない）。

そこで一度**Commit Status（Statuses API, `createCommitStatus`）**に切り替え、GitHub Actionsの
check run（CI結果）と同じ「チェック一覧」（`gh pr checks`で見えるもの＝PR下部のマージボックスが
参照するのと同じデータ）にクリック可能なリンクとして表示されることを確認した。ただし、
マージボックス付近は「レビュアーが見落としやすい」という指摘があり、さらに一段確実な手段として
**PR本文（Description）へのマーカー付き追記**を追加した。PRを開いた瞬間にタイトル直下へ
必ず表示されるため、現状これが最も見落としにくい。マーカー
（`<!-- docs-preview:start -->` 〜 `<!-- docs-preview:end -->`）で囲むことで、再pushのたびに
重複追記せず同じ箇所を上書きし、PRクローズ時にはブロックごと削除する。

Deployments API自体は使用していない（`permissions`も`deployments: write`ではなく
`statuses: write`）。Commit StatusとPRコメントは、チェック一覧や履歴からも追えるようにする
補助的な手段として引き続き残している。

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
- [x] PR本文にマーカー付きでプレビューリンクが追記され、タイトル直下に表示される
      （スクリーンショットで目視確認済み）
- [x] 再pushしても重複追記されず同じブロックが上書きされる
- [x] PRクローズでPR本文からブロックが削除され、元の本文だけが残る

## 未確認（要ユーザー側での実機確認）

- [ ] EMU未認証のブラウザセッションからのアクセスブロック（本番相当のOrganization/Private環境が必要）

## ローカルでのビルド確認方法

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-docs.txt
mkdocs build --strict --site-dir site
```
