# MonjuDB
DynamoDBライクなキーバリュー型分散データベースです

## 名前の由来
『三人寄れば文殊の知恵』から、Raftクラスターが3台揃ってデータ（知恵）を保存するイメージ

## 特徴
- DynamoDBライクなパーティション分散  
  - 一貫ハッシュでPKを分割し、ホットパーティションが出にくいよう仮想ノードで均等配置。  
  - ノード追加・削除時も再配置範囲を最小化し、スケールアウト/インを滑らかに実行。
- Raftによる強いレプリケーションを持ったクラスター  
  - 各パーティションは独立したRaftグループで多数決コミットし、強整合な書き込みを保証。  
  - リーダー切り替えはヘルスチェックと任期管理で素早く収束し、フェイルオーバー時のダウンタイムを抑制。
- PKとSKによるシンプルで高速な検索  
  - PKでパーティション確定、SKで範囲/プレフィックス検索が可能なDynamoDB風の2次元キー。  
  - クライアントからは単純なGet/Put/Query APIで扱え、内部の分散配置を意識せず使える。
- MVCC追記型アーキテクチャとWALによる信頼性  
  - 書き込みはWALに同期後、LSMツリーへ追記するMVCCでスナップショットも安全。  
  - ガベージコレクションとコンパクションで古いバージョンを整理しつつ、読み取り一貫性を維持。

## 技術スタック
- Go: 高パフォーマンスで並行処理を扱いやすい言語基盤
- Raft: 強整合なレプリケーションを支える分散合意形成アルゴリズム
- gRPC: バイナリRPCとIDLで型安全なサービス間通信
- Storage/LSM: Pebble or BadgerDB
- Gossip: HashiCorp memberlist/Serf

## アーキテクチャ
```
                     +------------------------+
                     |      Gossip Layer      |
                     | (Node Discovery & Fail |
                     |        Detection)      |
                     +------------+-----------+
                                  |
                                  v
                     +------------------------+
                     |  Consistent Hash Ring  |
                     |    (Partitioning)      |
                     +---+---------+---------+
                         |         |         
                         |         |         
         ----------------          |          ----------------
        |                           |                           |
        v                           v                           v
+---------------+        +---------------+        +---------------+
| Partition P1  |        | Partition P2  |        | Partition P3  |
|  (vnode 101)  |        |  (vnode 204)  |        |  (vnode 330)  |
+-------+-------+        +-------+-------+        +-------+-------+
        |                        |                        |
        v                        v                        v
+------------------+  +------------------+  +------------------+
|   Raft Group 1   |  |   Raft Group 2   |  |   Raft Group 3   |
| Leader | F | F   |  | Leader | F | F   |  | Leader | F | F   |
|  (N=3 replicas)  |  |  (N=3 replicas)  |  |  (N=3 replicas)  |
+---------+--------+  +---------+--------+  +---------+--------+
          |                     |                     |
          v                     v                     v
+------------------+  +------------------+  +------------------+
|   Storage (LSM)  |  |   Storage (LSM)  |  |   Storage (LSM)  |
|     + WAL        |  |     + WAL        |  |     + WAL        |
+------------------+  +------------------+  +------------------+
```

## 整合性モデル
- 結果整合性: 読み取りは最寄りノード(Leader/Followers)から返し、低レイテンシを優先。
- 書き込みは Raft グループでのクォーラムコミット (例: N=3/W=2) を基本とし、不可時は hinted handoff キューに退避。

## API仕様 (gRPC)
- シンプルなKV API。値はバイナリで任意フォーマットを格納可能。
- 本APIは結果整合性のみを対象とし、読み取りはフォロワーからも返される。

```proto
syntax = "proto3";
package monju.v1;

service MonjuKV {
  rpc Put(PutRequest) returns (PutResponse);
  rpc Get(GetRequest) returns (GetResponse);
  rpc Query(QueryRequest) returns (QueryResponse);
}

message PutRequest {
  string pk = 1;
  string sk = 2;
  bytes value = 3;
}

message PutResponse {
  int64 version = 1; // コミットされたバージョン
}

message GetRequest {
  string pk = 1;
  string sk = 2;
}

message GetResponse {
  bool found = 1;
  bytes value = 2;
  int64 version = 3;
}

message QueryRequest {
  string pk = 1;
  string sk_prefix = 2; // 先頭一致
  string start_sk = 3;  // ページング開始位置
  int32 limit = 4;
  bool ascending = 5;
}

message QueryResponse {
  repeated Item items = 1;
  string next_start_sk = 2; // 次ページの開始位置（なければ空）
}

message Item {
  string pk = 1;
  string sk = 2;
  bytes value = 3;
  int64 version = 4;
}
```

## この開発で学べること
- 一貫ハッシュ＋仮想ノードによる分散設計と、ノード増減時の再配置最適化
- Raft実装の実践: 任期管理、ログ複製、リーダー選出、スナップショット/ログ圧縮
- DynamoDBライクなPK/SK設計でのアクセスパターン設計とクエリ効率化
- MVCCとWALを組み合わせた耐障害性の高いストレージ(LSM)の設計・実装
- Gossipによるノード検出/障害検知とリングメタデータの収束戦略
- gRPCベースの分散API設計と運用監視（ヘルスチェック、メトリクス、トレーシング）
