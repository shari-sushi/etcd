やりたかった
https://etcd.io/docs/v3.4/dev-guide/interacting_v3/
~~とあまりにコマンドが違う(etcd のコマンドがどのサービスのどのメソッドに該当するのか変換する必要があった)ことに途中で気づいたのでボツにした。~~
~~[API reference](https://etcd.io/docs/v3.4/dev-guide/api_reference_v3/)~~
~~proto でやってるぽいので API というより gRPC?~~

と思ったら、それをhttps://github.com/etcd-io/etcd/blob/main/etcdctl/README.md#etcdctl で動かすらしい。

---

## メモ

- revision は全体のバージョンで、１つ変更があるたびにインクリメントするのか？
- put で宣言、再代入できる
- (同じ key に複数の value を入れれない代わり？に、ナンバリングで管理できる。)
- put
- get, del
  - prefix 接頭辞で検索
  - limit
  - --from-key 「byte がそれ以上の物」を引っ張れる。
  - 範囲検索は「以上～未満」検索…byte で見てるっぽい
  - --from-key
- del
  --prev-kv 削除時の kv を取得
- 最初から入ってるっぽい？kv <br>
  \x86\x88\x1e <br>
  \x86\x88\x1e\x86\x88\x1e <br>
- watch ... key 変更の監視
  - 現在、他の端末での操作の監視
  - 複数監視(追加可能かな)　　　？
  - --rev={No. }　変更履歴の監視(取得)　　？
  - progress
- compact ... revision のコンパクト化…compact で指定した以前の revision の value を取得６できなくなる
  ※現在の revision であれば存在の有無に無関係に出力できる。
- リース(lease: 契約、貸す) ？

  > リースの TTL が経過すると、リースは失効し、アタッチされた鍵はすべて削除される。

  - lease grant {seconds} ... {秒}のリース ID を出力(生成？)
  - put --lease={lease ID} ... 別の key に一時的に値をアタッチ(コピー的な)しておける
  - lease revolke {lease ID} ... 取り消し
  - lease keep-alive {lease ID} ... 延長
  - lease timetolive {lease ID} ... リース情報の取得　設定 TTL と残時間、添付 key

？...動作未確認

---

## 準備

参考： https://qiita.com/rin1208/items/7005207adb0dc9bff9f2

- 1: ネットワーク作成
  ```
  docker network create etcd-network --subnet=172.16.0.0/24 --ip-range=172.16.0.0/24
  ```
  ※やっちゃダメかも
- 2: 起動
  ```
  docker-compose build
  docker-compose up -d
  ```
  1 のせいか、
  ```
  $ docker-compose up
    WARN[0000] a network with name etcd-network exists but was not created by compose.
    Set `external: true` to use an existing network
  　network etcd-network was found but has incorrect label com.docker.compose.network set to ""
  ```
  となったので ↓ の後に build, up し直したら起動した。
  ```
  // 実務で作ったものか、今作ったものかcreated_atで確認
  docker network inspect etcd-network
  // 削除
  docker network rm etcd-network
  ```
  なお、rm しなくても `external: true`を設定すればいけるのかと思いきや、別のエラーが発生したので諦めた。
- 3: 動作確認
  ※ https://etcd.io/docs/v3.4/dev-guide/api_reference_v3/ の service KV を参考にしてると思われる

  - write
    ```
    $ curl -X POST -d '{"key": "hoge","value": "hogehoge"}' http://localhost:12379/v3/kv/put
    {"header":{"cluster_id":"10899443767562795482","member_id":"10670599892879733166","revision":"2","raft_term":"2"}}
    ```
  - read
    ```
    $ curl -L http://localhost:12379/v3/kv/range -X POST -d '{"key": "hoge"}'
    {"header":{"cluster_id":"10899443767562795482","member_id":"10670599892879733166","revision":"2","raft_term":"2"},"kvs":[{"key":"hoge","create_revision":"2","mod_revision":"2","version":"1","value":"hogehoge"}],"count":"1"}
    ```
    ※ `-L`=リダイレクト有効にし、リダイレクト先の情報を取得。今回はおまじないだと思う。
  - result <br>
    `"key":"hoge"`に対して、以下が保存されていた。
    - `"create_revision":"2"`
    - `"mod_revision":"2"`
    - `"version":"1"`
    - `"value":"hogehoge"`

- etcdctl (CLI ツール)の準備
  参考？：https://gist.github.com/skynet86/451c42ec3dc883e190aa7c57bc6c2acc

  ```
  # 最新バージョンとダウンロードurlをshel変数にセット(使ってるのbashだけど)
  ETCD_VER=v3.4.26
  DOWNLOAD_URL=https://github.com/etcd-io/etcd/releases/download

  # ダウンロードとインストール
  curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o etcd-${ETCD_VER}-linux-amd64.tar.gz
  tar xzvf etcd-${ETCD_VER}-linux-amd64.tar.gz
  sudo mv etcd-${ETCD_VER}-linux-amd64/etcdctl /usr/local/bin/
  rm -rf etcd-${ETCD_VER}-linux-amd64*
  ```

  ↓ インストールできたことを確認

  ```
  $ etcdctl version
  etcdctl version: 3.4.26
  API version: 3.4
  ```

  ```go
  // ↓シェル変数なので起動ごと、windowごとに必要かな
  $ export ETCDCTL_ENDPOINTS=http://localhost:12379,http://localhost:12381
  $ etcdctl endpoint health
  http://localhost:12379 is healthy: successfully committed proposal: took = 2.395484ms
  http://localhost:12381 is healthy: successfully committed proposal: took = 2.60832ms
  ```

### etcd 触ってみる

参考：https://etcd.io/docs/v3.4/dev-guide/interacting_v3/

- 基本的な動作

  ```
  $ etcdctl put foo bar
  OK
  ```

  ```
  $ etcdctl get foo
  foo
  bar
  ```

  ```
  $ etcdctl get foo --hex
  \x66\x6f\x6f
  \x62\x61\x72
  ```

  ```
  $ etcdctl get foo --print-value-only
  bar
  ```

  ```go
  // foo しか入れてないので取得できるのは１つ
  $ etcdctl get foo foo3
  foo
  bar

  // foo1~foo3 まで put
  $ etcdctl put foo1 bar1
  OK
  $ etcdctl put foo2 bar2
  OK
  $ etcdctl put foo3 bar3
  OK
  ```

- 範囲指定

  > 範囲が半開区間 foo3 を除いた[foo, foo3) に及ぶので、foo3 は除外されることに注意してください。

  ```go
  // foo ~ foo3 まで指定すると、foo2まで取得できた
  $ etcdctl get foo foo3
  foo
  bar
  foo1
  bar1
  foo2
  bar2

  // foo3はもちろんある
  $ etcdctl get foo3
  foo3
  bar3
  ```

  ```go
  // prefix
  $ etcdctl get --prefix foo
  foo
  bar
  foo1
  bar1
  foo2
  bar2
  foo3
  bar3
  ```

- 個数制限

  ```go
  // limit
  $ etcdctl get --prefix --limit=2 foo
  foo
  bar
  foo1
  bar1
  ```

- 過去のバージョンのキーを読む<br>
  挙動分からん過ぎてやばい<br>
  **revision とはなんぞや**

  ```go
  $ etcdctl put foo barbar
  OK
  $ etcdctl get foo
  foo
  barbar

  // 1, 2は出力無し
  $ etcdctl get foo --rev=1
  $ etcdctl get foo --rev=2

  // 3~6までこれ
  $ etcdctl get foo --rev=5
  foo
  bar
  // 7のみ
  $ etcdctl get foo --rev=7
  foo
  barbar
  // 8以降はエラー
  ```

  revision は全体のバージョンで、１つ変更があるたびにインクリメントするのか？

- 指定されたキーのバイト値以上のキーを読む…？

  ```go
  // fでも同じ結果。gだと��だけになる
  etcdctl get --from-key fo --keys-only
  foo

  foo1

  foo2

  foo3

  ��

  ```

- 削除

  ```
  $ etcdctl get --from-key 0 --keys-only
  foo1

  foo2

  foo3

  goo

  hoo

  ioo

  ��

  ```

  範囲は byte で見てるっぽい

  ```

  $ etcdctl get --from-key 0 --keys-only
  foo1

  foo2

  foo3

  goo

  hoo

  ioo

  ��

  $ etcdctl del foo hoo
  4
  $ etcdctl get --from-key 0 --keys-only
  hoo

  ioo

  ��

  ```

  ```

  etcdctl watch foo4
  // ここで別 bash で etcdctl put foo4 bar4bar
  PUT
  foo4
  bar4bar

  ```

  ```go
  // 都度改行
  　etcdctl watch -i
  　watch foo
  　watch foo1
  ```
