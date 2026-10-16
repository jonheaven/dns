# DNS — Dogecoin Names System

A `.doge` name is a Doginals inscription. The first valid inscription wins, and holding that inscription output is owning the name. The carrier is an `ord` inscription (P2SH commit → reveal), not an OP_RETURN.

**Implementing a minter?** Read [IMPLEMENT.md](IMPLEMENT.md) before [spec.md](spec.md). An OP_RETURN `DNS1|reg|<name>|<address>` does not register a name. Explorers call that output “Data Embedded in Transaction with Unknown Protocol.” The name has to be the body of an `ord` inscription (P2SH commit → reveal).

| | |
| ---: | --- |
| **Version** | v1.1 carrier lock |
| **Class** | Transferable name object + site records |
| **Payload** | Doginals inscription: plain `label.namespace` or JSON `"p": "dns"` |
| **User-facing name** | ÐNS · Dogecoin Names System · `.doge` names |
| **Protocol / wire identifier** | `p: "dns"` · hero suffix `.doge` |
| **Source of truth** | [spec.md](spec.md) |
| **Rarity** | [rarity.md](rarity.md) — trait → score → tier (`rarity-v1`) |
| **Indexer** | dogex `/api/dns/*` and `/api/doginals/dns/*` |
| **Product** | [dogex.dog/names](https://dogex.dog/names) · mirror `/dns` · gateway `/n/jon.doge` |
| **Wallet** | Dojak Local Browser Wallet (`signInscriptionTxs` / `publishDnsOnChain`) |

DNS is a **first-come inscription** of a human-readable name (`jon.doge`). The inscription UTXO **is** the name. Transfer the UTXO, transfer the name. Point it at a website, an on-chain HTML inscription, or a Dogecoin address.

It is **not** ÐN05 (Nostr localpart), **not** Ð𝕏 (X handle bind), and **not** an OP_RETURN lease. Those protocols are identity statements. A `.doge` name is a holdable object.

### Naming (informative)

- **Display name:** **ÐNS** (Dogecoin Names System) — *Names*, not Name (SNS-aligned)
- **Product copy:** `.doge` names · `jon.doge`
- **Repository path:** `spec/protocols/dns/` · public copy [github.com/jonheaven/dns](https://github.com/jonheaven/dns)
- **On-chain marker:** `"p": "dns"` (JSON) or plain-text `label.namespace`
- **Product paths:** `/names` (primary) · `/dns` (mirror) · `/n/:name` (gateway)
