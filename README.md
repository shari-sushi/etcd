# etcd マルチメンバーで触ってみた。

- 実務で使用しているが、良くわかっていない
- サーバーサイドの cache に使う kvs について、web で redis が人気(?)ってことくらいしか知らない。
- write/read 両方の可用性、負荷分散を軽量的に実現
  - マルチノード
  - Raft でリーダーとメンバーと候補者がすごい
  - ミラーリングとストライピング。

※ 動作確認のファイル、結果は`api-multi-server`の README.md を参照のこと。

## 参考、参考になるらしいもの

- [etcd から始める分散合意システム入門](https://zenn.dev/honahuku/articles/introduction_to_consensus_algorithm)
  初見だとムズイ。
- 本家
  - \*1: https://etcd.io/docs/v3.4/dev-guide/local_cluster/
  - https://etcd.io/docs/v3.4/dev-guide/interacting_v3/

## docker-compose 解説

- etcd ノードを 2 つ起動し、network で繋げている。
- network はサブネットであり(表現の正確性自信無い)、その中の IP アドレス１つを各ノードが使用している。
- `listen-client-urls http://0.0.0.0:12379`で公開ポート指定
-
