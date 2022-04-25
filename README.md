# https-front
This is an nginx server definiton for digicre web services.

## これは何
デジクリの運用するサーバでは、ドメイン名の異なる複数のWebサービスが一つのサーバで動いています。

このリポジトリはデジクリの運用するサーバで複数ドメインへのアクセスをすべて受け取り、適切なサービスに転送するnginxサーバを定義しています。

転送先のコンテナを再起動した場合、コンテナ名とIPアドレスの関係が変わる可能性があるので、必ずこのDockerコンテナも再起動するようにしてください。

## これは何故必要
前述のとおり、デジクリでは一つのサーバでドメインの異なる複数のWebサービスが動いており、リクエストからドメイン名を見て、リクエストを適切なサーバ（Dockerコンテナ）へ振り分ける必要があります。
このリポジトリでは、その為に必要なnginxサーバを定義しています。

また、現在、Webサービスの提供にはhttpsへの対応が必須と言っても過言ではありませんし、デジクリのサービスの中には機密情報を扱うものもあることから、Webサーバのhttps対応が必要です。
httpsを利用するためには、適切な証明書を認証局に発行してもらう必要があります。
数年前までは、無料で証明書を発行してくれる認証局の存在はかなり薄いものでした。
現在では、Let's Encryptがhttpsに最低限必要な証明書を自動で発行してくれることで有名です。
デジクリにおいても、このLet's Encryptを利用して証明書を利用しています。

Let's Encryptでの証明書の発行は、それを行うためのスクリプトを書くよりできる限り既存のものを利用した方が良いですし、既にその為のツールが存在しています。

[SteveLTN/https-portal](https://github.com/SteveLTN/https-portal)はDockerコンテナに環境変数を渡して起動するだけで、証明書の発行、nginxサーバへの適用から証明書の更新までを自動行ってくれるnginxサーバが立ち上がるDockerイメージを定義しています。
このリポジトリはこれを利用してnginxサーバを立ち上げることで、https対応への負担を減らし、また、一つのサーバで複数のWebサービスを動かしています。

## 設定の説明
ここでは、docker-compose.ymlファイルの設定の説明を行います。

```
  https-portal:
    image: steveltn/https-portal:1
```
ここでは、先述のSteveLTN/https-portalのイメージを利用することを宣言しています。

```
    volumes:
      - ./ssl_certs:/var/lib/https-portal
```
ここでは、発行された証明書を `ssl_certs` ディレクトリ下に配置することで、起動時に毎回証明書を取得せずに、取得した証明書を使いまわすようにしています。

```
    ports:
      - 80:80
      - 443:443
```
ここでは、Dockerコンテナの80番ポートと443番ポートをバインドし、外部からのアクセスを受け付けるようにしています。

```
    environment:
      DOMAINS: "core.digicre.net -> http://web:80, core2.digicre.net -> http://core_next:3000, blog.digicre.net -> http://blog_next:3000, bolide.digicre.net -> http://bolide_frontend:80"
      STAGE: "production"
      WEBSOCKET: "true"
      CLIENT_MAX_BODY_SIZE: "200M"
```
ここでは、Dockerコンテナに環境変数を渡しています。

`DOMAINS` 環境変数には、ドメイン名と転送先の関係を記述します。
例えば、 `core.digicre.net -> http://web:80` という設定は、 `core.digicre.net` へのアクセスがあった場合には、 `http://web:80` にリクエストを転送します。
また、ここで記述されたドメイン名の証明書を自動で取得します。
新たに設定を追加する場合には、カンマ区切りで設定を記述します。

```
    networks:
      - https_network
```
ここでは、Dockerコンテナが接続するネットワーク設定を行っています。
転送先となるDockerコンテナも必ずこのネットワークに接続されている必要があります。
