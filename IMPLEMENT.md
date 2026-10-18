# Implement a Dogecoin name mint

Plain text and JSON are both valid inscription bodies.

Read this, then [spec.md](spec.md). If they disagree, spec.md wins. [rarity.md](rarity.md) is a display score. It is not part of minting.

You are here because a minter wrote a name into an OP_RETURN and the explorer said **Data Embedded in Transaction with Unknown Protocol**. That transaction did not register a domain.

## The bug

This is not a Dogecoin name:

```text
DNS1|reg|example.doge|D8mZsgKwmSQWYRPEMCcm8KzNFj1JVF5UpA
```

Hex of that prefix is `444e53317c7265677c` (`DNS1|reg|`). It sits in a nulldata output (`OP_RETURN`, script starting `6a`). Explorers that do not know the prefix file it as unknown embedded data. There is no inscription, so there is no `text/plain` body and no `application/json` body.

Delete that codec. Do not rename `DNS1` to something else and keep the OP_RETURN. Do not wrap the same string in JSON and put *that* in an OP_RETURN.

## What Bitcoin sats names do, and what you do on Dogecoin

Sats names are an `ord` inscription. The name is the inscription **content**. The first one wins. Whoever holds that output owns the name.

Dogecoin is the same object with a different carrier. The name is still inscription content, in plain text or in JSON (`p:"dns"`). Dogecoin Core rejects the Bitcoin witness script (`OP_FALSE OP_IF … OP_ENDIF`) when it is placed in a legacy `scriptSig` (`scriptsig-not-pushonly` / CLEANSTACK). The inscription Dogecoin will relay, and that indexers read, is a **P2SH commit → reveal**:

1. Commit transaction pays a P2SH output.
2. Reveal transaction spends it. The name is pushes inside `vin[0].scriptSig`.
3. One reveal, one name. Inscription id = `<reveal-txid>i0`.
4. The address paid by the reveal’s inscription output owns the name. Spending that output transfers the name. There is no `transfer` opcode.

`vin[0].scriptSig` is push-only. The bytes `6f7264` (`ord`) appear there, not in an OP_RETURN.

## Body (pick one)

**Normal mint.** Content type `text/plain`. Body is the name and nothing else:

```text
example.doge
```

**Mint that also sets records.** Content type `application/json`. Body is one JSON object, UTF-8, no `DNS1`, no pipes:

```json
{"p":"dns","op":"register","name":"example.doge","url":"https://example.com"}
```

`op` may be `register` or `reg`. `p` is the three letters `dns`, lowercase.

**Later record change.** A second inscription, same envelope, content type `application/json`, from the current holder:

```json
{"p":"dns","op":"config","name":"example.doge","url":"https://example.com"}
```

`update` and `upd` mean config. A config is not a second claim of the name. Omitted keys stay as they were. `""` clears a scalar key. `addresses: []` clears extra pay-tos. `txt: null` clears the TXT map.

Optional record keys, all ignored when omitted: `nostr` (64 hex or `npub1…`), `lightning` (`user@host` or `lnurl1…`), `addresses` (up to 8 extra Dogecoin pay-tos), `txt` (short free-form keys), `subdomains` (64 hex Merkle root of off-chain child labels). The root does not create `pay.jon.doge` on chain. A proof against that root is product, later.

Two claims of the same name in one block: the earlier transaction wins. A config is judged after the spends in its own transaction, so a config that rides along with a transfer does not stick to the buyer unless the config output is paid to that buyer too.

The Dogecoin address that holds the **register** inscription is the owner. JSON `address` (alias `rev`) is only a pay-to record. It can differ from the holder. It does not mint the name to that address. To mint to someone else, pay the reveal’s inscription output to them.

**Chosen name.** Holding many names does not pick one. The holder inscribes `{"p":"dns","op":"primary","name":"jon.xdg"}` from the address that holds `jon.xdg`. The latest valid one wins. `name: ""` clears it. Sending the name away drops it. Indexers ignore a primary whose owner does not hold that name.

## Envelope bytes

For one short chunk, the pushes **before** the signature are:

```text
PUSH("ord")
OP_1
PUSH(content-type)
OP_0
PUSH(body)
```

Plain `example.doge`:

```text
036f7264
51
0a746578742f706c61696e
00
0c6578616d706c652e646f6765
```

Redeem script (`n` = number of those pushes; the example above has 5):

```text
<33-byte compressed pubkey> OP_CHECKSIGVERIFY OP_DROP×n OP_TRUE
```

Full reveal `scriptSig`:

```text
<pushes above>
<DER signature || 0x01>
<redeem script>
```

Commit output script:

```text
OP_HASH160 <HASH160(redeem script)> OP_EQUAL
```

Reference wallets put the inscription on reveal vout 0 and pay it 100000 koinu (0.001 DOGE). Any spendable output above dust that this script reveals is the name. A fee to a registrar is not required.

Broadcast the commit to Dogecoin Core. After Core knows that txid, broadcast the reveal. The reveal’s input 0 spends the commit output. A public HTTP “push” API that returns a txid which `getrawtransaction` cannot see did not mint anything.

## Name check before you sign

Accept the name only when all of these hold:

- Exactly one `.`. Left part is the label, right part is the namespace.
- ASCII in each part is `[a-z0-9-]` after lowercasing. No underscore, no leading or trailing hyphen.
- No space, `/ ? # @ :`, or a second dot (`a.b.doge` is rejected).
- Each part is 1–63 Unicode scalars.
- Namespace is not `bitmap` and does not start with `dogemap`.
- Hero suffix for a product UI is `.doge`. Other suffixes are still valid names (`jon.eth` is a different name from `jon.doge`).

Store and inscribe the lowercased form (`Jon.Doge` → `jon.doge`).

## Done when

All of these are true on the reveal transaction Core accepted:

- `getrawtransaction <reveal-txid> true` returns the transaction.
- `vin[0].scriptSig` hex contains `036f7264` (`PUSH "ord"`).
- The body push is `example.doge` or the JSON object. It is not `DNS1|`.
- You can point at inscription id `<reveal-txid>i0`.
- The owner is the address holding that output, not a trailing address in an OP_RETURN.

If the only new output is nulldata and the explorer says unknown protocol, the mint failed. Fix the carrier and mint again. The failed transaction did not reserve the name.

## Leave these alone

- Do not register with `dns gift …` or `dns offer …`. Those are optional memos (80 bytes, value 0) on a reveal that **already** has the inscription. `dns gift <message>` is a note. `dns offer <name> <amount> [note]` is a bid ping. Neither moves the name.
- Do not put several names in one inscription body. Extra text after the first token is ignored, not extra mints.
- Do not implement `transfer` or `renew`. Transfer by spending the name UTXO.
- Do not require a yearly payment. Holding the output is ownership.
