# DNS — Dogecoin Names System

**Version:** v1.1 (carrier lock)  
**Status:** Launch draft — registration carrier is normative  
**Last Updated:** 2026-10-15  
**Chain:** Dogecoin  
**Protocol Marker:** `p: "dns"` (JSON) or plain `label.namespace`  
**Hero namespace:** `.doge`  
**Public copy:** <https://github.com/jonheaven/dns>

---

## 1. Introduction

DNS maps a human-readable name such as `jon.doge` to records the owner publishes: a Dogecoin address, an HTTPS site, an on-chain HTML inscription, an avatar, and free-form text.

Names are **Doginals inscriptions**. The first valid inscription of a given name wins. The product mint is plain text, so the tradeable inscription stays the name itself. The current holder of that inscription UTXO is the owner. Moving the UTXO transfers the name. A later `op:"config"` inscription from that holder updates the site and bio. It does not replace the name.

The mint follows Bitcoin’s Sats Names System: any `name.namespace` can be a name, and plain text is the name inscription. Dogecoin adds `op:"config"` so records can change without rewriting that inscription. The marker is `p:"dns"`. `.doge` is the hero suffix, not a gate. A pay-to address inside JSON is not the owner. JSON `{"p":"dns","op":"register"|"reg",...}` and a later `{"p":"dns","op":"config"|"update"|"upd",...}` are valid inscription bodies.

Optional. Orthogonal to ÐN05, Ð𝕏, and ÐPFP. Product UIs MAY show a `.doge` name next to those binds.

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
- **Gateway:** an HTTPS path on a Dogenals host (`/n/jon.doge`) that opens `url` or the `site` inscription. Not ICANN DNS.

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

1. **Commit.** One output pays a P2SH whose redeem script commits to the inscription pushes:
   `OP_HASH160 <HASH160(redeemScript)> OP_EQUAL`.
2. **Reveal.** Spend that output. `vin[0].scriptSig` is **push-only**. Dogecoin era-1 indexers read **`vin[0].scriptSig` only**, so one reveal carries one name. The inscription id is `<reveal-txid>i0`. The address that receives the reveal’s inscription output (reference wallets use vout 0) is the owner.

Reference redeem script for the partial that holds the envelope pushes (`n` = how many pushes are in that partial, not how many bytes are in the name):

```text
<33-byte compressed pubkey> OP_CHECKSIGVERIFY OP_DROP×n OP_TRUE
```

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

A config counts only when the name already exists and the config inscription’s owner is the current holder of the name inscription. Omitted keys do not clear existing values. Set a field to `""` to clear it.

### 5.4 Transfer

There is no `op:"transfer"`. Spend the register inscription UTXO to the new owner. Indexers MUST resolve owner from the inscription location, not from a protocol opcode.

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

---

## 6. Name rules

Indexers MUST:

1. Lowercase the name (ASCII).
2. Split on the first `.` into exactly two parts: `label` and `namespace`.
3. Reject (ignore) anything else — no `a.b.doge`, no leading/trailing dots.

**Label** and **namespace** follow Bitcoin SNS spirit with DNS-like ASCII hygiene:

- **Unicode / emoji** allowed (non-ASCII). ZWJ + emoji variation selectors allowed for sequences.
- **ASCII** only `[a-z0-9-]` after **fullwidth ASCII fold** (U+FF01..FF5E → ASCII). Letters indexed lowercase. **No underscore** (avoids `self.total_supply`). Hyphen ok mid-label (`doge-labs`), not leading/trailing.
- No `/ ? # @ :` or path strings like `d.gg/invite`.
- Reject whitespace, controls, invisible format/bidi junk, and slash/dot lookalikes (`／` `。` fraction slash, …).
- Cap each part at **63 Unicode scalars**.
- Stored form is folded + lowercased so `Ｊｏｎ.doge` and `jon.doge` are one name.

Open namespaces mean SNS-style leftovers like `readme.md`, `1.dogimap`, and doge-speak `such.123` can mint — intentional FCFS. No reserved-label list.

**Carve-outs (Dogecoin-native only):** `bitmap` and `dogemap` / `dogemap*` (ÐMaps). Do **not** denylist other `*map` or digit suffixes — that fights reverse names and open SNS.

Product UIs SHOULD default new registrations to `.doge`, and MAY feature **doge-speak** names (`such.jon`, `very.cool`, `doge.you`) where the meme word is the **left** label. That is the same `label.namespace` grammar — not a second protocol. `such.jon` and `jon.such` are different names.

Indexers MAY expose `GET /api/dns/list?prefix=such` (match on the left label) so product browse does not treat every right-hand word as a first-class TLD in the UI. Suffix `namespace` indexing is unchanged.

---

## 7. Records

| Field | Meaning | Validation |
| --- | --- | --- |
| `address` | Pay-to. SNS calls this `rev`. Not the owner. | Dogecoin address length 26–35 |
| `url` | Off-chain website | `http://` or `https://`, max 512 chars |
| `site` | On-chain site. SNS calls this `relay`. | Inscription id `txid` + `i` + index |
| `avatar` | Image inscription or URL | Inscription id or `https://` URL, max 128 chars |
| `content` | Bio / text | UTF-8, max 280 chars |
| `cname` | Alias hint | Valid name; product MAY redirect `/n/a` → `/n/b` |

`url` is the PepeNet-style “point your name at a website.” `site` is “put a site on chain” (HTML Doginal). A name MAY have both; the gateway prefers `url` for a live redirect and still exposes `site` as an on-chain copy.

---

## 8. Indexer state

For each name:

- `name`, `namespace`
- `inscription_id` (register)
- `sequence_number`, `inscription_number`, `height`, `timestamp` of the register
- `records` from the winning register, then any later config from the current holder
- `owner` from the register inscription’s current `owner_hash160` / satpoint

**FCFS:** if `dns_name` already has a key, ignore later registers of the same name.

**Reorg:** delete names created in the rolled-back block.

Indexers MUST NOT treat a config inscription as a second name.

---

## 9. HTTP (dogex)

Chain truth:

| Method | Path |
| --- | --- |
| GET | `/api/dns/status` |
| GET | `/api/dns/list?namespace=doge&offset=&limit=&hasContent=` |
| GET | `/api/dns/recent?limit=` — newest registrations (activity feed) |
| GET | `/api/dns/name/:name` |
| GET | `/api/dns/address/:address` |
| GET | `/api/dns/resolve/:name` |

`/api/doginals/dns/name/:name`, `/namespace/:ns`, `/stats` remain; they MUST return the same record object plus owner/records (additive).

`resolve` is an alias of `name` with `ok` / `available` for product clients.

---

## 10. Product (command.dog + web-com)

command.dog MAY probe `url` (HEAD/GET, short timeout, no private IPs) and attach `live: true|false` to a directory. That probe is **not** chain truth.

Storefront (`dogenals/web-com`):

- `/names` — search, register, recent claim feed, explore, my names (primary)
- `/names/:name` — name profile + point-at-site
- `/dns` · `/dns/:name` — same product (tech mirror)
- `/n/:name` — gateway
- `/n/:name` — gateway: redirect to `url`, or iframe `/cdn/content/:site`

A local TLS helper that makes `https://jon.doge` open in the address bar is **optional product**, not protocol. v1 MUST work with the HTTPS gateway alone.

Wallet: Dojak Local Browser Wallet. Sign with `signInscriptionTxs` (P2SH commit → reveal). Broadcast command.dog → Core. Never public relays.

---

## 11. Security

- FCFS is economic (fee + squat). There is no UDRP.
- Stolen keys transfer the name. Same as any Doginal.
- `url` / `site` can phish. The gateway MUST show the destination before an automatic redirect on first visit in a session, and MUST sandbox on-chain HTML (`iframe` sandbox).
- Live probes MUST block loopback, RFC1918, link-local, and metadata IPs (SSRF).
- Config from a non-owner is ignored (not an error that forks the name).

---

## 12. Examples

See [`examples/`](examples/) and [`vectors/conformance.json`](vectors/conformance.json).

---

## 13. Product notes (not protocol)

Studio mint UIs (`/names`, mirror `/dns`) MAY attach a registrar fee on the register reveal
(e.g. 1.1 DOGE to an operator house address). That fee is **product-only**. Indexers MUST
NOT require it for a valid FCFS register. DIY / CLI mints without the fee remain valid.

A checkout of several names is one commit transaction with one P2SH output per name, then
one reveal per name. Dogecoin era-1 indexers read `vin[0].scriptSig` only, so one reveal
carries one inscription. Extra lines in a plain-text body are not extra registrations.

A gift message is a value-0 OP_RETURN on the reveal: `dns gift <message>` (80-byte cap,
message clipped). It is not part of the name. The gift inscription body is the plain name
only. Site, bio, and other records are a JSON register the minter keeps, or a later
`config` from the holder. Indexers MUST ignore the OP_RETURN for FCFS. A list of
other `label.namespace` tokens in an inscription body is not a gift message and not a mint.

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
- Subnames (`pay.jon.doge`) as child inscriptions
- Escrow / atomic accept-offer settlement
- Empirical `#rank / supply` from indexer histograms
