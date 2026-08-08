# operator quickstart — oil-coverage

**所要 3 分。ここに書いた 5 手順は 2026-08-09 に実際に実行し、出力を確認したもの
だけである。** 踏めない手順（`actor-manifest.test.ts` の実行など）は載せていない
——なぜ載せられないかは手順 5 で確かめる。

この repo は **descriptor + gate の snapshot** であって、動くサービスではない
（[README](../README.md) / [ADR-0001](adr/0001-descriptor-snapshot-not-an-executor.md)）。
したがって quickstart の目的は「起動する」ことではなく、**宣言されている内容と、
その宣言が現実とどれだけ合っているかを自分で測る**ことにある。

必要なもの: `git` · `jq` · `curl` · `dig` · [`nbb`](https://github.com/babashka/nbb)。
graph も PDS も k8s も要らない。

---

## 0. 手に入れる

superproject（`com-junkawasaki/root`）の west 管理下にいるなら:

```bash
cd <superproject>
west update --fetch smart oil-coverage
cd orgs/cloud-itonami/oil-coverage
```

単体で見るなら:

```bash
git clone git@github.com:cloud-itonami/oil-coverage.git
cd oil-coverage
```

**`src/oil_coverage/murakumo.cljc` があることを確かめる。** 無ければ checkout が
`main` より古い（west pin が 2026-07-18 の rescue commit より前を指していた時期がある）:

```bash
ls src/oil_coverage/murakumo.cljc && git log --oneline -1
```

---

## 1. 何を宣言しているか（形）

```bash
jq -r '"pipelines=\(.pipelines|length) sub-actors=\(.actors|length) capabilities=\(.capabilities|length) subscribeRepos=\(.triggers.subscribeRepos.collections|length)"' actor-manifest.jsonld
```

```
pipelines=5 sub-actors=6 capabilities=4 subscribeRepos=12
```

`sub-actors=6` / `pipelines=5` がこの repo の指紋である。**6 つの `oil-*`
segment repo は `actors=4` / `pipelines=8`** なので、ここで見分けがつく:

```bash
for r in oil-upstream oil-midstream oil-refining oil-trading oil-shipping oil-distribution; do
  printf '%-18s ' "$r"
  jq -r '"actors=\(.actors|length) pipelines=\(.pipelines|length)"' "../$r/actor-manifest.jsonld"
done
```

（superproject の `orgs/cloud-itonami/` 配下にいる場合。単体 clone なら省略）

---

## 2. 何を撃つと宣言しているか（trigger）

```bash
jq -r '.pipelines[].trigger | "\(.type)\t\(.cron // .nsid)"' actor-manifest.jsonld
```

```
cron	15 */6 * * *
cron	0 */6 * * *
xrpc	com.etzhayyim.apps.oil.coverage.get
xrpc	com.etzhayyim.apps.oil.coverage.listTargets
xrpc	com.etzhayyim.apps.oil.coverage.listBackbone
```

**この cron を撃つ scheduler はこの repo に無い。** `.github/workflows/` も
`crontab` も無く、`runtime: k8s-langserver` が指す k8s も無い。宣言だけがある。

数える対象の 9 label:

```bash
jq -r '[.pipelines[].steps[].args.sql // empty | select(test("OilCompany"))][0]
       | capture("UNWIND \\[(?<l>[^]]*)\\]").l' actor-manifest.jsonld
```

```
'OilCompany','OilField','OilPipeline','OilTerminal','Refinery','OilCargo','CrudeGrade','ProductGrade','PricingBenchmark'
```

---

## 3. 名乗りが解決するか測る（ここが一番重要）

この repo は自分を **2 通りに名乗る**。まず両方を出す:

```bash
jq -r '"manifest @id : \(.["@id"])"' actor-manifest.jsonld
jq -r '"did.json  id : \(.id)"'      .well-known/did.json
```

```
manifest @id : did:web:oil-coverage.etzhayyim.com
did.json  id : did:web:etzhayyim.com:actor:oil-coverage
```

**同じ actor の別表記ではない。片方は DNS に存在しない。** 測る:

```bash
dig +short oil-coverage.etzhayyim.com          # → 何も返らない
curl -sS -o /dev/null -w '%{http_code}\n' --max-time 12 \
  https://oil-coverage.etzhayyim.com/.well-known/did.json      # → 000（接続前に失敗）

curl -sS -o /dev/null -w '%{http_code}\n' --max-time 12 \
  https://etzhayyim.com/actor/oil-coverage/did.json            # → 200
```

`did:web` の解決規則では、`did:web:etzhayyim.com:actor:oil-coverage` は
`https://etzhayyim.com/actor/oil-coverage/did.json` に対応する（パスを持つ形は
`.well-known` を挟まない）。**解決するのはこちらだけ**である。

次に、**live が返す文書と repo の中の文書が一致するか**を見る:

```bash
curl -sS --max-time 15 https://etzhayyim.com/actor/oil-coverage/did.json -o /tmp/live-did.json
diff <(jq -S . /tmp/live-did.json) <(jq -S . .well-known/did.json)
```

一致しない。2026-08-09 時点で違うのは 4 か所:

| | repo | live |
|---|---|---|
| crypto suite context | `ed25519-2020` | `jws-2020` |
| `alsoKnownAs` | 4 件 | 空 |
| PDS endpoint | `pds.etzhayyim.com` | `pds.aozora.app` |
| 2 つ目の service | `#aozora` | `#xrpc-libp2p` |

endpoint の生死も測れる（**live 側だけが生きている**）:

```bash
curl -sS -o /dev/null -w 'pds.etzhayyim.com %{http_code}\n' --max-time 12 https://pds.etzhayyim.com/xrpc/_health
curl -sS -o /dev/null -w 'pds.aozora.app    %{http_code}\n' --max-time 12 https://pds.aozora.app/xrpc/_health
```

```
pds.etzhayyim.com 530
pds.aozora.app    200
```

**結論として、repo の `.well-known/did.json` を「配信されている DID document」と
して読んではいけない。** 配信もされていない（`.nojekyll` があるが Pages は 404）:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' --max-time 12 \
  https://etzhayyim.github.io/com-etzhayyim-oil-coverage/.well-known/did.json   # → 404
```

---

## 4. gate を動かす（この repo で唯一「実行できる」もの）

`src/oil_coverage/murakumo.cljc` は 11 cell × 7 gate の deny-by-default。
**attestation が 1 つでも欠ければ effect は 0 本**であることを、実際に見る:

```bash
nbb --classpath src -e "
(require '[oil_coverage.murakumo :as m])
(println \"actor-did:\" m/actor-did)
(println \"cells:\" (count m/cell-specs) \" gates:\" (count m/common-gates))
;; (a) 何も attest しない → 止まる
(let [r (m/cell-plan :get {:attestations {}})]
  (println \"none      ->\" (:status r) \"effects=\" (count (:effects r)) \"missing=\" (count (:missing-gates r))))
;; (b) 7 つ中 6 つだけ attest → まだ止まる（緩む方向に壊れていないこと）
(let [r (m/cell-plan :get {:attestations (zipmap (butlast m/common-gates) (repeat true))})]
  (println \"6-of-7    ->\" (:status r) \"effects=\" (count (:effects r)) \"missing=\" (count (:missing-gates r))))
;; (c) 7 つ全部 → 計画が出る。ただし出るのは data であって書き込みではない
(let [r (m/cell-plan :get {:attestations (zipmap m/common-gates (repeat true))
                           :computed-at \"2026-08-09T00:00:00Z\" :request-id \"demo\"})]
  (println \"all       ->\" (:status r) \"effects=\" (count (:effects r)))
  (println \"effect     :\" (select-keys (first (:effects r)) [:op :actor :collection])))
"
```

```
actor-did: did:web:oil-coverage.etzhayyim.com
cells: 11  gates: 7
none      -> :blocked effects= 0 missing= 7
6-of-7    -> :blocked effects= 0 missing= 1
all       -> :ready effects= 1
effect     : {:op :mst/put-record, :actor did:web:oil-coverage.etzhayyim.com, :collection com.etzhayyim.oil-coverage.get}
```

ここで 2 つ確認しておくこと:

1. **`:ready` でも書き込みは起きていない。** 返るのは
   `{:op :mst/put-record ...}` という **data** で、これを実行する者はこの repo に
   居ない。gate は「書くとしたら何をどこに書くか」しか答えない。
2. **`:actor` に載るのは解決しない方の DID** である（手順 3）。gate の
   `actor-did` は `did:web:oil-coverage.etzhayyim.com` に固定されている。

> namespace は `oil_coverage.murakumo`（**アンダースコア**）。
> `(require '[oil-coverage.murakumo])` はハイフンでは通らない。

---

## 5. 走らないものを確認する

```bash
ls package.json node_modules 2>&1        # → どちらも No such file or directory
ls actor-manifest.test.ts                # → 在る
```

`actor-manifest.test.ts` は vitest 前提の 12 assertion を持つが、`package.json`
も vitest も無いので**実行経路が無い**。`.ts` なのでこの workspace の nbb 経路にも
載らない。**「テストを走らせる」手順として案内できないのはこのため**で、
同じ検査を nbb の `test/` として書き直すのは未了（ADR-0001 参照）。

手順 1〜4 で読んだ数（pipeline 5 / sub-actor 6 / cell 11 / gate 7）は、いま
**README がそう主張しているだけ**で、実体が動いても自動では赤くならない。
数が合わなければ README が古い。

---

## 次に何を読むか

| 知りたいこと | 読む場所 |
|---|---|
| この repo が何で、何でないか | [README.md](../README.md) |
| なぜ identity の不整合を直さないのか | [ADR-0001](adr/0001-descriptor-snapshot-not-an-executor.md) |
| 測られる側の 6 segment | `../oil-{upstream,midstream,refining,trading,shipping,distribution}` |
| 同型の repo で test まで在る先例 | `../marine-insurance`（`run_tests.cljs` + `test/`） |
