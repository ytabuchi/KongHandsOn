# Kong ハンズオンドキュメント

このドキュメントでは [Kong 公式のドキュメント](https://getkong.org/docs/) に沿って [Kong Community Edition（以下 Kong）](https://konghq.com/kong-community-edition/) のハンズオンを行います。

## 事前準備

ハンズオンでは、Kong の Docker イメージ（Linux）を使用します。また、Kong へのアクセスに `Curl` または [Postman](https://www.getpostman.com/) を使用します。

- Docker が動作する環境（お持ちでない場合は Docker が動作している環境を用意してありますので、お知らせください）
- Linux にリモートログインする SSH クライアント
    - Windows の方は [TeraTerm](https://ttssh2.osdn.jp/)／[PuTTY](https://www.putty.org/)／[Poderosa](http://ja.poderosa-terminal.com/)／[rlogin](http://nanno.dip.jp/softlib/man/rlogin/) などをご用意ください。
- POST できる TCP クライアントツール
    - Windows の方は [Postman](https://www.getpostman.com/apps) が良いでしょう。
    - macOS／Linux の方は Curl が標準でインストールされています。必要に応じて Postman を使用してください。



## インストール

まずは Postgres のデータベースを起動します。Docker 環境にログインし、以下を実行します。

```bash
docker run -d --name kong-database \
    -p 5432:5432 \
    -e "POSTGRES_USER=kong" \
    -e "POSTGRES_DB=kong" \
    postgres:9.4
```

次にデータベースを用意します。`KONG_DATABASE` の環境変数で利用しているデータベース（Casandora と Postgres があり、今回は Postgres を使用しています）を指定します。

```bash
docker run --rm \
    --link kong-database:kong-database \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_HOST=kong-database" \
    -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
    kong:latest kong migrations up
```

`XX migrations ran` というようなメッセージが表示され、データベースの準備ができます。準備が完了したら、Kong のコンテナを起動します。

```bash
docker run -d --name kong \
    --link kong-database:kong-database \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_HOST=kong-database" \
    -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
    -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
    -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
    -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_LISTEN=0.0.0.0:8001" \
    -e "KONG_ADMIN_LISTEN_SSL=0.0.0.0:8444" \
    -p 8000:8000 \
    -p 8443:8443 \
    -p 8001:8001 \
    -p 8444:8444 \
    kong:latest
```

起動を確認します。

```bash
$ curl -i http://localhost:8001/
```

> こちらで用意したコンテナを使用する場合は、実行時に表示されるコンテナ名とポート番号をメモしておいてください。これ以降もポートは適宜読み替えてください。

## API の追加と確認

では実際に API を追加します。API の開発は、Admin API ポートの各種エンドポイントに POST／UPDATE／DELETE などでデータを送って行っていきます。ローカルの Docker で動かしている場合は、標準で `localhost:8001` が Admin API のポートで、API 操作のエンドポイントは `/apis/` となります。

```bash
curl -i -X POST \
    --url http://localhost:8001/apis/ \
    --data 'name=test' \
    --data 'uris=/test' \
    --data 'upstream_url=http://httpbin.org'
```

`HTTP/1.1 201 Created` の JSON が返ってくれば成功です。

`name` が作成した API の名前（一意である必要があります）、`uris` がエンドポイント、`upstream_url` が管理する（転送する）API のエンドポイントです。今回は [httpbin](http://httpbin.org) を使用します。


アクセスして確かめてみます。動作しているユーザー向けポートはデフォルトでは `8000` で、先ほど作成した `uris` の `/test/` が httpbin が転送されているエンドポイントです。

```bash
curl -i -X GET \
    --url http://localhost:8000/test/get?data=value
```


次のような JSON が返ってきて、正しく転送されていることがわかります。

```bash
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 303
Content-Type: application/json
Date: Fri, 09 Jun 2017 09:02:59 GMT
Server: meinheld/0.6.1
Via: kong/0.10.3
X-Kong-Proxy-Latency: 41
X-Kong-Upstream-Latency: 355
X-Powered-By: Flask
X-Processed-Time: 0.000751972198486
 
{
  "args": {
  "data": "value"
  },
  "headers": {
  "Accept": "*/*",
  "Accept-Encoding": "gzip, deflate",
  "Connection": "close",
  "Host": "httpbin.org",
  "User-Agent": "HTTPie/0.9.2"
  },
  "origin": "172.17.0.1, XXX.XXX.XXX.XXX",
  "url": "http://httpbin.org/get?data=value"
}
```


## Plugin で認証機能を追加

次に、ユーザー認証によるアクセス制限を行います。

```bash
curl -i -X POST \
    --url http://localhost:8001/apis/test/plugins/ \
    --data 'name=key-auth'
```

> `/test/plugins/` として、API 別に Plugin を追加することも可能ですし、グローバルに追加することも可能です。


もう一度、同じ API にアクセスしてみます。

```bash
curl -i -X GET \
    --url http://localhost:8000/test/get?data=value
```

今度は以下のように認証エラーが返ってきました。


```bash
HTTP/1.1 401 Unauthorized
Connection: keep-alive
Content-Type: application/json; charset=utf-8
Date: Wed, 07 Mar 2018 11:50:04 GMT
Server: kong/0.12.2
Transfer-Encoding: chunked
WWW-Authenticate: Key realm="kong"

{
    "message": "No API key found in request"
}
```

正しく Plugin が動作していることが分かりましたので、アクセスできるユーザーを追加します。

```bash
curl -i -X POST \
    --url http://localhost:8001/consumers/ \
    --data "username=Jason"
```

Jason さんが追加されました。

```bash
HTTP/1.1 201 Created
Content-Type: application/json
Connection: keep-alive

{
  "username": "Jason",
  "created_at": 1428555626000,
  "id": "bbdf1c48-19dc-4ab7-cae0-ff4f59d87dc9"
}
```

そのまま Jason に対して `API Key` を発行します。

```bash
curl -i -X POST \
    --url http://localhost:8001/consumers/Jason/key-auth/ \
    --data 'key=ENTER_KEY_HERE'
```

これで、この APIKEY で最初の API にアクセスできるようになりました。

再度アクセスしてみます。

```bash
curl -i -X GET \
    --url http://localhost:8000/test/get?data=value \
    --header "apikey: ENTER_KEY_HERE"
```

次の結果が返ってきて、無事アクセスできたことが分かります。

```bash
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 440
Connection: keep-alive
Server: meinheld/0.6.1
Date: Wed, 07 Mar 2018 12:29:18 GMT
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
X-Powered-By: Flask
X-Processed-Time: 0
Via: kong/0.12.2
X-Kong-Upstream-Latency: 157
X-Kong-Proxy-Latency: 26

{
  "args": {
    "data": "value"
  },
  "headers": {
    "Accept": "*/*",
    "Apikey": "ENTER_KEY_HERE", # <- Apikey のヘッダー
    "Connection": "close",
    "Host": "httpbin.org",
    "User-Agent": "curl/7.54.0",
    "X-Consumer-Id": "0eb28a3c-4c43-4dd8-9471-ea3005ec72fb",
    "X-Consumer-Username": "Jason", # <- Consumer のヘッダー
    "X-Forwarded-Host": "XXX.XXX.XXX.XXX"
  },
  "origin": "XXX.XXX.XXX.XXX, XXX.XXX.XXX.XXX",
  "url": "http://localhost/get?data=value"
}
```

## おまけ：レートリミット

時間に余裕があれば、Rate Limit を掛けてみましょう。

Plugin 名は `rate-limiting` で今回は秒単位で 2回、分単位で 10回の制限を掛けてみます。


```bash
curl -i -X POST \
    --url http://localhost:8001/apis/test/plugins \
    --data "name=rate-limiting" \
    --data "config.second=1" \
    --data "config.minute=5"
```


エンドポイントにアクセスしてみます。


```bash
curl -i -X GET \
    --url http://localhost:8000/test/get?data=value1&data=value2
```

例えば 1秒間に 2回アクセスすると、次のような JSON が返ってきて、1秒内の制限に引っ掛かっていますが、1分内には残り 2回アクセスできることが分かります。

```bash
HTTP/1.1 429 
Date: Thu, 08 Mar 2018 07:06:06 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
X-RateLimit-Limit-second: 1
X-RateLimit-Remaining-second: 0
X-RateLimit-Limit-minute: 5
X-RateLimit-Remaining-minute: 2
Server: kong/0.12.2

{"message":"API rate limit exceeded"}
```

`consumer_id` などと組み合わせることで柔軟な制限を指定できます。

いかがでしたでしょうか！？これで今回のハンズオンは終了です。お疲れ様でした。


## 参考資料

本日一部使用した Kong の Admin API の一覧は

[Admin API \- v0\.12\.x \| Kong \- Open\-Source API Management and Microservice Management](https://getkong.org/docs/0.12.x/admin-api/)

に記載されています。

例えば、以下のようなコマンドが用意されています。

**作成された API 一覧**

```bash
curl -i -X GET \
    --url http://localhost:8001/apis/
```

**作成した API の削除**

```bash
curl -i -X DELETE \
    --url http://localhost:8001/apis/{API}
```

**Plugin の変更**

```bash
curl -i -X PATCH \
    --url http://localhost:8001/apis/{API}/plugins/{id} \
    --data "config.{property}"
```

などです。非常に簡単に操作できる API Management の Kong をぜひ触ってみてください。Kong についての詳細は

- [Kong 本家](https://konghq.com/kong-community-edition/)
- [エクセルソフト Kong ページ](https://www.xlsoft.com/jp/products/kong/index.html?r=ytgh)
- [エクセルソフトブログ Kong タグ](http://www.xlsoft.com/jp/blog/blog/tag/kong/?r=ytgh)

をご覧ください。

以上です。




