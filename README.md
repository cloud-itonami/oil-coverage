# oil-coverage

**石油サプライチェーンの *被覆率を測る* actor の descriptor と、その書き込みを止める
deny-by-default gate。石油そのものを扱う 6 つの segment actor は、ここには無い。
実装ではない。**

`oil-coverage` という名前は、読み手に **7 番目の segment** を期待させる。違う。
これは **他の 6 つ（upstream / midstream / refining / trading / shipping /
distribution）を上から測る meta actor** で、「国 × segment のうちどこがまだ
グラフに載っていないか」を 6 時間ごとに数えると宣言している。

| | ここにあるか |
|---|---|
| actor が**何を名乗り、何を要求し、どの pipeline を持つと宣言しているか** | **ある**（`actor-manifest.jsonld` / `.well-known/did.json`） |
| **gate**（attestation が 7 つ揃わなければ effect を 1 つも出さない判断） | **ある**（`src/oil_coverage/murakumo.cljc`、191 行） |
| 被覆率を実際に数える graph、cron を撃つ scheduler、XRPC を受ける server | **無い** |
| 石油の実データ（field / pipeline / refinery / cargo / benchmark） | **無い** |

**ここには動くサービスは無い。** `cell-plan` が返すのは「書くとしたら何をどこに
書くか」という**計画**であって、書き込みそのものではない。`:effects` は
`{:op :mst/put-record ...}` という data であり、それを実行する者はこの repo に居ない。

経緯は [docs/adr/0001-descriptor-snapshot-not-an-executor.md](docs/adr/0001-descriptor-snapshot-not-an-executor.md)。
手順は [docs/operator-quickstart.md](docs/operator-quickstart.md)。

## 名前が 2 つあり、解決するのは片方だけ

この repo は自分を 2 通りに名乗っている。**同じ actor の別表記ではなく、
一方は DNS に存在しない。**

| 出所 | 名乗り | 2026-08-09 実測 |
|---|---|---|
| `actor-manifest.jsonld` の `@id`<br>`src/oil_coverage/murakumo.cljc` の `actor-did` | `did:web:oil-coverage.etzhayyim.com` | **解決しない**。`oil-coverage.etzhayyim.com` に A/AAAA レコードが無く、`curl` は `000`（接続前に失敗） |
| `.well-known/did.json` の `id` | `did:web:etzhayyim.com:actor:oil-coverage` | **解決する**。`https://etzhayyim.com/actor/oil-coverage/did.json` が `200` |

**gate が名乗るのは解決しない方**である（`murakumo.cljc:6`）。effect の
`:actor` フィールドに載るのもそちら。これは既知の不整合で、直していない —
どちらを正とするかは etzhayyim 側の identity 決定であって、この snapshot が
勝手に決めてよいことではない。

### この repo の `.well-known/did.json` は配信されていない

`.nojekyll` が置かれているので GitHub Pages 配信を意図していたと読めるが、
2026-08-09 時点で `etzhayyim.github.io/com-etzhayyim-oil-coverage` も
`cloud-itonami.github.io/oil-coverage` も **404**。

さらに、解決する方の URL が実際に返す文書は、**この repo の中の did.json とは
別物**である:

| | repo の `.well-known/did.json` | live（`etzhayyim.com/actor/oil-coverage/did.json`） |
|---|---|---|
| crypto suite context | `ed25519-2020` | `jws-2020` |
| `alsoKnownAs` | 4 件（at:// · github · rad: · github.io） | **空** |
| `verificationMethod` | **キー無し**（フィールド自体が無い） | 空配列（`_meta` に「ERC725 mirror pending」と注記） |
| PDS endpoint | `https://pds.etzhayyim.com` → **530** | `https://pds.aozora.app` → **200** |
| 2 つ目の service | `#aozora`（AozoraAppView） | `#xrpc-libp2p`（libp2p multiaddr） |

つまり **repo の did.json は live の写しではなく、live より古い（あるいは別系統の）
文書**。ここを「配信されている DID document」として読まないこと。

## 何を測ると宣言しているか

`actor-manifest.jsonld` は 5 pipeline を宣言する（cron 2 / xrpc 3）。

| trigger | 何を宣言しているか |
|---|---|
| cron `15 */6 * * *` | **目標行列を撒く。** 20 か国 × 6 segment の組から選んだ **32 target** を `OilCoverageTarget` として MERGE し、社会面に digest を 1 本流す |
| cron `0 */6 * * *` | **測る。** ノード総数 → segment 別 target 数 → backbone 9 label の実在数 を数え、`agent.chat` に要約させ、`OilCoverageSnapshot` として書き戻す |
| xrpc `…coverage.get` | 最新 snapshot と target 総数を返す |
| xrpc `…coverage.listTargets` | target 行列を最大 200 行返す |
| xrpc `…coverage.listBackbone` | backbone 9 label の実在数を返す |

撒かれる 32 target の内訳（`upstream` に偏っているのは宣言どおり）:

| segment | 目標数 |
|---|---|
| upstream | 10 |
| midstream | 5 |
| refining | 5 |
| shipping | 4 |
| trading | 4 |
| distribution | 4 |

backbone として数える 9 label: `OilCompany` `OilField` `OilPipeline`
`OilTerminal` `Refinery` `OilCargo` `CrudeGrade` `ProductGrade`
`PricingBenchmark`。

## 最近接 repo との境界

`triggers.subscribeRepos` は **12 collection** を購読すると宣言する。その宛先は
`cloud-itonami` に**すべて実在する別 repo** で、この repo の仕事ではない:

| 購読先 collection | 実装を持つ（はずの）repo | この repo との関係 |
|---|---|---|
| `…oilUpstream.coverageSnapshot` | `oil-upstream` | 測られる側 |
| `…oilMidstream.coverageSnapshot` | `oil-midstream` | 測られる側 |
| `…oilRefining.coverageSnapshot` | `oil-refining` | 測られる側 |
| `…oilTrading.coverageSnapshot` | `oil-trading` | 測られる側 |
| `…oilShipping.coverageSnapshot` | `oil-shipping` | 測られる側 |
| `…oilDistribution.coverageSnapshot` | `oil-distribution` | 測られる側 |
| `…bunker.coverageSnapshot` | `bunker` | 燃料油。石油 segment ではないが供給網で接する |
| `…vessel.coverageSnapshot` | `vessel` | 船体 |
| `…port.coverageSnapshot` | `port` | 港湾 |
| `…marineInsurance.coverageSnapshot` | `marine-insurance` | 海上保険 |
| `…oil.coverageSnapshot` / `…oil.coverageTarget` | **この repo 自身** | 自分の出力を読み返す |

**6 つの `oil-*` segment repo と混同しないこと。** それらは
`actors` が 4・`pipelines` が 8 で、担当 segment の実体（field, pipeline,
refinery, cargo …）を扱うと宣言している。この repo だけが `actors` 6 ×
`pipelines` 5 で、**中身ではなく被覆率**を扱う。

## gate は何を止めるか

`src/oil_coverage/murakumo.cljc` は **11 cell × 7 gate** の deny-by-default。
7 つの attestation が 1 つでも欠けると `:status :blocked` で `:effects` は空になる。

7 gate: `:council-charter-attestation` `:no-platform-held-key-baseline`
`:no-probing-baseline` `:murakumo-only-inference-baseline`
`:did-primary-baseline` `:append-only-gate-baseline`
`:kotoba-only-substrate-baseline`

実行して確かめられる（[quickstart](docs/operator-quickstart.md) の手順 4）。

⚠ **namespace が `oil_coverage.murakumo`（アンダースコア）である。** Clojure の
慣習では `oil-coverage.murakumo` と書いてファイル側を `oil_coverage/` にする。
現状でも load はできるが、`(require '[oil-coverage.murakumo])` は**通らない** —
`oil_coverage` と綴る必要がある。scaffold 生成器の産物で、直すと呼び出し側が
壊れうるためこの反復では触っていない。

## 2026-05-21 の snapshot であること

etzhayyim monorepo の `20-actors/oil-coverage` から descriptor だけを写した
snapshot。`actor-manifest.jsonld` が `runtime: k8s-langserver` /
`edge: sveltekit-proxy` と宣言していても、**その runtime も edge もここには無い。**
`complianceDocs` が指す 2 本（`90-docs/rules/compliance/…` /
`90-docs/platform/…`）も、この repo には存在しない**元 monorepo のパス**である。

`actor-manifest.test.ts` は **走らない** — `package.json` も vitest も無く、
`node_modules` も無い。vitest 前提で書かれた 12 assertion が、実行されないまま
置かれている。`.ts` なのでこの workspace の nbb 経路にも載らない。

## 既知のギャップ（この反復で埋めていないもの）

- **test が無い。** 上の表の数（pipeline 5 / sub-actor 6 / cell 11 / gate 7 /
  target 32）と 2 つの DID は、いま**この README が主張しているだけ**で、実体が
  動いても赤くならない。`marine-insurance` は `test/…/docs_test.cljs` でこれを
  固定している —— 同じものがここにも要る。
- **west pin が遅れていた。** superproject の pin は `36703a1`（2026-07-02）で、
  `src/oil_coverage/murakumo.cljc` を含む `9532319` を指していなかった。この
  README を書く時点で main に合わせている。
- **identity の不整合を直していない**（上記 2 名の DID、live との差分）。
