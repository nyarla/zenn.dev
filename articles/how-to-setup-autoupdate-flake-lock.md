---
title: "nix flakeを使ったプロジェクトでflake.lockをGitHub Actionsで自動的に更新してみた"
emoji: "❄️"
type: "tech"
topics:
  - githubactions
  - nix
  - nixflakes
  - nixos
published: false
---

## 話の前提

私はここ最近、何かしらの開発を始める際に、必ずと言っていいほど `nix flake` を活用しています。

`nix flake` はnixpkgsのリポジトリのリビジョンなどを固定するために `flake.lock` と言うロックファイルを作成するのですが、このロックファイルは `nix flake update` などのコマンドを打たない限り、自動的には更新されません。

そのため各種ソフトウェアを最新の状態へ保つためには、適宜`flake.lock`を更新しなければならないのですが、今回はその `flake.lock` をGitHub Actionsを用いて自動更新できるようにしたため、その際にどのような設定をしたかを解説していきます。

## 最初に結論を書け

はい。

ということで最初に結論を出すと、下記のようなファイルをGitHub Actionsのworkflowsに仕込めば、GitHub Actionsによる `flake.lock` の自動更新が可能になります：

```yaml:autoupdate-nix-flake.yaml
name: Automatic update to nix flake.lock

on:
  schedule:
    - cron: "45 16 * * 5"
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  autoupdate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - uses: cachix/install-nix-action@fc6e360bedc9ee72d75e701397f0bb30dce77568 # v31.5.2
        with:
          nix_path: nixpkgs=channel:nixpkgs-unstable

      - name: Update nix flake
        run: nix flake update

      - name: Make metadata
        id: meta
        run: |
          {
            echo "date=$(date +%Y-%m-%d)"
          } >> $GITHUB_OUTPUT

      - uses: peter-evans/create-pull-request@271a8d0340265f705b14b6d32b9829c1cb33d45e # v7.0.8
        with:
          title: "chore(flake): update nix flake at ${{ steps.meta.outputs.date }}"
          commit-message: "chore(flake): update nix flake at ${{ steps.meta.outputs.date }}"
          branch: "update-nix-flake-at-${{ steps.meta.outputs.date }}"
          add-paths: |
            flake.lock
```

なお上記の中身は、

https://github.com/nyarla/edgefeed/blob/c27e03234f292ed4fae4da53d1cea24f25b92af2/.github/workflows/autoupdate-nix-flake.yml

から抜き出したものですが、特に著作権などを主張するつもりもないため、利用したい方は[ISCライセンス](https://spdx.org/licenses/ISC.html)に準ずる形で扱ってください。

また今回のworkflow全体の流れとしては：

1. リポジトリからコード一式をcheckoutする
2. workflowのコンテナに `nix` を導入する
3. workflowの中で `nix flake update` を発行して `flake.lock` を更新する
4. その更新した中身をPull Requestとして出す

という形になります。

## 解説

まず`on.schedule`と`on.workflow_dispatch`については、GitHub Actionsでのcronや手動での `flake.lock` を実現するために仕込んであります。

この辺りの設定については、

https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows

を見た方が早いので、ここでは解説しません。

次いで `permissions` についてですが、これはこのworkflowで用いる `peter-evans/create-pull-request` がこの権限を要求するため設定しています。

そしてもっとも重要な点が`jobs.autoupdate.steps`の中身となりますが、これについては順を追って説明します。

### 1. `actions/checkout`

これについては大抵のworkflowで用いられている、GitHub上のリポジトリをcheckoutするものですね。

今回の場合、GitHub Actionsに起因するセキュリティリスクを下げるために、`@v4`と言った指定ではなくcommit hashを直接指定しています。

確かに `@v4` といったような、タグ経由でGitHub Actionsを利用することは便利です。

ただこのタグ経由でのActionの利用は、利用先のActionがアカウント侵害を受けた場合、利用しているタグを悪意あるcommit hashに書き換える攻撃が実際に起きているため、私はその攻撃を避けるためにcommit hashを直接利用する方法を採っています。

### 2. `cachix/install-nix-action`

これはGitHub Actionsの環境へNix package managerを導入するために使っていて、公式のインストーラーよりも高速に `nix` コマンドを扱うために利用しています。

この際 `with.nix_path` の指定で `nixpkgs=channel:nixpkgs-unstable` を利用していますが、これは `flake.nix` で `nixpkgs-unstable`を利用していて、ここへ合わせる形でこういった指定になっています。

### 3. `Update nix flake`

これについては `flake.lock` の更新（パッケージ定義をアップデート）を行うためのコマンドである、

```shell
$ nix flake update
```

をGitHub Actions内で実行する、という意味合いの `step` になります。

### 4. `Make metadata`

これは完全に用途に応じて……という流れにはなるのですが、この `step` は `peter-evans/create-pull-request` でPull Requestを作る際に、そこで使う変数を定義するためのものとなっています。

そのためこの `step` の中で、Pull Requestで扱う変数をすべて作る、という形になりますね。

### 5. `peter-evans/create-pull-request`

最期にこの `step` で `flake.lock` を更新するPull Requestを作り、それをリポジトリへ提出する、と言った辺りが今回解説した全体の流れになります。

この際に最重要なのは `with.add-paths` の値で、これに `flake.lock` を含めていないと、肝心の `flake.lock` がアップデートされたPull Requestが作成されません。

またこの `peter-evans/create-pull-request` を使ってPull Requestを作る際には、workflowからPull Requestを作成できるよう、GitHub側のリポジトリ設定を変更する必要があり、これは：

1. リポジトリ上の`Settings`タブを開く
2. `Settings`タブから`Actions`→`General`を開く
3. `Workflow permissions`セクションまで移動する
4. `Allow GitHub Actions to create and approve pull requests` にチェックを入れる

という流れで設定を行なえます。

この点についてはZennにある、

https://zenn.dev/kenghaya/articles/d7f766e5db6437

と言う記事で触れられているので、これについてはそちらを参考にしてください。

また今回のworkflowファイルではPull Requestに含まれる説明文など、細かいところまでは詰めていないため、細かい設定を詰めたい場合は、

https://github.com/peter-evans/create-pull-request?tab=readme-ov-file#action-inputs

も参照してください。

## 以上

以上が今回行なった、

> GitHub Actionsで `nix flake`の `flake.lock` を自動更新する

と言う中身の解説となります。

こうやって解説を書いてみると思いのほか単純だったので拍子抜けしているのですが、実際のところここまで辿り付くために、そこそこ苦戦した部分もあります。

特にworkflowからPull Requestを出すためにGitHubの設定を変更しなければならない、と言った点についてはまったく知らなかったため、そこで一回詰まっていました。

とは言えそこさえ抜けてしまえば後は普通に一発動作に近い形で動いたので、そこは目的を達成できて良かったな、と、ほっとしているところです。

---

個人的に、近年ではソフトウェアの開発リポジトリに `flake.nix` が含まれている事が増えているように感じています。

とは言え今回のように、 `flake.lock` を自動更新するところまでは行なわれていないようなので、もし `nix flake`を利用していて `flake.lock`の自動更新が必要なのであれば、今回の記事を参考にしてみて下さい。
