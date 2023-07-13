---
title: "Tailscale ネットワーク内に Fediverse を構築する"
emoji: "🪐"
type: "tech"
topics: ["fediverse", "activitypub", "mastodon", "misskey"]
published: true
---

## 始めに

最近 （2023-07-07 現在）Twitter の零落により Fedivese（Mastodon や Misskey など）への移住が進み始めていたり、Meta 社の threads が ActivityPub に対応予定であったりと、ActivityPub によって Server-to-Server (S2S）の通信する Web アプリケーション・サービスが注目を集める様になっています。

となると我々開発者としては ActivityPub 自体を話す Bot の開発や ActivityPub によって S2S 通信をする Web アプリケーションを開発したい！と言う意欲が湧いてくると考えているのですが（個人の感想です）、この時に問題となるのが、

> Fediverse と通信する Web アプリを構築するためのテスト環境はどう用意したら……？

と言う点です。

Fediverse ではない一般的な Web アプリケーションであれば、 Docker Compose などを用いて手軽に開発環境を用意することが出来ますが、Fedivese をローカルネットワーク上に構築しようとすると、下記の様な問題に遭遇します：

- SSRF 攻撃対策がローカルでの S2S 構築の障害になる（特に IP 範囲規制）
- Fedivese は https が前提なためローカルに https 環境構築する必要がある
- Fedivese App が他種多様であるため複数のインスタンスを立ち上げる必要がある

そのため今回の記事では上記の問題を（一部は強引に）回避しつつ、Fedivese ネットワークをローカルに構築する方法を解説したいと思います。

## 今回構築するネットワークの概要

まず Fedivese ネットワーク構築のためのソフトウェアは下記を用います：

- [Mastodon v4.1.3](https://github.com/mastodon/mastodon/releases/tag/v4.1.3)
- [Misskey master ブランチ](https://github.com/misskey-dev/misskey/tree/master)
- [GoToSocial commit d9c69f6](https://github.com/superseriousbusiness/gotosocial/commit/d9c69f6ce05daddf2f6d5d8c6c03b4c3d55df93a)

またネットワークの配置先として [Tailscale](https://tailscale.com/) を対象とし、Tailnet と DNS の HTTPS Certificates を用いて Fedivese ネットワークを構築します。

最後に Fedivese を構成するインスタンスは Docker Compose を用いて立ち上げることとし、Tailscale のクライアント（`tailscaled`）は各インスタンス用の Docker Compose ファイルで配置する事とします。

なお Fedivese ネットワーク構築の流れは下記の通りです：

1. 各コンテナ群に `tailscaled` を配置し Tailscale VPN へ参加させる
2. 各サービス群に `network_mode` を指定し通信を Tailscale コンテナ経由にする
3. 各アプリの SSRF 攻撃対策を外し Tailscale の `100.64.0.0/10` で動作できる様にる

## Fedivese ネットワーク構築の手順

それでは実際に Fedivese ネットワークを構築して行きます。

### Tailscale のコンテナを用意する

まず最初は通信の要となる Tailscale がインストールされたコンテナを用意します。

私の場合、次の様な Dockerfile と bootstrap スクリプトを用意し、`docker-compose.yml` から利用できる様にしました。

なおこれらの `Dockerfile` や `run.sh`は次のリポジトリを参考にしています：

https://github.com/lpasselin/tailscale-docker

```dockerfile:Dockerfile
FROM tailscale/tailscale:unstable
RUN mkdir -p /opt/bin
COPY run.sh /opt/bin/run.sh
RUN chmod +x /opt/bin/run.sh
```

```sh:run.sh
#!/bin/ash
trap 'kill -TERM $PID' TERM INT
echo "Starting Tailscale daemon"

/usr/local/bin/containerboot &
PID=$!
wait ${PID}

until tailscale --socket /tmp/tailscaled.sock serve https:443 / http://127.0.0.1:3000 ; do
  sleep 0.1
done

wait ${PID}
```

```yaml:docker-compose.yml
services:
  tailscale:
    # ここは各環境の Dockerfile があるディレクトリへ読み替えて下さい
    build: ../tailscale
    environment:
      # Tailscale の Auth Key は次から発行できます
      # https://login.tailscale.com/admin/settings/keys
      TS_AUTHKEY: "tskey-auth-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
      TS_STATE_DIR: /var/lib/tailscale
      TS_USERSPACE: 1
      # ホスト名を指定します。これは GoToSocial の場合です
      TS_EXTRA_ARGS: --hostname=gotosocial
    volumes:
      # tailscale の設定を永続化するためのディレクトリを指定して下さい
      - ../tailscale/state/gotosocial:/var/lib/tailscale
    command:
      - /opt/bin/run.sh
```

#### 補足

上記の `run.sh` では `/usr/local/bin/containerboot` をバックグラウンド実行した後で、

```sh
tailscale --socket /tmp/tailscaled.sock serve https:443 / http://127.0.0.1:3000
```

と言うコマンドを発行していますが、これは tailscaled を reverse proxy として動作させ、443 port の `/` へのアクセスを `127.0.0.1:4000` へマップするために実行しています。

また URL へのマップについては各 Fediverse ソフトウェアによって違い、Mastodon の様に複数のアプリを複数のパスに割り当てる場合にはスクリプトを次の様にする必要があります：

```sh:mastodon.sh
#!/bin/ash
trap 'kill -TERM $PID' TERM INT
echo "Starting Tailscale daemon"

/usr/local/bin/containerboot &
PID=$!
wait ${PID}

until tailscale --socket /tmp/tailscaled.sock serve https:443 / http://127.0.0.1:3000 ; do
  sleep 0.1
done

until tailscale --socket /tmp/tailscaled.sock serve https:443 /api/v1/streaming http://127.0.0.1:4000 ; do
  sleep 0.1
done

wait ${PID}
```

この設定については各 Fediverse アプリケーションの Revese Proxy 設定に準じるので、その場合は各種アプリケーションの設定を読み替えて下さい。

### ネットワーク通信を Tailscale コンテナ経由に集約する

次にネットワークへのアクセスを Tailscale コンテナ経由にする方法ですが、これは `docker-compose.yml` で次の様な設定をします。

なおこの例は GoToSocial のものです：

```yaml:docker-compose.yml
# この docker-compose.yml は gotosocial を git clone したディレクトリに配置されており
# gotosocial を build する Dockerfle は gotosocial リポジトリ内のものをそのまま使っています
services:
  tailscale:
    build: ../tailscale
    environment:
      TS_AUTHKEY: "tskey-auth-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
      TS_STATE_DIR: /var/lib/tailscale
      TS_USERSPACE: 1
      TS_EXTRA_ARGS: --hostname=gotosocial
    command:
      - /opt/bin/run.sh

  gotosocial:
    build: .
    container_name: gotosocial
    user: 1000:1000
    environment:
      # ここで 100.64.0.0/10 を信頼できるプロキシアドレスとして指定する
      GTS_TRUSTED_PROXIES: "127.0.0.1/32,100.64.0.0/10,::1"
      # ここで GoToSocial のホスト名を指定します
      # なお `tailXXXXX.ts.net` の実際の値は次の URL で確認できます：
      # - https://login.tailscale.com/admin/dns
      GTS_HOST: gotosocial.tailXXXXX.ts.net
      GTS_PROTOCOL: https
      GTS_BIND_ADDRESS: 0.0.0.0
      GTS_PORT: 3000
      GTS_DB_TYPE: sqlite
      GTS_DB_ADDRESS: /gotosocial/storage/sqlite.db
      GTS_LETSENCRYPT_ENABLED: "false"
      GTS_LETSENCRYPT_EMAIL_ADDRESS: ""
      GTS_LOG_LEVEL: trace
    volumes:
      # GoToSocial のデータを永続化するための設定
      - ./data:/gotosocial/storage
    restart: "always"
    # ここは **最重要ポイント** でネットワーク通信を tailscale コンテナ経由のみにします
    # GoToSocial の場合 `network_mode` の指定はこの一箇所だけですが、
    # 他のソフトウェアだと `db` や `redis` といった middleware を同じ設定をします
    # また `docker-compose.yml` では他のネットワーク関連の設定はすべて削除します
    network_mode: "service:tailscale"
    depends_on:
      - tailscale
```

### 各種 Fedivese アプリケーションの SSRF 攻撃対策を無効化する

ここまでの設定でインスタンスの立ち上げ自体は可能になりますが、前述の通り ActivityPub を話す Web Application は SSRF 攻撃防止のために ローカルネットワークでの S2S を禁じているため、この制約を外す必要があります。

この S2S ブロックを外すためには、

- 設定ファイルの書き換えで実現できるもの
- ソースコードを改変して実現できるもの

の二種類があり、それぞれやり方が違うのでこの辺りは各アプリケーションのソースコードを見て判断するしかありません。

とは言え今回の Mastodon、Misskey、GoToSocial ではパッチを当てることが容易であるため、S2S の制約を外すためのパッチは下記に記載しておきます。

ただし下記のパッチは **Mastodon などのセキュリティに穴を開けるパッチ** であり、間違っても **開いたネットワーク上にあるプロダクション環境では利用しない** で下さい。

#### Mastodon

```diff:mastodon.patch
diff --git a/app/lib/request.rb b/app/lib/request.rb
index c76ec6b64..e899f3dea 100644
--- a/app/lib/request.rb
+++ b/app/lib/request.rb
@@ -310,9 +310,7 @@ class Request
       alias new open

       def check_private_address(address, host)
-        addr = IPAddr.new(address.to_s)
-        return if private_address_exceptions.any? { |range| range.include?(addr) }
-        raise Mastodon::PrivateNetworkAddressError, host if PrivateAddressCheck.private_address?(addr)
+        return
       end

       def private_address_exceptions
```

#### GoToSocial

```diff:gotosocial.patch
diff --git a/internal/httpclient/sanitizer.go b/internal/httpclient/sanitizer.go
index 46540fd8..19ee8865 100644
--- a/internal/httpclient/sanitizer.go
+++ b/internal/httpclient/sanitizer.go
@@ -20,8 +20,6 @@ package httpclient
 import (
 	"net/netip"
 	"syscall"
-
-	"github.com/superseriousbusiness/gotosocial/internal/netutil"
 )

 type sanitizer struct {
@@ -32,7 +30,7 @@ type sanitizer struct {
 // Sanitize implements the required net.Dialer.Control function signature.
 func (s *sanitizer) Sanitize(ntwrk, addr string, _ syscall.RawConn) error {
 	// Parse IP+port from addr
-	ipport, err := netip.ParseAddrPort(addr)
+	_, err := netip.ParseAddrPort(addr)
 	if err != nil {
 		return err
 	}
@@ -41,27 +39,5 @@ func (s *sanitizer) Sanitize(ntwrk, addr string, _ syscall.RawConn) error {
 		return ErrInvalidNetwork
 	}

-	// Seperate the IP
-	ip := ipport.Addr()
-
-	// Check if this is explicitly allowed
-	for i := 0; i < len(s.allow); i++ {
-		if s.allow[i].Contains(ip) {
-			return nil
-		}
-	}
-
-	// Now check if explicity blocked
-	for i := 0; i < len(s.block); i++ {
-		if s.block[i].Contains(ip) {
-			return ErrReservedAddr
-		}
-	}
-
-	// Validate this is a safe IP
-	if !netutil.ValidateIP(ip) {
-		return ErrReservedAddr
-	}
-
 	return nil
 }
```

##### 補足

GoToSocial の場合バイナリをコンパイルする必要がるため、

```
$ VERSION=dev-mod ./scripts/build.sh
```

と言うようなコマンドを発行し、`gotosocial` のバイナリをコンパイルしてください。

#### Misskey

Misskey の場合ソフトウェア本体のソースコードにパッチを当てる必要はなく、設定ファイルである `.config/defau.yml` に下記を設定すれば良いです。

```yaml:.config/default.yml
# 一部抜粋
allowedPrivateNetworks: [
  '127.0.0.1/32',
  '100.64.0.0/10'
]
```

## Fedivese アプリのセットアップ

ここから先は各インスタンスを Docker Compose で立てる際と変わりは無いですが、GoToSocial や Misskey はともかく、Mastodon は若干手順が複雑であったのでその点を含めてインスタンスの立て方の手順を記載しておきます。

### Mastodon

基本的には、

https://docs.joinmastodon.org/admin/install/

の内容に従って作業を進めて行くと良いです。

ただしハマり所として、

- Postgres や Redis のコンテナは `rake mastodon:setup` 前に起動して起く必要がある
- `rake mastodon:setup` で生成した `.env.production` はコンテナを終了すると蒸発する
- `tailscaled` で reverse proxy する path が複数ある（`/` と `/api/v1/streaming`）

と言った辺りには注意が必要です。

また `RAILS_ENV=production bundle exec rake mastodon:setup` で設定する URL については tailnet で利用されるもの指定する必要があるので、その点にも注意が必要だと思われます。

### Misskey

https://misskey-hub.net/docs/install/docker.html

Misskey の場合ほぼハマりどこは無く、インスタンスで利用される URL を tailnet の物にするだけで OK です。

それと私の場合、プライベート環境の検証用ネットワークを構築することが主目的であっため、DB のパスワードなどはかなりの手抜きをして設定ファイルの初期値をそのまま使ったりしていますが、本来であればこの点は変更すべき物なので、そこは注意しましょう。

### GoToSocial

https://docs.gotosocial.org/en/latest/getting_started/installation/container/

GoToSocial は一点注意事項として、附属の Dockerfile では `gotosocial` のバイナリが Dockerfile と同じディレクトリに存在することを前提とするので、`gotosocial` のバイナリをコンパイルしておく必要があります。

```
$ VERSION=dev-mod ./scripts/build.sh
```

## Fedivese ネットワークをテストする

ここまでのインタンス構築が上手く行えていられていれば、各 Fediverse インスタンスで相互に follow や unfollow のテストが行なえる様になっています。ただし、この時にインスタンス同士が上手く相互通信できない時には、各インタンスのログを見て色々と切り分けをする必要があります。

その際の問題の切り分けとしては次の通りです。

### tailnet で名前解決が出来るか確認する

まず最初に行うことは tailnet 経由の名前解決が出来ているかどうかの確認です。

具体的に tailnet 経由で名前解決が出来るかどうかの確認は、次の方法で行います：

- `docker compose exec {container} {shell}` と言うコマンドでコンテナへ入る
- busybox の wget や ping 、もしくは curl や nc などで http か ping が通るか確認する

もしここで tailnet での名前解決が出来ない場合、コンテナ内の DNS で `100.100.100.100` が利用されていないと考えられるので、この場合 docker 本体の方の dns 設定で `100.100.100.100` を利用するように設定します。（なお Docker の設定方法については環境によって異なると思われるのでここでは省略します）

### Fedivdrse ソフトウェア同士の互換性の問題が無いか確認する

私の知る限り GoToSocial は Mastodon の `AUTHORIZED_FETCH=1`（Secure Mode）相当の挙動をしているため、`AUTHORIZED_FETCH=1` に対応しないインスタンスとは S2S による連合が組めません。

そのため Misskey だと `signToActivityPubGet: true` の設定が有効でない場合や、古い Mastodon のインタンスなどとは連合が組めないため、そう言ったソフトウェア間の互換性によって連合が機能しない場合もあります。

## 以上

Fediverse のネットワークを Tailscale 上に構築する手順としては以上になります。

ただこの記事を書き始めた 2023-07-07 からこれを書き上げた 2023-07-13 日の間で [GoToSocial の IP Block 周りに変更が入ったようです](https://github.com/superseriousbusiness/gotosocial/commit/2a99df0588e168660d3b528209d8f51689ca92b7)し Mastodon や Misskey でも日々更新行なわれているので、ここで記載した情報はいずれ古くなると思います。

また ActivityPub 実装を作る際には、[Windymelt](https://scrapbox.io/activitypub/windymelt) さんが立ち上げ有志たちで更新している、

https://scrapbox.io/activitypub/

も参考に出来るとかと思いますので、これらの情報を利用しつつ ActivitPub で様々な実装を作りましょう！
