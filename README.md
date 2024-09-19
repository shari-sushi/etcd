# etcd マルチメンバーで触ってみた。

- 実務で使用しているが、良くわかっていない
- そもそも kvs について、web で redis が人気(?)なことくらいしか分かってない。
- write/read 両方の可用性、負荷分散を軽量的に実現
  - マルチノード
  - Raft でリーダーとメンバーと候補者がすごい
  - ミラーリングとストライピング。

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
  docker network inspect etcd-network
  docker network rm etcd-network
  ```
  - 3: 動作確認
    write
  ```
  $ curl -X POST -d '{"key": "hoge","value": "hogehoge"}' http://localhost:12379/v3/kv/put
  {"header":{"cluster_id":"10899443767562795482","member_id":"10670599892879733166","revision":"2","raft_term":"2"}}
  ```
  read
  ```
  $ curl -L http://localhost:12379/v3/kv/range -X POST -d '{"key": "hoge"}'
  {"header":{"cluster_id":"10899443767562795482","member_id":"10670599892879733166","revision":"2","raft_term":"2"},"kvs":[{"key":"hoge","create_revision":"2","mod_revision":"2","version":"1","value":"hogehoge"}],"count":"1"}
  ```
  `"key":"hoge"`に対して、以下が保存されていたが保存されていた。
  - `"create_revision":"2"`
  - `"mod_revision":"2"`
  - `"version":"1"`
  - `"value":"hogehoge"`
