# ネーミング検討

Snowflake-like 分散ID生成システムの名称候補を検討する。

## 現状

- リポジトリ名: `kazahana`
- 設計書での呼称: 「本システム」「Snowflake-like ID生成システム」

## 命名の方向性

### A. 雪・氷関連（Snowflake との関連性を示す）

| 名前 | 読み | 意味 | 備考 |
|------|------|------|------|
| kazahana | かざはな | 風花（晴天に舞う雪） | 現リポジトリ名。Snowflake より軽やかなイメージ |
| fubuki | ふぶき | 吹雪 | 力強い、大量生成のイメージ |
| koyuki | こゆき | 小雪 | 軽量・シンプルさを表現 |
| awayuki | あわゆき | 淡雪 | 儚く消える雪、軽量感 |
| konayuki | こなゆき | 粉雪 | 細かい粒子、分散を想起 |
| sassayuki | ささゆき | 細雪 | 谷崎潤一郎の小説で有名 |
| mizore | みぞれ | 霙 | 雨と雪の中間 |
| arare | あられ | 霰 | 粒状で分散を想起 |
| hyoga | ひょうが | 氷河 | 大規模・永続的なイメージ |
| hyosho | ひょうしょう | 氷晶 | 結晶構造、IDの構造を想起 |

### B. 花びら・舞い散る系（雪が舞う様子との連想）

| 名前 | 読み | 意味 | 備考 |
|------|------|------|------|
| hanafubuki | はなふぶき | 花吹雪 | 桜が舞い散る様子、雪と花の融合 |
| hirahira | ひらひら | ひらひら | 舞い落ちる様子の擬態語 |
| chiru | ちる | 散る | シンプル、儚さ |
| rakka | らっか | 落花 | 花が散る、詩的 |
| maihana | まいはな | 舞花 | 舞う花びら |
| hanabi | はなび | 花火 | 一瞬で散る、ID生成の瞬間性 |
| chirari | ちらり | ちらり | 一瞬見える様子 |

### C. 基本名 + id サフィックス

| 名前 | 元の意味 |
|------|----------|
| kazahanaid | 風花 + id |
| fubukid | 吹雪 + id（fubuki + id の縮約） |
| fubukiid | 吹雪 + id |
| koyukid | 小雪 + id |
| flakeid | flake + id |
| snowid | snow + id |
| driftid | drift + id（Snowflake派生を示唆） |
| chronoid | chrono + id（時刻順ソート可能） |

### D. 英語系

| 名前 | 意味 | 備考 |
|------|------|------|
| snowlet | 小さな Snowflake | 64bit（より小さい）を表現 |
| flurry | 小雪、にわか雪 | 軽やかなイメージ |
| flurryid | flurry + id | |
| sleet | みぞれ | |
| frost | 霜 | |
| frostid | frost + id | |
| iceflake | 氷の欠片 | |
| drift | 吹きだまり、漂流 | Snowflake からの派生を示唆 |
| blizzard | 猛吹雪 | 強力だが攻撃的 |
| snowdrift | 吹きだまり | |
| flakelets | 小さな flake の複数形 | |

### E. 造語・組み合わせ

| 名前 | 由来 | 備考 |
|------|------|------|
| leaseflake | lease + flake | 動的リース機構を強調 |
| microflake | micro + flake | 64bit の小ささを表現 |
| nanoflake | nano + flake | さらに小さい印象 |
| swirl | 渦巻き | ID が渦のように生成される |
| swirlid | swirl + id | |
| sparkid | spark + id | 軽量・高速 |
| glint | きらめき | 一瞬の輝き |
| glintid | glint + id | |
| shimmer | ゆらめき | |

---

## パッケージ名の空き状況

調査日: 2026-02-02

### 総合結果

| 名前 | npm | crates.io | PyPI | Go | Maven | RubyGems | GitHub | 総合 |
|------|-----|-----------|------|----|-------|----------|--------|------|
| **kazahana** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ⚠️小規模 | ◎ |
| fubuki | ❌ | ❌ | ✅ | ❌ | ✅ | ❌ | ⚠️ | × |
| koyuki | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ⚠️小規模 | △ |
| **awayuki** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◎ |
| **konayuki** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ⚠️小規模 | ◎ |
| **hanafubuki** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◎ |
| **hirahira** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◎ |
| **kazahanaid** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◎ |
| **fubukid** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◎ |
| **fubukiid** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◎ |
| flakeid | ❌ | ✅ | ✅ | ❌ | ✅ | ✅ | ⚠️同種 | × |
| snowid | ✅ | ❌ | ✅ | ❌ | ✅ | ✅ | ⚠️同種 | × |
| **driftid** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◎ |
| **chronoid** | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ⚠️小規模 | △ |
| **snowlet** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◎ |
| flurry | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌有名 | × |
| **flurryid** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◎ |
| drift | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌有名 | × |
| **leaseflake** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◎ |
| **microflake** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ⚠️無関係 | ◎ |
| nanoflake | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ⚠️同種 | △ |
| **sparkid** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◎ |
| **glintid** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◎ |
| rakka | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ❌有名 | × |
| **maihana** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◎ |

### 凡例

- ✅ 空き
- ❌ 使用中
- ⚠️ 存在するが小規模/無関係
- ◎ 全プラットフォームで使用可能（推奨）
- △ 一部で競合あり
- × 主要プラットフォームで競合

---

## 全プラットフォームで空いている候補（推奨）

### 日本語系（雪・花）

| 名前 | 読み | 意味 | コメント |
|------|------|------|----------|
| **kazahana** | かざはな | 風花 | 現リポジトリ名。詩的で覚えやすい |
| **awayuki** | あわゆき | 淡雪 | 軽量感を表現、全て空き |
| **konayuki** | こなゆき | 粉雪 | 細かい粒子、分散を想起 |
| **hanafubuki** | はなふぶき | 花吹雪 | 雪と花の融合、華やか |
| **hirahira** | ひらひら | ひらひら | 舞い落ちる様子、可愛らしい |
| **maihana** | まいはな | 舞花 | 舞う花びら、優雅 |

### id サフィックス付き

| 名前 | コメント |
|------|----------|
| **kazahanaid** | 明確にID生成ライブラリと分かる |
| **fubukid** | fubuki + id の縮約、語呂が良い |
| **fubukiid** | fubuki + id |
| **driftid** | Snowflake派生を示唆 |
| **flurryid** | flurry（にわか雪）+ id |

### 英語系

| 名前 | 意味 | コメント |
|------|------|----------|
| **snowlet** | 小さな Snowflake | 64bitの小ささを表現、覚えやすい |
| **leaseflake** | リース + flake | 動的リース機構を強調（特徴的） |
| **microflake** | 小さな flake | 軽量さを表現 |
| **sparkid** | きらめき + id | 高速・軽量を表現 |
| **glintid** | きらめき + id | 一瞬の輝き |

---

## 避けるべき候補

| 名前 | 理由 |
|------|------|
| fubuki | npm, crates.io, Go, RubyGems で使用中 |
| flakeid | npm で同種のID生成ライブラリが存在 |
| snowid | crates.io で同種のID生成ライブラリが存在 |
| flurry | 全プラットフォームで使用中、GitHub で 574 stars |
| drift | 全プラットフォームで使用中、GitHub で 3138 stars |
| rakka | crates.io で使用中、GitHub で rakkasjs が 1098 stars |
| nanoflake | Go で競合、GitHub に同種プロジェクトあり |

---

## 評価軸

- [x] 発音しやすいか
- [x] スペルしやすいか
- [x] 既存パッケージと重複しないか（npm, crates.io, PyPI, Go, Maven, RubyGems, GitHub）
- [ ] Snowflake との関連性が伝わるか
- [ ] 特徴（軽量、動的リース、フォールバック）が伝わるか
- [ ] 覚えやすいか

---

## 最終候補（個人的おすすめ順）

1. **kazahana** - 現リポジトリ名。詩的で日本語らしく、Snowflake の「舞う雪」を連想させる
2. **snowlet** - 英語圏でも通じやすく、「小さな Snowflake」という意味が明確
3. **leaseflake** - このシステムの特徴（動的リース）を名前で表現
4. **fubukid** - 力強く、id サフィックスで用途が明確
5. **hanafubuki** - 「花吹雪」は美しく、雪と花の融合で詩的

---

## 決定

**kazahana** (風花)

- 意味: 晴天に舞う雪
- Snowflake より軽やかで詩的なイメージ
- 全パッケージレジストリで空き
- GitHub に同名プロジェクト（アニメクライアント）があるが、ジャンルが全く異なるため問題なし

決定日: 2026-02-02
