# Kong ハンズオンドキュメント

> Updated on 2022/3/23

ようこそ！

このドキュメントでは [Kong の公式ドキュメント](https://docs.konghq.com/gateway/) に沿って [Kong Gateway（旧 Kong Community Edition／以下 Kong）](https://konghq.com/kong/) のハンズオンを行います。

## 事前準備

ハンズオンでは、Kong の Docker イメージ（Linux）を使用します。また、Kong へのアクセスに `Curl` または [Postman](https://www.getpostman.com/) を使用します。

- Docker が動作する環境
- Linux にリモートログインする SSH クライアント
    - Windows の方は [TeraTerm](https://ttssh2.osdn.jp/)／[PuTTY](https://www.putty.org/)／[rlogin](http://nanno.dip.jp/softlib/man/rlogin/) などをご用意ください。
    - macOS／Linux の方は標準のターミナルをご利用ください。
- POST できる TCP クライアントツール
    - Windows の方は [Postman](https://www.getpostman.com/apps) が良いでしょう。どうしてもコマンドで操作したい方は、PowerShell 3.0 以上で使える [`Invoke-WebRequest`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest?view=powershell-6) コマンドが `Curl` に似た動きをするので使ってみても良いかもしれません。
    - macOS／Linux の方は Curl が標準でインストールされています。必要に応じて Postman を使用してください。



## インストール

[Install Kong Gateway on Docker \- v2\.8\.x \| Kong Docs](https://docs.konghq.com/gateway/2.8.x/install-and-run/docker/) のドキュメントの意訳です。

最初にコンテナー同士が発見や通信ができるようにカスタムネットワークをを作ります。Docker 環境で以下を実行します。

```sh
docker network create kong-net
```

次に Postgres のデータベースを起動します。

```sh
docker run -d --name kong-database \
  --network=kong-net \
  -p 5432:5432 \
  -e "POSTGRES_USER=kong" \
  -e "POSTGRES_DB=kong" \
  -e "POSTGRES_PASSWORD=kongpass" \
  postgres:9.6
```

1行バージョン：

```sh
docker run -d --name kong-database --network=kong-net -p 5432:5432 -e "POSTGRES_USER=kong" -e "POSTGRES_DB=kong" -e "POSTGRES_PASSWORD=kongpass" postgres:9.6
```

次にデータベースを用意します。`KONG_DATABASE` の環境変数で利用しているデータベースを指定します。`KONG_PASSWORD` （エンタープライズのみ）は Kong Gateway の Admin Super User のデフォルトのパスワードです。

```sh
docker run --rm --network=kong-net \
  -e "KONG_DATABASE=postgres" \
  -e "KONG_PG_HOST=kong-database" \
  -e "KONG_PG_PASSWORD=kongpass" \
  -e "KONG_PASSWORD=test" \
 kong/kong-gateway:2.8.0.0-alpine kong migrations bootstrap
```

1行バージョン：

```sh
docker run --rm --network=kong-net -e "KONG_DATABASE=postgres" -e "KONG_PG_HOST=kong-database" -e "KONG_PG_PASSWORD=kongpass" -e "KONG_PASSWORD=test" kong/kong-gateway:2.8.0.0-alpine kong migrations bootstrap
```


以下のようなメッセージが表示され、データベースの準備ができます。

```sh
XX migrations processed
XX executed
database is up-to-date
```

準備が完了したら、Kong のコンテナーを起動します。

> Note: Enterprise のライセンスをお持ちの場合は次のようにライセンスを export してください。その際に正しい JSON になるように `’` や `“` ではなく `'` や `"` のようなまっすぐなクォートを利用してください。
>     
> `export KONG_LICENSE_DATA='{"license":{"payload":{"admin_seats":"1","customer":"Example Company, Inc","dataplanes":"1","license_creation_date":"2017-07-20","license_expiration_date":"2017-07-20","license_key":"00141000017ODj3AAG_a1V41000004wT0OEAU","product_subscription":"Konnect Enterprise","support_plan":"None"},"signature":"6985968131533a967fcc721244a979948b1066967f1e9cd65dbd8eeabe060fc32d894a2945f5e4a03c1cd2198c74e058ac63d28b045c2f1fcec95877bd790e1b","version":"1"}}'` 


```sh
docker run -d --name kong-gateway \
  --network=kong-net \
  -e "KONG_DATABASE=postgres" \
  -e "KONG_PG_HOST=kong-database" \
  -e "KONG_PG_USER=kong" \
  -e "KONG_PG_PASSWORD=kongpass" \
  -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
  -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
  -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_LISTEN=0.0.0.0:8001" \
  -e "KONG_ADMIN_GUI_URL=http://localhost:8002" \
  -e KONG_LICENSE_DATA \
  -p 8000:8000 \
  -p 8443:8443 \
  -p 8001:8001 \
  -p 8444:8444 \
  -p 8002:8002 \
  -p 8445:8445 \
  -p 8003:8003 \
  -p 8004:8004 \
  kong/kong-gateway:2.8.0.0-alpine
```

1行バージョン：

```sh
docker run -d --name kong-gateway --network=kong-net -e "KONG_DATABASE=postgres" -e "KONG_PG_HOST=kong-database" -e "KONG_PG_USER=kong" -e "KONG_PG_PASSWORD=kongpass" -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" -e "KONG_PROXY_ERROR_LOG=/dev/stderr" -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" -e "KONG_ADMIN_LISTEN=0.0.0.0:8001" -e "KONG_ADMIN_GUI_URL=http://localhost:8002" -e KONG_LICENSE_DATA -p 8000:8000 -p 8443:8443 -p 8001:8001 -p 8444:8444 -p 8002:8002 -p 8445:8445 -p 8003:8003 -p 8004:8004 kong/kong-gateway:2.8.0.0-alpine
```

起動を確認します。

```sh
curl -i -X GET --url http://localhost:8001/services
```

以下のようなステータスコード 200 のレスポンスが返ってくれば成功です。

```json
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: http://localhost:8002
Connection: keep-alive
Content-Length: 23
Content-Type: application/json; charset=utf-8
Date: Wed, 23 Mar 2022 01:58:54 GMT
Server: kong/2.8.0.0-enterprise-edition
X-Kong-Admin-Latency: 2
X-Kong-Admin-Request-ID: HhwSxGvLHi0akP5MhAjInq01oNPBGmWl
vary: Origin

{
  "data": [],
  "next": null
}
```

続いて `KONG_ADMIN_GUI_URL` で指定した URL にアクセスし、Kong Manager が実行されていることを確認します。

```text
http://localhost:8002
```

Kong Gateway のテストが終了し、コンテナーが不要になった場合は、次のコマンドを使用してコンテナーをクリーンアップできます。

```sh
docker kill kong-dbless
docker container rm kong-dbless
docker network rm kong-net
```





## API の追加と確認

[Configuring a Service \- v2\.8\.x \| Kong Docs](https://docs.konghq.com/gateway/2.8.x/get-started/quickstart/configuring-a-service/) の意訳です。

API の開発は、Admin API ポートの各種エンドポイントに POST／UPDATE／DELETE などでデータを送って行っていきます。ローカルの Docker で動かしている場合は、標準で `localhost:8001` が Admin API のポートです。（[Kong の公式ドキュメント](https://docs.konghq.com/) とは多少内容が異なりますが、より理解しやすい内容になっていると思います。）

> このドキュメントでは主に Curl を使用していますが、Postman のリクエストのコレクションを
> 
> https://www.getpostman.com/collections/f4c17a70d4a43cd2e26f
> 
> で公開しています。Postman を使っている方はメニューの「Import＞Import from Link」から上記 URL を読み込むとコレクションを取得できます。


最初に Service を追加します。

```sh
curl -i -X POST \
  --url http://localhost:8001/services/ \
  --data "name=httpbin" \
  --data "url=http://httpbin.org/anything"
```

1行バージョン：

```sh
curl -i -X POST --url http://localhost:8001/services/ --data "name=httpbin" --data "url=http://httpbin.org/anything"
```


以下のようなレスポンスが返ってくれば成功です。

```json
HTTP/1.1 201 Created
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: http://localhost:8002
Connection: keep-alive
Content-Length: 375
Content-Type: application/json; charset=utf-8
Date: Wed, 23 Mar 2022 02:09:14 GMT
Server: kong/2.8.0.0-enterprise-edition
X-Kong-Admin-Latency: 13
X-Kong-Admin-Request-ID: KFtl34KAroolt1JheY7oly42JJ89hlV7
vary: Origin

{
  "ca_certificates": null,
  "client_certificate": null,
  "connect_timeout": 60000,
  "created_at": 1648001354,
  "enabled": true,
  "host": "httpbin.org",
  "id": "e90f3a07-58bc-4d5c-a25b-1c0fc1c46d33",
  "name": "httpbin",
  "path": "/anything",
  "port": 80,
  "protocol": "http",
  "read_timeout": 60000,
  "retries": 5,
  "tags": null,
  "tls_verify": null,
  "tls_verify_depth": null,
  "updated_at": 1648001354,
  "write_timeout": 60000
}
```


POST データに含める `name` が作成した Service の名前（一意である必要があります）で、`url` が転送先の API のエンドポイントです。今回は [httpbin](http://httpbin.org) を使用します。

次に作成したサービスにルートを追加します。

```sh
curl -i -X POST \
  --url http://localhost:8001/services/httpbin/routes \
  --data "hosts[]=example.com"
```

1行バージョン：

```sh
curl -i -X POST --url http://localhost:8001/services/httpbin/routes --data "hosts[]=example.com"
```


次のようなレスポンスが返ってくれば成功です。

```json
HTTP/1.1 201 Created
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: http://localhost:8002
Connection: keep-alive
Content-Length: 480
Content-Type: application/json; charset=utf-8
Date: Wed, 23 Mar 2022 02:11:24 GMT
Server: kong/2.8.0.0-enterprise-edition
X-Kong-Admin-Latency: 13
X-Kong-Admin-Request-ID: pPuOuDPcwPkvYXpOTGWgke7va65D5bfv
vary: Origin

{
  "created_at": 1648001484,
  "destinations": null,
  "headers": null,
  "hosts": [
    "example.com"
  ],
  "https_redirect_status_code": 426,
  "id": "e5fb4ac5-3cbe-45ca-9368-6be0672ec6fc",
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
  "request_buffering": true,
  "response_buffering": true,
  "service": {
    "id": "e90f3a07-58bc-4d5c-a25b-1c0fc1c46d33"
  },
  "snis": null,
  "sources": null,
  "strip_path": true,
  "tags": null,
  "updated_at": 1648001484
}
```

Routes は指定した Host がリクエストヘッダーにあった場合に、紐付けた Service の返信を返す設定を行います。

アクセスして確かめてみます。動作しているユーザー向けポートはデフォルトでは `8000` で、先ほど作成した Route では `example.com` のヘッダーを受け付けるので、次のコマンドでアクセスしてみましょう。

```sh
curl -i -X GET \
  --url http://localhost:8000/?arg=value \
  --header "host: example.com"
```

1行バージョン：

```sh
curl -i -X GET --url http://localhost:8000/?arg=value --header "host: example.com"
```

次のようなレスポンスが返ってきて、正しく転送されていることがわかります。

```json
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 500
Content-Type: application/json
Date: Wed, 23 Mar 2022 02:12:33 GMT
Server: gunicorn/19.9.0
Via: kong/2.8.0.0-enterprise-edition
X-Kong-Proxy-Latency: 164
X-Kong-Upstream-Latency: 535

{
  "args": {
    "arg": "value"
  },
  "data": "",
  "files": {},
  "form": {},
  "headers": {
  "Accept": "*/*",
  "Host": "httpbin.org",
  "User-Agent": "curl/7.68.0",
    "X-Amzn-Trace-Id": "Root=1-XXXXXXXX-XXXXXXXXXXXXXXXXXXXXXXXX",
    "X-Forwarded-Host": "example.com",
    "X-Forwarded-Path": "/"
  },
  "json": null,
  "method": "GET",
  "origin": "172.20.0.1, XXX.XXX.XXX.XXX",
  "url": "http://example.com/anything?arg=value"
}
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
$response = iwr -method post -uri "http://localhost:8001/services" -body 'name=httpbin&url=http://httpbin.org/anything'
$response.statuscode
```

とすると、成功していれば StatusCode として 201 が返ってきているのが取得できます。

`Invoke-WebRequest` については、[Microsoft の公式ページ](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest?view=powershell-6)に結構詳しくパラメーターや使い方が載っていますので、興味のある方は見てみてください。




## Plugin で認証機能を追加

次に、ユーザー認証によるアクセス制限を行います。

```sh
curl -i -X POST \
  --url http://localhost:8001/services/httpbin/plugins/ \
  --data "name=key-auth"
```

1行バージョン：

```sh
curl -i -X POST --url http://localhost:8001/services/httpbin/plugins/ --data "name=key-auth"
```

> `/test/plugins/` として、API 別に Plugin を追加することも可能ですし、グローバルに追加することも可能です。

次のようなレスポンスが返ってくれば成功です。

```json
HTTP/1.1 201 Created
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: http://localhost:8002
Connection: keep-alive
Content-Length: 404
Content-Type: application/json; charset=utf-8
Date: Wed, 23 Mar 2022 02:13:46 GMT
Server: kong/2.8.0.0-enterprise-edition
X-Kong-Admin-Latency: 19
X-Kong-Admin-Request-ID: lb7Xk3gTLa3WaOc26oElMmpdrvXnny0i
vary: Origin
vary: Origin

{
  "config": {
    "anonymous": null,
    "hide_credentials": false,
    "key_in_body": false,
    "key_in_header": true,
    "key_in_query": true,
    "key_names": [
      "apikey"
    ],
    "run_on_preflight": true
  },
  "consumer": null,
  "created_at": 1648001626,
  "enabled": true,
  "id": "88065c94-3fd4-40e4-a91f-869a8951d16e",
  "name": "key-auth",
  "protocols": [
    "grpc",
    "grpcs",
    "http",
    "https"
  ],
  "route": null,
  "service": {
    "id": "e90f3a07-58bc-4d5c-a25b-1c0fc1c46d33"
  },
  "tags": null
}
```



もう一度、同じ API にアクセスしてみます。

```sh
curl -i -X GET \
  --url http://localhost:8000/?arg=value \
  --header "host: example.com"
```

1行バージョン：

```sh
curl -i -X GET --url http://localhost:8000/?arg=value --header "host: example.com"
```

今度は以下のように認証エラーが返ってきました。


```json
HTTP/1.1 401 Unauthorized
Connection: keep-alive
Content-Length: 45
Content-Type: application/json; charset=utf-8
Date: Wed, 23 Mar 2022 02:14:39 GMT
Server: kong/2.8.0.0-enterprise-edition
WWW-Authenticate: Key realm="kong"
X-Kong-Response-Latency: 41

{
  "message": "No API key found in request"
}
```

正しく Plugin が動作していることが分かりましたので、アクセスできるユーザーを追加します。

```sh
curl -i -X POST \
  --url http://localhost:8001/consumers/ \
  --data "username=Jason"
```

1行バージョン：

```sh
curl -i -X POST --url http://localhost:8001/consumers/ --data "username=Jason"
```

次のようなレスポンスが返ってくれば成功です。

```json
HTTP/1.1 201 Created
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: http://localhost:8002
Connection: keep-alive
Content-Length: 151
Content-Type: application/json; charset=utf-8
Date: Wed, 23 Mar 2022 02:15:20 GMT
Server: kong/2.8.0.0-enterprise-edition
X-Kong-Admin-Latency: 34
X-Kong-Admin-Request-ID: Ih9bVG2WnfCmuCCjD6ogegqisbWfpnp0
vary: Origin

{
  "created_at": 1648001720,
  "custom_id": null,
  "id": "2ff17742-0c27-45ee-922f-5a01ca3313b7",
  "tags": null,
  "type": 0,
  "username": "Jason",
  "username_lower": "jason"
}
```

そのまま Jason に対して `API Key` を発行します。

```sh
curl -i -X POST \
  --url http://localhost:8001/consumers/Jason/key-auth/ \
  --data "key=ENTER_KEY_HERE"
```

1行バージョン：

```sh
curl -i -X POST --url http://localhost:8001/consumers/Jason/key-auth/ --data "key=ENTER_KEY_HERE"
```

> key をランダム生成する場合は、`--data ''` と、実データなしでリクエストを送ります。

次のような json が返ってきて、`id` と `key` が発行されたことがわかります。

```json
HTTP/1.1 201 Created
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: http://localhost:8002
Connection: keep-alive
Content-Length: 172
Content-Type: application/json; charset=utf-8
Date: Wed, 23 Mar 2022 02:19:06 GMT
Server: kong/2.8.0.0-enterprise-edition
X-Kong-Admin-Latency: 11
X-Kong-Admin-Request-ID: 9R9hUnCLSerb4rF2qPpHsvtff0Fg6Rl3
vary: Origin
vary: Origin

{
  "consumer": {
    "id": "2ff17742-0c27-45ee-922f-5a01ca3313b7"
  },
  "created_at": 1648001946,
  "id": "69683811-8805-41d9-b078-0db948527b2b",
  "key": "ENTER_KEY_HERE",
  "tags": null,
  "ttl": null
}
```

これで、この apikey で最初の API にアクセスできるようになりました。



```sh
curl -i -X GET \
  --url http://localhost:8000/?arg=value \
  --header "Host: example.com" \
  --header "apikey: ENTER_KEY_HERE"
```

1行バージョン：

```sh
curl -i -X GET --url http://localhost:8000/?arg=value --header "Host: example.com" --header "apikey: ENTER_KEY_HERE"
```


次のようなレスポンスが返ってくれば成功です。

```json
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 704
Content-Type: application/json
Date: Wed, 23 Mar 2022 02:19:48 GMT
Server: gunicorn/19.9.0
Via: kong/2.8.0.0-enterprise-edition
X-Kong-Proxy-Latency: 171
X-Kong-Upstream-Latency: 431

{
  "args": {
    "arg": "value"
  },
  "data": "",
  "files": {},
  "form": {},
  "headers": {
    "Accept": "*/*",
    "Apikey": "ENTER_KEY_HERE",
    "Host": "httpbin.org",
    "User-Agent": "curl/7.68.0",
    "X-Amzn-Trace-Id": "Root=1-XXXXXXXX-XXXXXXXXXXXXXXXXXXXXXXXX",
    "X-Consumer-Id": "2ff17742-0c27-45ee-922f-5a01ca3313b7",
    "X-Consumer-Username": "Jason",
    "X-Credential-Identifier": "69683811-8805-41d9-b078-0db948527b2b",
    "X-Forwarded-Host": "example.com",
    "X-Forwarded-Path": "/"
  },
  "json": null,
  "method": "GET",
  "origin": "172.20.0.1, XXX.XXX.XXX.XXX",
  "url": "http://example.com/anything?arg=value"
}
```

Key-Auth プラグインの詳細は、[公式ドキュメント](https://docs.konghq.com/hub/kong-inc/key-auth/)をご覧ください。





## おまけ：レートリミット

時間に余裕があれば、Rate Limit を掛けてみましょう。Rate Limit は秒、分、時間、日、月、または年などに HTTP リクエストの数を制限するプラグインです。Service や Route に認証プラグインがない場合は、クライアント　IP　アドレスが使用されます。認証プラグインが構成されている場合は Consumer が使用されます。詳細はドキュメント [Rate Limiting plugin \| Kong](https://docs.konghq.com/hub/kong-inc/rate-limiting/) を参照してください。

作成していた `httpbin` サービスにプラグインを追加しましょう。Plugin 名は `rate-limiting` で今回は秒単位で 1回、分単位で 5回の制限を掛けてみます。

```sh
curl -i -X POST http://localhost:8001/services/httpbin/plugins \
  --data "name=rate-limiting"  \
  --data "config.second=1" \
  --data "config.minute=5"
```

1行バージョン：

```sh
curl -i -X POST http://localhost:8001/services/httpbin/plugins  --data "name=rate-limiting" --data "config.second=1" --data "config.minute=5"
```

次のようなレスポンスが返ってくれば成功です。

```sh
HTTP/1.1 201 Created
Date: Wed, 23 Mar 2022 02:45:51 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: http://localhost:8002
X-Kong-Admin-Request-ID: mjGHayFe2jKuaLQ1hsEpUlsPz0L7erFF
vary: Origin
Access-Control-Allow-Credentials: true
Content-Length: 639
X-Kong-Admin-Latency: 19
Server: kong/2.8.0.0-enterprise-edition

{
  "config": {
    "limit_by": "consumer",
    "second": 1,
    "minute": 5,
    "hour": null,
    "day": null,
    "month": null,
    "year": null,
    "policy": "cluster",
    "fault_tolerant": true,
    "path": null,
    "header_name": null,
    "redis_timeout": 2000,
    "redis_ssl": false,
    "redis_ssl_verify": false,
    "redis_server_name": null,
    "redis_database": 0,
    "redis_password": null,
    "redis_username": null,
    "hide_client_headers": false,
    "redis_host": null,
    "redis_port": 6379
  },
  "service": {
    "id": "e90f3a07-58bc-4d5c-a25b-1c0fc1c46d33"
  },
  "route": null,
  "id": "087dc0f4-015f-44d8-9c2f-53d2eb969ce2",
  "protocols": [
    "grpc",
    "grpcs",
    "http",
    "https"
  ],
  "enabled": true,
  "tags": null,
  "consumer": null,
  "created_at": 1648003551,
  "name": "rate-limiting"
}
```


エンドポイントにアクセスします。1 秒以内に 2回、または 1分以内に 6回アクセスすると拒否されるはずですので、以下のコマンドを 2行分一気に貼り付けたりしてみましょう。

```sh
curl -i -X GET \
  --url "http://localhost:8000?arg=value" \
  --header "apikey: ENTER_KEY_HERE" \
  --header "host: example.com"
```

1行バージョン：

```sh
curl -i -X GET --url "http://localhost:8000?arg=value" --header "apikey: ENTER_KEY_HERE" --header "host: example.com"
```

1秒間に 2回アクセスすると、次のような JSON が返ってきて、1秒内の制限に引っ掛かっていますが、1分内には残り 4回アクセスできることが分かります。

```json
HTTP/1.1 429 Too Many Requests
Date: Wed, 23 Mar 2022 02:50:19 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
X-RateLimit-Limit-Second: 1
X-RateLimit-Limit-Minute: 5
X-RateLimit-Remaining-Second: 0
X-RateLimit-Remaining-Minute: 4
RateLimit-Limit: 1
RateLimit-Remaining: 0
RateLimit-Reset: 1
Retry-After: 1
Content-Length: 41
X-Kong-Response-Latency: 7
Server: kong/2.8.0.0-enterprise-edition

{
  "message":"API rate limit exceeded"
}
```

`consumer_id` などと組み合わせることで柔軟な制限を指定できます。

これで今回のハンズオンは終了です。お疲れ様でした。


## 参考資料

本日一部使用した Kong の Admin API の一覧は

[Admin API - v2.8.x | Kong Docs](https://docs.konghq.com/gateway/2.8.x/admin-api/)

に記載されています。（リンク先のドキュメントが古い場合は右上から最新バージョンを参照してください。）

例えば、以下のようなコマンドが用意されています。

### 作成された API 一覧

```sh
curl -i -X GET \
  --url http://localhost:8001/services/
```

### 作成した API の削除

```sh
curl -i -X DELETE \
  --url http://localhost:8001/services/{API}
```

### Plugin の変更

```sh
curl -i -X PATCH \
  --url http://localhost:8001/services/{API}/plugins/{id} \
  --data "config.{property}"
```

などです。非常に簡単に操作できる API Management の Kong をぜひ触ってみてください。Kong についての詳細は

- [Kong 本家](https://konghq.com/)
- [エクセルソフト Kong ページ](https://www.xlsoft.com/jp/products/kong/index.html?utm_source=external&utm_medium=github&utm_campaign=ytabuchi_kong-handson)
- [エクセルソフトブログ Kong タグ](http://www.xlsoft.com/jp/blog/blog/tag/kong/)

をご覧ください。

以上です。
