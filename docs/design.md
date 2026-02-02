# kazahana 設計書

## 1. 概要

本システムは Twitter Snowflake にインスパイアされた64bit分散ID生成システムである。
Snowflake の利点（時刻順ソート可能、高性能、64bit）を維持しつつ、マシンID管理の運用負荷を軽減することを目的とする。

### 1.1 設計目標

1. **DBアクセスなしでユニークID生成** - 中央DBへの負荷集中を回避
2. **インデックス性能の維持** - 時刻順でソート可能、B-tree効率良好
3. **連番の非公開** - 外部から推測困難
4. **運用容易性** - 静的なマシンID設定が不要
5. **柔軟性** - スタンドアロンから大規模分散まで対応
6. **障害耐性** - リースサーバ障害時もID生成継続可能

---

## 2. ID構造

### 2.1 ビット配置（デフォルト）

```
64bit 全体構成:
┌─────────┬──────────────┬──────────────┬──────────────┐
│ reserve │  timestamp   │  machine id  │   sequence   │
│  1bit   │    41bit     │    14bit     │    8bit      │
└─────────┴──────────────┴──────────────┴──────────────┘

0_[41bit timestamp]_[14bit machineId]_[8bit sequence]
```

### 2.2 各フィールド詳細

| フィールド | ビット数 | 説明 |
|-----------|---------|------|
| reserve | 1bit | 常に0。signed 64bit整数で正の数を保証 |
| timestamp | 41bit | カスタムエポックからのミリ秒。約69年分 |
| machineId | 14bit | リースサーバから動的に取得。0-8191は通常、8192-16383はフォールバック用 |
| sequence | 8bit | 同一ミリ秒内のシーケンス番号。0-255（複数リースで拡張可能） |

### 2.3 フォールバックID構造

リースサーバと通信できない場合、以下の優先順位でフォールバック:

**優先1: キャッシュされたリースを使用**

```
0_[41bit timestamp]_1_[13bit cached machineId]_[8bit sequence]
                    ↑
                    先頭bit=1でフォールバックを示す
```

直前まで有効だったリースのmachineIdを継続使用。
リース期限内であれば他ノードが同一IDを取得することはないため、
実質的に衝突リスクは極めて低い。

**優先2: 完全ランダム**

```
0_[41bit timestamp]_1_[21bit random]
```

キャッシュがない場合（初回起動時等）のみ使用。
同一ミリ秒に複数ノードがフォールバック状態になった場合のみ衝突の可能性あり。

通常IDとフォールバックIDは名前空間が分離されており、衝突しない。

- 通常モード: machineId 0〜8191（先頭bit=0）
- フォールバックモード: machineId 8192〜16383（先頭bit=1）

### 2.4 タイムスタンプ範囲

| ビット数 | 期間 | 用途 |
|---------|------|------|
| 40bit | 約35年 | やや短い |
| 41bit | 約69年 | **推奨（デフォルト）** |
| 42bit | 約139年 | 十分すぎる |

カスタムエポックを使用することで、サービス開始時点から69年間利用可能。

### 2.5 予約ビット（reserve）の必要性

先頭1bitを常に0とする理由:

```
64bit signed整数の範囲:
  最小: -9223372036854775808（先頭bit=1）
  最大:  9223372036854775807（先頭bit=0）
```

先頭bitが1になると:
- signed整数として扱う言語/DBで負の数になる
- ソート順序が期待と異なる
- 一部のJSONパーサーで問題が発生する

41bitタイムスタンプが全て1になる（約69年後）時点でも、先頭0を維持することでsigned 64bit整数の最大値に収まる。

---

## 3. システム構成

### 3.1 コンポーネント

```
┌─────────────────┐     ┌─────────────────┐
│  Application    │────▶│    Kazahana     │
│  Server         │     │    Client       │
└─────────────────┘     └────────┬────────┘
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │    LeaseProvider       │
                    │   （抽象インターフェース） │
                    └────────────┬───────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        ▼                        ▼                        ▼
┌──────────────┐      ┌──────────────────┐      ┌──────────────┐
│ HTTP Server  │      │  Redis Direct    │      │   SQLite     │
│ Provider     │      │  Provider        │      │   Provider   │
└──────────────┘      └──────────────────┘      └──────────────┘
```

### 3.2 動作モード

| モード | 説明 | ユースケース |
|--------|------|-------------|
| スタンドアロン | provider未指定。常にフォールバック方式でID生成 | 開発環境、小規模サービス |
| マネージド | providerからリース取得してID生成 | 本番環境、大規模サービス |
| マネージド+フォールバック | 通常はマネージド、障害時にフォールバック | 高可用性が必要な環境 |

---

## 4. リースプロトコル

### 4.1 インターフェース定義

```typescript
interface LeaseProvider {
  /**
   * 新しいマシンIDをリースする
   */
  acquire(options: AcquireOptions): Promise<AcquireResponse>;

  /**
   * リースを返却する
   */
  release(options: ReleaseOptions): Promise<void>;
}

interface AcquireOptions {
  /** サービス識別子 */
  serviceId?: string;
  /** メタ情報（ホスト名、プロセスID等） */
  meta?: Record<string, string>;
  /** 追加で必要なスループット/ms */
  throughputPerMs?: number;
}

interface ReleaseOptions {
  /** マシンID */
  id: number;
  /** HMAC署名: hmac(secret, `${id}:${timestamp}`) */
  signature: string;
  /** 署名生成時のタイムスタンプ（Unix ms） */
  timestamp: number;
}
```

### 4.2 レスポンス定義

```typescript
interface AcquireResponse {
  /** 取得したリース（throughputPerMsに応じて1つ以上） */
  leases: LeaseInfo[];
}

interface LeaseInfo {
  // リース固有情報
  id: number;           // 割り当てられたマシンID
  created: number;      // リース開始時刻（Unix ms）
  expired: number;      // リース期限（Unix ms）
  secret: string;       // release用認証キー

  // ビット配置（運用開始後は変更不可）
  customEpoch: number;  // カスタムエポック（Unix ms）
  bitReserve: number;   // 予約ビット数（通常1）
  bitTs: number;        // タイムスタンプビット数（通常41）

  // ビット配置（運用中に調整される可能性あり）
  bitId: number;        // マシンIDビット数（通常14）
  bitSeq: number;       // シーケンスビット数（通常8）
}
```

**設計ポイント**:
- ビット配置情報は `LeaseInfo` に含める（各リースが単体でID生成に必要な情報を全て持つ）
- `customEpoch`, `bitReserve`, `bitTs` は運用開始後に変更不可（変更すると過去のIDと互換性がなくなる）
- `bitId`, `bitSeq` は運用中にProvider側で動的に調整される可能性がある
- Providerの実装によっては `bitSeq` を動的に調整し、返却されるリースの `bitSeq` が不揃いになることがある
- クライアントは不揃いの `bitSeq` を持つリースを保持することを前提に設計されており、要求されたスループットを満たす点は変わらない

### 4.3 throughputPerMs による複数リース取得

クライアントは、コンフィグで指定された `maxThroughputPerMs` から現在保持しているリースのスループット合計を差し引いた分を `acquire` で要求する。

```typescript
// クライアント側の計算
const currentThroughput = leases.reduce((sum, l) => sum + (1 << l.bitSeq), 0);
const needed = config.maxThroughputPerMs - currentThroughput;

if (needed > 0) {
  const response = await provider.acquire({ throughputPerMs: needed });
  // ...
}
```

```typescript
// サーバ側の計算（bitSeqが均一の場合の例）
const maxPerLease = 1 << defaultBitSeq;  // 256 (bitSeq=8)
const count = Math.ceil(throughputPerMs / maxPerLease);
```

| 要求 throughputPerMs | サーバ側計算 | 返却リース数 |
|---------------------|--------------|-------------|
| 1（デフォルト） | ceil(1/256) | 1 |
| 256 | ceil(256/256) | 1 |
| 257 | ceil(257/256) | 2 |
| 1024 | ceil(1024/256) | 4 |

**注意**: クライアントは各リースの `bitSeq` を参照してスループットを計算する必要がある（不揃いの可能性があるため）。

### 4.4 リースライフサイクル

```
1. acquire: 新規リース取得
   └─▶ 空きIDを探索 → 割り当て → secret生成 → 返却

2. 通常運用: ID生成
   └─▶ リース情報に基づきローカルでID生成

3. 期限接近時: 新規リース取得
   └─▶ 期限の90%経過で新しいacquire実行
   └─▶ 古いリースは期限切れまで併用可能

4. release: リース返却（graceful shutdown時）
   └─▶ secret検証 → リース解放 → IDが即座に再利用可能に

5. expire: リース期限切れ（release忘れの救済）
   └─▶ 自動的にIDが再利用可能に
```

### 4.5 セキュリティ

- `secret` はリース発行時にサーバ側で生成し、クライアントに返却
- `release` 時は `secret` を直接送信せず、HMAC署名を使用
- 署名生成: `hmac(secret, "${id}:${timestamp}")`
- サーバ側でtimestamp検証（±30秒程度）によりリプレイ攻撃を防止
- secretがネットワーク上を流れないため、ログに残っても安全

---

## 5. クライアント実装

### 5.1 設定

```typescript
interface KazahanaClientConfig {
  // === よく使う設定 ===

  /** リースプロバイダ。未指定でスタンドアロンモード */
  provider?: LeaseProvider;

  /** サービス識別子 */
  serviceId?: string;

  /**
   * 1ミリ秒あたりの最大ID生成数
   * この値に達するまでリースを追加取得し、超過時は次のミリ秒まで待機
   * @default 256 (1リース分)
   */
  maxThroughputPerMs?: number;

  /**
   * フォールバックID生成を無効化
   * @default false
   */
  disableFallback?: boolean;

  // === あまり変更しない設定 ===

  /**
   * リース取得失敗後の初回リトライ間隔（ミリ秒）
   * @default 1000
   */
  acquireRetryInterval?: number;

  /**
   * リトライ間隔の上限（ミリ秒）
   * @default 60000
   */
  acquireRetryMaxInterval?: number;

  /**
   * 時刻巻き戻り許容量（ミリ秒）
   * - 正の値: 指定ms以内なら待機、超過でエラー
   * - 0: 即エラー
   * - 負の値: 無制限待機
   * @default 5000
   */
  maxBackwardMs?: number;

  // === 通常は変更不要 ===

  /** タイムスタンプビット数 @default 41 */
  bitTs?: number;

  /** デフォルトエポック（スタンドアロンモード用） @default LIB_DEFAULT_EPOCH */
  defaultEpoch?: number;
}
```

### 5.2 エポック決定ロジック

```
優先順位:
1. LeaseInfo.customEpoch（API正常時、リースから取得）
2. キャッシュされたエポック（API障害時、過去に取得成功していれば）
3. config.defaultEpoch（LIB_DEFAULT_EPOCHがデフォルト）
```

### 5.3 エポック不一致検出

```typescript
const newEpoch = response.leases[0]?.customEpoch;
if (cachedEpoch && newEpoch && cachedEpoch !== newEpoch) {
  console.error(`Epoch mismatch! cached: ${cachedEpoch}, server: ${newEpoch}`);
}
```

設定ミスやサーバ側変更を早期に検出するため、不一致時に警告。

### 5.4 ID生成フロー

```
nextId() 呼び出し
    │
    ▼
┌─────────────────────────────┐
│ リースあり かつ 期限内？       │
└─────────────┬───────────────┘
              │
    ┌─────────┴─────────┐
    │ Yes              │ No
    ▼                  ▼
通常生成          ┌─────────────────┐
    │            │ provider あり？   │
    └────┐       └────────┬────────┘
         │          │         │
         │     Yes  │         │ No
         │          ▼         │
         │    acquire()       │
         │       │            │
         │  ┌────┴────┐       │
         │  │成功│失敗│       │
         │  ▼    ▼    │       │
         │ 通常 フォール◀──────┘
         │ 生成  バック
         │  │    │
         ▼  ▼    ▼
         ID 返却
```

### 5.5 リース更新戦略

```typescript
// 期限の90%経過で新規リース取得開始
const ACQUIRE_THRESHOLD = 0.9;

const elapsed = now - lease.created;
const duration = lease.expired - lease.created;

if (elapsed > duration * ACQUIRE_THRESHOLD) {
  // バックグラウンドで新規acquire（ID生成はブロックしない）
  this.acquireInBackground();
}
```

### 5.6 時刻巻き戻り対策

#### 検出と対応

| 巻き戻り量 | 対応 | 理由 |
|-----------|------|------|
| maxBackwardMs以内 | await して待機 | NTPステップ調整の典型的な範囲 |
| maxBackwardMs超 | エラーをスロー | 異常事態の可能性。手動介入を促す |

#### 設定値

| maxBackwardMs | 挙動 | ユースケース |
|---------------|------|-------------|
| 正の値（デフォルト5000） | 指定ms以内なら待機、超過でエラー | 一般的な本番環境 |
| 0 | 巻き戻り即エラー | 厳格な時刻管理が必要な環境 |
| 負の値（-1等） | 無制限待機 | VM環境、時刻ジャンプが頻発する環境 |

#### 実装

```typescript
async nextId(): Promise<bigint> {
  let timestamp = Date.now();

  if (timestamp < this.lastTimestamp) {
    const backward = this.lastTimestamp - timestamp;
    const limit = this.maxBackwardMs < 0
      ? Number.MAX_SAFE_INTEGER
      : this.maxBackwardMs;

    if (backward <= limit) {
      await this.sleep(backward);
      timestamp = Date.now();
    } else {
      throw new ClockBackwardError(backward, limit);
    }
  }

  // ... 以降ID生成処理
}
```

#### 運用推奨

- NTPは slew-only モード（`ntpd -x` または chrony デフォルト）を推奨
- VMでは時刻同期の設定を確認
- 監視: 巻き戻り検出時にメトリクス/ログを出力

### 5.7 リース取得リトライ戦略

リースサーバ障害時の負荷集中を防ぐため、Exponential Backoffでリトライする。

#### バックオフ計算

```
interval = min(acquireRetryInterval × 2^(failures-1), acquireRetryMaxInterval)
```

| 連続失敗回数 | 次回リトライまでの間隔 |
|-------------|----------------------|
| 1 | 1秒 |
| 2 | 2秒 |
| 3 | 4秒 |
| 4 | 8秒 |
| 5 | 16秒 |
| 6 | 32秒 |
| 7+ | 60秒（上限） |

#### 状態遷移

```
正常 ──acquire失敗──▶ フォールバック中
  ▲                      │
  │                      │ (interval経過後)
  │                      ▼
  └──acquire成功──── リトライ
```

リトライ待機中はフォールバックモードでID生成を継続（disableFallback=falseの場合）。
リース取得成功で失敗カウンタはリセットされる。

### 5.8 スループット制限

単一クライアントによるリース大量取得を防ぐため、スループット上限を設定できる。

#### 計算方法

各リースは異なるbitSeqを持つ可能性があるため、スループットはリース単位で計算する:

```typescript
currentThroughput = Σ(2^lease.bitSeq) for each lease
```

| リース | bitSeq | スループット |
|--------|--------|-------------|
| lease1 | 8 | 256 |
| lease2 | 8 | 256 |
| lease3 | 8 | 256 |
| lease4 | 8 | 256 |
| **合計** | - | **1,024** |

#### 更新タイミング

`currentThroughput` は以下のタイミングで再計算:

- `acquire` 成功時
- `release` 成功時

#### シーケンス枯渇時の挙動

```
シーケンス枯渇
    │
    ▼
┌─────────────────────────────────┐
│ currentThroughput < maxThroughputPerMs? │
└───────────────┬─────────────────┘
                │
    ┌───────────┴───────────┐
    │ Yes                   │ No
    ▼                       ▼
追加リース取得を試行      次のミリ秒まで待機
    │                       │
    ├─成功→ 新リースで生成   │
    │                       │
    └─失敗→ フォールバック ──┘
           または待機
```

#### 使用例

```typescript
// 低スループット環境（デフォルト: 1リースのみ）
const client1 = new KazahanaClient({ provider });

// 高スループット環境（最大1024ID/ms）
const client2 = new KazahanaClient({
  provider,
  maxThroughputPerMs: 1024,  // 4リース取得
});
```

### 5.9 リースのライフサイクル管理

#### 期限切れリースの除去

期限切れリースは `acquire` 成功時にのみ除去する。

**理由**:
- acquire成功直後なら必ず1つ以上のリースが存在
- 自動pruneで空になることを防ぐ
- フォールバック用のキャッシュを確実に維持

#### フォールバック用リースキャッシュ

```
優先順位:
1. 有効なリース（leases[0]）
2. 期限切れだがキャッシュされたリース（cachedLeaseForFallback）
3. なし → 完全ランダム
```

期限切れリースを除去する際、最後のリースを `cachedLeaseForFallback` に保存。
これによりリースサーバ障害時も衝突リスクの低いフォールバックが可能。

### 5.10 高スループット環境での動的リース取得

単一リースでシーケンスが不足する高性能サーバでは、複数リースを動的に取得して並列使用できる:

```typescript
class KazahanaClient {
  private leases: LeaseInfo[] = [];
  private currentThroughput: number = 0;
  private cachedLeaseForFallback: LeaseInfo | null = null;

  async nextId(): Promise<bigint> {
    const timestamp = await this.getValidTimestamp();

    // 使用可能なリースを探す（id昇順で走査）
    for (const lease of this.leases) {
      if (lease.isValidAt(timestamp) && !lease.isSequenceExhaustedAt(timestamp)) {
        return lease.generate(timestamp);
      }
    }

    // 全て枯渇 → 追加リース取得または待機
    if (this.canAcquireMoreLeases() && this.provider && this.shouldRetryAcquire()) {
      try {
        await this.acquire();
        return this.nextId();
      } catch (e) {
        this.onAcquireFailure();
      }
    }

    // フォールバックまたは次のミリ秒まで待機
    if (!this.disableFallback && this.fallbackLease) {
      return this.generateFallbackId(timestamp);
    }

    await this.waitNextMillis(timestamp);
    return this.nextId();
  }

  private async acquire(): Promise<void> {
    const needed = this.maxThroughputPerMs - this.currentThroughput;
    const response = await this.provider!.acquire(
      this.serviceId,
      this.getMeta(),
      needed  // 不足分だけ要求
    );

    for (const lease of response.leases) {
      this.leases.push(new LeaseInfo(lease));
    }
    this.leases.sort((a, b) => a.id - b.id);
    this.pruneExpiredLeases();
    this.updateThroughput();
    this.onAcquireSuccess();
  }

  private pruneExpiredLeases(): void {
    const now = Date.now();
    const expiredLeases = this.leases.filter(l => !l.isValidAt(now));

    if (expiredLeases.length > 0) {
      this.cachedLeaseForFallback = expiredLeases[expiredLeases.length - 1];
    }

    this.leases = this.leases.filter(l => l.isValidAt(now));
  }

  private updateThroughput(): void {
    this.currentThroughput = this.leases.reduce(
      (sum, lease) => sum + (1 << lease.bitSeq),
      0
    );
  }

  private get fallbackLease(): LeaseInfo | null {
    return this.leases[0] ?? this.cachedLeaseForFallback;
  }
}
```

**ID純増性の保証**: リースをid昇順でソートし、常に小さいidから使用することで、同一ミリ秒内でもmachineIdの小さい順にIDが生成される。これにより生成されるIDは常に純増となる。

---

## 6. サーバ実装

### 6.1 データモデル（Redis例）

```
Key: lease:{machineId}
Value: Hash {
  serviceId: string
  secret: string
  expired: number
  holder: string      // ホスト名等のメタ情報
  createdAt: number
}

Key: lease:last_assigned
Value: number (最後に割り当てたmachineId)
```

### 6.2 acquire処理（Luaスクリプト）

```lua
-- ラウンドロビン方式で空きIDを探索してアトミックに確保
-- 前回割り当てたID+1から開始し、全体を均等に使用する
local lastId = redis.call("GET", "lease:last_assigned") or -1
lastId = tonumber(lastId)
local start = (lastId + 1) % (maxMachineId + 1)
local count = tonumber(ARGV.count)  -- 要求リース数
local results = {}

for i = 0, maxMachineId do
  local id = (start + i) % (maxMachineId + 1)
  local key = "lease:" .. id
  local expires = redis.call("HGET", key, "expired")
  if not expires or tonumber(expires) < tonumber(ARGV.now) then
    local secret = ARGV.secret .. "_" .. #results
    redis.call("HMSET", key,
      "serviceId", ARGV.serviceId,
      "secret", secret,
      "expired", ARGV.expired,
      "holder", ARGV.holder,
      "createdAt", ARGV.now
    )
    redis.call("SET", "lease:last_assigned", id)
    table.insert(results, {id, secret})

    if #results >= count then
      break
    end
  end
end

if #results == 0 then
  return nil  -- 空きなし
end
return results
```

### 6.3 フォールバック領域の予約

```typescript
// machineId の先頭bitが0のみ発行
// bitId=14 の場合: 0-8191 を発行、8192-16383 はフォールバック用
const maxMachineId = (1 << (bitId - 1)) - 1;
```

### 6.4 サービス別設定

```typescript
interface ServiceConfig {
  serviceId: string;
  customEpoch: number;
  bitReserve: number;
  bitTs: number;
  bitId: number;
  bitSeq: number;
  leaseDuration: number;  // ミリ秒
}

// デフォルト設定
const DEFAULT_CONFIG: ServiceConfig = {
  serviceId: 'default',
  customEpoch: Date.parse("2026-01-01T00:00:00Z"),
  bitReserve: 1,
  bitTs: 41,
  bitId: 14,
  bitSeq: 8,
  leaseDuration: 10 * 60 * 1000,  // 10分
};
```

---

## 7. プロバイダ実装例

### 7.1 HTTP Provider

```typescript
class HttpLeaseProvider implements LeaseProvider {
  constructor(private endpoint: string) {}

  async acquire(options: AcquireOptions): Promise<AcquireResponse> {
    const res = await fetch(`${this.endpoint}/lease`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(options),
    });
    if (!res.ok) throw new Error(`acquire failed: ${res.status}`);
    return res.json();
  }

  async release(options: ReleaseOptions): Promise<void> {
    const { id, signature, timestamp } = options;
    const res = await fetch(`${this.endpoint}/lease/${id}`, {
      method: 'DELETE',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ signature, timestamp }),
    });
    if (!res.ok) throw new Error(`release failed: ${res.status}`);
  }
}
```

### 7.2 Redis Provider

```typescript
class RedisLeaseProvider implements LeaseProvider {
  constructor(
    private redis: Redis,
    private config: ServiceConfig = DEFAULT_CONFIG
  ) {}

  async acquire(options: AcquireOptions): Promise<AcquireResponse> {
    const { serviceId, meta, throughputPerMs = 1 } = options;
    const config = this.getServiceConfig(serviceId);
    const now = Date.now();
    const secret = crypto.randomBytes(16).toString('hex');
    const expired = now + config.leaseDuration;
    const maxId = (1 << (config.bitId - 1)) - 1;

    // 必要リース数を計算
    const maxPerLease = 1 << config.bitSeq;
    const count = Math.ceil(throughputPerMs / maxPerLease);

    const results = await this.redis.evalScript(
      ACQUIRE_SCRIPT,
      0,
      maxId, now, expired, secret, serviceId || '', JSON.stringify(meta || {}), count
    );

    if (!results || results.length === 0) {
      throw new Error('No machine ID available');
    }

    return {
      leases: results.map(([id, leaseSecret]: [number, string]) => ({
        id,
        created: now,
        expired,
        secret: leaseSecret,
        customEpoch: config.customEpoch,
        bitReserve: config.bitReserve,
        bitTs: config.bitTs,
        bitId: config.bitId,
        bitSeq: config.bitSeq,
      })),
    };
  }

  async release(options: ReleaseOptions): Promise<void> {
    const { id, signature, timestamp } = options;

    // タイムスタンプ検証（±30秒）
    if (Math.abs(Date.now() - timestamp) > 30000) {
      throw new Error('Timestamp expired');
    }

    const stored = await this.redis.hget(`lease:${id}`, 'secret');
    if (!stored) {
      throw new Error('Lease not found');
    }

    // HMAC検証
    const expected = crypto
      .createHmac('sha256', stored)
      .update(`${id}:${timestamp}`)
      .digest('hex');
    if (signature !== expected) {
      throw new Error('Invalid signature');
    }

    await this.redis.del(`lease:${id}`);
  }
}
```

### 7.3 Memory Provider（テスト用）

```typescript
class MemoryLeaseProvider implements LeaseProvider {
  private leases = new Map<number, { secret: string; expired: number }>();
  private lastAssigned = -1;
  private readonly config = {
    customEpoch: Date.parse("2026-01-01T00:00:00Z"),
    bitReserve: 1,
    bitTs: 41,
    bitId: 14,
    bitSeq: 8,
  };

  async acquire(options: AcquireOptions): Promise<AcquireResponse> {
    const { throughputPerMs = 1 } = options;
    const now = Date.now();
    const expired = now + 600000;  // 10分
    const maxPerLease = 1 << this.config.bitSeq;  // 256
    const count = Math.ceil(throughputPerMs / maxPerLease);
    const results: LeaseInfo[] = [];

    for (let i = 0; i < 8192 && results.length < count; i++) {
      const id = (this.lastAssigned + 1 + i) % 8192;
      const existing = this.leases.get(id);
      if (!existing || existing.expired < now) {
        const secret = Math.random().toString(36).slice(2);
        this.leases.set(id, { secret, expired });
        this.lastAssigned = id;
        results.push({
          id,
          created: now,
          expired,
          secret,
          ...this.config,
        });
      }
    }

    if (results.length === 0) {
      throw new Error('No machine ID available');
    }

    return { leases: results };
  }

  async release(options: ReleaseOptions): Promise<void> {
    const { id, signature, timestamp } = options;

    if (Math.abs(Date.now() - timestamp) > 30000) {
      throw new Error('Timestamp expired');
    }

    const lease = this.leases.get(id);
    if (!lease) {
      throw new Error('Lease not found');
    }

    const crypto = require('crypto');
    const expected = crypto
      .createHmac('sha256', lease.secret)
      .update(`${id}:${timestamp}`)
      .digest('hex');
    if (signature !== expected) {
      throw new Error('Invalid signature');
    }

    this.leases.delete(id);
  }
}
```

---

## 8. 外部ID変換

### 8.1 概要

内部IDは時刻順ソート可能だが、連番が推測可能。
外部公開用には可逆な変換を施し、連番を隠蔽する。

```
内部ID（システム用）        外部ID（ユーザー向け）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
・時刻順ソート可能    ←→    ・連番が見えない
・高速生成                   ・公開可能
・DB インデックス効率
```

### 8.2 変換方式

| モード | シークレット | 用途 |
|--------|------------|------|
| デフォルト | 不要 | 連番隠蔽のみ（開発・一般用途） |
| 暗号化 | 必要 | 予測不可能性が重要な場合 |

### 8.3 デフォルトモード（シークレットなし）

ビット分散のみ行う。暗号学的安全性はないが、連番は見えなくなる。

- 可逆変換（内部ID ↔ 外部ID）
- シークレット設定不要で動作
- 連番の推測を困難にする

### 8.4 暗号化モード（シークレット指定時）

64bitブロック暗号で難読化。アルゴリズムは実装依存（XTEA等）。

- 暗号学的に安全な変換
- シークレット漏洩時は全外部IDが予測可能になるリスクあり
- 金融系など予測不可能性が重要な場合に使用

### 8.5 出力形式: base64url

64bit → 8バイト → base64url → 11文字（パディングなし）

```
内部ID: 0x0123456789ABCDEF
外部ID: "ASNFZ4mrze8"
```

- URL-safe（`+/` の代わりに `-_` を使用）
- パディング（`=`）なし
- 固定長11文字

### 8.6 使用パターン

```typescript
// DB保存: 内部IDのまま（ソート・インデックス効率）
await db.insert({ id: internalId, ... });

// APIレスポンス: 外部IDに変換
return { id: toExternalId(internalId), ... };  // "Abc123XyZ90"

// APIリクエスト: 内部IDに復元
const internalId = toInternalId(req.params.id);
const record = await db.findById(internalId);
```

---

## 9. エラークラス

```typescript
class ClockBackwardError extends Error {
  constructor(
    public readonly backwardMs: number,
    public readonly limitMs: number
  ) {
    super(
      `Clock moved backward by ${backwardMs}ms (limit: ${limitMs}ms). ` +
      `Check NTP configuration or system time settings.`
    );
    this.name = 'ClockBackwardError';
  }
}

class LeaseAcquisitionError extends Error {
  constructor(public readonly cause: unknown) {
    super('Failed to acquire lease and fallback is disabled');
    this.name = 'LeaseAcquisitionError';
  }
}

class NoProviderError extends Error {
  constructor() {
    super('No provider configured and fallback is disabled');
    this.name = 'NoProviderError';
  }
}
```

---

## 10. 運用ガイドライン

### 10.1 ビット配分の調整指針

| 状況 | 推奨調整 |
|------|---------|
| サーバ台数少、バースト多 | bitId↓ bitSeq↑ |
| サーバ台数多、バースト少 | bitId↑ bitSeq↓ |
| 長期運用（50年超） | bitTs=42 |
| 短期サービス | bitTs=40 でも可 |

### 10.2 リース期間の設定

| 期間 | メリット | デメリット |
|------|---------|-----------|
| 短い（1-5分） | ID早期回収、枯渇しにくい | サーバ負荷高 |
| 中程度（10-15分） | バランス良好 | - |
| 長い（1-8時間） | サーバ負荷低 | 障害時の回収遅い |

推奨: **10分**（デフォルト）。更新閾値は90%（9分で更新開始）。

| TTL | 更新開始 | 更新頻度 |
|-----|----------|----------|
| 10分 | 9分後 | 6回/時間 |

### 10.3 監視項目

- アクティブリース数
- リース取得失敗率
- フォールバック発生率
- シーケンス枯渇発生率
- サービス別ID消費量
- 時刻巻き戻り検出回数

### 10.4 障害対応

| 障害 | 影響 | 対応 |
|------|------|------|
| リースサーバダウン | フォールバックモードに移行 | サーバ復旧、衝突確率は低いので許容可能 |
| リース取得失敗（ID枯渇） | ID生成不可 | bitId増加、リース期間短縮 |
| 時刻巻き戻り | maxBackwardMs以内なら待機 | NTP同期確認 |

### 10.5 厳格モード設定

| 設定 | デフォルト | 厳格モード例 | 説明 |
|------|-----------|-------------|------|
| maxBackwardMs | 5000 | 0 | 時刻巻き戻り許容量 |
| disableFallback | false | true | フォールバック無効化 |

```typescript
// 金融システム等、ID重複が許されない環境
const client = new KazahanaClient({
  provider: new HttpLeaseProvider('https://id.example.com'),
  maxBackwardMs: 0,
  disableFallback: true,
});
```

**注意**: 厳格モードでは可用性よりも正確性を優先する。
リースサーバ障害時はID生成自体が停止するため、
アプリケーション側で適切なエラーハンドリングが必要。

---

## 11. 使用例

### 11.1 基本的な使用

```typescript
// スタンドアロン（最小構成）
const client = new KazahanaClient();
const id = await client.nextId();

// HTTPサーバ使用
const client = new KazahanaClient({
  provider: new HttpLeaseProvider('https://id.example.com'),
  serviceId: 'payment-service',
});
const id = await client.nextId();

// Redis直接使用
const client = new KazahanaClient({
  provider: new RedisLeaseProvider(redis),
});
const id = await client.nextId();
```

### 11.2 graceful shutdown

```typescript
process.on('SIGTERM', async () => {
  await client.shutdown();
  process.exit(0);
});
```

### 11.3 高スループット環境

```typescript
// 最大1024ID/msまで自動でリース追加取得
const client = new KazahanaClient({
  provider: new HttpLeaseProvider('https://id.example.com'),
  maxThroughputPerMs: 1024,  // 4リース取得
});
```

### 11.4 VM環境（時刻ジャンプ許容）

```typescript
const client = new KazahanaClient({
  provider: new HttpLeaseProvider('https://id.example.com'),
  maxBackwardMs: -1,  // 無制限待機
});
```

### 11.5 外部IDの使用

```typescript
// ID生成
const internalId = await client.nextId();

// 外部公開用に難読化
const externalId = toExternalId(internalId);
console.log(`Public ID: ${externalId}`);

// 外部IDから復元
const restored = toInternalId(externalId);
assert(restored === internalId);
```

---

## 12. 比較

### 12.1 Snowflakeとの比較

| 項目 | Twitter Snowflake | 本システム |
|------|------------------|-----------|
| マシンID管理 | Zookeeper等で静的管理 | リースサーバで動的管理 |
| 障害耐性 | マシンID設定必須 | フォールバックで継続可能 |
| ビット配分 | 固定 | サーバ側で動的調整可能 |
| 運用負荷 | 高い | 低い |
| 時刻巻き戻り対策 | 待機 | 待機（設定可能） |

### 12.2 UUIDv7との比較

| 項目 | UUIDv7 | 本システム |
|------|--------|-----------|
| サイズ | 128bit | 64bit |
| 衝突回避 | ランダム（74bit） | マシンID + シーケンス |
| インデックス効率 | 良い | より良い（半分のサイズ） |
| 中央サーバ | 不要 | オプション |

### 12.3 ULIDとの比較

| 項目 | ULID | 本システム |
|------|------|-----------|
| サイズ | 128bit | 64bit |
| タイムスタンプ | 48bit（ミリ秒） | 41bit（ミリ秒） |
| ランダム部 | 80bit | マシンID+シーケンス |
| ソート可能 | Yes | Yes |
| 中央サーバ | 不要 | オプション |

---

## 13. 定数定義

```typescript
const LIB_DEFAULT_EPOCH = Date.parse("2026-01-01T00:00:00Z");
const DEFAULT_BIT_RESERVE = 1;
const DEFAULT_BIT_TS = 41;
const DEFAULT_BIT_ID = 14;
const DEFAULT_BIT_SEQ = 8;
const DEFAULT_LEASE_DURATION = 10 * 60 * 1000;  // 10分
const DEFAULT_ACQUIRE_THRESHOLD = 0.9;
const DEFAULT_MAX_BACKWARD_MS = 5000;
const DEFAULT_ACQUIRE_RETRY_INTERVAL = 1000;
const DEFAULT_ACQUIRE_RETRY_MAX_INTERVAL = 60000;
const DEFAULT_MAX_THROUGHPUT_PER_MS = 256;  // 1リース分
```

---

## 付録A: 設計判断の経緯

本セクションでは、設計過程で検討されたが採用しなかった選択肢と、その理由を記録する。
これは後続の開発者が同じ検討を繰り返すことを防ぎ、設計意図を正確に伝えるためである。

### A.1 renew APIを設けない理由

**検討内容**: リース期限を延長する `renew(id, secret)` APIの導入

**採用しなかった理由**:

1. **運用上の圧力の発生**
   - renewがあると「このIDをずっと使い続けたい」という要望が生まれる
   - 特定のIDに依存するシステム設計を誘発する
   - 一度固定IDに依存すると、変更が困難になる

2. **障害時の問題**
   - 「いつものIDが取れない」という障害モードが増える
   - IDは本来交換可能であるべき

3. **シンプルさの維持**
   - 期限切れたら新しいIDを取得すればよい
   - クライアント実装がシンプルになる
   - サーバ実装もシンプルになる

**代替設計**: 期限の90%経過時点で新規 `acquire` を実行し、シームレスに新しいリースへ移行する。

### A.2 release時にHMAC署名を採用した理由

**検討内容**: secretをそのまま送信する方式

```typescript
// secret固定方式の案
provider.release(id, secret);
```

**採用しなかった理由**:

1. **セキュリティ向上**
   - secretがネットワーク上を流れない
   - ログに記録されても安全
   - 漏洩時の影響が限定的

2. **リプレイ攻撃の防止**
   - timestamp検証（±30秒）により、過去の通信を再利用した攻撃を防止
   - staleなプロセスによる誤操作も防げる

3. **実装コストが低い**
   - HMAC計算は数行で実装可能
   - 標準ライブラリで対応
   - パフォーマンスへの影響は無視できる程度

**採用した設計**:
```typescript
const timestamp = Date.now();
const signature = hmac(secret, `${id}:${timestamp}`);
provider.release(id, signature, timestamp);
```

### A.3 defaultEpochのデフォルト値をLIB_DEFAULT_EPOCHとした理由

**検討内容**: defaultEpochを0（Unixエポック）とする案

**採用しなかった理由**:

1. **不要なビット消費**
   - Unixエポック（1970年）から2026年まで約56年分のビットが無駄になる
   - 41bitタイムスタンプの有効期間が実質的に短くなる

2. **設計時期との整合性**
   - 本システムの発案が2026年である
   - 2026年以前のIDは生成されることがない

**結論**: `LIB_DEFAULT_EPOCH = Date.parse("2026-01-01T00:00:00Z")` をデフォルトとする。

### A.4 ビット配分を動的に変更可能とした理由

**検討内容**: ビット配分を固定（Snowflakeと同様）にする案

**採用した理由**:

1. **運用柔軟性**
   - サービス規模の変化に対応できる
   - 初期は保守的な設定、実績を見て調整

2. **実装への影響がない**
   - ビットシフトの値が変数になるだけ
   - パフォーマンスへの影響はゼロ

3. **クライアントコード変更不要**
   - サーバ側の設定変更だけで対応可能
   - デプロイなしで調整できる

### A.5 フォールバックモードでmachineIdの先頭bitを1とした理由

**検討内容**: フォールバック用に別のID体系を使用する案

**採用した理由**:

1. **名前空間の完全分離**
   - 通常ID（先頭bit=0）とフォールバックID（先頭bit=1）は絶対に衝突しない
   - 同じ64bit空間内で共存可能

2. **識別可能性**
   - IDを見るだけでフォールバックかどうか判別可能
   - 障害調査時に有用

3. **シンプルな実装**
   - 特別なフラグやメタデータ不要
   - ID自体に情報が埋め込まれている

### A.6 LeaseProviderを抽象インターフェースとした理由

**検討内容**: HTTPエンドポイントのみをサポートする案

**採用した理由**:

1. **デプロイ柔軟性**
   - 小規模: SQLiteやインメモリで十分
   - 中規模: Redis直接アクセス
   - 大規模: 専用HTTPサーバ

2. **テスト容易性**
   - MemoryProviderでユニットテストが容易
   - モック不要

3. **段階的な移行**
   - スタンドアロン → Redis → HTTPサーバと段階的にスケール可能
   - クライアントコードの変更は最小限

### A.7 高スループット対応をクライアント側の複数リース取得とした理由

**検討内容**: サーバ側でビット配分を動的に調整してシーケンス幅を増やす案

**採用した理由**:

1. **シンプルさ**
   - ビット配分変更は全クライアントに影響する
   - 複数リース取得は必要なクライアントだけが行う

2. **柔軟性**
   - 高性能サーバは多くのリースを取得
   - 低性能サーバは1つで十分
   - サーバスペックに応じた自動適応

3. **公平性**
   - ID空間を効率的に使用
   - 使わないサーバが無駄にID空間を占有しない

### A.8 複数リース取得時にid順でソートする理由

**検討内容**: リースを取得順のまま使用する案

**採用した理由**:

1. **ID純増性の保証**
   - 同一ミリ秒内で複数リースを使用する場合、machineIdの大小がID全体の大小に影響する
   - id昇順でソートし、小さいidのリースから優先使用することで、生成されるIDが常に純増となる

2. **インデックス効率の最大化**
   - B-treeインデックスは順序付きinsertで最も効率が良い
   - 純増性が崩れるとページ分割が発生しやすくなる

3. **デバッグ容易性**
   - IDの大小関係と時間順序が一致する
   - 「このIDより後に生成されたID」という検索が正確になる

**実装**:
```typescript
this.leases.push(newLease);
this.leases.sort((a, b) => a.id - b.id);
```

### A.9 フォールバック時にキャッシュされたリースを優先使用する理由

**検討内容**: フォールバック時は常にランダムIDを生成する案

**採用した理由**:

1. **衝突確率の低減**
   - キャッシュされたmachineIdはリース期限内なら他ノードが取得不可
   - 実質的に衝突リスクゼロ

2. **IDの連続性**
   - 同一machineIdを継続使用するためソート順が維持されやすい

3. **障害調査の容易さ**
   - machineIdが変わらないため、ログ追跡が容易

### A.10 期限切れリースのpruneをacquire成功時のみとした理由

**検討内容**: 定期的またはnextId呼び出し時にpruneする案

**採用しなかった理由**:

1. **フォールバックキャッシュの保護**
   - 自動pruneでリースが空になると、フォールバック時に完全ランダムになる
   - acquire成功時ならprune後も必ず1つ以上のリースが存在

2. **シンプルさ**
   - 定期タイマーや複雑な条件分岐が不要
   - acquire成功という明確なタイミングで実行

### A.11 ビット配分を14+8（machineId+sequence）とした理由

**検討内容**: 従来の10+12配分を維持する案

**採用した理由（14+8）**:

1. **複数リース方式との相性**
   - シーケンス枯渇時は追加リースを取得する設計がある
   - 8bit（256/ms）で単一リース25.6万/秒、十分な性能
   - 4リース取得で102万/秒まで拡張可能

2. **大規模分散環境への対応**
   - 14bit で 16,384 台（通常用 8,192 台）をサポート
   - 10bit の 1,024 台から16倍に拡大
   - K8s等のコンテナ環境で多数のPodが起動するケースに対応

3. **効率的なID空間利用**
   - 低負荷サーバは1リースで十分
   - 高負荷サーバのみ追加リース取得
   - ID空間を使う分だけ消費する公平な設計

### A.12 リースTTLを10分とした理由

**検討内容**: 5分または1時間とする案

**採用した理由（10分）**:

1. **バランス**
   - 更新頻度: 6回/時間（サーバ負荷許容範囲）
   - 回収時間: 最大10分（障害時の影響限定的）

2. **K8s等との相性**
   - Pod再起動が頻繁な環境でも問題ない
   - graceful shutdownでrelease呼べば即座に回収

3. **ID枯渇リスクの低減**
   - 8,192台 × 10分 = 短時間でID回収
   - 穴あきが長期間残らない

### A.13 マシンID割り当てをラウンドロビン方式とした理由

**検討内容**: 0から順に探索する方式

**採用した理由（前回割り当てID+1から開始）**:

1. **フォールバックとの衝突確率均等化**
   - 0から順だと小さいIDに偏る
   - フォールバック時のランダム分布との重なりが不均等になる
   - ラウンドロビンで全体を均等に使用し、衝突確率を分散

2. **実装コストが低い**
   - `lease:last_assigned` キーを1つ追加するだけ
   - Luaスクリプトに2行追加

3. **デバッグへの影響なし**
   - IDの時系列順序はtimestamp部分で保証される
   - machineIdの割り当て順序はデバッグに影響しない

### A.14 外部ID変換のアルゴリズム選定

具体的なアルゴリズムは実装時に比較検証して決定する。

**要件**:
- 64bitブロックの可逆変換
- デフォルトモード: ビット分散（シークレット不要）
- 暗号化モード: 64bitブロック暗号（XTEA等）

**出力形式**: base64url（11文字固定）

### A.15 acquire に throughputPerMs パラメータを追加した理由

**検討内容**: クライアントが必要なリース数を直接指定する案

**採用した理由（throughputPerMsで抽象化）**:

1. **抽象化**
   - クライアントは性能要件だけ指定
   - リース数の計算はサーバ側で行う
   - bitSeqが変わってもクライアント変更不要

2. **効率的な更新**
   - リース期限がバラけてきた時に不足分だけ要求可能
   - `currentThroughput` と `maxThroughputPerMs` の差分を指定

3. **ネットワーク効率**
   - 1リクエストで必要な複数リースをまとめて取得
   - 個別に acquire を呼ぶより効率的

---

## 付録B: パフォーマンス特性

### B.1 Date.now()の呼び出し頻度

実測値（環境依存）:

```javascript
const arr = new Array(1000000).fill(0).map(() => Date.now());
// 結果例: 100万回で約63ユニーク値、最大25722回/ms
```

これは「ID生成処理のみ」の場合。実際のアプリケーションでは:

- DBアクセス
- バリデーション
- ビジネスロジック

が含まれるため、実効的なID生成レートは大幅に低下する（100倍程度遅くなると想定）。

### B.2 シーケンス幅の目安

| bitSeq | 最大値/ms | 1リース/秒 | 想定用途 |
|--------|----------|-----------|---------|
| 6bit | 64 | 6.4万 | 小規模、低トラフィック |
| **8bit** | 256 | **25.6万** | **デフォルト（複数リースで拡張可能）** |
| 10bit | 1,024 | 102万 | 高トラフィック |
| 12bit | 4,096 | 409万 | 非常に高トラフィック |

**複数リースによる拡張**:

| リース数 | 8bit時の最大/秒 |
|----------|----------------|
| 1 | 25.6万 |
| 2 | 51.2万 |
| 4 | 102万 |
| 8 | 204万 |

### B.3 エンディアンについて

ID生成自体はエンディアンに依存しない（純粋な数値演算）。

注意が必要なケース:
- バイナリプロトコルでの送受信
- 異なる言語間でのバイト列交換

推奨: バイト列として扱う場合はBig Endian（ネットワークバイトオーダー）で統一。

---

## 付録C: 用語集

| 用語 | 説明 |
|------|------|
| カスタムエポック | タイムスタンプの基準となる時刻。Unixエポック（1970年）ではなくサービス開始時点を使用することで、41bitで69年間利用可能にする |
| リース | マシンIDの一時的な使用権。期限付きで発行され、期限切れで自動解放される |
| フォールバック | リースサーバと通信できない場合の代替ID生成方式。キャッシュされたmachineId、またはランダム性で衝突を回避 |
| シーケンス | 同一ミリ秒内でのID重複を防ぐカウンター |
| スループット | 1ミリ秒あたりに生成可能なID数。リースのbitSeqから計算される |
| 外部ID | 内部IDを変換した外部公開用のID。base64urlで11文字 |

