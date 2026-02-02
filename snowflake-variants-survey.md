# Snowflake派生分散ID生成ライブラリ調査結果

調査日: 2026-02-02

## 比較表

| ライブラリ | マシンID管理方法 | 動的マシンID割り当て | フォールバック機能 | 動的リース機構 |
|:--|:--|:--|:--|:--|
| **Twitter Snowflake** (オリジナル) | ZooKeeper または 設定ファイル | △ ZooKeeperで起動時に割当 | × なし | × なし（永続ノード） |
| **Sony Sonyflake** | プライベートIPの下位16bit（デフォルト） | × 静的（IP依存） | × なし | × なし |
| **Baidu uid-generator** | データベース（WORKER_NODE テーブル） | ○ 起動時にDB自動割当 | × なし | × なし（使い捨て方式） |
| **美团 Leaf** | ZooKeeper（永続順序ノード） | ○ ZK順序ノードで自動割当 | × なし | × なし（永続ノード） |
| **Discord Snowflake** | 静的設定（worker ID + process ID） | × 静的設定 | × なし | × なし |
| **CosId** | Redis/ZK/JDBC/MongoDB/K8s StatefulSet | ○ 複数バックエンドで動的割当 | △ LocalMachineStateStorageでキャッシュ | × 明示的リースなし |
| **Icicle** | Redis + Lua（手動shard ID設定） | × 静的設定（0-1023） | △ 複数Redisでラウンドロビン | × なし |
| **Didi Tinyid** | データベース（号段モード） | ○ DB自動割当 | ○ 複数DBサポート | × なし（号段方式） |
| **Seata IdWorker** | MACアドレス下位10bit | × 静的（MAC依存）、失敗時ランダム | × なし | × なし |

---

## 各ライブラリの詳細

### 1. Twitter Snowflake（オリジナル）

- **リポジトリ**: アーカイブ済み
- **マシンID管理**: 10bit（5bit datacenter + 5bit worker）。起動時にZooKeeperで取得可能だが、設定ファイルでの静的設定も可
- **動的割り当て**: ZooKeeperで起動時に割り当て可能
- **フォールバック**: なし（中央サーバ不要の分散設計）
- **リース機構**: なし

参考: [Announcing Snowflake](https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake), [Wikipedia - Snowflake ID](https://en.wikipedia.org/wiki/Snowflake_ID)

---

### 2. Sony Sonyflake

- **リポジトリ**: [github.com/sony/sonyflake](https://github.com/sony/sonyflake)
- **マシンID管理**: 16bit。デフォルトはプライベートIPアドレスの下位16bit
- **動的割り当て**: なし（IPベースで自動決定だが、動的な中央管理はなし）
- **フォールバック**: なし
- **リース機構**: なし
- **特徴**: `CheckMachineID`コールバックで外部検証可能。`awsutil`パッケージでEC2メタデータ取得をサポート

参考: [Understanding Distributed ID Generation with Sonyflake](https://medium.com/@sanhdoan/understanding-distributed-id-generation-with-sonyflake-a-twitter-snowflake-implementation-in-go-e4aab981bfb2)

---

### 3. Baidu uid-generator

- **リポジトリ**: [github.com/baidu/uid-generator](https://github.com/baidu/uid-generator)
- **マシンID管理**: 22bit（最大420万ノード）。データベース（WORKER_NODEテーブル）で管理
- **動的割り当て**: ○ 起動時に`DisposableWorkerIdAssigner`がDBから自動割り当て
- **フォールバック**: なし
- **リース機構**: なし（「使い捨て」方式 - 再起動時は新しいIDを取得）
- **特徴**: RingBufferによるID事前生成でCachedUidGeneratorが高性能

参考: [Baidu uid-generator README](https://github.com/baidu/uid-generator/blob/master/README.md)

---

### 4. 美团 Leaf

- **リポジトリ**: [github.com/Meituan-Dianping/Leaf](https://github.com/Meituan-Dianping/Leaf)
- **マシンID管理**: ZooKeeperの永続順序ノード（PERSISTENT_SEQUENTIAL）で管理
- **動的割り当て**: ○ 起動時にZKで自動的に順序IDを取得
- **フォールバック**: なし（ZKが必須）
- **リース機構**: なし（永続ノード使用、エフェメラルノードではない）
- **特徴**: 号段モード（Segment Mode）とSnowflakeモードの両方をサポート。時刻回帰問題への対策あり（3秒ごとにタイムスタンプ更新）

参考: [Leaf README_CN](https://github.com/Meituan-Dianping/Leaf/blob/master/README_CN.md)

---

### 5. Discord Snowflake

- **マシンID管理**: 10bit（5bit worker + 5bit process）、静的設定
- **動的割り当て**: なし
- **フォールバック**: なし
- **リース機構**: なし
- **特徴**: エポックは2015年1月1日。各サーバで独立してID生成

参考: [Discord Snowflake Explained](https://medium.com/netcord/discord-snowflake-explained-id-generation-process-a468be00a570)

---

### 6. CosId

- **リポジトリ**: [github.com/Ahoo-Wang/CosId](https://github.com/Ahoo-Wang/CosId)
- **マシンID管理**: 複数のMachineIdDistributorをサポート
  - `ManualMachineIdDistributor`: 手動設定
  - `StatefulSetMachineIdDistributor`: K8s StatefulSetのホスト名から取得
  - `RedisMachineIdDistributor`: Redis経由で動的割り当て
  - `JdbcMachineIdDistributor`: RDB経由で動的割り当て
  - `ZookeeperMachineIdDistributor`: ZK経由で動的割り当て
  - `MongoMachineIdDistributor`: MongoDB経由で動的割り当て
- **動的割り当て**: ○ 複数バックエンドで対応
- **フォールバック**: △ `LocalMachineStateStorage`でローカルファイルに状態キャッシュ
- **リース機構**: 明示的なリースはないが、状態保存により再起動時の一貫性を確保
- **特徴**: 時刻回帰検知、Machine-Id-Safe-Guard機能

参考: [CosId Official Documentation](https://cosid.ahoo.me/), [SnowflakeId | CosId](https://cosid.ahoo.me/guide/snowflake.html)

---

### 7. Icicle

- **リポジトリ**: [github.com/intenthq/icicle](https://github.com/intenthq/icicle)
- **マシンID管理**: 10bit shard ID。各Redisインスタンスに手動でshard IDを設定（0-1023）
- **動的割り当て**: なし（静的設定）
- **フォールバック**: △ 複数Redisノードにラウンドロビンで分散
- **リース機構**: なし
- **特徴**: Luaスクリプトで原子的にID生成。最低2台のRedis推奨

参考: [Distributed ID generation with Redis and Lua](https://medium.com/@marketing_81824/distributed-id-generation-with-redis-and-lua-35c1b4bc671a)

---

### 8. Didi Tinyid

- **リポジトリ**: [github.com/didi/tinyid](https://github.com/didi/tinyid)
- **マシンID管理**: 号段（Segment）モード。データベースで号段を管理
- **動的割り当て**: ○ DBから号段を取得
- **フォールバック**: ○ 複数DB対応（奇数/偶数で分離）
- **リース機構**: なし（号段の事前取得とダブルバッファリング）
- **特徴**: 高可用性のため複数DBをサポート。tinyid-clientで完全ローカル生成可能

参考: [Tinyid Wiki](https://github.com/didi/tinyid/wiki)

---

### 9. Seata IdWorker

- **ドキュメント**: [Seata UUID Generator Analysis](https://seata.apache.org/blog/seata-analysis-UUID-generator/)
- **マシンID管理**: 10bit。MACアドレスの下位10bitを使用、失敗時はランダム
- **動的割り当て**: なし
- **フォールバック**: × （ただしMAC取得失敗時はランダム値にフォールバック）
- **リース機構**: なし
- **特徴**: タイムスタンプと位置を入れ替えた改良版Snowflake

参考: [Q&A on the New Version of Snowflake Algorithm](https://seata.apache.org/blog/seata-snowflake-explain/)

---

## まとめ

### 動的マシンID割り当てを持つライブラリ

- Baidu uid-generator（DB起動時割当）
- 美团 Leaf（ZK順序ノード）
- CosId（Redis/ZK/JDBC/MongoDB）
- Didi Tinyid（DB号段）

### フォールバック機能を持つライブラリ

- **Didi Tinyid**: 複数DBサポートにより、1台障害時も継続可能
- **Icicle**: 複数Redisへのラウンドロビン
- **CosId**: LocalMachineStateStorageによるローカルキャッシュ

### 動的リース機構を持つライブラリ

調査した範囲では、明示的な「リース」機構（定期的な更新が必要で、失効するとIDが解放される仕組み）を持つものは見つからなかった。

多くのライブラリは「使い捨て」（disposable）方式か「永続ノード」方式を採用しており、中央サーバの障害時には再起動後の再登録に依存している。これはSnowflake方式の根本的な設計思想（各ノードが独立してID生成できる）に起因している。

---

## kazahana との比較

| 特徴 | 既存ライブラリ | kazahana |
|------|---------------|----------|
| マシンID割当 | 起動時に取得して固定 | **定期的にリース更新** |
| 障害時 | 再起動必要 or 停止 | **フォールバックで継続** |
| ID回収 | 手動 or なし | **リース期限で自動回収** |
| 運用負荷 | ZK/DB の管理必要 | **リースサーバのみ** |

**結論**: kazahana の「動的リース」＋「フォールバック」＋「自動ID回収」の組み合わせは、調査した範囲では見当たらない独自のコンセプトである。
