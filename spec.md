# DNS — Dogecoin Names System

**Version:** v1.3 — records belong to a holding; one priority key for every indexer  
**Ordering:** commit-first (§5.7) — one rule, no activation height  
**Status:** Launch draft — registration carrier is normative  
**Last Updated:** 2026-11-24  
**Chain:** Dogecoin  
**Protocol Marker:** `p: "dns"` (JSON) or plain `label.namespace`  
**Hero namespace:** `.doge`  
**Public copy:** <https://github.com/jonheaven/dns>

---

## 1. Introduction

DNS maps a human-readable name such as `jon.doge` to records the owner publishes: a Dogecoin address, an HTTPS site, an on-chain HTML inscription, an avatar, and free-form text.

Names are **Doginals inscriptions**. The first valid inscription of a given name wins. The product mint is plain text, so the tradeable inscription stays the name itself. The current holder of that inscription UTXO is the owner. Moving the UTXO transfers the name. A later `op:"config"` inscription from that holder updates the site and bio. It does not replace the name.

The mint follows Bitcoin’s Sats Names System: any `name.namespace` can be a name, and plain text is the name inscription. Dogecoin adds `op:"config"` so records can change without rewriting that inscription. The marker is `p:"dns"`. `.doge` is the hero suffix, not a gate. A pay-to address inside JSON is not the owner. JSON `{"p":"dns","op":"register"|"reg",...}` and a later `{"p":"dns","op":"config"|"update"|"upd",...}` are valid inscription bodies.

Optional. Orthogonal to ÐN05, Ð𝕏, ÐPFP, and @ handles. Product UIs MAY show a `.doge` name next to those binds.

`@jon` is not a DNS name. §6 rejects `@`. A holdable handle is its own inscription ([github.com/jonheaven/at](https://github.com/jonheaven/at)): plain text `@jon`, marker `p:"at"`. `jon.at` remains an ordinary open-namespace DNS name and does not create `@jon`.

### If your mint is an OP_RETURN, it did not register

This payload is **not** DNS. Indexers MUST ignore it. It does not create a name, an owner, or an inscription:

```text
DNS1|reg|example.doge|D8mZsgKwmSQWYRPEMCcm8KzNFj1JVF5UpA
```

UTF-8 hex begins `444e53317c7265677c` (`DNS1|reg|`). Block explorers that do not recognize the prefix label the output **Data Embedded in Transaction with Unknown Protocol**. That label means “nulldata we do not know.” It is not a content type, not a witness, and not a Doginal.

`DNS1` is not a protocol id. `|` is not a field separator. The address at the end is not the owner. The same string is still invalid if it is pasted into an inscription body: `|` cannot appear in a label, and the text is not `label.namespace`.

Bitcoin sats names put the name in an `ord` **witness** inscription. Dogecoin names are that same object — inscription content, first one wins, the UTXO is the name — carried the way Dogecoin Core will actually relay. See §5.

---

## 2. Status and Scope

- **Status:** Launch draft
- **In scope:** Name grammar, FCFS, plain-text register, JSON config updates by the holder, transfer-by-UTXO, indexer APIs, product gateway
- **Out of scope:** ICANN, UDRP, a public CA, a required desktop TLS helper, yearly rent, OP_RETURN names, the pipe codec `DNS1|reg|…`
- **Compatibility:** Existing plain-text `label.namespace` inscriptions remain valid registrations. A JSON register is valid only when it is the first inscription of that name.

---

## 3. Terminology

- **Name:** lowercase `label.namespace` (example `jon.doge`).
- **Label:** the left part (`jon`).
- **Namespace:** the right part (`doge`). Hero product is `.doge`.
- **Register inscription:** the FCFS inscription that created the name. Its UTXO **is** the name.
- **Owner:** P2PKH/P2SH address currently holding the register inscription UTXO.
- **Config:** a later inscription from the current holder. It updates records. It is not the name.
- **Site:** an HTTPS URL (`url`) and/or an inscription id of `text/html` content (`site`).
- **Gateway:** an HTTPS path on a Doginals host (`/n/jon.doge`) that opens `url` or the `site` inscription. Not ICANN DNS.

---

## 4. Design Goals

### Goals

- Names you can hold, sell, and gift — like a Doginal.
- Cheap FCFS from chain data; no registrar account.
- Point a name at a website or an on-chain HTML inscription (`jon.doge` as a site).
- Honor names already inscribed as plain `label.namespace`.
- Deterministic indexers: dogex is the reference.

### Non-Goals

- OP_RETURN names, including `DNS1|reg|<name>|<address>`. Nulldata cannot be transferred as a Doginal. Gift and offer memos in §13 are the only OP_RETURN this protocol defines, and neither one registers a name.
- Yearly leases / protocol rent (PepeNet). Ownership is the UTXO, not a calendar.
- Making `.doge` resolve in every browser without a helper or gateway. Product MAY ship a local helper later; v1 is the HTTPS gateway.
- Trademark protection.

---

## 5. Envelope format

A registration exists only when the name is the **body of an `ord` inscription**. Content type is `text/plain` or `application/json` (a `;charset=utf-8` parameter is accepted). The lab `dog` marker is for other protocols — DNS MUST use `ord`.

### 5.0 Carrier (normative) — P2SH commit, then reveal

Dogecoin Core rejects a Bitcoin tapscript envelope in a legacy scriptSig (`OP_FALSE OP_IF … OP_ENDIF` → `scriptsig-not-pushonly` / CLEANSTACK). Do not copy that script into `scriptSig`. Do not put the name in an OP_RETURN.

The mint that creates the inscription is two transactions:

1. **Commit.** One output pays a P2SH: `OP_HASH160 <HASH160(redeemScript)> OP_EQUAL`. With the reference redeem below that hash commits to the reveal **key only**, not to the inscription pushes (see the warning after the redeem). With the committed redeem from `committed-envelope.md` (Doginals spec set) it commits to the exact pushes.
2. **Reveal.** Spend that output. `vin[0].scriptSig` is **push-only**. Dogecoin historical indexers read **`vin[0].scriptSig` only**, so one reveal carries one name. The inscription id is `<reveal-txid>i0`. The address that receives the reveal’s inscription output (reference wallets use vout 0) is the owner.

Reference redeem script for the partial that holds the envelope pushes (`n` = how many pushes are in that partial, not how many bytes are in the name):

```text
<33-byte compressed pubkey> OP_CHECKSIGVERIFY OP_DROP×n OP_TRUE
```

> **This redeem does not bind the name.** `OP_DROP` discards every push and a legacy signature never covers a scriptSig, so whoever relays or mines the reveal can swap the envelope pushes (same count) and the transaction stays valid. A config's records can be rewritten in flight this way. Wallets SHOULD use the **committed redeem** instead (one `<literal> OP_EQUALVERIFY` or `OP_HASH160 <h> OP_EQUALVERIFY` per push, `committed-envelope.md` (Doginals spec set)). The scriptSig layout is the same, so every indexer that reads this section reads both. Only committed reveals rank by their commit (§5.7); a classic reveal ranks by its reveal.

Reveal `scriptSig`, in order:

```text
<envelope pushes…>
<DER signature || SIGHASH_ALL (0x01)>
<redeem script>
```

**Envelope pushes implementers MUST produce** (doginals.js multipart; this is what the reference indexer tries first). For a body that fits in one chunk:

```text
PUSH("ord")
OP_1                         # parts count = 1
PUSH(content-type)           # "text/plain" or "application/json"
OP_0                         # countdown = 0
PUSH(body)                   # UTF-8 name, or UTF-8 JSON
```

Worked plain-text registration of `example.doge` (`text/plain`). These bytes are the envelope only — the signature and redeem script follow them:

```text
03 6f7264
51
0a 746578742f706c61696e
00
0c 6578616d706c652e646f6765
```

That is `ord`, parts `1`, `text/plain`, countdown `0`, body `example.doge`. A name is registered when a Core-accepted reveal’s `vin[0].scriptSig` contains this envelope (or the classic form below) and the body parses as §5.1 or §5.2. It is not registered when the only payload is nulldata.

Indexers MUST also accept the classic single-chunk form, so older inscriptions still count:

```text
PUSH("ord") PUSH(content-type) PUSH(0x00) PUSH(body)
```

Chunks longer than one piece use the same multipart rules: parts count `N`, then `N` pairs of a descending countdown and a data push (countdown starts at `N-1` and ends at `0`). A short name is one chunk. Do not split a name across reveals.

Broadcast **commit, then reveal**, to Dogecoin Core (`sendrawtransaction` or a proxy that forwards to Core). Wait until Core returns the commit txid before broadcasting the reveal — the reveal spends the commit. A txid from a public relay that never appears in `getrawtransaction` is not a mint.

The reference inscription output value is 100000 koinu (0.001 DOGE) to a standard address. Any spendable output above dust that carries this inscription is the name UTXO. A studio registrar fee on the reveal is product-only (§13) and MUST NOT be required for validity.

A tapscript witness envelope is not the mint path for this spec. If one is confirmed and the body parses, an indexer MAY treat it as an inscription. A website mint MUST still use the P2SH reveal above.

### 5.1 Plain-text register (normal mint)

Body is UTF-8. Indexers MUST take the first whitespace-delimited token and ignore the rest of the inscription:

```text
jon.doge
```

Indexers MUST treat this as a register with empty records. No config fields.

### 5.2 JSON register (advanced mint)

```json
{
  "p": "dns",
  "op": "register",
  "name": "jon.doge",
  "address": "D...",
  "url": "https://example.com",
  "site": "<inscription_id>",
  "avatar": "<inscription_id>",
  "content": "hello from chain",
  "cname": "jon.doge"
}
```

Required: `p` = `"dns"`, `op` = `"register"` or `"reg"`, `name` a valid name. All record fields are optional. Omit `address` unless the minter is setting a pay-to that can differ from the holder.

`rev` is an alias of `address` (pay-to, not owner). `relay` is an alias of `site`. `website` is an alias of `url`. `insc` / `html` are aliases of `site`. A trailing comma before `}` is accepted.

### 5.3 JSON config (update records)

```json
{
  "p": "dns",
  "op": "config",
  "name": "jon.doge",
  "url": "https://example.com",
  "content": "updated bio"
}
```

`op` MAY be `"update"` or `"upd"`. The config inscription is not the name. It does not transfer.

Record fields MAY also sit inside a `config` object. Indexers MUST read that object as the same fields. A top-level key wins when both are present. A body with `p:"dns"`, a valid `name`, and record fields but no `op` is a config, not a second registration.

Dogecoin inscriptions split a large body across 520-byte pushes. Indexers MUST concatenate every body push before parsing. A config that does not fit in one push is still one JSON document. Dropping the last short chunk, or keeping only the last chunk, is non-compliant.

A config counts only when the name already exists and the config inscription’s owner is the holder of the name **after every spend in that same transaction has been applied**. A config in the transaction that gives the name away does not stick to the new owner. The new owner may publish their own config in that transaction (their config output and the name output both pay them) or in a later one.

**Records belong to a holding, not to the name.** When the register output moves and the address that can spend it changes (a sale, a gift, a move to another address in the same wallet, or a burn), every record resets to empty, including records set by a JSON register (§5.2), and the name counts as never configured. A move that pays the same address keeps them. The reset happens when the spend is applied, so a config in that same transaction that pays the new owner (above) is the new holder’s first config. The old holder’s primary is already dropped (§5.6). This keeps a seller’s `address` record from receiving the buyer’s payments.

Omitted keys do not clear existing values. Set a scalar field to `""` to clear it. Set `addresses` to `[]` to clear extra pay-tos. Set `txt` to `null` to clear the whole TXT map. A TXT value of `""` deletes that one key.

**Order.** Inscriptions are applied in block order: lower height, then lower transaction index, then lower inscription index in the transaction. Registers rank by the §5.7 priority key; a register whose commit does not qualify ranks by this order (so when two such registers share a block, the earlier one wins and the later one is a missed claim). Configs use this order. The last valid config from the current holder wins. An older config replayed after a newer one is ignored. Indexers MUST see earlier writes from the same block before they judge a later one (a write batch that is still uncommitted is not a reason to miss the earlier name).

### 5.4 Transfer

There is no `op:"transfer"`. Spend the register inscription UTXO to the new owner. Indexers MUST resolve owner from the inscription location, not from a protocol opcode. A transfer to a different owner resets the records (§5.3).

`op:"renew"` is **not** defined. Ignore it.

### 5.5 Invalid carriers (normative)

Indexers MUST NOT create a name from any of the following:

| What was broadcast | Why it is not a name |
| --- | --- |
| OP_RETURN / nulldata `DNS1\|reg\|<name>\|<address>` | Unrecognized embedded data. No inscription body, no content type, no inscription id. |
| OP_RETURN of the JSON object in §5.2 or §5.3 | Same. JSON counts only as inscription content. |
| `OP_FALSE OP_IF "ord" … OP_ENDIF` inside a legacy scriptSig | Core rejects the input. Nothing is indexed. |
| Inscription body `DNS1\|reg\|…` | Not `label.namespace` and not `p:"dns"` JSON. |
| Pipe fields, `DNS1`, `op:"transfer"`, `op:"renew"` | Undefined. Ignore. |

The only OP_RETURN strings this protocol defines are the memos in §13 (`dns gift …`, `dns offer …`). They ride **beside** a real inscription. They never replace it.

### 5.6 Primary (the name this address wears)

SNS `rev` is a pay-to record on a name. It does not choose which of several names an address displays. Bitcoin’s later `.btc` drafts call that choice `primary-name`. Dogecoin uses one inscription:

```json
{
  "p": "dns",
  "op": "primary",
  "name": "jon.xdg"
}
```

`name` is required and MUST be a valid `label.namespace`. This inscription is not a registration and does not transfer the name.

It counts only when all of these are true:

1. `name` is already registered (first inscription won).
2. The primary inscription’s owner is the current holder of that name.
3. It is the latest such inscription from that holder (higher block, then later in the block). A later valid primary replaces the earlier one.

`{"p":"dns","op":"primary","name":""}` clears the choice. Sending the name away clears it for the previous holder. The new holder does not inherit it. A primary from someone who does not hold the name is ignored, so a wallet cannot wear a name it does not own.

One address has at most one primary. A wallet that holds many names wears only the one it set.

---

### 5.7 Priority: commit first (normative)

This is the one ordering rule for every register, from genesis. Pure reveal order would let anyone who sees a reveal in the mempool copy the name into their own commit and reveal and win if they are mined first. So a register ranks by its **commit** position when the reveal proves the commit fixed its content, and by its reveal position otherwise.

**Priority key** of a register, compared as a tuple, lower first:

- `(commit_height, commit_tx_index, commit_vout)` when the reveal is a single-transaction inscription, `vin[0]` is a committed reveal (`committed-envelope.md` (Doginals spec set) §3), and the output it spends confirmed at most **W = 30** blocks before the reveal;
- otherwise `(reveal_height, reveal_tx_index, 0)`.

**Rule.** A register for a free name wins. For a taken name, it wins (and the existing register becomes a missed claim) only when its key is lower than the existing row's key and the tip has not passed `existing_key_height + W` (the row is still **pending**). Once a row is past that height it is **final**: no later reveal could still out-rank it, because a commit older than `W` blocks no longer gives priority. Final rows are never displaced. A register with no qualifying commit simply carries its reveal key, so reveal order still decides between such registers.

**Bitwork.** A JSON register MAY carry `"bitworkc"` (commit txid) and/or `"bitworkr"` (reveal txid), `bitwork.md` (Doginals spec set). A claim the txid does not meet voids the register (logged, not a name). Apps MAY show the commit txid's leading zeros as a badge.

**Wallets** SHOULD broadcast the commit, wait for one confirmation, then broadcast the reveal. Before the reveal is public nobody can learn the name; after the commit confirms nobody can get an earlier commit.

**Reorgs** undo displacement: the pre-block row, its inscription→name map and owner index come back.

**API.** `GET /api/dns/name/:name` adds `status` (`pending` | `final`) and `priority { key, from_commit, bound, commit_txid, bitworkc, bitworkr, commit_zeros, final_after_height }`. `key` is the tuple above.

**Indexers built before this rule** stored names in reveal order with no priority key. Such a row is treated as final and never displaced until the indexer reindexes from genesis under this rule.

**One key for every indexer.** An indexer that replays history (a catch-up lane, a second implementation, a rebuild) MUST compute this same key. It keeps the P2SH outputs of the last `W` blocks so a committed reveal ranks by its commit, exactly as a live indexer does. Ranking a replay by reveal height alone is not conformant: it disagrees with a live indexer whenever a committed reveal wins inside the window.

When a replay meets a row that a live indexer wrote from a **later** block, it applies the rule in block order: the earlier reveal is the incumbent, and the later row keeps the name only if its key could have displaced that earlier reveal under the rule above.

A JSON register whose `bitworkc` or `bitworkr` claim the txid does not meet is void everywhere, including in a replay. It is not even a missed claim.

| Vector | Setup | Winner |
| --- | --- | --- |
| Mempool copy | A commits `alice.doge` (committed) at h=1; B copies it with a classic reveal at h=2; A reveals at h=3 | A (key 1 < 2); B becomes a missed claim |
| Late commit | A commits+reveals at h=1; B commits+reveals at h=2 | A |
| Slow reveal | A commits at h=1, reveals at h=33 (> W); B reveals at h=2 | B (A ranks by its reveal) |
| Classic vs classic | A reveals `bob.doge` (classic) at h=1; B reveals it (classic) at h=2 | A (reveal order) |
| Same block | A and B both commit at h=1 (A at tx 1, B at tx 2) and reveal at h=2 | A (lower commit tx index) |
| Window edge | A commits at h=1 and reveals at h=31 (31 − 1 = W) | A ranks by its commit (key height 1); revealed at h=32 it ranks by its reveal |
| Bitwork | `{"p":"dns","op":"reg","name":"erin.doge","bitworkc":"0"}`, commit txid starts `0` | valid; same with `"0000000000"` → void |
| Replay, committed | A commits `carol.doge` (committed) at h=99 and reveals at h=110; B reveals it (classic) at h=100; an indexer replays both | A, the same as a live indexer |
| Replay meets a later row | A live indexer stored B (classic) from h=6,000,000; a replay then reaches A (classic) at h=4,700,000 | A; B becomes a missed claim |

---

## 6. Name rules

Indexers MUST:

1. Lowercase the name (ASCII).
2. Split on the first `.` into exactly two parts: `label` and `namespace`.
3. Reject (ignore) anything else — no `a.b.doge`, no leading/trailing dots.

**Label** and **namespace** follow Bitcoin SNS spirit with DNS-like ASCII hygiene:

- **Non-ASCII** is a letter, a number, a combining mark (`M`), or an emoji. ASCII punctuation is not a name character.
- **Emoji** means Unicode `Extended_Pictographic` (UTS #51 emoji-data, Unicode 18.0, 2026-01-30), plus `Emoji_Modifier` (skin tone) and `Regional_Indicator` (flags). ZWJ (U+200D), variation selectors (U+FE0E, U+FE0F), and the keycap combiner (U+20E3) may join a sequence. A hand-listed block range is not conformant: it drops emoji that live outside those blocks. `⭐` (U+2B50) is a name character. `⭐.doge` is a valid registration.
- Do **not** substitute the binary property `Emoji=Yes`. That property also marks ASCII `#`, `*`, and digits, which stay outside the name character set.
- **ASCII** only `[a-z0-9-]` after **fullwidth ASCII fold** (U+FF01..FF5E → ASCII). Letters indexed lowercase. **No underscore** (avoids `self.total_supply`). **No apostrophe** — straight `'`, curly `’` `‘`, or modifier `ʼ`. Hyphen ok mid-label (`doge-labs`), not leading/trailing.
- No `/ ? # @ :` or path strings like `d.gg/invite`. A leading `@` and no dot is the @ handle protocol, not a label. Indexers MUST keep rejecting it here.
- Reject whitespace, controls, invisible format/bidi junk, and slash/dot lookalikes (`／` `。` fraction slash, …).
- Cap each part at **63 Unicode scalars**.
- Stored form is folded + lowercased so `Ｊｏｎ.doge` and `jon.doge` are one name.
- **One letter-script per part.** Digits, hyphen, combining marks, and emoji may sit with that script. A label that mixes letter scripts (Latin `j` plus Cyrillic `о`) is rejected. This is the homoglyph control. Do not fold Cyrillic or Greek onto ASCII — that would let a lookalike squat the Latin name.
- A name that is entirely an ASCII lookalike in another script (Cyrillic `сорех.doge`) is still a distinct name. Indexers MUST set `confusable: true` on it. Product UIs SHOULD show a lookalike warning and SHOULD prefer ASCII in search suggestions. The stored key does not change.

Open namespaces mean SNS-style leftovers like `readme.md`, `1.dogimap`, and doge-speak `such.123` can mint — intentional FCFS. No reserved-label list.

**Carve-outs (Dogecoin-native only):** `bitmap` and `dogemap` / `dogemap*` (ÐMaps). Do **not** denylist other `*map` or digit suffixes — that fights reverse names and open SNS.

Product UIs SHOULD default new registrations to `.doge`, and MAY feature **doge-speak** names (`such.jon`, `very.cool`, `doge.you`) where the meme word is the **left** label. That is the same `label.namespace` grammar — not a second protocol. `such.jon` and `jon.such` are different names.

Indexers MAY expose `GET /api/dns/list?prefix=such` (match on the left label) so product browse does not treat every right-hand word as a first-class TLD in the UI. Suffix `namespace` indexing is unchanged.

---

## 7. Records

| Field | Meaning | Validation |
| --- | --- | --- |
| `address` | Pay-to. SNS calls this `rev`. Not the owner, and not the name the address displays. Resets when the name changes owner (§5.3). | Dogecoin address length 26–35 |
| `url` | Off-chain website | `http://` or `https://`, max 512 chars |
| `site` | On-chain site. SNS calls this `relay`. | Inscription id `txid` + `i` + index |
| `avatar` | Image inscription or URL | Inscription id or `https://` URL, max 128 chars |
| `content` | Bio / text | UTF-8, max 280 chars |
| `cname` | Alias hint | Valid name; product MAY redirect `/n/a` → `/n/b` |
| `nostr` | Nostr pubkey | 64 hex characters, or `npub1…` |
| `lightning` | Lightning address | `user@host`, or `lnurl1…`. Alias `lud16` |
| `addresses` | Extra pay-tos | Array of up to 8 Dogecoin addresses. `address` stays the primary. `[]` clears |
| `txt` | Free-form notes | Object, up to 8 keys per inscription. Key is `[a-z0-9-]{1,32}`. Value max 280 UTF-8 chars. Later configs overlay per key. `null` clears the map |
| `subdomains` | Off-chain child commitment | 64 hex characters (32-byte Merkle root). Alias `merkle`. Not a registration |
| `tlsa` | Optional DANE certificate pins | Array of up to 4 strings `"<usage> <selector> <matching> <hex>"`: usage `2` or `3`, selector `0` or `1`, matching `1` (64 hex) or `2` (128 hex), lowercase. `[]` clears |

Every record is the current holder’s. All of them reset to empty when the name changes owner (§5.3). After a sale, a wallet that resolves `jon.doge` pays the owner address until the buyer writes a config. It never pays the seller’s old `address`.

`url` is the PepeNet-style “point your name at a website.” `site` is “put a site on chain” (HTML Doginal). A name MAY have both; the gateway prefers `url` for a live redirect and still exposes `site` as an on-chain copy.

`tlsa` is optional. It carries DANE TLSA associations (RFC 6698 / RFC 7671 presentation form) for the
HTTPS server behind `url`, or for a local TLS helper that serves `https://<name>` (§10, §15). A client
that supports it MUST NOT accept a certificate that matches none of the entries; a client that does not
support it ignores it. It is a record like the others: it changes nothing about who owns the name, and an
invalid value is ignored like any other invalid record field. Example:
`"tlsa":["3 1 1 0c72ac70b745ac19998811b131d662c9ac69dbdbe7cb23e5b514b56664c5d3d6"]`.

`subdomains` is optional and off by default. It commits a Merkle root of child labels (`pay` under `jon.doge` → `pay.jon.doge` as a proof, not as a second L1 name). Resolving a child is a proof against that root. Indexers store the root and do not walk children. A name with no root has no subhandles. This is the Spaces-style scale hatch: pure FCFS names ignore it, and it is not a namespace the indexer sells.

---

## 8. Indexer state

For each name:

- `name`, `namespace`
- `inscription_id` (register)
- `sequence_number`, `inscription_number`, `height`, `timestamp` of the register
- `records` from the winning register, then any later config from the current holder; empty again after every change of owner (§5.3)
- `owner` from the register inscription’s current `owner_hash160` / satpoint
- `primary` for an address: the latest valid §5.6 inscription, cleared when that address no longer holds the name
- `supply` for that namespace and label length: how many names of each charset are taken, plus the finite ASCII pool (`26^n` letters, `10^n` digits, `36^n` letters-and-digits) for lengths 1–6. Emoji and 7+ have no cap. This is remaining supply, not a price rank.

**FCFS:** the register with the lower §5.7 priority key wins — its commit position when bound and committed within 30 blocks, else its reveal position. A later register does not change the owner. Indexers keep that later inscription as a missed claim (`dns_lost`) so a name page can show how many copies lost the race. Two registers in one block are not a tie the later write may overwrite.

**Reorg:** delete names created in the rolled-back block, and restore any row a rolled-back register displaced (§5.7).

A pending row (§5.7) can be displaced by a lower key; a final row cannot.

Indexers MUST NOT treat a config inscription as a second name.

---

## 9. HTTP (dogex)

Chain truth:

| Method | Path |
| --- | --- |
| GET | `/api/dns/status` |
| GET | `/api/dns/list?namespace=doge&offset=&limit=&hasContent=&pattern=&market=&signals=` |
| GET | `/api/dns/recent?limit=` — newest registrations (activity feed) |
| GET | `/api/dns/stats/audit` — holders, patterns, lengths, market counts, missed FCFS |
| GET | `/api/dns/offers?limit=` — names with on-chain bids, busiest first |
| GET | `/api/dns/name/:name` |
| GET | `/api/dns/name/:name/claims` — later registers that lost FCFS |
| GET | `/api/dns/address/:address` |
| GET | `/api/dns/primary/:address` — chosen name, or `name: null` |
| GET | `/api/dns/resolve/:name` |

`/api/doginals/dns/name/:name`, `/namespace/:ns`, `/stats` remain; they MUST return the same record object plus owner/records (additive).

`resolve` is an alias of `name` with `ok` / `available` for product clients. An available name still includes `supply` for that length so a mint screen can show how many are left before the claim.

---

## 10. Product (command.dog + web-com)

command.dog MAY probe `url` (HEAD/GET, short timeout, no private IPs) and attach `live: true|false` to a directory. That probe is **not** chain truth.

Storefront (`dogenals/web-com`):

- `/names` — search, register, recent claim feed, explore, my names (primary)
- `/names/stats` — holder, pattern, and missed-FCFS audit
- `/names/:name` — name profile + point-at-site. The name is shown as the same domain card as the inscription page, including how many later claims missed FCFS
- `/dns` · `/dns/:name` — same product (tech mirror)
- `/n/:name` — gateway
- `/n/:name` — gateway: redirect to `url`, or iframe `/cdn/content/:site`

A local TLS helper that makes `https://jon.doge` open in the address bar is **optional product**, not protocol. v1 MUST work with the HTTPS gateway alone.

Wallet: Dig Local Browser Wallet. Sign with `signInscriptionTxs` (P2SH commit → reveal). Broadcast command.dog → Core. Never public relays.

---

## 11. Security

- FCFS is economic (fee + squat). There is no UDRP.
- Mempool copying: a classic reveal can be copied and out-raced. Use a committed reveal and wait for the commit to confirm (§5.7).
- In-flight rewriting: the classic redeem (§5.0) lets a relayer swap envelope pushes. The committed redeem prevents it.
- Stolen keys transfer the name. Same as any Doginal.
- A sale resets the records (§5.3). A seller cannot leave an `address` record behind to catch the buyer’s payments.
- `url` / `site` can phish. The gateway MUST show the destination before an automatic redirect on first visit in a session, and MUST sandbox on-chain HTML (`iframe` sandbox).
- Live probes MUST block loopback, RFC1918, link-local, and metadata IPs (SSRF).
- Config from a non-owner is ignored (not an error that forks the name).

---

## 12. Examples

See [`examples/`](examples/) and [`vectors/conformance.json`](vectors/conformance.json).

---

## 13. Product notes (not protocol)

Studio mint UIs (`/names`, mirror `/dns`) MAY attach a registrar fee on the register reveal
(e.g. 1 DOGE to an operator house address). That fee is **product-only**. Indexers MUST
NOT require it for a valid FCFS register. DIY / CLI mints without the fee remain valid.

A checkout of several names is one or more commit transactions, each with one P2SH
output per name, then one reveal per name. One commit carries at most 20 names.
Dogecoin Core rejects a mempool package past 25 transactions, and 20 leaves room
when the funding coin is still unconfirmed. A larger cart is several of those
payments in one wallet approval. Each payment spends a different confirmed coin,
so a single fat coin has to be split and that split has to confirm before the
payments are signed. The mint page accepts a pasted list (spaces, commas, or lines). Taken
and invalid names are dropped before they enter the cart. A word with no dot is claimed in
the `.doge` namespace. Dogecoin historical indexers read `vin[0].scriptSig` only, so one reveal
carries one inscription. Extra lines in a plain-text body are not extra registrations.

A gift message is a value-0 OP_RETURN on the reveal: `dns gift <message>` (80-byte cap,
message clipped). It is not part of the name. The gift inscription body is the plain name
only. Site, bio, and other records are a JSON register the minter keeps, or a later
`config` from the holder. Indexers MUST ignore the OP_RETURN for FCFS. A list of
other `label.namespace` tokens in an inscription body is not a gift message and not a mint.

**`.doge` on other systems (not protocol).** DogeNet, the Dogecoin deployment of PepeNet, keeps its
own name registry, and it can also hold `.doge` names. The same label can have different holders in
DNS and in DogeNet. DNS ownership is decided only by this spec (permanent names, commit-first priority,
§5.7); a DogeNet registration never changes a DNS name, and DNS never reads DogeNet state. Products that
show both SHOULD label the source of every record and MUST NOT merge records across the two systems. A
user who wants one name in both registers it in both. A DANE-aware DogeNet browser can use a DNS name's
`tlsa` record when that browser resolves the name through DNS.

Offers are product attention signals: a micro DOGE payment to the current owner plus an
OP_RETURN that states the bid the buyer is willing to pay. The bid amount is not sent as
the payment. They do not transfer the name and are not indexed as DNS ownership ops.

---

## 14. Name rarity (product overlay)

Deterministic trait → score → tier → clubs for every `label.namespace`.
Works on unminted names. Not an on-chain field. See [`rarity.md`](rarity.md)
(`rarity-v1`). Tier names match Rare Koinu: mythic → … → common.

## 15. Deferred

- Wildcard ICANN-style `*.doge.dogecoin.dog` host
- Desktop TLS helper (PepeNet-desktop analogue)
- ÐMP name market UI (the inscription is already listable)
- Subnames as child inscriptions (`pay.jon.doge` on L1). The optional `subdomains` Merkle root is the scale path; proof checks are product, not a second indexer
- On-chain proof verification of that root (v1.2 stores the commitment only)
- Escrow / atomic accept-offer settlement
- Empirical `#rank / supply` sort among taken names (dogex publishes remaining ASCII supply per namespace and length; it does not order one name ahead of another)

## 16. Changelog

- **v1.3 (2026-11-24).** Records belong to a holding: they reset when the owner changes (§5.3, §7). Replays MUST rank with the §5.7 key, keep the last `W` blocks of P2SH outputs, and honor bitwork (§5.7). `vectors/conformance.json` gains `sequences`.
- **2026-11-20.** Commit-first priority (§5.7) became the one ordering rule, with no activation height.
- **v1.2 (2026-10-18).** Records, lookalike rules, and same-block ordering.
