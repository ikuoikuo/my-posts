# Qiita記事の管理

Node.js 22.22.1以上を使用します。

```sh
npm ci
npm run login
npm run preview
```

ログインにはQiitaの個人用アクセストークン（`read_qiita` / `write_qiita`）を使用します。
https://qiita.com/settings/tokens/new?read_qiita=1&write_qiita=1&description=qiita-cli

トークンはCLIの入力欄に入力します。認証情報は通常 `~/.config/qiita-cli/credentials.json` に保存されます。
プレビューは http://localhost:8888 です。初回起動時はQiita上の投稿済み記事も同期されます。

## 今回の記事

編集元は `qiita/drafts/IBM Bob にパーソナルトレーナーになってもらう.md` です。
CLI用の `qiita/public/bob-training.md` は編集元へのシンボリックリンクなので、本文を二重管理しません。
`qiita/posts` にある既存のアーカイブはそのまま保持します。

画像は公開リポジトリ `ikuoikuo/my-posts` の `qiita/assets/bob-training` に保存し、本文では `raw.githubusercontent.com` のURLを参照します。コミットIDでURLを固定しているため、別ブランチで作業しても表示できます。

画像を更新するときは、先に画像をコミット・pushし、そのコミットIDを使ったURLに記事の参照を変更します。

## 手動投稿

画像の公開URLへの差し替えとプレビュー確認を終えた後、今回の記事だけを投稿します。

```sh
npm run publish:bob
```

このコマンドは公開記事を作成します。Qiita上の未公開下書きを更新する操作ではありません。
初回投稿後はCLIが記事IDを保存し、同じコマンドでその記事を更新できます。
GitHub Actionsによる自動投稿は設定していません。

新しい記事は `npx qiita new 記事名 --root qiita` で作成できます。
