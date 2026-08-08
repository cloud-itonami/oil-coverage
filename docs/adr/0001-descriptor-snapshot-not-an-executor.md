# ADR-0001 — この repo は descriptor snapshot であって executor ではない

- **status**: accepted
- **date**: 2026-08-09
- **scope**: `cloud-itonami/oil-coverage`
- **上位**: superproject ADR-2608080000（成熟度を 1 段ずつ上げる loop）/
  ADR-2608052000（測り方）。先例は `cloud-itonami/marine-insurance` の同番 ADR。

## 文脈

この repo は etzhayyim monorepo の `20-actors/oil-coverage` から 2026-05-21 に
descriptor だけを写した snapshot である。2026-07-18 に `src/oil_coverage/murakumo.cljc`
（191 行の deny-by-default gate）が rescue commit として足された（PR #1）が、
**README が 1 バイトも無かった**ため、この repo を開いた者には次が分からなかった:

1. `oil-coverage` が 7 番目の石油 segment なのか、6 segment を測る meta actor なのか
2. `actor-manifest.jsonld` が宣言する `runtime: k8s-langserver` /
   `edge: sveltekit-proxy` / 5 pipeline が、ここで**動く**のかどうか
3. 自分を 2 通りに名乗っている DID のどちらが実在するのか
4. `actor-manifest.test.ts` を走らせる方法（**無い**）

3 は特に危うい。`actor-manifest.jsonld` と gate は
`did:web:oil-coverage.etzhayyim.com` を名乗るが、2026-08-09 に測ったところ
そのホストには DNS レコードが無く解決しない。解決するのは
`.well-known/did.json` が名乗る `did:web:etzhayyim.com:actor:oil-coverage` の方で、
しかも **live が返す文書は repo 内の did.json と別物**だった（crypto suite
context・`alsoKnownAs`・PDS endpoint・2 つ目の service がすべて違う）。

## 決定

1. **この repo を「descriptor + gate の snapshot」として明示的に位置づける。**
   README の冒頭で「実装ではない」と名乗り、「ここにあるか」の表で
   *宣言* と *実行主体* を分ける。名前が機能を示さない repo は README 冒頭で
   名乗る、という workspace 規則（superproject CLAUDE.md）の適用である。
   `oil-coverage` は主題を言うが、**meta actor であること**を言わない。

2. **境界を最近接 repo に対して明示する。** `triggers.subscribeRepos` の 12
   collection は、`cloud-itonami` に実在する 10 の別 repo（`oil-upstream`
   `oil-midstream` `oil-refining` `oil-trading` `oil-shipping`
   `oil-distribution` `bunker` `vessel` `port` `marine-insurance`）を指す。
   6 つの `oil-*` segment repo は `actors` 4 × `pipelines` 8、この repo は
   `actors` 6 × `pipelines` 5 —— **中身ではなく被覆率**を扱う側である。

3. **identity の不整合は記録するが、この repo では直さない。** どちらの DID を
   正とするかは etzhayyim 側の identity 決定であり、snapshot が単独で決めてよい
   ことではない。live との差分も同じ理由で埋めない。**黙って辻褄を合わせず、
   測った値をそのまま書く。**

4. **`actor-manifest.test.ts` は走らないものとして扱う。** `package.json` も
   vitest も無い。この workspace は script host を nbb に寄せており
   （superproject ADR-2607173000）、`.ts` を走らせるために npm 依存を持ち込むより、
   同じ検査を nbb の `test/` として書き直す方が筋がよい。**この反復では
   書いていない**（下記）。

5. **quickstart は踏める手順だけを書く。** 書いた 5 手順はすべて 2026-08-09 に
   実際に実行し、出力を確認したものに限る。走らない `actor-manifest.test.ts`
   を「テストの走らせ方」として案内しない。

## この反復でやらなかったこと

- **test を書いていない。** README が主張する数（pipeline 5 / sub-actor 6 /
  cell 11 / gate 7 / target 32）と 2 つの DID を実体と突き合わせるものが無く、
  実体が動いても README は赤くならない。`marine-insurance` の
  `test/marine_insurance/docs_test.cljs` が先例で、同型のものがここにも要る。
  成熟度 loop は 1 反復 1 軸なので、これは次の反復（`axis-test`）に残す。
- **namespace の `oil_coverage`（アンダースコア）を直していない。** 慣習は
  `oil-coverage.murakumo` だが、直すと `require` の綴りが変わるので snapshot の
  範囲を越える。README に注記した。
- **DID document を live に合わせていない**（決定 3）。
- **`complianceDocs` の 2 パスが解決しない**ことは記録したが、元 monorepo 側の
  文書をここへ写してはいない。

## 帰結

- この repo を開いた者は、**何が宣言で何が実行か**を README の最初の表で 1 回で
  読める。以前は `actor-manifest.jsonld` を全部読んで推測するしかなかった。
- `oil-coverage` を 7 番目の segment と誤読する経路が塞がる。
- identity の不整合が**記録として残る**ので、etzhayyim 側が決めるときの入力になる。
- README の数値は現時点で機械検査されていない。**次の反復まではドリフトしうる**
  ——このことも README に書いてある。
