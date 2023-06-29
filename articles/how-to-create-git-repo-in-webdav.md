---
title: "Git リポジトリを WebDAV サーバ上に作る方法"
emoji: "🕸️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - git
  - webdav
published: true
---

昨今では Git リポジトリを作る場合、 GitHub に作ったり self-hosted な GitHub clone 上で作ったり、と言うケースが多いと思いますが、しばらく前に Git リポジトリを WebDAV 上で作る機会があったので、今回はその点をメモっておきます。

## やり方

Git リポジトリを WebDAV 上で作る場合の流れはおおよそこんな感じです。

```bash
# git init に --bare を指定し剥き身の Git リポジトリを作る
$ git init --bare git-repo-in-webdav.git

# Git リポジトリを WebDAV上で使えるようにする
$ cd git-repo-in-webdav.git
$ git update-server-info

# WebDAV サーバ上へ Git リポジトリをアップロードする。例えば：
$ cd ..
$ rclone sync git-repo-in-webdav.git WeDAV:Git/git-repo-in-webdav.git

# WebDAV サーバから Git リポジトリを clone してくる
$ GIT_SMART_HTTP=0 git clone https://account@webdav/Git/git-repo-in-webdav.git webdav

# WebDAV サーバへローカルの変更を push する
$ cd webdav
$ GIT_SMART_HTTP=0 git push origin main
```

## 注意点

まず Git リポジトリを WebDAV サーバで利用する場合、作成した Git リポジトリに対して `git update-server-info` を実行する事は必須です。と言うかこのコマンドを打たないと Git リポジトリを WebDAV サーバから `git clone` してくることが出来ません。

次に必要な工夫としては、 WebDAV 上 の Git リポジトリに対して何かしらのコマンド（例えば `git clone`など）を打つ場合、`GIT_SMART_HTTP=0` を指定して Git の Smart HTTP プロトコルを無効にする必要があります。これは Git が HTTP で通信する際に smart http 対応サーバを前提として http 通信をするので、その点への対処です。

最後に注意すべき点として、WebDAV 上の Git リポジトリへ `git push` する際のアカウントやパスワードについては `.netrc` や `.git/config` 内の `url` の値として指定する必要がある、と言う点です。

通常 WebDAV サーバにパスワードが掛かってない事はイントラネットでもあまり無いと思いますが、WebDAV にパスワード認証が掛かっている場合、そのパスワード認証のアカウントを何らかの方法で Git に伝える必要があります。

そのため Git に対してセキュアにパスワードを伝える方法は、推奨できる方法として `.git/config` の `url` 値に アカウント名だけを指定し、残りのパスワード部分については Git の credential helper を使う、と言う手法を取る事が良いかと思います。

## 以上

ちなみに何故 Git リポジトリを WebDAV サーバ上に作ったかと言うと、私個人のブログ記事は Git で管理しているのですが、これが GitHub で対応できる容量を超えてしまい、最終的に `git push` が失敗するようになった、と言う事が理由です。

また今回のように WebDAV 上で Git リポジトリを管理する上での欠点として、

- WebDAV サーバへ push する際の挙動が遅い
- 多人数で管理するには機能が足りなさすぎる
- `git push` する際に色々と手間がかかる

と言う事柄があるので、私個人の感想として WebDAV で Git リポジトリを作る時には、

- Git LFS を使うほどではないが容量を消費するファイルが大量にある
- プライベートで一人管理するファイル類である
- 便利な管理 UI が無くともなんとかなる

と言う条件を満たす場合にのみ行うのが良いのではないか？と思っています。
