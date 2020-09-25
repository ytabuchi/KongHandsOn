# Kong ハンズオンドキュメント

> Updated on 2020/9/25

ようこそ！

このドキュメントでは [Kong の公式ドキュメント](https://docs.konghq.com/) に沿って [Kong Gateway（旧 Kong Community Edition／以下 Kong）](https://konghq.com/kong/) のハンズオンを行います。

## 事前準備

ハンズオンでは、Kong の Docker イメージ（Linux）を使用します。また、Kong へのアクセスに `Curl` または [Postman](https://www.getpostman.com/) を使用します。

- Docker が動作する環境
- Linux にリモートログインする SSH クライアント
    - Windows の方は [TeraTerm](https://ttssh2.osdn.jp/)／[PuTTY](https://www.putty.org/)／[Poderosa](http://ja.poderosa-terminal.com/)／[rlogin](http://nanno.dip.jp/softlib/man/rlogin/) などをご用意ください。
    - macOS／Linux の方は標準のターミナルをご利用ください。
- POST できる TCP クライアントツール
    - Windows の方は [Postman](https://www.getpostman.com/apps) が良いでしょう。どうしてもコマンドで操作したい方は、PowerShell 3.0 以上で使える [`Invoke-WebRequest`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest?view=powershell-6) コマンドが `Curl` に似た動きをするので使ってみても良いかもしれません。
    - macOS／Linux の方は Curl が標準でインストールされています。必要に応じて Postman を使用してください。



## インストール

最初にコンテナ同士が発見や通信ができるようにカスタムネットワークをを作ります。Docker 環境で以下を実行します。

```bash
docker network create kong-net
```

次に Postgres のデータベースを起動します。

```bash
docker run -d --name kong-database \
               --network=kong-net \
               -p 5432:5432 \
               -e "POSTGRES_USER=kong" \
               -e "POSTGRES_DB=kong" \
               -e "POSTGRES_PASSWORD=kong" \
               postgres:9.6
```

1行バージョン：

```bash
docker run -d --name kong-database --network=kong-net -p 5432:5432 -e "POSTGRES_USER=kong" -e "POSTGRES_DB=kong" -e "POSTGRES_PASSWORD=kong" postgres:9.6
```

次にデータベースを用意します。`KONG_DATABASE` の環境変数で利用しているデータベース（Casandora と Postgres があり、今回は Postgres を使用しています）を指定します。

```bash
docker run --rm \
     --network=kong-net \
     -e "KONG_DATABASE=postgres" \
     -e "KONG_PG_HOST=kong-database" \
     -e "KONG_PG_USER=kong" \
     -e "KONG_PG_PASSWORD=kong" \
     -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
     kong:latest kong migrations bootstrap
```

1行バージョン：

```bash
docker run --rm --network=kong-net -e "KONG_DATABASE=postgres" -e "KONG_PG_HOST=kong-database" -e "KONG_PG_USER=kong" -e "KONG_PG_PASSWORD=kong" -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" kong:latest kong migrations bootstrap
```


以下のようなメッセージが表示され、データベースの準備ができます。

```bash
XX migrations processed
XX executed
database is up-to-date
```

準備が完了したら、Kong のコンテナを起動します。

```bash
docker run -d --name kong \
     --network=kong-net \
     -e "KONG_DATABASE=postgres" \
     -e "KONG_PG_HOST=kong-database" \
     -e "KONG_PG_USER=kong" \
     -e "KONG_PG_PASSWORD=kong" \
     -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
     -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
     -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
     -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
     -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
     -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
     -p 8000:8000 \
     -p 8443:8443 \
     -p 127.0.0.1:8001:8001 \
     -p 127.0.0.1:8444:8444 \
     kong:latest
```

1行バージョン：

```bash
docker run -d --name kong --network=kong-net -e "KONG_DATABASE=postgres" -e "KONG_PG_HOST=kong-database" -e "KONG_PG_USER=kong" -e "KONG_PG_PASSWORD=kong" -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" -e "KONG_PROXY_ERROR_LOG=/dev/stderr" -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" -p 8000:8000 -p 8443:8443 -p 127.0.0.1:8001:8001 -p 127.0.0.1:8444:8444 kong:latest
```

起動を確認します。

```bash
curl -i http://localhost:8001/
```





## API の追加と確認

API の開発は、Admin API ポートの各種エンドポイントに POST／UPDATE／DELETE などでデータを送って行っていきます。ローカルの Docker で動かしている場合は、標準で `localhost:8001` が Admin API のポートです。

> このドキュメントでは主に Curl を使用していますが、Postman のリクエストのコレクションを
> 
> https://www.getpostman.com/collections/f4c17a70d4a43cd2e26f
> 
> で公開しています。Postman を使っている方はメニューの「Import＞Import from Link」から上記 URL を読み込むとコレクションを取得できます。


最初に Service を追加します。

```bash
curl -i -X POST \
  --url http://localhost:8001/services/ \
  --data 'name=example-service' \
  --data 'url=http://mockbin.org'
```

1行バージョン：

```bash
curl -i -X POST --url http://localhost:8001/services/ --data 'name=example-service' --data 'url=http://mockbin.org'
```


以下のようなレスポンスが返ってくれば成功です。

```bash
HTTP/1.1 201 Created
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 361
Content-Type: application/json; charset=utf-8
Date: Fri, 25 Sep 2020 08:34:09 GMT
Server: kong/2.1.4
X-Kong-Admin-Latency: 5

{
    "ca_certificates": null,
    "client_certificate": null,
    "connect_timeout": 60000,
    "created_at": 1601022849,
    "host": "mockbin.org",
    "id": "8bd4fae3-4878-4cdc-9feb-4b9369373bab",
    "name": "example-service",
    "path": null,
    "port": 80,
    "protocol": "http",
    "read_timeout": 60000,
    "retries": 5,
    "tags": null,
    "tls_verify": null,
    "tls_verify_depth": null,
    "updated_at": 1601022849,
    "write_timeout": 60000
}
```


POST データに含める `name` が作成した Service の名前（一意である必要があります）で、`url` が転送先の API のエンドポイントです。今回は [httpbin](http://httpbin.org) を使用します。

次に作成したサービスにルートを追加します。

```bash
curl -i -X POST \
  --url http://localhost:8001/services/example-service/routes \
  --data 'hosts[]=example.com'
```

1行バージョン：

```bash
curl -i -X POST --url http://localhost:8001/services/example-service/routes --data 'hosts[]=example.com'
```


次のようなレスポンスが返ってくれば成功です。

```bash
HTTP/1.1 201 Created
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 429
Content-Type: application/json; charset=utf-8
Date: Fri, 25 Sep 2020 09:00:53 GMT
Server: kong/2.1.4
X-Kong-Admin-Latency: 6

{
    "created_at": 1601024453,
    "destinations": null,
    "headers": null,
    "hosts": [
        "example.com"
    ],
    "https_redirect_status_code": 426,
    "id": "be9ad0a3-2beb-4019-8deb-3686ee60d39c",
    "methods": null,
    "name": null,
    "path_handling": "v0",
    "paths": null,
    "preserve_host": false,
    "protocols": [
        "http",
        "https"
    ],
    "regex_priority": 0,
    "service": {
        "id": "8bd4fae3-4878-4cdc-9feb-4b9369373bab"
    },
    "snis": null,
    "sources": null,
    "strip_path": true,
    "tags": null,
    "updated_at": 1601024453
}
```

Routes は指定した Host がリクエストヘッダーにあった場合に、紐付けた Service の返信を返す設定を行います。

アクセスして確かめてみます。動作しているユーザー向けポートはデフォルトでは `8000` で、先ほど作成した Route では `example.com` のヘッダーを受け付けるので、次のコマンドでアクセスしてみましょう。

```bash
curl -i -X GET \
  --url http://localhost:8000/ \
  --header 'Host: example.com'
```

1行バージョン：

```bash
curl -i -X GET --url http://localhost:8000/ --header 'Host: example.com'
```

次のようなレスポンスが返ってきて、正しく転送されていることがわかります。

```bash
HTTP/1.1 200 OK
Connection: keep-alive
Content-Encoding: gzip
Content-Type: text/html; charset=utf-8
Date: Fri, 25 Sep 2020 09:06:45 GMT
Etag: W/"29c7-XG+PICJmz/J+UYWt5gkKqqAUXjc"
Kong-Cloud-Request-ID: d882b1d9b65eceeb0b0fd9c74bc90671
Server: Cowboy
Transfer-Encoding: chunked
Vary: Accept-Encoding
Via: kong/2.1.4
X-Kong-Proxy-Latency: 285
X-Kong-Upstream-Latency: 559
X-Kong-Upstream-Status: 200

<!DOCTYPE html><html>...
```



### Power Shell ユーザー向け TIPS

Windows で Power Shell を使う方は、curl と同じような `Invoke-WebRequest` というメソッドが用意されています。例えば

```powershell
Invoke-WebRequest -uri "http://localhost:8001/services"
```

でレスポンスのオブジェクトを取れるので、ドットシンタックスでオブジェクトの内部の値を確認したり、上記の戻り値を変数に格納して確認したりできます。（`iwr` は `Invoke-WebRequest` のエイリアスです。）

**ヘッダーを表示**

```powershell
(iwr -uri "http://localhost:8001/services").Headers
```

**Response を取得して Content を表示**

```powershell
$response = iwr -uri "http://localhost:8001/services"
$response.Content
```

また、Service を追加する時は

```powershell
$response = iwr -method post -uri "http://localhost:8001/services" -body 'name=example-service&url=http://httpbin.org'
$response.statuscode
```

とすると、成功していれば StatusCode として 201 が返ってきているのが取得できます。

`Invoke-WebRequest` については、[Microsoft の公式ページ](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest?view=powershell-6)に結構詳しくパラメーターや使い方が載っていますので、興味のある方は見てみてください。




## Plugin で認証機能を追加

次に、ユーザー認証によるアクセス制限を行います。

```bash
curl -i -X POST \
  --url http://localhost:8001/services/example-service/plugins/ \
  --data 'name=key-auth'
```

1行バージョン：

```bash
curl -i -X POST --url http://localhost:8001/services/example-service/plugins/ --data 'name=key-auth'
```

> `/test/plugins/` として、API 別に Plugin を追加することも可能ですし、グローバルに追加することも可能です。

次のようなレスポンスが返ってくれば成功です。

```bash
HTTP/1.1 201 Created
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 363
Content-Type: application/json; charset=utf-8
Date: Fri, 25 Sep 2020 09:11:27 GMT
Server: kong/2.1.4
X-Kong-Admin-Latency: 6

{
    "config": {
        "anonymous": null,
        "hide_credentials": false,
        "key_in_body": false,
        "key_names": [
            "apikey"
        ],
        "run_on_preflight": true
    },
    "consumer": null,
    "created_at": 1601025087,
    "enabled": true,
    "id": "ca2af9d6-e83e-4fc4-866b-579370397e31",
    "name": "key-auth",
    "protocols": [
        "grpc",
        "grpcs",
        "http",
        "https"
    ],
    "route": null,
    "service": {
        "id": "8bd4fae3-4878-4cdc-9feb-4b9369373bab"
    },
    "tags": null
}
```



もう一度、同じ API にアクセスしてみます。

```bash
curl -i -X GET \
  --url http://localhost:8000/ \
  --header 'Host: example.com'
```

1行バージョン：

```bash
curl -i -X GET --url http://localhost:8000/ --header 'Host: example.com'
```

今度は以下のように認証エラーが返ってきました。


```bash
HTTP/1.1 401 Unauthorized
Connection: keep-alive
Content-Length: 45
Content-Type: application/json; charset=utf-8
Date: Fri, 25 Sep 2020 09:14:11 GMT
Server: kong/2.1.4
WWW-Authenticate: Key realm="kong"
X-Kong-Response-Latency: 11

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

1行バージョン：

```bash
curl -i -X POST --url http://localhost:8001/consumers/ --data "username=Jason"
```

Jason さんが追加されました。

```bash
HTTP/1.1 201 Created
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 117
Content-Type: application/json; charset=utf-8
Date: Fri, 25 Sep 2020 09:16:17 GMT
Server: kong/2.1.4
X-Kong-Admin-Latency: 5

{
    "created_at": 1601025377,
    "custom_id": null,
    "id": "96af1e5a-45e4-419e-9dc5-18ed28aabd9b",
    "tags": null,
    "username": "Jason"
}
```

そのまま Jason に対して `API Key` を発行します。

```bash
curl -i -X POST \
  --url http://localhost:8001/consumers/Jason/key-auth/ \
  --data 'key=ENTER_KEY_HERE'
```

1行バージョン：

```bash
curl -i -X POST --url http://localhost:8001/consumers/Jason/key-auth/ --data 'key=ENTER_KEY_HERE'
```

次のような json が返ってきて、`id` と `key` が発行されたことがわかります。

```bash
HTTP/1.1 201 Created
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 172
Content-Type: application/json; charset=utf-8
Date: Fri, 25 Sep 2020 09:19:11 GMT
Server: kong/2.1.4
X-Kong-Admin-Latency: 5

{
    "consumer": {
        "id": "96af1e5a-45e4-419e-9dc5-18ed28aabd9b"
    },
    "created_at": 1601025551,
    "id": "5c743e87-22cb-4d1d-9f0b-dc5b0fcdc198",
    "key": "ENTER_KEY_HERE",
    "tags": null,
    "ttl": null
}
```

これで、この apikey で最初の API にアクセスできるようになりました。

> key をランダム生成する場合は、`--data ''` と、実データなしでリクエストを送ります。

```bash
curl -i -X GET \
  --url http://localhost:8000 \
  --header "Host: example.com" \
  --header "apikey: ENTER_KEY_HERE"
```

1行バージョン：

```bash
curl -i -X GET --url http://localhost:8000 --header "Host: example.com" --header "apikey: ENTER_KEY_HERE"
```


次の結果が返ってきて、無事アクセスできたことが分かります。

```bash
HTTP/1.1 200 OK
Connection: keep-alive
Content-Encoding: gzip
Content-Type: text/html; charset=utf-8
Date: Fri, 25 Sep 2020 09:22:04 GMT
Etag: W/"29c7-XG+PICJmz/J+UYWt5gkKqqAUXjc"
Kong-Cloud-Request-ID: 96772f7d2c07a855da0161b5556c845d
Server: Cowboy
Transfer-Encoding: chunked
Vary: Accept-Encoding
Via: kong/2.1.4
X-Kong-Proxy-Latency: 116
X-Kong-Upstream-Latency: 497
X-Kong-Upstream-Status: 200

<!DOCTYPE html><html>...
```

Key-Auth プラグインの詳細は、[公式ドキュメント](https://docs.konghq.com/hub/kong-inc/key-auth/)をご覧ください。





## おまけ：レートリミット（まだアップデートしていません。すみません。）

時間に余裕があれば、Rate Limit を掛けてみましょう。

Plugin 名は `rate-limiting` で今回は秒単位で 1回、分単位で 5回の制限を掛けてみます。

```bash
curl -i -X POST \
    --url http://localhost:8001/services/test/plugins \
    --data "name=rate-limiting" \
    --data "config.second=1" \
    --data "config.minute=5"
```

1行バージョン：

```bash
curl -i -X POST --url http://localhost:8001/services/test/plugins --data "name=rate-limiting" --data "config.second=1" --data "config.minute=5"
```


エンドポイントにアクセスします。1 秒以内に 2回、または 1分以内に 6回アクセスすると拒否されるはずですので、以下のコマンドを 2行分一気に貼り付けたりしてみましょう。

```bash
curl -i -X GET \
    --url "http://localhost:8000/test/get?data=value1&data=value2" \
    --header "apikey: ENTER_KEY_HERE"
```

1行バージョン：

```bash
curl -i -X GET --url "http://localhost:8000/test/get?data=value1&data=value2" --header "apikey: ENTER_KEY_HERE"
```

1秒間に 2回アクセスすると、次のような JSON が返ってきて、1秒内の制限に引っ掛かっていますが、1分内には残り 2回アクセスできることが分かります。

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

### 作成された API 一覧

```bash
curl -i -X GET \
    --url http://localhost:8001/services/
```

### 作成した API の削除

```bash
curl -i -X DELETE \
    --url http://localhost:8001/services/{API}
```

### Plugin の変更

```bash
curl -i -X PATCH \
    --url http://localhost:8001/services/{API}/plugins/{id} \
    --data "config.{property}"
```

などです。非常に簡単に操作できる API Management の Kong をぜひ触ってみてください。Kong についての詳細は

- [Kong 本家](https://konghq.com/kong-community-edition/)
- [エクセルソフト Kong ページ](https://www.xlsoft.com/jp/products/kong/index.html?r=ytgh)
- [エクセルソフトブログ Kong タグ](http://www.xlsoft.com/jp/blog/blog/tag/kong/?r=ytgh)

をご覧ください。

以上です。
