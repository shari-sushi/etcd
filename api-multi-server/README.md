やりたかった
https://etcd.io/docs/v3.4/dev-guide/interacting_v3/
~~とあまりにコマンドが違う(etcd のコマンドがどのサービスのどのメソッドに該当するのか変換する必要があった)ことに途中で気づいたのでボツにした。~~
~~[API reference](https://etcd.io/docs/v3.4/dev-guide/api_reference_v3/)~~
~~proto でやってるぽいので API というより gRPC?~~

と思ったら、それをhttps://github.com/etcd-io/etcd/blob/main/etcdctl/README.md#etcdctl で動かすらしい。

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
    ※ `-X`=
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

  ```
  $ export ETCDCTL_ENDPOINTS=http://localhost:12379,http://localhost:12381
  $ etcdctl endpoint health
  http://localhost:12379 is healthy: successfully committed proposal: took = 2.395484ms
  http://localhost:12381 is healthy: successfully committed proposal: took = 2.60832ms
  ```
